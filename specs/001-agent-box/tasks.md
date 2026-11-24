# Tasks: agent-box

順に進める。失敗するテストは、失敗を確かめたら実装より先にコミットする。テストを消す・弱める・スキップするのは禁止。

## GitHub の側

- [ ] T001 GitHub App を作る（spec.md の「GitHub App」の権限）。`.pem` を囲いの外に 600 で置き、App ID と Bot User ID を記録する
- [ ] T002 使うリポジトリの main に ruleset を掛ける（PR・承認1つ・CI の通過を必須、force push と削除を禁止、Bypass list は本人だけ）。あわせて secret scanning と push protection を有効にし、捨ててよいリポジトリで、GitHub のトークン（`ghs_…` の形）と、Anthropic などエージェントの認証情報の形を含む push が断られるかを確かめて `gate.md` に記録する（本物の秘密は使わない。見本の形を使い、試したあとは消す）
- [ ] T003 Woodpecker で、秘密を渡すイベントを main への push と tag に絞り、PR の pipeline に承認を要るようにする。CODEOWNERS で `.woodpecker/**` と `.woodpecker.y*ml` を守る
- [ ] T004 Woodpecker のトークンを `~/.config/agent-box/woodpecker-token`（600）に置き、`.pem` を `~/.config/agent-box/github-app.pem`（600）に置く。どちらも chezmoi とリポジトリに入れない

## リポジトリの土台

- [ ] T010 `mise.toml`（node 26・gitleaks・hk・oxlint・oxfmt・shellcheck・shfmt・bats。node・gitleaks・hk は版を固定する）、`hk.pkl`（リント・整形・bats・`node --test`）、`tsconfig.json`（`erasableSyntaxOnly`・`noEmit`）、`.woodpecker.yaml`（PR では秘密を使わない）を置く

## 設定のファイル

- [ ] T020 [テスト先] `key = value` の読み方（コメント、key の繰り返し、知らない key でのエラー、空白を含む値）を、sh と TS で同じ見本のファイルに当てて確かめるテストを書く
- [ ] T021 T020 を通す sh と TS の読み手を書く。`agent-box check`（知らない key、無いパス、読むだけと書けるの重なり）も足す

## 囲い（bin/agent-box）

- [ ] T030 [テスト先] bats で、組み立てた bwrap の引数を確かめる: ホームが tmpfs、`/tmp` が tmpfs、書けるフォルダーが先で読むだけのファイルが後、読むだけの設定、無いものを作るときの中身（`.json` は `{}`、ほかのファイルは空、フォルダーは空のフォルダー）、`/init`・`/run/WSL`・`/mnt` を見せないこと、環境変数を全部消して決めたものだけを渡すこと（`WSL_INTEROP` と `WOODPECKER_TOKEN` が渡らない）、プロジェクトの決め方と起動を断る場所（git の外、ホームとその上、`/mnt` の下）、同じ絶対パスに mount すること、worktree の元の `.git` を mount すること、T3 から起動したときに足すもの、`AGENT_BOX=1` のときは包まずに本物を exec すること、外側のラッパーのフォルダーを `PATH` から外すこと、`GIT_CONFIG_*`（身元・`credential.helper = agent-box`・insteadOf・`core.hooksPath`。git から `git credential fill` を呼んで helper に届くこと）、`--unshare-pid` と新しい `/proc`・`--die-with-parent` を付け `--new-session` を付けないこと、`/mnt/wsl/resolv.conf` があるときだけ mount すること、`.git/hooks`（worktree なら元のものも）を読むだけで重ねること、終わったときに `.git/config` の残してよいもの以外の変更を写しへ戻すこと（`branch.*` と、新しいリモートの `url`・`fetch` だけを残す。url は `https://`・`git@github.com:`・`ssh://git@github.com/` だけ。消えたリモートを指す `branch.<x>.remote` は戻す）、写しを `~/.local/state/agent-box/<hash>/`（`<hash>` は `--git-common-dir` の実パスから）に置き、`kill -9` のあと次に起動したときに戻ること、`flock` の中で PID と起動時刻が合う生きているセッションを数え、0 のときだけ戻してから写しを取り、1 つ以上のときは写しも戻しもせず、最後のセッションだけが戻すこと、同じリポジトリの worktree どうしが写しを共有すること、戻すとき（`bin/agent-box-restore-config.ts`）に `--list -z` で読んで値に `=` や改行があっても崩れず、値の無いキーを `true` として足すこと、`branch.<x>.remote`・`pushRemote` は最後に残るリモートの名前を指すときだけ残すこと、`AGENT_BOX_NODE`・`AGENT_BOX_GITLEAKS`・`AGENT_BOX_HK` をフルパスで渡すこと、`PATH` の順（`inner-bin` → mise の shim → `/usr/bin`）、submodule と worktree の `hooks` と `config` も同じに扱うこと、`HK_CHECK=1` と `HK_STASH_UNTRACKED=0` を渡すこと、`MISE_STATE_DIR`・`MISE_CACHE_DIR`・`MISE_TRUSTED_CONFIG_PATHS`、Claude の `managed-settings.json` を mount すること、書けるのは親のフォルダーごとで単一のファイルを書けるように mount しないこと、`trap` が `EXIT`・`INT`・`TERM`・`HUP` で片付けること
- [ ] T031 [テスト先] bats で `.env` を確かめる: `.env`・`.env.*`・`.envrc`・`.npmrc`・`.pypirc` に重ねる、シンボリックリンクならリンクの先にも重ねる、見本を除く、`.git`・`target`・`node_modules`・`.venv` を探さない、追跡されているものには重ねずに警告する
- [ ] T032 T030・T031 を通す `bin/agent-box` と `bin/agent-box-restore-config.ts`（`.git/config` を `--list -z` で読み、残してよいものを `--add` で足して、付け替えで入れ替える）を書く（位置パラメータで組み立てる、`exec` で置き換えない、`trap` で片付ける）。secret-scan が見る認証のファイルと環境変数の名前の一覧を、`profiles/*.conf` から作って読むだけで mount する（値は入れない）
- [ ] T033 本人の端末で、本物の bwrap を使って `~/.ssh` が見えないこと、`.env` が読めないこと、`cmd.exe` が動かないこと、DNS と HTTPS（CA の証明書）が通ること、`/proc` から囲いの外のプロセスが見えないこと、`.git/hooks` に書けないこと、Ctrl-C がエージェントに届くことを確かめ、`gate.md` に記録する

