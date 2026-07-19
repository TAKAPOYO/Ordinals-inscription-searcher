# Claude Code 音声通知セットアップ手順（紅谷社長向け）

タスクが完了したら「タスクが完了したのだ」、確認待ちになったら「ClaudeCodeが呼んでいるのだ」と、ずんだもんの声で知らせてくれる仕組みです。VSCode + WSL2でClaude Codeを使っている前提の手順です。

---

## 事前準備（TAKA側で用意するもの）

- `notification.wav`（通知音声）
- `task_completion.wav`（完了音声）

この2ファイルをTAKAから受け取ってください。

---

## セットアップ手順

### ① 音声ファイルの配置

VSCodeの内蔵ターミナルで以下を実行し、Windows側の保存フォルダを作ります。

```bash
mkdir -p /mnt/c/temp/voice
```

受け取った `notification.wav` と `task_completion.wav` を、エクスプローラーで `C:\temp\voice\` にコピーしてください。

### ② hookスクリプトの配置

VSCodeの内蔵ターミナルで以下を順番に実行します。

```bash
mkdir -p ~/.claude/hooks-scripts
```

続けて、下記3ファイルを作成します（`cat > ファイル名 << 'EOF' ... EOF` の形でそのまま貼り付けて実行してください）。

```bash
cat > ~/.claude/hooks-scripts/play-voice.sh << 'EOF'
#!/bin/bash
set -euo pipefail

HOOK_NAME="$1"
VOICE_DIR="/mnt/c/temp/voice"

case "$HOOK_NAME" in
  Notification) VOICE_FILE="notification.wav" ;;
  Stop)         VOICE_FILE="task_completion.wav" ;;
  *)            exit 0 ;;
esac

FULL_PATH="${VOICE_DIR}/${VOICE_FILE}"
[ ! -f "$FULL_PATH" ] && exit 0

WINDOWS_PATH=$(echo "$FULL_PATH" | sed 's#/mnt/c#C:#')
powershell.exe -Command "(New-Object Media.SoundPlayer '${WINDOWS_PATH}').PlaySync()" 2>/dev/null || true
EOF

cat > ~/.claude/hooks-scripts/Stop.sh << 'EOF'
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
"${SCRIPT_DIR}/play-voice.sh" "Stop"
EOF

cat > ~/.claude/hooks-scripts/Notification.sh << 'EOF'
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
"${SCRIPT_DIR}/play-voice.sh" "Notification"
EOF

chmod +x ~/.claude/hooks-scripts/*.sh
```

### ③ settings.jsonにhookを登録

`~/.claude/settings.json` をVSCodeで開きます（内蔵ターミナルで `code ~/.claude/settings.json`）。

`"hooks"` という項目がすでにあればその中に、なければファイル全体を囲む `{ }` の中に、以下を追記してください。

```json
"hooks": {
  "Notification": [
    {
      "matcher": "",
      "hooks": [
        { "type": "command", "command": "~/.claude/hooks-scripts/Notification.sh" }
      ]
    }
  ],
  "Stop": [
    {
      "matcher": "",
      "hooks": [
        { "type": "command", "command": "~/.claude/hooks-scripts/Stop.sh" }
      ]
    }
  ]
}
```

既存の設定と併記する場合はJSONのカンマ区切りに注意してください（不安な場合はTAKAに画面共有で見てもらうのが確実です）。

### ④ 動作確認

VSCodeを再起動し、Claude Codeで何か適当な指示を出して完了するのを待ちます。「タスクが完了したのだ」と聞こえればOKです。

鳴らない場合は以下を確認：

1. `ls /mnt/c/temp/voice/` でWAVファイルが2つとも存在するか
2. Windows側のスピーカー音量・既定の再生デバイスがミュートになっていないか
3. 一度新しいターミナル（新規セッション）を開き直して試す

---

## 補足

- セリフは固定です（カスタマイズなし版）。将来セリフを変えたい場合は別途VOICEVOX MCP方式への切り替えが必要になります。
- この仕組みはTAKA自身の環境（WSL2 + PowerShell再生）と同一構成です。
