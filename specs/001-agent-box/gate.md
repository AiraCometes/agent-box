# agent-box の関門の記録

**日付**: 2026-11-24
**環境**: WSL2（Linux 6.18.58.1-microsoft-standard-WSL2）、T3 Code 0.1.11、Node 24（26 は mise で取れる。26.13.1）

## 決めたこと（対話で決めた Q1〜Q36）

| # | 決めたこと |
|---|---|
| Q1 | 守る: 手元の鍵、作業リポジトリの `.env`、WSL の interop、ほかのリポジトリとホーム、囲いの規則と設定だけのファイル、main。受け入れる: 作業中のプロジェクトを壊されること、そのコードを外へ送られること |
| Q2 | エージェントのプロセスを丸ごと包み、中では確認なしで任せる。ログイン情報が中にあることは受け入れる |
| Q3 | GitHub での身元は GitHub App。machine user と deploy key は使わない |
| Q4 | main の ruleset: PR・承認1つ・CI を必須、force push と削除を禁止。Bypass list は本人だけ（App は入れない） |
| Q5 | 手入れの上限: 設定は1つ、エージェントの追加は設定のファイル1つ、版を上げたときだけ確かめる |
| Q6 | 対象は WSL の中の CLI のエージェント（T3 からも端末からも）。Windows 側は範囲の外 |
| Q7 | bwrap を使う POSIX sh のスクリプト1本。見せるものだけを mount する |
| Q8 | ホストのネットワークを使う（T3 の MCP に届くため）。LAN は塞がない |
| Q9 | 起動のたびに `.env` と `.env.*` に空のファイルを重ねる。見本は除き、`.git`・`target`・`node_modules`・`.venv` は探さない。追跡されているものは警告だけ |
| Q10 | interop は囲いの中だけで塞ぐ（G1 で塞げると確かめた） |
| Q11 | `.pem` は外に置き、外の窓口が installation token を作る。中には Unix ソケットを1本だけ見せる |
| Q12 | App の権限: Contents・Pull requests・Issues は読み書き、Metadata・Checks・Statuses は読む。Workflows・Administration・Actions の書き込み・Secrets は与えない |
| Q13 | Woodpecker: 秘密は push と tag だけ、PR の pipeline には承認、CODEOWNERS で `.woodpecker/**` と `.woodpecker.y*ml` |
| Q14・Q19 | 設定だけのファイルは読むだけ、混ざったものと状態は書ける |
| Q15 | 道具は読むだけ、取得のキャッシュは囲い専用、`target` は共有 |
| Q16 | 残す: hk・check-file・cc-safety-net。やめる: verify-commits・agent-git・SPEC-025・srt・in-srt・拒否の一覧 |
| Q17 | 本人用に、シェルだけを開く `box sh` を用意する |
| Q18 | ホームは毎回、空の tmpfs から組み立てる |
| Q20 | 起動スクリプトと囲いの中の道具は POSIX sh、窓口は Node 26 で動かす依存なしの TypeScript。リントは oxlint・oxfmt・shellcheck・shfmt |
| Q21 | 順番: GitHub の側 → このリポジトリで実装 → Claude・Pi・Antigravity・Codex を1つずつ移す → dotfiles の古い仕組みを外す |
| Q22 | T3 の Binary path は画面で1回設定する。chezmoi は `~/.t3/userdata` に触れない |
| Q23 | テストは bats と node:test だけ。受け入れはエージェントを1つ移すごと |
| Q24 | コミットの作者は App の bot（Bot User ID を使う） |
| Q25・Q36 | このリポジトリ（公開）で、`specs/001-agent-box` に書く |
| Q26 | 外側のラッパーは `~/.local/share/agent-box/bin`、囲いの中の道具は `inner-bin`。中の `PATH` から外側を外す |
| Q27 | 中では自動更新を切り、外で更新する |
| Q28 | ソケットは、外の操作用と、起動ごとのセッション用の2種類。操作用のつながりが切れたらセッションを消す |
| Q29 | git は credential helper（`protocol=https` と `host=github.com` だけ）と `insteadOf`。git のラッパーは作らない |
| Q30 | Woodpecker のログは、窓口の読むだけの口で読む |
| Q31・Q32 | Codex を4つ目に足す。組み込みの囲いは切る |
| Q33 | Claude・Pi・Antigravity は公式の curl の入れ方のまま。Codex は公式が勧める入れ方 |
| Q35 | 設定は `key = value` の `.conf`（key の繰り返しで一覧、知らない key はエラー）。`agent-box check` を付ける |

