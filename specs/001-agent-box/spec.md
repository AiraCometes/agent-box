# Feature Specification: agent-box

**Issue**: #1
**状態**: 設計を確定した（2026-11-24）。決めた経緯と、実機で確かめた結果は `gate.md` に書く
**環境**: WSL2（Linux 6.18、メモリの割り当てはおよそ 7.8GB、Windows 側のディスクの空きはおよそ 30GB）、T3 Code 0.1.11、bwrap、systemd のユーザーのセッション（使える。`Linger=yes`）

## 目的と範囲

- WSL の中で動く CLI のコーディングエージェントを、1つの囲いで包む。対象は Claude Code・Pi・Antigravity（`agy`）・Codex と、今後足すもの。T3 Code から起動しても、端末から起動しても包む
- エージェントのプロセスを丸ごと包み、中では確認なしで任せる。エージェント自身のログイン情報（`~/.claude/.credentials.json`、Pi の `auth.json`、Codex の `auth.json` など）が囲いの中にあることは受け入れる
- Windows 側で動くエージェントやエディターの拡張機能は範囲の外
- 手入れの上限:
  - 囲いの設定は1つにまとめる
  - エージェントを1つ足すのは、設定のファイルを1つ足すだけで済む
  - 普段は触らない。エージェントの版を上げたときだけ、受け入れの確かめを1回流す

## 守るものと受け入れるもの

- **守る**:
  - 手元の鍵（`~/.ssh`、GitHub と Woodpecker のトークン、GitHub App の `.pem`、クラウドの認証、`/mnt/c`）
  - 作業リポジトリの中の `.env`
  - WSL の interop（Windows の実行ファイルを起動させない）
  - ほかのリポジトリと、ホームのほかの場所
  - 囲いの規則と、エージェントの設定だけのファイル（中から書き換えさせない）
  - GitHub の main
- **受け入れる**:
  - 作業中のプロジェクトを壊されること（git で戻す）
  - 作業中のプロジェクトのコードを外へ送られること（外への通信は絞らない）
  - 設定と状態が1つのファイルに混ざっているものを、エージェントが書き換えること（下の「エージェントの置き場」）

## 囲いの作り

### 仕組み

- POSIX sh のスクリプト `bin/agent-box` が bwrap の引数を組み立てて起動する。引数は文字列をつなげず、位置パラメータ（`set -- "$@" …`）で組み立てる
- `exec` で bwrap に置き換えない。bwrap が終わったら `trap` でセッション用のソケットを消す
- 囲いの中でもう一度ラッパーが呼ばれたら（環境変数 `AGENT_BOX=1`）、包まずに本物を exec する（bwrap は入れ子にできない）

### ホームとファイル

- ホームは毎回、空の tmpfs から組み立てる。そこに mount するのは、エージェントの状態の置き場（書ける）、設定（読むだけ）、道具（読むだけ）、囲い専用のキャッシュ（書ける）
- mount の順は、書けるフォルダーが先、その中の読むだけのファイルが後
- 読むだけで mount するファイルやフォルダーが、ホストに無いときは、起動の前にホストの側に作る。ファイルの中身は `{}`。bwrap は、元のパスが無いと起動に失敗するため
- システムは `/usr`・`/etc` などを読むだけで mount する。`/init`・`/run/WSL`・`/mnt` は mount しない
- 作業フォルダー（起動したときのプロジェクト）は書けるように mount する

### `.env`

- 起動のたびに、プロジェクトの中の `.env` と `.env.*` を探し、見つかったものに中身の無いファイルを重ねて読めなくする
- 除く: `*.example`・`*.sample`・`*.template`
- 探さない: `.git`・`target`・`node_modules`・`.venv`
- git に追跡されているものには重ねず、警告を出す（中身は `.git` の履歴から読めるので、重ねても守りにならず、checkout や stash が落ちるだけ）
- 起動したあとに作られた `.env` は、次に起動するまで塞がれない

### WSL の interop

- `/run/WSL` を mount せず、環境変数 `WSL_INTEROP` を消す
- `binfmt_misc` を隠すやり方は使わない。WSL の登録には `F` のフラグが付いていて、`/init` を見せなくても `.exe` を起動すると `/init` が動くため。塞げるのは、interop のソケットに届かないから（`gate.md` の G1）

### ネットワーク

- ホストのネットワークをそのまま使う。T3 のツール（MCP）は `http://127.0.0.1:<port>/mcp` に届く必要があるため
- LAN と、同じ PC のほかのサービスは塞がない。困ったら、ネットワークを分けて T3 にだけ中継する形に移す

