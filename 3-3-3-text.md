# 3-3-3. パフォーマンス監視

#### 1. OpenTelemetry Operator インストール

MuShop が稼働している OKE クラスタに OpenTelemetry Operator をインストールします。  
OCI Cloud Shell を開いて、以下のコマンドを実行します。最初に 「open-telemetry」の Helm リポジトリを登録します。

```sh
$ helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/<your-directory>/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /home/<your-directory>/.kube/config
"open-telemetry" has been added to your repositories
```

OpenTelemetry Operator をインストールします。

```sh
$ helm install opentelemetry-operator open-telemetry/opentelemetry-operator --namespace mushop \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/<your-directory>/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /home/<your-directory>/.kube/config
NAME: opentelemetry-operator
LAST DEPLOYED: Wed Apr 30 07:36:17 2025
NAMESPACE: mushop
STATUS: deployed
REVISION: 1
NOTES:
[WARNING] No resource limits or requests were set. Consider setter resource requests and limits via the `resources` field.


opentelemetry-operator has been installed. Check its status by running:
  kubectl --namespace mushop get pods -l "app.kubernetes.io/name=opentelemetry-operator"

Visit https://github.com/open-telemetry/opentelemetry-operator for instructions on how to create & configure OpenTelemetryCollector and Instrumentation custom resources by using the Operator.
```

インストールされていることを確認します。

```sh
$ kubectl get deploy opentelemetry-operator -n mushop
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
opentelemetry-operator   1/1     1            1           2m
```

以上で、OpenTelemetry Operator のインストールは完了です。

#### 2. 自動計装の設定

OCI Cloud Shell のホームディレクトリにある「mushop-devops」ディレクトリ内の「otel.yaml」マニフェストを開いて、事前に取得したエンドポイントとデータキー（プライベート）を設定します。  
この「otel.yaml」マニフェストが、先に説明した Instrumentation カスタムリソースの定義ファイルです。  
ファイル内の「< your-data-upload-endpoint >」と「< your-private-data-key >」を取得した内容に書き換えて、保存します。

※ < your-data-upload-endpoint > と < your-private-data-key > の <>は設定値には不要です。以降の手順でも同様です。  
例1：< your-data-upload-endpoint > ⇒ https://xxxxxxxxxxxxxxx.apm-agt.ap-tokyo-1.oci.oraclecloud.com  
例2：< your-private-data-key > ⇒ CLQKHXXXXXXXXXXXXXXXXXXXXXXXXX26  

```sh
$ cd
```
```sh
$ vim mushop-devops/deploy/complete/opentelemetry/otel.yaml
```
```yaml
・
・＜省略＞
・
      - name: OTEL_com_oracle_apm_agent_data_upload_endpoint
        value: <your-data-upload-endpoint>
      - name: OTEL_com_oracle_apm_agent_private_data_key
        value: <your-private-data-key>
      - name: OTEL_com_oracle_apm_agent_config_trace_jvmargs_createChildSpans
        value: "false"
      - name: OTEL_com_oracle_apm_agent_config_trace_jvmargs_createSpan
        value: "false"
```

保存した「otel.yaml」を適用します。

```sh
$ kubectl apply -f mushop-devops/deploy/complete/opentelemetry/otel.yaml
instrumentation.opentelemetry.io/inst-apm-java created
```

インストールされていることを確認します。

```sh
$ kubectl get instrumentation -n mushop
NAME            AGE     ENDPOINT   SAMPLER                    SAMPLER ARG
inst-apm-java   6m18s              parentbased_traceidratio   1
```
MuShop アプリケーションの内、トレース対象となる Java 実装のコンポーネント（orders、carts、fulfillment）を再起動します。

```sh
$ kubectl rollout restart deploy mushop-carts -n mushop
deployment.apps/mushop-carts restarted
```
```sh
$ kubectl rollout restart deploy mushop-fulfillment -n mushop
deployment.apps/mushop-fulfillment restarted
```
```sh
$ kubectl rollout restart deploy mushop-orders -n mushop
deployment.apps/mushop-orders restarted
```

再起動後、orders、carts、fulfillment の Pod が全て Running であることを確認します。  
さらに、OpenTelemetry Operator と Instrumentation によって OCI APM Agent と OCI APM Agent が利用する環境変数がインジェクションされていることを確認するために、ここでは、orders の Pod 名を取得しておきます。carts、fulfillment も同じ手順で確認できます。