## 決めたこと（仕様のレビューと、認証情報の扱いで足したもの）

| # | 決めたこと |
|---|---|
| Q43 | エージェント自身の認証は、囲いの中から読めることを受け入れる。漏れたときの取り消し方を README に書く |
| Q46・Q47 | 2段階で作る。1 段目は囲いの中の hk（gitleaks と、値をそのまま比べる secret-scan）と cc-safety-net で止める。2 段目で、GitHub への書き込みを全部窓口に通す（素の `git push` は `pushInsteadOf` と remote helper、gh は `http_unix_socket`） |
| S1 | push protection は三つ目の網として扱い、頼りにしない。Claude のログインのトークンなどは、hk が流す gitleaks に自分で足す規則で止める |
| S2 | 無いものを先に作るとき、`.json` は `{}`、ほかのファイルは空、フォルダーは空のフォルダー |
| S3 | Woodpecker の秘密は、main への push と tag だけで渡す |
| S4 | プロジェクトは git の最上位。git の外、ホームとその上、`/mnt` の下では起動を断る |
| S5 | プロジェクトは同じ絶対パスに mount する。worktree なら元の `.git` も mount する |
| S6 | T3 から起動したときは、Antigravity の `--gemini-dir`（書ける）、`T3_ACP_MCP_NODE` と `T3_PI_MCP_EXTENSION_PATH`（読むだけ）を足す |
| S7 | 環境変数は全部消して、決めたものだけを渡す |
| S8 | フックが呼ぶ道具（check-file、hk、cc-safety-net、deja）は読むだけで mount する |
| S9・C3 | cc-safety-net の rulebook で、フックを飛ばす書き方を止める（`--no-verify`・`-n`・`git -c`・`--config-env`・`unset GIT_CONFIG_COUNT` など）。止められないものは 1 段目では受け入れる |
| S10 | `/tmp` は起動ごとの tmpfs |
| S12 | 囲いの外のあなたの commit と push は、agent-box の範囲に入れない |
| C1 | cc-safety-net の policy と rulebook は agent-box が持ち、`agent-box check` で各エージェントへの組み込みを確かめる |
| C2 | cc-safety-net は `strict`。読ませない場所に `~/.config/agent-box` と `~/.config/gh` を足す |
| C4 | gitleaks は囲いの中用の hk の設定で流す。規則に Claude のログイン・Google の OAuth・OpenCode・Woodpecker・Codex の形を足す |

## 決めたこと（Sonnet 5.5 のレビューで足したもの、V1〜V11）

| # | 決めたこと |
|---|---|
| V1 | `[ -e /mnt/wsl/resolv.conf ]` のときだけ、それを同じパスに読むだけで mount する |
| V2 | `.git/hooks` は読むだけで重ねる。`.git/config` は書けるままにし、終わったときに `branch.*` 以外のキーの変更を、起動したときの写しへキーごとに戻す（残してよいものだけを残す） |
| V3 | `core.hooksPath` を agent-box の読むだけのフックのフォルダーに向け、hk を流す。リポジトリ自身の `.git/hooks` のフックは囲いの中では動かない |
| V4 | secret-scan が見る認証のファイルと変数の名前の一覧を、起動スクリプトが作って読むだけで mount する（値は入れない） |
| V5 | `profiles/*.conf` には環境変数の名前だけを書き、値は起動したときの環境から引き継ぐ |
| V6 | secret-scan の値は引数・環境変数・一時ファイルに出さない（パイプからファイル記述子で渡す形は、W1 で node の中で完結させる形に置き換えた）。gitleaks には `--config`・`--redact`・`--ignore-gitleaks-allow`・`-i` を必ず付ける |
| V7 | 書けるのは親のフォルダーごと。`~/.claude.json` は `CLAUDE_CONFIG_DIR` で移せるかを確かめ、だめなら写して戻す |
| V8 | `--unshare-pid` と新しい `/proc`、`--die-with-parent` を付ける。`--new-session` は付けない |
| V9 | `MISE_STATE_DIR`・`MISE_CACHE_DIR` を囲い専用の場所に、`MISE_TRUSTED_CONFIG_PATHS` にプロジェクトを入れる |
| V10 | Claude は `/etc/claude-code/managed-settings.json` でフックを固定する。ほかのエージェントは移すときに確かめる |
| V11 | `.envrc`・`.npmrc`・`.pypirc` も塞ぐ。`.env` がシンボリックリンクならリンクの先にも重ねる |

