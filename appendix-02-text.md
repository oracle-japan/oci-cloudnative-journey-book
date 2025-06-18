# Helm の基礎知識

## Chart と values ファイルの関係性

1. Chartの作成

以下のコマンドで、新しい Helm Chart を作成します。

```sh
$ helm create my-pod-chart
```

このコマンドにより、my-pod-chart という名前のディレクトリが作成され、その中にデフォルトの Chart テンプレート構造が生成されます。

2. Chart テンプレートの編集
デフォルトの templates ディレクトリ内のファイルを削除した後に、Pod の Chart テンプレートを作成します。

Chart テンプレートファイル：my-pod-chart/templates/pod.yaml

```sh
$ rm -rf my-pod-chart/templates/*
```
```sh
$ vim my-pod-chart/templates/pod.yaml
```
```sh
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.appName }}
  labels:
    app: {{ .Values.appName }}
spec:
  containers:
  - name: {{ .Values.appName }}
    image: {{ .Values.image }}
    ports:
    - containerPort: {{ .Values.containerPort }}
    env:
    - name: LOG_LEVEL
      value: {{ .Values.logLevel }}
```

- Chart テンプレートの役割 :  
Chart テンプレートは、Kubernetesリソースの定義をテンプレート化したものです。{{ .Values.XXX }}の形式で変数を指定し、values ファイルの値を動的に埋め込むことで、環境や条件に応じたリソースを簡単に生成できます。
- サンプル Chart テンプレートの内容:
Pod の構成を定義しており、名前やイメージ、ポート、環境変数などが values ファイルから動的に設定されます。

3. valuesファイルの編集

デフォルトで作成される values.yaml を以下の通りに編集します。

ファイル：my-pod-chart/values.yaml

```sh
$ vim my-pod-chart/values.yaml
```
```sh
appName: my-app
image: nginx:latest
containerPort: 80
logLevel: debug
```

- values ファイルの役割 :  
values ファイルは、テンプレートで使用する具体的な値を設定するためのファイルです。環境ごとに異なる値を指定することで、テンプレートを再利用できます。更新する場合も、values ファイルの値を変更するだけで反映できるため、運用効率が高まります。
- サンプル values ファイルの内容:  
appName: Podやコンテナの名前に使用される値を定義  
image: 使用するコンテナイメージ（例: nginx:latest）を指定  
containerPort: コンテナが公開するポート番号を指定  
logLevel: アプリケーションのログ出力レベルを指定  

4. Helm Chart のインストール

以下のコマンドでHelm Chartをインストールします。

```sh
$ helm install my-pod-release ./my-pod-chart
```
```sh
NAME: my-pod-release
LAST DEPLOYED: Mon Jan  6 12:33:32 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- my-pod-release: このインストールのリリース名（任意の名前に変更可能）
- ./my-pod-chart: Chartディレクトリのパス

5. インストールの確認

インストールされたリソースを確認します。

```sh
$ kubectl get pods
```
```sh
NAME                              READY   STATUS    RESTARTS   AGE
my-app                            1/1     Running   0          11s
```

my-app という名前の Pod がデプロイされていることを確認できます。

補足

- 設定の変更:  
values ファイルを編集後、以下のコマンドで変更を反映できます。更新毎に表示結果の REVISION の値が加算されます。

```sh
$ helm upgrade my-pod-release ./my-pod-chart
```
```sh
Release "my-pod-release" has been upgraded. Happy Helming!
NAME: my-pod-release
LAST DEPLOYED: Mon Jan  6 12:37:06 2025
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

- リリースの削除:
Helm でインストールしたリソースを削除するには以下のコマンドを使用します。

```sh
$ helm uninstall my-pod-release
```
```sh
release "my-pod-release" uninstalled
```