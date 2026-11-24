# Tasks: agent-box

順に進める。失敗するテストは、失敗を確かめたら実装より先にコミットする。テストを消す・弱める・スキップするのは禁止。

## GitHub の側

- [ ] T001 GitHub App を作る（spec.md の「GitHub App」の権限）。`.pem` を囲いの外に 600 で置き、App ID と Bot User ID を記録する
- [ ] T002 使うリポジトリの main に ruleset を掛ける（PR・承認1つ・CI の通過を必須、force push と削除を禁止、Bypass list は本人だけ）。あわせて secret scanning と push protection を有効にし、捨ててよいリポジトリで、GitHub のトークン（`ghs_…` の形）と、Anthropic などエージェントの認証情報の形を含む push が断られるかを確かめて `gate.md` に記録する（本物の秘密は使わない。見本の形を使い、試したあとは消す）
- [ ] T004 Woodpecker のトークンを `~/.config/agent-box/woodpecker-token`（600）に置き、`.pem` を `~/.config/agent-box/github-app.pem`（600）に置く。どちらも chezmoi とリポジトリに入れない
- [ ] T003 Woodpecker で、秘密を渡すイベントを push と tag に絞り、PR の pipeline に承認を要るようにする。CODEOWNERS で `.woodpecker/**` と `.woodpecker.y*ml` を守る

## リポジトリの土台

- [ ] T010 `mise.toml`（node 26・oxlint・oxfmt・shellcheck・shfmt・bats）、`hk.pkl`（リント・整形・bats・`node --test`）、`tsconfig.json`（`erasableSyntaxOnly`・`noEmit`）、`.woodpecker.yaml`（PR では秘密を使わない）を置く

## 設定のファイル

- [ ] T020 [テスト先] `key = value` の読み方（コメント、key の繰り返し、知らない key でのエラー、空白を含む値）を、sh と TS で同じ見本のファイルに当てて確かめるテストを書く
- [ ] T021 T020 を通す sh と TS の読み手を書く。`agent-box check`（知らない key、無いパス、読むだけと書けるの重なり）も足す

## 囲い（bin/agent-box）

- [ ] T030 [テスト先] bats で、組み立てた bwrap の引数を確かめる: ホームが tmpfs、`/tmp` が tmpfs、書けるフォルダーが先で読むだけのファイルが後、読むだけの設定、無いものを作るときの中身（`.json` は `{}`、ほかのファイルは空、フォルダーは空のフォルダー）、`/init`・`/run/WSL`・`/mnt` を見せないこと、環境変数を全部消して決めたものだけを渡すこと（`WSL_INTEROP` と `WOODPECKER_TOKEN` が渡らない）、プロジェクトの決め方と起動を断る場所（git の外、ホームとその上、`/mnt` の下）、同じ絶対パスに mount すること、worktree の元の `.git` を mount すること、T3 から起動したときに足すもの、`AGENT_BOX=1` のときは包まずに本物を exec すること、外側のラッパーのフォルダーを `PATH` から外すこと、`GIT_CONFIG_*`（身元・credential.helper・insteadOf）
- [ ] T031 [テスト先] bats で `.env` を確かめる: `.env` と `.env.*` に重ねる、見本を除く、`.git`・`target`・`node_modules`・`.venv` を探さない、追跡されているものには重ねずに警告する
- [ ] T032 T030・T031 を通す `bin/agent-box` を書く（位置パラメータで組み立てる、`exec` で置き換えない、`trap` で片付ける）
- [ ] T033 本人の端末で、本物の bwrap を使って `~/.ssh` が見えないこと、`.env` が読めないこと、`cmd.exe` が動かないことを確かめ、`gate.md` に記録する

## 窓口（broker/token-broker.ts）

- [ ] T040 [テスト先] node:test で、偽の GitHub を相手に確かめる: JWT（RS256）、installation token の `repositories` と `permissions` を絞ること、残り5分までのキャッシュと再発行、セッション用のソケットがほかのリポジトリを頼めないこと、操作用のつながりが切れたらセッションを消すこと
- [ ] T041 [テスト先] node:test で、偽の Woodpecker を相手に確かめる: 番号と step の ID を `/^\d+$/` で検証、lookup の repo_id のキャッシュ、ログをそのまま中継すること
- [ ] T044 [テスト先] node:test で、`.pem` と Woodpecker のトークンのファイルの権限が 600 より広いと窓口が起動しないことを確かめる
- [ ] T042 T040・T041・T044 を通す窓口を書く。systemd のユーザーの `.socket` と `.service` のテンプレートを置く
- [ ] T043 窓口が `--max-old-space-size=32 --jitless` で動くことを、大きなログを流して確かめる

## 囲いの中の道具（inner-bin）

- [ ] T050 [テスト先] bats で `git-credential-agent-box` を確かめる: `protocol=https` と `host=github.com` 以外にはトークンを渡さない
- [ ] T051 T050 を通す `git-credential-agent-box`、`gh`、`agent-box-ci` を sh と `curl --unix-socket` で書く

## 認証情報を GitHub に上げないための確かめ（1 段目）

- [ ] T052 [テスト先] bats で、`gitleaks.toml`（既定の規則と、Claude のログイン・Google の OAuth・OpenCode・Woodpecker・Codex の形）と `agent-box-secret-scan`（認証のファイルの値と `grep -F` で比べる）が、見本の値を見つけ、値を出さずにファイルと行だけを出すことを確かめる。見本の値は、本物の形に似せた偽物にする
- [ ] T053 T052 を通す `gitleaks.toml`・`inner-bin/agent-box-secret-scan`・囲いの中用の `hk/config.pkl`（dotfiles の設定を読み込み、pre-commit と pre-push に gitleaks と secret-scan を足す）を書く
- [ ] T054 cc-safety-net の `policy.json`（`strict`、読ませない場所に `~/.config/agent-box` と `~/.config/gh`）と rulebook（`--no-verify`・`-n`・`git -c`・`--config-env`・`unset GIT_CONFIG_COUNT`・`env -u GIT_CONFIG_COUNT`・`gh pr merge`・`gh repo delete`・`gh repo edit --visibility`）を、止める見本と通る見本つきで書き、CI で `rule verify` を流す
- [ ] T055 `agent-box check` に、cc-safety-net が各エージェントに組み込まれているかの確かめを足す
- [ ] T056 README に、エージェントの認証が漏れたときの取り消し方（Claude のログアウトとセッションの取り消し、API キーの作り直し、Codex のログアウト）を書く

## 移す（1つごとに、記録用の包みで実際の動きを記録してから受け入れる）

- [ ] T060 Claude Code を移す。組み込みの囲いを切り、T3 の Binary path を設定する。受け入れを `gate.md` に記録する
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