```sh
$ kubectl get pods -n mushop
NAME                                      READY   STATUS    RESTARTS   AGE
・
・＜省略＞
・
mushop-carts-5796446dd-dxwjh              1/1     Running   0          5m14s
mushop-fulfillment-8478cf4969-t2kcr       1/1     Running   0          5m4s
mushop-orders-5c8c7f7d7-69czt             1/1     Running   0          4m57s
・
・＜省略＞
・
```

orders Pod の詳細を確認します。アプリケーションの環境変数に Instrumentation で定義した環境変数がインジェクションされています。

```sh
$ kubectl describe pod mushop-orders-5c8c7f7d7-69czt -n mushop
Name:             mushop-orders-5c8c7f7d7-69czt
Namespace:        mushop
・
・＜省略＞
・
      OTEL_com_oracle_apm_agent_data_upload_endpoint:                   https://aaaadfuerh3x6aaaaaaaaadu4m.apm-agt.ap-tokyo-1.oci.oraclecloud.com
      OTEL_com_oracle_apm_agent_private_data_key:                       ZZM3A73KNZKDGGH2KL7N32WYNK54IN32
・
・＜省略＞
・
```

また、OCI APM Agent は、Init Container としてアプリケーションにインジェクションされており、以下の部分で確認できます。

```sh
・
・＜省略＞
・
Init Containers:
  opentelemetry-auto-instrumentation-java:
    Container ID:  cri-o://6b1b9a234a2b5b00fbe500f8c2f8312c78fc7a4d5fe6fe1dc5d58afdd95296bc
    Image:         container-registry.oracle.com/oci_observability_management/oci-apm-java-agent
    Image ID:      container-registry.oracle.com/oci_observability_management/oci-apm-java-agent@sha256:4e682177b7e45720968981b9fc5af487329ef1aa7768434d07fa3355bbc5e0a0
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      /javaagent.jar
      /otel-auto-instrumentation-java/javaagent.jar
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 30 Apr 2025 08:07:24 +0000
      Finished:     Wed, 30 Apr 2025 08:07:24 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  256Mi
    Requests:
      cpu:        50m
      memory:     64Mi
    Environment:  <none>
    Mounts:
      /otel-auto-instrumentation-java from opentelemetry-auto-instrumentation-java (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-cjh9p (ro)
・
・＜省略＞
・
```

この設定により、OCI APM に対して、JMX メトリクスおよびトレース、スパンをアップロードできます。  
以上で、自動計装の設定は完了です。

### 3. フロントエンドアプリケーションへのエンドポイントとパブリックデータキーの設定

フロントエンドアプリケーションが OCI APM のブラウザエージェントを利用し、モニタリング情報をアップロードする設定を行います。  
OCI Cloud Shell のホームディレクトリにある「mushop-devops」ディレクトリ内の「_apm.pug」ファイルを開いて、事前に取得したエンドポイントとデータキー（パブリック）を設定します。  
ファイル内の「< your-data-upload-endpoint >」と「< your-public-data-key >」を取得した内容に書き換えて、保存します。

```sh
$ vim mushop-devops/src/storefront/src/templates/partials/_apm.pug
```
```html
script.
    window.apmrum = (window.apmrum || {}); 
    window.apmrum.serviceName='Mushop-Service';
    window.apmrum.webApplication='Mushop';
    window.apmrum.ociDataUploadEndpoint='<your-data-upload-endpoint>';
    window.apmrum.OracleAPMPublicDataKey='<your-public-data-key>';
    window.apmrum.traceSupportingEndpoints =  [ { headers: [ 'APM' ], hostPattern: '.*' }, { headers: [ ], hostPattern: '.*' }]; 
script(async, crossorigin='anonymous', src='<your-data-upload-endpoint>/static/jslib/apmrum.min.js')
```

OCI DevOps のコードリポジトリにプッシュするため、バージョンファイル内の値「0.0.1」を「0.0.2」に変更して保存します。

```sh
$ vim mushop-devops/appVersion
```
```sh
0.0.1 ⇒ 0.0.2
```

更新したソースコードを OCI DevOps のコードリポジトリにプッシュします。

```sh
$ cd mushop-devops
```
```sh
$ git add .
```
```sh
$ git commit -m "add apm info"
[main 61db9fe] add apm info
 3 files changed, 6 insertions(+), 8 deletions(-)
```

