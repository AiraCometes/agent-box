# Tasks: agent-box

順に進める。失敗するテストは、失敗を確かめたら実装より先にコミットする。テストを消す・弱める・スキップするのは禁止。

## GitHub の側

- [ ] T001 GitHub App を作る（spec.md の「GitHub App」の権限）。`.pem` を囲いの外に 600 で置き、App ID と Bot User ID を記録する
- [ ] T002 使うリポジトリの main に ruleset を掛ける（PR・承認1つ・CI の通過を必須、force push と削除を禁止、Bypass list は本人だけ）
- [ ] T003 Woodpecker で、秘密を渡すイベントを push と tag に絞り、PR の pipeline に承認を要るようにする。CODEOWNERS で `.woodpecker/**` と `.woodpecker.y*ml` を守る

## リポジトリの土台

- [ ] T010 `mise.toml`（node 26・oxlint・oxfmt・shellcheck・shfmt・bats）、`hk.pkl`（リント・整形・bats・`node --test`）、`tsconfig.json`（`erasableSyntaxOnly`・`noEmit`）、`.woodpecker.yaml`（PR では秘密を使わない）を置く

## 設定のファイル

- [ ] T020 [テスト先] `key = value` の読み方（コメント、key の繰り返し、知らない key でのエラー、空白を含む値）を、sh と TS で同じ見本のファイルに当てて確かめるテストを書く
- [ ] T021 T020 を通す sh と TS の読み手を書く。`agent-box check`（知らない key、無いパス、読むだけと書けるの重なり）も足す

## 囲い（bin/agent-box）

- [ ] T030 [テスト先] bats で、組み立てた bwrap の引数を確かめる: ホームが tmpfs、書けるフォルダーが先で読むだけのファイルが後、読むだけの設定、無いファイルを `{}` で作ること、`/init`・`/run/WSL`・`/mnt` を見せないこと、`WSL_INTEROP` と `WOODPECKER_TOKEN` を消すこと、`AGENT_BOX=1` のときは包まずに本物を exec すること、外側のラッパーのフォルダーを `PATH` から外すこと、`GIT_CONFIG_*`（身元・credential.helper・insteadOf）
- [ ] T031 [テスト先] bats で `.env` を確かめる: `.env` と `.env.*` に重ねる、見本を除く、`.git`・`target`・`node_modules`・`.venv` を探さない、追跡されているものには重ねずに警告する
- [ ] T032 T030・T031 を通す `bin/agent-box` を書く（位置パラメータで組み立てる、`exec` で置き換えない、`trap` で片付ける）
- [ ] T033 本人の端末で、本物の bwrap を使って `~/.ssh` が見えないこと、`.env` が読めないこと、`cmd.exe` が動かないことを確かめ、`gate.md` に記録する

## 窓口（broker/token-broker.ts）

- [ ] T040 [テスト先] node:test で、偽の GitHub を相手に確かめる: JWT（RS256）、installation token の `repositories` と `permissions` を絞ること、残り5分までのキャッシュと再発行、セッション用のソケットがほかのリポジトリを頼めないこと、操作用のつながりが切れたらセッションを消すこと
- [ ] T041 [テスト先] node:test で、偽の Woodpecker を相手に確かめる: 番号と step の ID を `/^\d+$/` で検証、lookup の repo_id のキャッシュ、ログをそのまま中継すること
- [ ] T042 T040・T041 を通す窓口を書く。systemd のユーザーの `.socket` と `.service` のテンプレートを置く
- [ ] T043 窓口が `--max-old-space-size=32 --jitless` で動くことを、大きなログを流して確かめる

## 囲いの中の道具（inner-bin）

- [ ] T050 [テスト先] bats で `git-credential-agent-box` を確かめる: `protocol=https` と `host=github.com` 以外にはトークンを渡さない
- [ ] T051 T050 を通す `git-credential-agent-box`、`gh`、`agent-box-ci` を sh と `curl --unix-socket` で書く

## 移す（1つごとに、記録用の包みで実際の動きを記録してから受け入れる）

- [ ] T060 Claude Code を移す。組み込みの囲いを切り、T3 の Binary path を設定する。受け入れを `gate.md` に記録する
- [ ] T061 Pi を移す。自動更新を切る方法と、本体の置き場を読むだけにしたときの動きを確かめる
- [ ] T062 Antigravity を移す
- [ ] T063 Codex を入れて移す（公式の入れ方を確かめる。組み込みの囲いを切る。`config.toml` の扱いを確かめる）

## 後片付け

- [ ] T070 dotfiles で、リンク・設定のファイル・systemd のユニットを chezmoi に入れ、CLAUDE.md・AGENTS.md・GEMINI.md の CI のログの読み方を `agent-box-ci` に書き換える
- [ ] T071 4つとも移ったあと、dotfiles から srt・in-srt・verify-commits・agent-git・SPEC-025 を外す
- [ ] T072 README にセットアップの手順（GitHub App、T3 の Binary path、更新のしかた）を書く