## 窓口（broker/token-broker.ts）

- [ ] T040 [テスト先] node:test で、偽の GitHub を相手に確かめる: JWT（RS256）、installation token の `repositories` と `permissions` を絞ること、残り5分までのキャッシュと再発行、セッション用のソケットがほかのリポジトリを頼めないこと、操作用のつながりが切れたらセッションを消すこと
- [ ] T041 [テスト先] node:test で、偽の Woodpecker を相手に確かめる: 番号と step の ID を `/^\d+$/` で検証、lookup の repo_id のキャッシュ、ログをそのまま中継すること
- [ ] T044 [テスト先] node:test（`broker/broker-perms.test.ts`）で、`.pem` と Woodpecker のトークンのファイルの権限が 600 より広いと窓口が起動しないことを確かめる
- [ ] T042 T040・T041・T044 を通す窓口を書く。systemd のユーザーのユニットのテンプレート `systemd/agent-box-broker.socket` と `systemd/agent-box-broker.service` を置く
- [ ] T043 窓口が `--max-old-space-size=32 --jitless` で動くことを確かめる。合否: 100MB のログを中継し終えるまで、窓口のプロセスの RSS が 64MB を超えず、落ちないこと

## 囲いの中の道具（inner-bin）

- [ ] T050 [テスト先] bats で `git-credential-agent-box` を確かめる: `protocol=https` と `host=github.com` 以外にはトークンを渡さない
- [ ] T051 T050 を通す `git-credential-agent-box`、`gh`、`agent-box-ci` を sh と `curl --unix-socket` で書く

## 認証情報を GitHub に上げないための確かめ（1 段目）