### 道具とキャッシュ

- 道具は読むだけで mount する: mise、rustup、cargo の本体、Go、`~/.local/share/claude/versions`、`~/.local/bin/agy`、`~/.pi/agent/bin`、Pi の本体の置き場
- 取得のキャッシュは、囲い専用の書ける場所に分ける: cargo の registry、npm、uv、pnpm、mise の取得物。分けるので、囲いの中で毒されたキャッシュを、ホストのビルドが使うことはない
- ビルドの成果物（`target` など）はプロジェクトの中にあるので、そのまま共有する
- 中では、エージェントの自動更新を切る（Claude は `DISABLE_AUTOUPDATER=1`。Antigravity は T3 が `AGY_CLI_DISABLE_AUTO_UPDATE=1` を渡す。Pi と Codex は受け入れで確かめる）。更新は囲いの外で、それぞれの公式の更新コマンドで流す

### エージェントの置き場

- **読むだけにするもの**（設定だけのファイル）:
  - Claude: `~/.claude/settings.json`、`~/.claude/CLAUDE.md`、hooks
  - Pi: 拡張、`AGENTS.md`、`mcp.json`
  - Antigravity: `~/.gemini/GEMINI.md`、`hooks.json`
  - 共通のスキル: `~/.agents/skills`
- **書けるようにするもの**（設定と状態が混ざっているもの、動いている最中に書かれるもの）:
  - Claude: `~/.claude.json`、memory、plugins、同期されるスキル、会話の記録などの状態
  - Pi: `settings.json`、`trust.json`、sessions などの状態
  - Antigravity: `~/.gemini/antigravity-cli/settings.json` と、状態の置き場
  - Codex: `config.toml`（信頼を書き込むため）と、状態の置き場
- `settings.json` を読むだけにするので、`/config` でテーマなどを変えたときの保存は落ちる。これは受け入れる
- エージェント組み込みの囲いは切る（入れ子にしない）:
  - Claude Code: `settings.json` の `sandbox`
  - Codex: `sandbox_mode = "danger-full-access"`、`approval_policy = "never"`

## 起動の入口

- **外側のラッパー**: `~/.local/share/agent-box/bin/{claude,pi,agy,codex,box}`。どれも `bin/agent-box` へのリンクで、名前でエージェントを選ぶ
  - `PATH` で `~/.local/bin` より前に置く
  - 本物は、設定に書いたフルパスで呼ぶ（`PATH` から探さない）
  - `~/.local/bin` には置かない。もう本物が入っていて、自動更新が張り替えるため
- **囲いの中の道具**: `inner-bin/{gh,git-credential-agent-box,agent-box-ci}`。読むだけで mount する。囲いの中の `PATH` からは、外側のラッパーのフォルダーを外す
- **T3 Code**: プロバイダーごとの Binary path を、画面で1回設定する。値は外側のラッパーのパス。Antigravity の分もファイル名が `agy` なので、T3 の判定（ファイル名が `agy` か `antigravity`）を通る。手順は README に書く
- **`box sh`**: エージェントを起動せずにシェルだけを開く。あなたが依存のインストール（`npm install` など）を囲いの中で流すための入口（使うかは任意）
- エージェントの入れ方:
  - Claude・Pi・Antigravity: 今の公式の curl の入れ方のまま
  - Codex: 移すときに公式の文書を確かめ、公式が勧める入れ方にする（npm なら mise の `npm:`）

## 鍵とトークンの窓口

### GitHub App

- エージェントの GitHub での身元は GitHub App にする（machine user と deploy key は使わない）。PR の作者が `<app>[bot]` になるので、あなたが承認できる
- 権限:
  - 与える: Contents（読み書き）、Pull requests（読み書き）、Issues（読み書き）、Metadata（読む）、Checks と Commit statuses（読む）
  - 与えない: Workflows、Administration、Actions への書き込み、Secrets
- `.pem` は囲いの外に置き、権限は 600。囲いには mount しない

### 窓口

- TypeScript の単一ファイル `broker/token-broker.ts`。Node 26 でそのまま動かす（変換しない構文だけを使う。`erasableSyntaxOnly`）
  - 依存は持たない（`package.json` と `node_modules` も持たない）
  - `fetch` ではなく `node:https` を使い、`node:crypto`・`node:net` も使う
  - `--max-old-space-size=32 --jitless` で起動する
- systemd のユーザーの `.socket` ユニットで受け付ける