「Unsername」は、「テナンシ名/メールアドレス」、「Password」は、認証トークンです。「3-2-4-2. OCI DevOps による CI/CD」で利用したものと同じものを使用します。

```sh
$ git push
Username for 'https://devops.scmservice.ap-tokyo-1.oci.oraclecloud.com': <your-tenacny-name>/<your-mailadress>
Password for 'https://<your-tenacny-name>/<your-mailadress>@devops.scmservice.ap-tokyo-1.oci.oraclecloud.com': <your-
authentication-token>
Enumerating objects: 25, done.
Counting objects: 100% (25/25), done.
Delta compression using up to 2 threads
Compressing objects: 100% (10/10), done.
Writing objects: 100% (13/13), 1003 bytes | 1003.00 KiB/s, done.
Total 13 (delta 9), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (9/9)
To https://devops.scmservice.ap-tokyo-1.oci.oraclecloud.com/namespaces/nryejxjmyppf/projects/mushop/repositories/mushop-devops
   d8b976e..61db9fe  main -> main
```

「3-2-4-2. OCI DevOps による CI/CD」と同様に、OCI DevOps の CI/CD パイプラインが自動的に実行されます。デプロイが完了するまで待ちます。  
これで、OCI APM によるトレースおよびリアル・ユーザー・モニタリングを実施できます。  
以上で、トレースおよびリアル・ユーザー・モニタリングの事前準備は完了です。

### 2. OpenTelemetry Java Agent によるトレースの可視化

OCI Cloud Shell のホームディレクトリにある「mushop-devops」ディレクトリ内の「otel.yaml」マニフェストを開いて、OpenTelemetry Java Agent を有効にする設定を行います。  
この設定によって、OCI APM で、OpenTelemetry Java Agent 使用したトレースを実施できます。

```sh
$ cd
```
```sh
$ vim mushop-devops/deploy/complete/opentelemetry/otel.yaml
```

本項「2. 自動計装の設定」で設定した項目と 「image: container-registry.oracle.com/oci_observability_management/oci-apm-java-agent」 部分をコメントアウトします。次に、上部6つの項目のコメントアウトを外します。さらに「< your-data-upload-endpoint >」と「< your-private-data-key >」を書き換えて、保存します。

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: inst-apm-java
  namespace: mushop
spec:
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "1"
  java:
    # image: container-registry.oracle.com/oci_observability_management/oci-apm-java-agent
    env:
      - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
        value: <your-data-upload-endpoint>/20200101/opentelemetry/private/v1/traces
      - name: OTEL_EXPORTER_OTLP_TRACES_PROTOCOL
        value: http/protobuf
      - name: OTEL_EXPORTER_OTLP_HEADERS
        value: Authorization=dataKey <your-private-data-key>
      - name: OTEL_EXPORTER_OTLP_TRACES_HEADERS
        value: Authorization=dataKey <your-private-data-key>
      - name: OTEL_METRICS_EXPORTER
        value: none
      - name: OTEL_LOGS_EXPORTER
        value: none
      # - name: OTEL_com_oracle_apm_agent_data_upload_endpoint
      # value: <your-data-upload-endpoint>
      # - name: OTEL_com_oracle_apm_agent_private_data_key
      #  value: <your-private-data-key>
      # - name: OTEL_com_oracle_apm_agent_config_trace_jvmargs_createChildSpans
      #  value: "false"
      # - name: OTEL_com_oracle_apm_agent_config_trace_jvmargs_createSpan
      #  value: "false"
```

更新した「otel.yaml」を適用します。

```sh
$ kubectl apply -f mushop-devops/deploy/complete/opentelemetry/otel.yaml
instrumentation.opentelemetry.io/inst-apm-java configured
```

次に、Mushop の Helm チャートを編集します。

```sh
$ vim mushop-devops/deploy/complete/helm-chart/mushop/values.yaml
``` 

carts、orders、fufillment の「.opentelemetry.provider」フィールドの「oci」を「otel」に変更して保存します。

```yaml
・
・＜省略＞
・
carts:
  enabled: true
  opentelemetry:
    provider: otel
・
・＜省略＞
・
orders:
  enabled: true
  image:
    repository: ocir.${REGION}.oci.oraclecloud.com/${NAMESPACE}/mushop-orders
    tag: ${VERSION}
    pullPolicy: Always
  streaming:
    secret: kafka-secret
    serverKey: server
    configKey: config
  opentelemetry:
    provider: otel
