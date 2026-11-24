# Appium／WDA 接続設計

## 初回導入と接続プロファイル

1. iPhoneを対象の接続ホストに信頼させ、開発者モードと必要なUI Automation設定を確認する。初期開発ではMacのXcodeでペアリングし、USBを外してネットワーク接続を確認する。
2. UDIDを開発チームに登録し、実機で有効な署名付きWDAを準備する。対象iOSとAppium・XCUITestドライバ・WDAの組み合わせは固定して台帳に残す。
3. iOS Driveに端末名、UDID、OSバージョン、Appium接続先、WDA起動方式、映像転送設定を登録し、セッションから画面取得と操作を確認する。

| プロファイル | 方針 |
| --- | --- |
| Mac初期開発 | Xcodeで準備したWDAをAppiumから起動。必要に応じて対象iOSに合うRemoteXPC構成を使う。 |
| Windows直接 | 署名済みWDAを用意し、Apple Mobile DeviceドライバとRemoteXPCを設定。各セッションにUDIDとOS版を指定し、導入済みWDA起動または起動済みWDAへの接続を使用する。 |
| 別Mac上のAppium | iOS DriveからリモートAppiumへ接続する。ホストから届く端末別のMJPEG転送先も用意する。 |

Windowsでの無線運用は、端末がネットワーク接続として認識され、USBなしでセッション準備、画面取得、タッチ操作、切断からの復帰が通ることを条件とする。本設計時点で実機確認は済んでいない。

## 端末操作と座標

| 端末接続層の処理 | 実装方針 |
| --- | --- |
| 状態取得 | セッションID、画面の向き、現在の論理寸法と表示領域を取得。再接続や向きの変化で更新する。 |
| 撮影 | 自動化判定用に通常のスクリーンショットを要求。画面の向き、実画像寸法、取得IDを付与する。 |
| プレビュー | WDAのMJPEGを選択端末について中継。ストリームが利用できない場合は低頻度の撮影に切り替え、画面に状態を表示する。 |
| タッチ入力 | AppiumのW3C Actionsなどを介して、タップ、長押し、スワイプを論理座標で送る。 |

正規化座標の原点は表示画像の左上とし、右・下を正方向にとる。ホストは、画像の向き、表示領域、実際の論理座標範囲を照合した後に変換する。画像と論理画面の対応が取れている場合、正規化点を対応矩形へ拡大し、右端・下端は範囲内の最大値に丸める。黒帯やブラウザの余白は座標計算から除く。セッションの画面情報が入力の参照フレームと食い違う場合は拒否し、フレームを取り直す。

端末間の差は、モデル・向き・論理寸法で管理する画面プロファイルで扱う。固定座標入力は作成時のプロファイルと検証済み互換プロファイルに限定する。画像から得た座標も同じ変換を通す。

## 並列接続

Appium接続先ごとにUDIDとポートの対応を管理し、ホスト側のWDA転送ポートとMJPEG転送ポートをセッション間で重複させない。各端末の再接続は段階的に試行し、同じAppiumホストに再起動処理を集中させない。ライブ映像、コマンド、認識用撮影の失敗を分けて表示する。

[AppiumのWindows対応](https://appium.github.io/appium-xcuitest-driver/latest/guides/non-macos-hosts/) · [RemoteXPCと並列セッション](https://appium.github.io/appium-xcuitest-driver/latest/guides/remotexpc-tunnels-real-devices/) · [MJPEG](https://appium.github.io/appium-xcuitest-driver/latest/guides/mjpeg/) · [Appleの無線ペアリング](https://help.apple.com/xcode/mac/current/en.lproj/devbc48d1bad.html)