### ソケット

- **操作用のソケット**: 囲いの外にあり、権限は 600。囲いには mount しない
- **セッション用のソケット**: 起動スクリプト（外で動く）が操作用のソケットに頼むと、窓口が、そのリポジトリにだけ効くものを作る。起動スクリプトは、それを囲いの中の決まったパスに mount する
  - どのリポジトリに効くかは、起動のときに決める。囲いの中からの頼みでは変えない
  - 起動スクリプトは、囲いが終わるまで操作用のソケットにつながったままにする。つながりが切れたら、窓口がセッションを消す（`kill -9` などで `trap` が動かないときのため）

### トークン

- installation token は、`repositories` を1つに、`permissions` を上の範囲に絞って発行する
- 期限の残りが5分になるまで、窓口のメモリでキャッシュする

### git

`GIT_CONFIG_*` で次を渡す（ホームが tmpfs なので、`~/.gitconfig` には頼らない）:

- `user.name` と `user.email`: bot の名前（`<app>[bot]`、`<Bot User ID>+<app>[bot]@users.noreply.github.com`。Bot User ID は `gh api '/users/<app>[bot]' --jq .id` の値）
- `credential.helper`: `git-credential-agent-box`
  - sh と `curl --unix-socket` で動く（Node を起動しない）
  - 標準入力の `protocol=https` と `host=github.com` を確かめ、それ以外にはトークンを渡さない
- `url.https://github.com/.insteadOf`: `git@github.com:` と `ssh://git@github.com/` の形を `https://github.com/` に書き換える

### gh

- `inner-bin/gh` が、窓口からトークンをもらって `GH_TOKEN` に入れ、本物の gh を exec する

### Woodpecker CI のログ

- 窓口が、セッションのリポジトリの pipeline の一覧とログだけを返す読むだけの口を持つ。Woodpecker のトークンは窓口だけが持ち、囲いには渡さない
- pipeline の番号と step の ID は `/^\d+$/` で確かめる
- repo_id は `GET /api/repos/lookup/{owner}/{repo}` で解決してキャッシュし、数値の repo_id に対して呼ぶ（API の正確な形は、実装のときに Woodpecker の版の文書と照らす）
- ログはメモリに溜めず、そのまま中継する（`upstreamRes.pipe(clientRes)`）
- 囲いの中には、全部のエージェントで共通の取得コマンド `inner-bin/agent-box-ci` を置く。CLAUDE.md・AGENTS.md・GEMINI.md は、`woodpecker-cli` の代わりにこれを使うように書き換える

### 秘密の置き場

秘密は、どのリポジトリにも入れない（agent-box にも dotfiles にも）。chezmoi の管理にも入れない。設定のファイルには、秘密のファイルのパスだけを書く。

| 秘密 | 置き場 | 囲いの中 |
|---|---|---|
| SSH の鍵 | `~/.ssh` | mount しない |
| GitHub App の `.pem` | `~/.config/agent-box/github-app.pem`（権限 600） | mount しない |
| Woodpecker のトークン | `~/.config/agent-box/woodpecker-token`（権限 600） | mount しない。窓口だけが読む |
| installation token | 窓口のメモリ | セッションのソケットから、使うたびにもらう（1時間で切れ、1つのリポジトリに絞る） |
| エージェント自身の認証 | ホームの今の場所（`~/.claude/.credentials.json`、Pi や Codex の `auth.json` など） | 状態として書けるように mount する（受け入れた危険） |

- 窓口は、起動したときに `.pem` と Woodpecker のトークンのファイルの権限を確かめる。600 より広ければ起動しない
- agent-box と dotfiles の `.gitignore` に、鍵・トークン・`.env` などの形を入れる（うっかり入れないため）。ただし、`.gitignore` は `git add -f` やエージェントがファイルに書き写すことを止められない
- **受け入れている危険**: 囲いの中のエージェントは、自分の認証と installation token を読める。これをファイルに書いてコミットし push すれば、リポジトリに入る。囲いでは止められないので、GitHub の push protection（secret scanning が push を断る仕組み）で止める。どの形の秘密まで見つけられるかは、T002 で確かめる

## GitHub と CI の側

- **push protection**: 使うリポジトリで、secret scanning と push protection を有効にする

- **main の ruleset**:
  - 掛ける: PR を必須、承認を1つ必須、CI の通過を必須、force push と削除を禁止
  - Bypass list には、あなたの個人のアカウントだけを入れる（App は入れない）
