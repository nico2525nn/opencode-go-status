このリポジトリには3つのツールが入っています:

| ファイル | 説明 |
|---|---|
| `opencode-go-status` | モニタ + セットアップ + 単発ベンチ |
| `go-usage` | OpenCode Go サブスク残量を**複数ワークスペース一括**で表示 |
| `ox-alpha-bench`(※) | ox-alpha の3プロバイダTPS/TTFTベンチ(※個人用のためリポジトリ外) |

## 特徴

- **毎分チェック**: 最小プロンプトのストリーミング要求(1回約12トークン)
- **性能記録**: TTFB(最初のSSEバイト)/ TTFT(最初のトークン)/ latency(完了まで) を JSONL でログ
- **TPS計測**: 1時間ごとに固定長生成プローブで tokens/sec を測定(間隔変更可、0で無効)
- **通知**: 状態変化(ok↔error)のみデスクトップ通知。新しい通知が前のを**置き換え**、最新1件だけ表示
- **自己完結セットアップ**: APIキー取得・保存、systemd ユーザーサービス登録を一括
  (systemd が無い環境は XDG autostart に自動フォールバック)

## 必要環境

- Python 3.8+ (標準ライブラリのみ・追加依存なし)
- `notify-send` (通知用。無くてもモニタ自体は動きます)
- OpenCode Go の API キー ([OpenCode Zen](https://opencode.ai/zen) で取得)
- systemd ユーザーセッション(自動起動用。無ければ autostart フォールバック)

## インストール

この1ファイルをPCにコピーして `--setup install` を実行するだけです:

```bash
cp opencode-go-status ~/.local/bin/
~/.local/bin/opencode-go-status --setup install
```

`--setup install` が行うこと:

1. 実行場所に関わらず、自分自身を `~/.local/bin/opencode-go-status` にコピー
2. APIキーを取得(`OPENCODE_GO_API_KEY` 環境変数 → 既存キーファイル → 対話入力の順)し
   `~/.config/opencode-go-status/key` (mode 600) に保存
3. 設定ファイル `~/.config/opencode-go-status/config` を作成
4. systemd ユーザーサービスを登録・起動(ログイン時に自動起動)

## 使い方

| コマンド | 説明 |
|---|---|
| `opencode-go-status` | モニタ実行(毎分チェック) |
| `opencode-go-status --once` | 1回だけチェックして終了 |
| `opencode-go-status --notify-now` | 1回チェックして必ず通知 |
| `opencode-go-status --bench --model <モデル>` | 任意のモデルで1回だけTPS/TTFTを計測 |
| `opencode-go-status --setup install [設定...]` | 設定保存 + サービス再登録 |
| `opencode-go-status --setup uninstall` | サービス登録を解除(設定・キーは保持) |
| `opencode-go-status --setup status` | 設定・サービス状態・最新ステータス表示 |
| `opencode-go-status --setup test-notify` | 最新ステータスでテスト通知 |
| `opencode-go-status --setup set-key` | APIキー再設定 |

## 設定

優先順位: **CLIフラグ > 設定ファイル > デフォルト**

設定ファイル: `~/.config/opencode-go-status/config` (INI形式)

| キー | デフォルト | 説明 |
|---|---|---|
| `interval` | `60` | チェック間隔(秒) |
| `timeout` | `30` | リクエストタイムアウト(秒) |
| `notify` | `change` | `change`(状態変化時)/ `always`(毎分)/ `off`(無効) |
| `tps_interval` | `3600` | TPS計測間隔(秒)。`0` で無効 |
| `log` | `~/opencode-go-status.log` | JSONLログのパス |
| `provider` | `go` | プロバイダプリセット: `go`(定額サブスク)/ `zen`(Zen残高・従量) |
| `endpoint` | (provider別) | APIエンドポイント(指定するとproviderを上書き) |
| `model` | `deepseek-v4-flash` | モデルID |

設定変更の例:

```bash
# 通知を毎分に、TPS計測を2時間おきに
opencode-go-status --setup install --notify always --tps-interval 7200
# 実行中のモニタへ反映
systemctl --user restart opencode-go-status.service
```

## ログ形式

`~/opencode-go-status.log` に1行=1チェックの JSONL で追記されます:

```json
{"ts": "2026-08-10T19:00:00+09:00", "status": "ok", "http": 200, "ttfb_ms": 825,
 "ttft_ms": 1265, "latency_ms": 1405, "reason": null, "tps": null, "probe": "ping",
 "model": "deepseek-v4-flash", "reply": "pong",
 "usage": {"prompt": 10, "completion": 2, "total": 12}}
```

- `status`: `ok` / `error`(エラー時は `reason` に理由)
- `ttfb_ms`: 最初のSSEバイトまで / `ttft_ms`: 最初のトークンまで / `latency_ms`: ストリーム完了まで
- `probe`: `ping`(毎分) / `tps`(TPS計測時)。`tps` は tokens/秒

## ベンチマーク

任意のモデルで **1回だけ** TPS / TTFT を計測します(モニタのログには書かれません):

```bash
opencode-go-status --bench --model deepseek-v4-pro
opencode-go-status --bench --model grok-4.5 --json           # 機械可読出力
opencode-go-status --bench --model opencode-go/kimi-k3       # プレフィックスは自動除去
opencode-go-status --bench --model deepseek-v4-flash --provider zen   # Zen(従量)経由
opencode-go-status --bench --model X --endpoint https://…  # 任意のOpenAI互換API
```

APIキーは `go` と `zen` で共通です。Zenの残高が無いと `CreditsError: Insufficient balance`
が返ります(残高は https://opencode.ai/zen で確認)。モニタをZen経由にする場合は
`opencode-go-status --setup install --provider zen` の後、サービスを再起動してください。

出力例:

```
model:    deepseek-v4-flash
endpoint: https://opencode.ai/zen/go/v1/chat/completions
status:   ok  (http 200)
ttfb:     1.22s
ttft:     8.62s  (first content token)
          first token: 1.22s (incl. reasoning)
latency:  10.44s
tokens:   93 prompt + 959 completion = 1052 total
tps:      104.0 tokens/s (959 tokens / 9.22s stream)
```

- 固定長プロンプトのストリーミング生成で計測。デフォルト最大1024トークン(`--max-tokens` で変更)
- `ttft` = 最初の**表示コンテンツ**トークンまで。reasoningが先行するモデルでは最初のトークン(推理込み)も併記
- `tps` = completionトークン数 ÷ ストリーム時間(最初のトークン〜完了)。reasoningトークンも含む実スループット
- thinkingパラメータは送信しません(Goエンドポイントは `thinking: disabled` を誤処理し、contentを返さなくなるため)

## 通知

- デフォルトは**状態変化時のみ**(`notify=change`)。DOWN時は critical で理由、
  復帰時は OK で latency / TTFT / 最新TPS を表示
- 新しい通知は古い通知を置き換えるため、**最新1件だけ**が画面に残ります
- 通知に使うのは `notify-send`(libnotify)。`-r` 非対応の古い環境ではスタック表示に自動フォールバック

## コスト

| プローブ | トークン/回 | 頻度 | コスト目安(月) |
|---|---|---|---|
| ping | 12 | 毎分 | ≈ $0.09 |
| TPS | 88 | 1時間ごと | ≈ $0.02 |

(DeepSeek V4 Flash: input $0.14 / output $0.28 per 1M tokens で計算)

## go-usage — サブスク残量の一括表示

OpenCode Go の利用枠(5時間 / 週次 / 月次)を**複数ワークスペース分まとめて**表示します。
サブスクを複数(メイン+サブワークスペース等)持っている場合に各アカウントのキーを登録して
一括確認できます。

```bash
go-usage                  # 一覧表示(使用率高は色分け: 緑<60% / 黄60-89% / 赤>=90%)
go-usage --add main       # キーを対話入力で登録(ラベル付き、ファイルは mode 600)
go-usage --add sub        # 2枠目以降も同じ
go-usage --remove sub     # 削除
go-usage --json           # 機械可読出力
```

キーは `~/.config/go-usage/config` の `[keys]` セクションに保存されます
(API: `GET https://opencode.ai/zen/go/v1/usage`、`percent` と `resetsAt` を返します)。

## ライセンス

## ライセンス

[MIT](LICENSE)