・
・＜省略＞
・
fulfillment:
  enabled: true
  image:
    repository: ocir.${REGION}.oci.oraclecloud.com/${NAMESPACE}/mushop-fulfillment
    tag: ${VERSION}
    pullPolicy: Always
  streaming:
    secret: kafka-secret
    serverKey: server
    configKey: config
  opentelemetry:
    provider: otel
```

OCI DevOps のコードリポジトリにプッシュするため、バージョンファイル内の値「0.0.2」を「0.0.3」に変更して保存します。

```sh
$ vim mushop-devops/appVersion
```
```sh
0.0.2 ⇒ 0.0.3
```

更新したソースコードを OCI DevOps のコードリポジトリにプッシュします。

```sh
$ cd mushop-devops
```
```sh
$ git add .
```

```sh
$ git commit -m "add apm info"
[main af0aedd] add apm info
 3 files changed, 22 insertions(+), 22 deletions(-)
```

「Unsername」は、「テナンシ名/メールアドレス」、「Password」は、認証トークンです。「3-2-4-2. OCI DevOps による CI/CD」で利用したものと同じものを使用します。

```sh
$ git push
Username for 'https://devops.scmservice.ap-tokyo-1.oci.oraclecloud.com': <your-tenacny-name>/<your-mailadress>
Password for 'https://<your-tenacny-name>/<your-mailadress>@devops.scmservice.ap-tokyo-1.oci.oraclecloud.com': <your-
authentication-token>
Enumerating objects: 19, done.
Counting objects: 100% (19/19), done.
Delta compression using up to 2 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (10/10), 900 bytes | 900.00 KiB/s, done.
Total 10 (delta 5), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (5/5)
To https://devops.scmservice.ap-tokyo-1.oci.oraclecloud.com/namespaces/nryejxjmyppf/projects/mushop/repositories/mushop-devops
   61db9fe..af0aedd  main -> main
```

「3-2-4-2. OCI DevOps による CI/CD」と同様に、OCI DevOps の CI/CD パイプラインが自動的に実行されます。デプロイが完了するまで待ちます。  
デプロイ完了後に更新された MuShop で、商品をカートに入れ、注文し、配送するまでのシナリオを実施します。（「3-2-4-3. MuShop 利用ガイダンス」の商品注文と配送処理）そして、トレース状況を確認します。


Browser Agent がインストールされている場合でも、リアル・ユーザー・モニタリングそのものを無効化できます。無効化するには、本項「3. フロントエンドアプリケーションへのエンドポイントとパブリックデータキーの設定」で編集したソースコードを、以下のように修正します。

```html
script.
    window.apmrum = (window.apmrum || {}); 
    window.apmrum.serviceName='Mushop-Service';
    window.apmrum.webApplication='Mushop';
    window.apmrum.ociDataUploadEndpoint='<your-data-upload-endpoint>';
    window.apmrum.OracleAPMPublicDataKey='<your-public-data-key>';
    window.apmrum.traceSupportingEndpoints =  [ { headers: [ 'APM' ], hostPattern: '.*' }, { headers: [ ], hostPattern: '.*' }]; 
    window.apmrum.obs = 0;　 // 追記
script(async, crossorigin='anonymous', src='<your-data-upload-endpoint>/static/jslib/apmrum.min.js')
```

以上で、OCI APM によるリアル・ユーザー・モニタリングは完了です。

環境削除について補足しておきます。OCI Cloud Shell から以下コマンドを実行して OpenTelemetry Operator をアンインストールしてください。そして、OCI Dashboard Console から作成した可用性モニタリング（合成モニタリング）のモニター「mushop」を手動で削除した後に、「3-2-4-4. ハンズオン環境の破棄」の手順を実行してください。また、OCI Resource Manager の破棄におけるリソースのコンフリクトとは関係ありませんが、ここではスクリプトでポリシーも作成しているので OCI APM を使用しない場合は OCI Dashboard Console から手動で削除してください。削除のタイミングは、「3-2-4-4. ハンズオン環境の破棄」の前または後のどちらでも構いません。  

```sh
$ helm uninstall opentelemetry-operator -n mushop
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/y_ocnj28/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /home/y_ocnj28/.kube/config
release "opentelemetry-operator" uninstalled
```