- **Woodpecker**:
  - 秘密を渡すのは push と tag のときだけにする（pull_request では渡さない）
  - PR の pipeline は、動かす前に承認を要るようにする
  - CODEOWNERS で `.woodpecker/**` と `.woodpecker.y*ml` を守る。ただし、CI は設定以外のコード（justfile、テスト、`build.rs` など）も動かすので、秘密を守る重心は、上の2つ（渡すイベントの制限と承認）に置く

## 設定のファイル

- 形: 1行に `key = value` を1つ書く
  - `#` で始まる行はコメント
  - 値は引用符を付けず、行の終わりまでを値にする（空白を含むパスもそのまま書ける）
  - 一覧は、同じ key を繰り返して書く
  - 知らない key はエラーにして、起動を止める
  - 拡張子は `.conf`。TOML の仲間としてではなく、独自の形として扱う
- エージェントごとの設定（`profiles/<エージェント>.conf`）: 書けるパス、読むだけのパス、本物のバイナリのフルパス、中で渡す環境変数
- 窓口の設定（`broker.conf`）: App ID、`.pem` のパス、Woodpecker のサーバー、Woodpecker のトークンのファイルのパス。秘密そのものは書かない（下の「秘密の置き場」）。窓口（TS）も同じ形を読む。sh と TS の読み方がずれないことはテストで確かめる
- 設定は chezmoi（dotfiles）が囲いの外に置き、囲いには mount しない
- `agent-box check` で、知らない key、無いパス、読むだけと書けるの重なりを、起動の前に一覧で出す
- パスごとの選択肢や入れ子が要るようになったら、JSON に移す

## 残すものとやめるもの

- **残す**: 囲いの中の hk（早く気づくための仕組み）、編集直後の check-file、cc-safety-net
- **やめる**（dotfiles から外す）: verify-commits と accept、agent-git、SPEC-025（Antigravity の hook での包み）、srt、in-srt、ツールごとの拒否の一覧。外すのは、4つのエージェントを全部移したあと

## 道具・テスト・進め方

- **リントと整形**: mise で管理する。TS は oxlint と oxfmt、sh は shellcheck と `shfmt -ln posix`。hk で回す
- **テスト**: 外部のヘルパーは使わない
  - **bats**:
    - mount の順（書けるフォルダーが先、読むだけのファイルが後）
    - 読むだけの設定
    - `.env` に重ねるもの
    - `/run/WSL` を見せないこと、`WSL_INTEROP` を消すこと
    - `WOODPECKER_TOKEN` が囲いに渡らないこと
    - 設定のファイルの読み方
  - **node:test**: 偽の GitHub と偽の Woodpecker を立てて、次を確かめる
    - トークンのリポジトリと権限が絞られること
    - キャッシュの期限が切れたら再発行すること
    - セッションのソケットを消すこと
    - 番号を検証すること
  - **本物の bwrap を使うテスト**: 本人の端末で流す
- **受け入れ**: エージェントを1つ移すごとに、まず記録用の包みで実際の入力と動きを記録する。そのうえで次を確かめる
  - `~/.ssh` が見えない
  - 作業リポジトリの `.env` が読めない
  - `cmd.exe` が動かない
  - `git push` が窓口を通る
  - T3 のツールが動く
- **順番**:
  1. GitHub の側: App を作る、ruleset を掛ける、Woodpecker の秘密と承認を設定する
  2. このリポジトリで、囲いと窓口を実装し、単体テストを書く
  3. Claude → Pi → Antigravity → Codex の順に移し、1つごとに受け入れを確かめる
  4. 4つとも移ったら、dotfiles から古い仕組みを外す
- **dotfiles（chezmoi）が持つもの**:
  - このリポジトリの置き場から `~/.local/share/agent-box/bin` へのリンク
  - 設定のファイル（`profiles/*.conf`、`broker.conf`）
  - systemd のユーザーのユニットを有効にすること

## まだ確かめていないこと（実装や受け入れで確かめる）

- Pi の自動更新を切る方法と、本体の置き場（`~/.pi/agent/install/`）を読むだけにしたときの動き
- 1つのリポジトリに絞った installation token で、ほかの public なリポジトリを読めるか
- 窓口が `--max-old-space-size=32 --jitless` で動くか
- Woodpecker の API の正確な形（lookup とログの取得先）
- 起動したときの `.env` の検索の速さ
- Codex の置き場（`~/.codex`、`CODEX_HOME`）、`config.toml` に信頼を書き込む動き、Codex 組み込みの囲いの切り方
