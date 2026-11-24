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