## 決めたこと（Sonnet 5.5 の2回目のレビューで足したもの、W1〜W6）

| # | 決めたこと |
|---|---|
| W1 | secret-scan は node の中で完結させる。値は 16 文字以上。pre-commit は `git diff --cached -U0`、pre-push は `<remote>..<local>`、新しいブランチは `<local> --not --remotes`、削除の push は飛ばす。見つかれば終了コード 1、落ちたときも通さない |
| W2 | `.git/config` の写しは `~/.local/state/agent-box/<hash>/` に置き、起動のときにも残りで戻す。同時のセッションは `flock` の中で、PID と起動時刻が合う生きているものだけを数え、最後のセッションが戻す。submodule と worktree の `hooks`・`config` も同じに扱う |
| W3 | gitleaks と secret-scan は、読むだけの `hooks/pre-commit`・`hooks/pre-push` が hk より先に直接流す（リポジトリの `hk.pkl` で上書きできないように） |
| W4 | テストの見本の値は、テストの中で作ってリポジトリに入れない。`.gitignore` は、秘密の形に当たるコードのファイルだけをファイルごとに残す |
| W5 | `.git/config` に残してよいのは `branch.*` と、新しいリモートの `url`・`fetch`（url は `https://`・`git@github.com:`・`ssh://git@github.com/` だけ）。消えたリモートを指す `branch.<x>.remote` は戻す。submodule は囲いの中では続けて使えないものとして受け入れる |
| W6 | 共通の環境変数に `HK_CHECK=1` と `HK_STASH_UNTRACKED=0` を入れる。T060 で、agent-box のリポジトリ固有の hk の段が囲いの中で通ることを確かめる |

## 決めたこと（Sonnet 5.5 の3回目のレビューで足したもの、X1〜X6）

| # | 決めたこと |
|---|---|
| X1 | 差分を読む git に `--no-ext-diff --no-textconv --no-color --no-renames` と、`-c diff.external= -c diff.noprefix=false -c diff.mnemonicPrefix=false -c color.ui=false` を付ける。gitleaks には `GIT_EXTERNAL_DIFF=` と、同じ設定を `GIT_CONFIG_*` の後ろに足して渡す（効くかは T052 で確かめる） |
| X2 | 新しいブランチの push は、起動したときの写しにある `refs/remotes/*` の SHA だけを除く。写しが 0 件なら `--not` を付けない。リモートの SHA がローカルに無いときも同じ扱い |
| X3 | `flock` の中で、生きているセッションが 0 のときだけ戻してから写しを取る。1 つ以上なら写しも戻しもしない。最後のセッションが戻す。写しの場所は `--git-common-dir` の実パスから決める |
| X4 | node・gitleaks・hk は agent-box の `mise.toml` で版を固定し、起動スクリプトがフルパスを決めて渡す。`PATH` は `inner-bin` → mise の shim → `/usr/bin` |
| X5 | gitleaks は pre-commit で `git --staged`、pre-push で `git --log-opts`。secret-scan は `spawn` で行ごとに読み、`-m` でマージの内容も見る。pre-push の標準入力は変数に取っておき、それぞれに渡す |
| X6 | `.git/config` は、囲いの外の node のスクリプトが、`--list -z` で読み、残してよいものを `--add` で足したファイルに付け替える。警告にはキーの名前だけを出す |

## 決めたこと（Sonnet 5.5 の4回目のレビューで足したもの、Y1〜Y5）

