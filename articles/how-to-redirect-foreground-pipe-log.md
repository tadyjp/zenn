---
title: "ローカルのサーバーログを Claude Code に読ませる"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "claudecode", "vite"]
published: true
---

[Vite](https://ja.vite.dev/) など、フォアグラウンドで起動するプロセスのログを、Claude Code や Gemini CLI 等に見てもらいたいときに使うコマンド。

標準出力・エラーをログファイルに書き出しつつ、もとのままターミナルにも出力するため、開発体験を損なわず、かつ Claude Code に認識してもらいやすい。

![](/images/how-to-redirect-foreground-redirect-log/image.png)

## スクリプト

以下のコードをお好きな場所へ (./bin/redirect-log など)


```shell
#!/bin/bash

# redirect-log: 画面にカラー表示しつつ、ログファイルにプレーンテキストで記録
# 使用方法: ./bin/redirect-log -- command [args...]
# 例: ./bin/redirect-log -- npm run dev

# 引数チェック
if [ "$1" != "--" ]; then
    echo "Usage: $0 -- command [args...]" >&2
    exit 1
fi
shift  # "--" を削除

if [ $# -eq 0 ]; then
    echo "Error: No command specified" >&2
    exit 1
fi

# プロジェクトルートとログディレクトリの設定
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
LOG_DIR="$PROJECT_ROOT/logs"
mkdir -p "$LOG_DIR"

# ログファイル名の生成: YYYY-MM-DD_HH-MM-SS_コマンド名.log
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
COMMAND_NAME=$(basename "$1" | tr '/' '_' | tr ' ' '_')
LOG_FILE="$LOG_DIR/${TIMESTAMP}_${COMMAND_NAME}.log"

# ヘッダー情報をログファイルに記録
{
    echo "========================================"
    echo "Log started at: $(date)"
    echo "Command: $*"
    echo "Working directory: $(pwd)"
    echo "========================================"
    echo ""
} >> "$LOG_FILE"


# フッターが既に書かれたかを記録
FOOTER_WRITTEN=false

# 終了時にフッターを確実に書き込む（重複防止付き）
write_footer() {
    if [ "$FOOTER_WRITTEN" = "false" ]; then
        FOOTER_WRITTEN=true
        {
            echo ""
            echo "========================================"
            echo "Log ended at: $(date)"
            echo "Exit status: ${EXIT_STATUS:-$?}"
            echo "========================================"
        } >> "$LOG_FILE" 2>/dev/null
    fi
}

# 終了時にフッターを書き込む（EXITのみで十分）
trap write_footer EXIT

# Node.js系ツールのカラー出力を強制
export FORCE_COLOR=1

# コマンド実行: 画面にカラー表示、ログにプレーンテキスト保存
"$@" 2>&1 | tee >(sed -u 's/\x1B\[[0-9;]*[a-zA-Z]//g' >> "$LOG_FILE")

# 終了ステータスを保持
EXIT_STATUS=${PIPESTATUS[0]}

exit $EXIT_STATUS
```

## 実行方法

```shell
./bin/redirect-log -- npm run dev
```


## ブラウザのコンソールログも一緒に見たい

Vite なら、 https://github.com/mitsuhiko/vite-console-forward-plugin をいれる。

(本当は、同じものを作ろうとしてたが、先駆者がいた。ぐぬぬ。)
