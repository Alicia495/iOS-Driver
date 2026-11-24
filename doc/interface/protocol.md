# HTTP・WebSocket仕様

## 共通

APIの起点は `/api/v1`。HTTPのデータ形式はJSON、画像は対応する画像メディア型、シナリオの移行形式はZIPとする。時刻はUTCのISO 8601、区間はミリ秒。端末やステップはホストのIDで参照する。エラーは `code`、`message`、必要に応じて `details` を返す。

## HTTP

| メソッド・パス | 用途 |
| --- | --- |
| `GET /appium-endpoints`・`POST /appium-endpoints`・`PATCH /appium-endpoints/{id}` | 接続先の一覧、登録、変更。端末は登録済み接続先のIDを参照 |
| `GET /devices`・`POST /devices` | 端末一覧と登録 |
| `PATCH /devices/{id}`・`POST /devices/{id}/connection-test` | 端末設定の変更と接続確認 |
| `GET /devices/{id}/screenshot` | 素材作成・確認用の静止画。フレームIDと寸法・向きはレスポンスヘッダーで返す |
| `POST /devices/{id}/control-lease`・`DELETE /devices/{id}/control-lease` | 手動操作権の取得と解放。取得時に操作セッションのIDと有効期限を返す |
| `POST /recordings`・`GET /recordings/{id}`・`POST /recordings/{id}/finish` | 記録開始、下書き再取得、記録終了 |
| `GET /scenarios`・`POST /scenarios`・`GET /scenarios/{id}` | シナリオ一覧、作成、詳細 |
| `POST /scenarios/{id}/versions` | 編集内容を検証して新しい保存版を作る |
| `POST /scenarios/import`・`GET /scenarios/{id}/versions/{versionId}/export` | シナリオと画像の移行 |
| `POST /templates`・`POST /templates/{id}/test` | 切り出し画像を登録し、保存画像または現在の画面で判定を試す |
| `GET /runs`・`POST /runs`・`GET /runs/{id}` | 履歴、実行の作成、端末別結果。作成時に版と対象端末IDの一覧を指定 |
| `POST /runs/{id}/cancel`・`POST /runs/{id}/devices/{deviceId}/cancel` | 全体または対象端末へ停止要求 |
| `GET /artifacts/{id}` | 閲覧権のある診断画像を取得 |

実行作成はクライアントが指定するリクエストIDで重複を確認し、同じID・同じ内容の再送では既存実行を返す。既存版と異なる編集競合、端末操作権の競合は `409` とする。形式や設定の検証エラーは `422`。接続試験など時間のかかる処理は `202` を返し、結果を状態イベントで通知する。

## 制御・状態のWebSocket

`/api/v1/ws/control` はJSONで通信する。操作者は `lease.renew` で有効期限を延長できる。入力イベント `gesture.submit` は操作ID、端末ID、操作セッションID、フレームID、種別、点、必要な場合は長押しやスワイプの時間を含む。タップの例を示す。

```json
{
  "type": "gesture.submit",
  "operationId": "6da2ca14-b8cb-4630-a984-927f1eeb7886",
  "deviceId": "device-a",
  "leaseId": "lease-a",
  "frameId": "frame-584",
  "gesture": { "type": "tap", "point": { "x": 0.5, "y": 0.72 } }
}
```

ホストから `gesture.result` で操作IDと結果を返す。結果は `applied`、`rejected`、`unknown`。結果通知が届かなかった場合、クライアントは `gesture.status` に同じ操作IDを指定して照会する。ホストは既知の状態を返し、端末コマンドは重複送信しない。ホストは `device.updated`、`recording.updated`、`run.updated`、`run.device.updated` を送る。WebSocketが切れたクライアントはHTTPで現在状態を再取得する。

## 映像用WebSocket

`/api/v1/ws/devices/{id}/preview` は1端末を購読する。1フレームにつきメタデータJSONを1件、その直後に対応するJPEGのバイナリメッセージを1件送る。メタデータには `frameId`、`width`、`height`、`orientation`、`receivedAt`、`geometryRevision` を含める。ブラウザは画像の描画が終わった時点でそのIDを入力に利用できるようにする。

遅い購読先にはフレーム単位で間引く。送信中のペアは途中で破棄せず、次に送る候補を最新の1枚へ更新する。ホストは端末セッション・画面の世代が一致する新しいフレームに限り操作を受け付ける。認証方式と接続範囲は[運用設計](../operations/deployment.md)に従う。