| # | 決めたこと |
|---|---|
| Y1 | 差分は `agent-box-secret-scan.ts` が1回だけ作り、自分の照合と `gitleaks stdin` の両方に流す。gitleaks は `share/empty/` で流し、0 以外の終了と `EPIPE` は全部通さない。差分を作る git に `--text --src-prefix=a/ --dst-prefix=b/ -c core.quotePath=false` を足す。`-c diff.external=` と `GIT_EXTERNAL_DIFF=` は使わない（X1 の一部を置き換えた） |
| Y2 | 新しいブランチの push で除く範囲は、`git ls-remote --branches --tags`（通らなければ `--heads --tags`）で push 先に聞き、`git cat-file -e "<sha>^{commit}"` でローカルにあるものだけにする。囲いの中の `refs/remotes/*` と写しは使わない（X2 を置き換えた） |
| Y3 | `.git/config` を戻すスクリプトは `bin/agent-box-restore-config.ts` で、T032 で書く。commit が止まる確かめは T057 として T053 のあとに回す |
| Y4 | `mise where` は agent-box の `mise.toml` のあるフォルダーで流す。版が入っていなければ起動を断る |
| Y5 | 値の無いキーは `true` として戻す |

## G8 4回目のレビューで、レビューの側が確かめたこと（2026-11-24、gitleaks 8.30.1・git 2.56.1）

- `--no-ext-diff --no-textconv --no-color --no-renames` と `-c diff.noprefix=false -c diff.mnemonicPrefix=false` で、`.git/config` の `diff.external`・textconv・`color.diff=always`・`noprefix`・`mnemonicPrefix` と `.gitattributes` の `diff=x` を打ち消せた
- `-c diff.external=`（空）や `GIT_EXTERNAL_DIFF=`（空）は「設定なし」にならず、`--no-ext-diff` の無い `git diff` は `cannot run` で落ちた。gitleaks は自分で `--no-ext-diff` を付けている
- `gitleaks git` は、`-i` を空のフォルダーに向けても、調べる場所の `.gitleaksignore` を読んだ。`color.diff=always`（`color.ui=false` より強い）・textconv・`-diff`・`binary`・NUL で、見つかる件数が 0 になった。`--log-opts` に `-m` が無いと、マージだけで入った内容が見えなかった
- 差分を作る側で `--text` などを付けて `gitleaks stdin` に渡し、作業フォルダーを空のフォルダーにすると、見つかり、リポジトリの `.gitleaksignore` は効かなかった
- `--no-ext-diff` などを付けても、`-diff`・`binary`・NUL のファイルは `--text` が無いと `Binary files … differ` になった。`diff.srcPrefix`・`diff.dstPrefix`・`core.quotePath` で、ファイル名の形が変わった
- `git config --list -z` は「キー、改行、値、NUL」の形で、値に `=` や改行があっても `--add` で崩れずに戻せた。値の無いキーは `key\0` で、改行が無い

## G7 3回目のレビューで、レビューの側が一時リポジトリで確かめたこと（2026-11-24）

- `diff.external` を置くと `git diff --cached` はそのプログラムを動かし、秘密の行が出なくなった。`.gitattributes` の `diff=x` と `diff.x.textconv` でも同じ。`--no-ext-diff --no-textconv` を付けると見つかった
- `git update-ref refs/remotes/fake/x HEAD` のあと、`git log -p -U0 HEAD --not --remotes` の出力は 0 行になった
- パイプを `grep -f /dev/fd/3` で2回読むと、GNU grep 3.12 では2回目に何も出ず、ugrep 7.8.4 では全行に当たった
- `branch.main.remote` に `ext::…` を入れて fetch すると、git は値を URL として扱い、`transport 'ext' not allowed` で止まった
- hk 2.13.0 に `HK_CHECK` と `HK_STASH_UNTRACKED` がある

## G6 レビューのときに確かめたこと（2026-11-24）

- `/etc/resolv.conf` は `/mnt/wsl/resolv.conf` へのシンボリックリンクで、リンクの先のファイルはある
- `/proc/sys/dev/tty/legacy_tiocsti` は `0`（TIOCSTI はカーネルで止まっている）
- gitleaks 8.30.1 のソース（`cmd/root.go`）: `--redact`・`--ignore-gitleaks-allow`・`-i`（`.gitleaksignore` の場所、既定は `.`）・`--baseline-path` がある。設定は `--config` → `GITLEAKS_CONFIG` → `GITLEAKS_CONFIG_TOML` → 調べる場所の `.gitleaks.toml` → 既定の順に使う。`--config` を渡さないと、エージェントが置いた `.gitleaks.toml` で規則を弱められる
- dotfiles の hk の設定（`~/.config/hk/config.pkl`）の頭の説明: hk の版が mise の hk と同じなら、hk に入っている Pkl の包みで解くので取りに行かない

