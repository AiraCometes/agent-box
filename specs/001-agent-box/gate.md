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
| V6 | secret-scan の値は引数・環境変数・一時ファイルに出さず、パイプからファイル記述子で渡す。gitleaks には `--config`・`--redact`・`--ignore-gitleaks-allow`・`-i` を必ず付ける |
| V7 | 書けるのは親のフォルダーごと。`~/.claude.json` は `CLAUDE_CONFIG_DIR` で移せるかを確かめ、だめなら写して戻す |
| V8 | `--unshare-pid` と新しい `/proc`、`--die-with-parent` を付ける。`--new-session` は付けない |
| V9 | `MISE_STATE_DIR`・`MISE_CACHE_DIR` を囲い専用の場所に、`MISE_TRUSTED_CONFIG_PATHS` にプロジェクトを入れる |
| V10 | Claude は `/etc/claude-code/managed-settings.json` でフックを固定する。ほかのエージェントは移すときに確かめる |
| V11 | `.envrc`・`.npmrc`・`.pypirc` も塞ぐ。`.env` がシンボリックリンクならリンクの先にも重ねる |

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
