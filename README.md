# agent-box

WSL2 の中で動く CLI のコーディングエージェント（Claude Code・Pi・Antigravity・Codex）を、1つの囲いで包むための道具です。囲いは bwrap を使い、見せるものだけを mount します。GitHub のトークンは、囲いの外の窓口が GitHub App から作って渡します。

仕様は [specs/001-agent-box](specs/001-agent-box) にあります。セットアップの手順は、実装に合わせてここに書き足します。
