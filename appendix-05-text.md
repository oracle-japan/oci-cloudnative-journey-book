# OpenTelemetry と OCI APM によるトレースとログの相関

## 「トレースとログの相関」の実践

### OCI Logging Analytics の設定

設定する内容は、表-05 の通りです。設定後、「テスト定義」ボタンをクリックして、「成功」と表示されていれば問題ありません。矢印のプルダウンを展開すると詳細を確認できます。「成功」を確認した後に、「追加」ボタンをクリックします。

| 設定項目 | 内容 |
| :--- | :--- |
| ベース・フィールド | Message |
| サンプルのベース・フィールド・コンテンツ |`2025-04-10T08:35:42.126660771+00:00 stdout F 08:35:42.126 [pool-3-thread-1] INFO  m.FulfillmentService#149  traceId: f700955b941efdc6672f66542f90e675 spanId: 5dd4792bad791127 Sending shipment update {\"orderId\":4053,\"shipment\":{\"id\":\"a55855c6-1c90-47a8-9d4c-bfd60a1c1afa\",\"name\":\"Shipped\"}}` |
| 式の抽出 | `traceId:\s+{Trace ID:\S+}\s+spanId:` |

表-05「拡張フィールド定義の追加」の設定内容

| 設定項目 | 内容 | 
| :--- | :--- | 
|名前 | トレースとログの相関 |
|URL | 「問い合わせ URL の取得」で作成したURL <br> https://cloud.oracle.com/loganalytics/explorer?viz=records&query=%27Log+Source%27+%3D+%27OCI+Unified+Schema+Logs%27+and+%27Trace+ID%27+%3D+<TraceId>&vizOptions=%7B%22customVizOpt%22%3A%7B%22primaryFieldIname%22%3A%22mbody%22%2C%22primaryFieldDname%22%3A%22Original%20Log%20Content%22%7D%7D&scopeFilters=lg%3Aroot%2Ctrue%3Brs%3Aroot%2Ctrue%3Brg%3Aap-tokyo-1&timeNum=15&timeUnit=MINUTES&region=ap-tokyo-1&tenant=xxxxxxxxxxxx |

表-06 「ドリルダウンの作成」の設定値

以上で、OCI APM のドリルダウン設定は完了です。