## G5 認証情報を止める仕組みで分かったこと（2026-11-24、公式の文書と道具のソースから）

- GitHub の push protection: Claude のログインのトークン（`sk-ant-oat01-`）は既定の検出に無い。Google と Gemini の API キーは、見つけても push のときに断らない。App の installation token での push に効くかは、文書に書かれていない。Issue・PR・コメントの本文は、上がったあとに知らせるだけ。個人の public なリポジトリでは、自分で規則を足せない
- gitleaks 8.30.1・betterleaks・trufflehog のどの既定の規則にも、`sk-ant-oat01-` は無い。gitleaks は Google の OAuth（`ya29.`・`GOCSPX`）も無い。gitleaks は MIT の単体のバイナリで、mise（`aqua:gitleaks/gitleaks`）で入れられ、`git --staged` と `--log-opts` で範囲を調べられる
- gh 2.104.0 には、API の通信を Unix ソケットに流す `http_unix_socket` の設定がある
- cc-safety-net: 自分で足す規則は、語を文字どおりに比べる（前方の一致や正規表現は使えない）。頭に付けた環境変数は見ない。組み込みで、coding CLI の認証のトークンと `.git/hooks` への書き込みを止める

## G1 WSL の interop を囲いの中で塞げるか（本人の端末、囲いの外）

- `cmd.exe` を `/tmp` に写し、bwrap の中から `cmd.exe /c echo INTEROP_WORKS` を流した
  - 共通の引数: `/usr`・`/etc` を読むだけ、`/proc`・`/dev`、`/tmp` と `/run` は tmpfs、`--unshare-pid`
  - ネットワークはホストのまま
- WSL の登録（`/proc/sys/fs/binfmt_misc/WSLInterop`）: `interpreter /init`、`flags: PF`、`magic 4d5a`
- 結果:
  - 囲いの外: 動いた
  - A（`/init` も `/run/WSL` も見せない）: 落ちた。`/init` が動き、`UtilConnectUnix:552: connect failed 2` で終わった
  - B（`/init` を見せる）: A と同じ
  - C（`/init` と `/run/WSL` を見せる）: `Invalid argument` で落ちた（理由は確かめていない）
  - D（C に加えて `binfmt_misc` に tmpfs を重ねる）: C と同じ
- 分かったこと: `F` のフラグがあるので、`/init` を見せなくても `.exe` の起動で `/init` が動く。`binfmt_misc` を隠しても変わらない。塞げたのは、`/run` を空にして interop のソケットに届かなかったから → `/run/WSL` を mount せず、`WSL_INTEROP` を消す

## G2 systemd のユーザーのセッション（本人の端末）

- `systemctl --user is-system-running` は `degraded`（バスにはつながる）、`Linger=yes` → `.socket` ユニットで窓口を受け付けられる

## G3 T3 Code 0.1.11 のソース（`pingdotgg/t3code` の v0.1.11。動かしてはいない）

- プロバイダーごとに `binaryPath` がある。シェルを通さず1つのコマンドとして起動するので、前に書き足す形はできない
- Antigravity の `binaryPath` は、ファイル名が `^(agy|antigravity)(\.(exe|cmd|bat))?$` に合う必要がある（`apps/server/src/provider/AntigravityInstallation.ts`）
- エージェントとのやりとりは標準入出力。T3 のツール（MCP）は、どのエージェントからも最後は `http://127.0.0.1:<port>/mcp` に届く
- Linux で使えるときは、T3 がエージェントを `systemd-run --user --scope` で包む

## G4 エージェントの置き場（読むだけで調べた。値は見ていない）

- 設定と状態が混ざっているもの:
  - Claude の `~/.claude.json`（MCP サーバー、プロジェクトの信頼、`/config` の値。中身は読めず、文書と backups の mtime から判断）
  - Pi の `settings.json` と `trust.json`
  - Antigravity の `~/.gemini/antigravity-cli/settings.json`（文書から判断）
- 動いている最中に書かれ、動きも変えるもの:
  - Claude の memory、同期されるスキル（`~/.claude/skills/synced/`）、plugins
- `~/.local/bin` には、もう本物が入っている:
  - `claude`: 自動更新が張り替えるリンク
  - `agy`: 本体のバイナリ
  - `pi`: `~/.pi/agent/bin/pi` へのリンク