- [ ] T052 [テスト先] bats で、`gitleaks.toml`（既定の規則と、Claude のログイン・Google の OAuth・OpenCode・Woodpecker・Codex の形）と `inner-bin/agent-box-secret-scan.ts`（差分を1回だけ作り、認証のファイルと環境変数の 16 文字以上の値との照合と、`gitleaks stdin` の両方に流す）が、見本の値を見つけ、値を出さずにファイルと行（ファイルの中の行の番号）だけを出し、終了コード 1 で止めることを確かめる。pre-commit（`git diff --cached -U0`）と pre-push（既存のブランチ、新しいブランチは `ls-remote` で聞いた SHA のうちローカルにあるものを除く、1つも無ければ `--not` を付けない、リモートの SHA がローカルに無いときは新しいブランチと同じ扱い、削除の push を飛ばす、マージコミットの内容も見る、`refs/pull/*` を除く範囲に入れない、注釈付きのタグを扱う、`ls-remote` が落ちたら通さない、`--branches` と `--heads` のどちらが通るか）の両方、囲いの中で作った偽の `refs/remotes/*` で範囲が空にならないこと、エージェントが `diff.external`・`.gitattributes` の textconv・`-diff`・`binary`・NUL を含むファイル・`color.diff=always`・`diff.noprefix=true`・`diff.mnemonicPrefix=true`・`diff.srcPrefix`・`core.quotePath` を置いても、secret-scan と gitleaks の両方が見つけ、ファイル名を正しく出すこと、1MB を超える差分で落ちないこと、pre-push の標準入力が secret-scan と `hk run pre-push` のそれぞれに届くこと、2 つ以上のファイル、node が落ちたとき・一覧が読めないとき・gitleaks が 0 以外で終わったとき・gitleaks への書き込みで `EPIPE` が起きたときに通さないこと、改行を含む値で何にでも当たらないことも確かめる。あわせて、gitleaks を `share/empty/` で `--config`・`--redact`・`--ignore-gitleaks-allow` を付けて流し、リポジトリの `.gitleaks.toml`・`.gitleaksignore`・`gitleaks:allow` が効かないこと、secret-scan の値が引数・環境変数・一時ファイルに出ないこと（`/proc/<pid>/cmdline` と `/proc/<pid>/environ` と `/tmp` を見る）、空の値で何にでも当たらないこと、API キーの環境変数の値も見つけることを確かめる。見本の値は、本物の形に似せた偽物を bats の `setup` で文字列をつなげて作り、`$BATS_TEST_TMPDIR` に置く（リポジトリには入れない）
- [ ] T053 T052 を通す `gitleaks.toml`・`inner-bin/agent-box-secret-scan.ts`・`hooks/pre-commit` と `hooks/pre-push`（secret-scan を先に流し、通れば `hk run` を呼ぶ。pre-push の標準入力は変数に取っておいて両方に渡す）・`share/empty/.gitkeep` を書く
- [ ] T054 cc-safety-net の `policy.json`（`strict`、読ませない場所に `~/.config/agent-box` と `~/.config/gh`）と rulebook（`--no-verify`・`-n`・`git -c`・`--config-env`・`unset GIT_CONFIG_COUNT`・`env -u GIT_CONFIG_COUNT`・`gh pr merge`・`gh repo delete`・`gh repo edit --visibility`）を、止める見本と通る見本つきで書き、CI で `rule verify` を流す。あわせて、Claude の `managed-settings.json`（cc-safety-net と check-file のフックを固定し、プロジェクトの設定から外せないようにする）を書く
- [ ] T055 `agent-box check` に、cc-safety-net が各エージェントに組み込まれているかの確かめを足す
- [ ] T057 本人の端末で、本物の bwrap の囲いの中から、見本の値を書いたファイルの commit が `hooks/pre-commit` で止まり、push が `hooks/pre-push` で止まることを確かめ、`gate.md` に記録する（T053 のあと）
- [ ] T056 README に、エージェントの認証が漏れたときの取り消し方（Claude のログアウトとセッションの取り消し、API キーの作り直し、Codex のログアウト）を書く

## 移す（1つごとに、記録用の包みで実際の動きを記録してから受け入れる）

- [ ] T060 Claude Code を移す。組み込みの囲いを切り、T3 の Binary path を設定する。agent-box のリポジトリで、リポジトリ固有の hk の段（check の側）が囲いの中で通ることも確かめる。受け入れを `gate.md` に記録する
- [ ] T061 Pi を移す。自動更新を切る方法と、本体の置き場を読むだけにしたときの動きを確かめる
- [ ] T062 Antigravity を移す
- [ ] T063 Codex を入れて移す（公式の入れ方を確かめる。組み込みの囲いを切る。`config.toml` の扱いを確かめる）

## 後片付け

- [ ] T070 dotfiles で、リンク・設定のファイル・systemd のユニットを chezmoi に入れ、CLAUDE.md・AGENTS.md・GEMINI.md の CI のログの読み方を `agent-box-ci` に書き換える
- [ ] T071 4つとも移ったあと、dotfiles から srt・in-srt・verify-commits・agent-git・SPEC-025 を外す
- [ ] T072 README にセットアップの手順（GitHub App、T3 の Binary path、更新のしかた）を書く

## 2 段目: GitHub への書き込みの窓口（囲いが落ち着いてから）

- [ ] T080 記録用の包みで、git の remote helper のやりとり（`list for-push`・`push` と、push の選択肢）と、gh の `http_unix_socket` の動き（トークンが無いときの扱い、使うサブコマンドごと）を記録して `gate.md` に書く
- [ ] T081 [テスト先] bats と node:test で、`git-remote-agentbox` と窓口の push（範囲を調べる、見つかれば断ってファイルと行だけを返す、フックと設定を使わずに決まった URL へ push する）と、gh の中継（本文を調べる）を確かめる
- [ ] T082 T081 を通す。切り替えたら、1 段目の書けるトークンを外し、囲いの中には読むだけのトークンだけを渡す
