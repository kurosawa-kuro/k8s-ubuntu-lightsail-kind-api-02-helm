以下では、先ほどの **Deployment/Service (ClusterIP) 構成** を Helm で管理する方法をご紹介します。  
マニフェストを Helm Chart 化し、`helm install` / `helm upgrade` / `helm uninstall` の流れを学べる **最小構成チュートリアル** となります。

---
# 目次

1. **Helm のインストール**  
2. **Helm Chart の作成**  
3. **Chart の構造・ファイル内容**  
4. **Helm install / upgrade / uninstall**  
5. **port-forward で動作確認**  

> - 引き続き **Ingress 不要・禁止**  
> - **NodePort も使わない**  
> - **サーバ内部でのみアクセス**  
> - イメージ: `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest`  
> - コンテナポート: **8080**  

---

## 1️⃣ Helm のインストール

Helm はパッケージ管理ツールのように、Chart（パッケージ）をインストールして Kubernetes リソースを管理できます。Ubuntu 22.04 での簡易インストール例を示します。

```bash
# Helm のバイナリを取得（例: v3.11.3）
curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.11.3-linux-amd64.tar.gz

# 展開して /usr/local/bin に移動
tar xf helm.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version
# => version.BuildInfo{Version:"v3.11.3", ...} などバージョン表示されればOK
```

> **メモ**: 公式ドキュメントにも各 OS 向けのインストール手順がまとまっています。  
> 参考: [Install Helm | Helm Docs](https://helm.sh/docs/intro/install/)

---

## 2️⃣ Helm Chart の作成

Helm にはテンプレートのひな型を作るコマンドが用意されています。`helm create` で空の Chart (サンプル) を作り、その中を調整する方法が最も簡単です。

```bash
cd ~/dev/k8s-ubuntu-lightsail-kind-api-01-basic
helm create container-nodejs-api-chart
# => container-nodejs-api-chart/ ディレクトリが作成される
```

作成直後のディレクトリ構成は以下のようになります（不要ファイルは削除してもOK）。

```
container-nodejs-api-chart
 ├── Chart.yaml          # チャートのメタデータ
 ├── values.yaml         # デフォルト値
 ├── charts/             # サブチャート格納用(今回は不要)
 ├── templates/
 │    ├── _helpers.tpl   # テンプレ用のヘルパー
 │    ├── deployment.yaml
 │    ├── service.yaml
 │    └── tests/         # テスト用テンプレ
 └── ...
```

---

## 3️⃣ Chart の構造・ファイル内容

ひな型の `deployment.yaml` や `service.yaml` を、先ほどの手動マニフェストに合わせて編集します。

### 3-1. Chart.yaml

`Chart.yaml` は基本的にそのままでも構いませんが、必要に応じて名前やバージョンを変更します。

```yaml
apiVersion: v2
name: container-nodejs-api
description: A Helm chart for container-nodejs-api
version: 0.1.0
appVersion: "1.0"
```

### 3-2. values.yaml

`values.yaml` に、Deployment や Service で使う各種パラメータを定義します。  
ここではシンプルに下記程度を記載します（`helm create` 実行直後の不要部分は削除/コメントアウト）。

```yaml
# replicas (Deployment)
replicaCount: 1

# イメージ設定
image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080
  tag: "latest"
  pullPolicy: Always

# コンテナのポート
containerPort: 8080

# Service設定
service:
  type: ClusterIP
  port: 8080

# リソース制限
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### 3-3. templates/deployment.yaml

`templates/deployment.yaml` のサンプルを、Helm テンプレートに合わせて書き換えます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "container-nodejs-api.fullname" . }}
  labels:
    app: {{ include "container-nodejs-api.name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "container-nodejs-api.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "container-nodejs-api.name" . }}
    spec:
      containers:
        - name: {{ include "container-nodejs-api.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.containerPort }}
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
```

### 3-4. templates/service.yaml

`templates/service.yaml` を編集します。NodePort なし、ClusterIP だけを指定。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "container-nodejs-api.fullname" . }}
  labels:
    app: {{ include "container-nodejs-api.name" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ include "container-nodejs-api.name" . }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
      protocol: TCP
      name: http
```

> **ポイント**: Helm のテンプレート言語 (Go テンプレート) を使って、`.Values.***` に定義した値を参照しています。

---

## 4️⃣ Helm install / upgrade / uninstall

編集が完了したら、Chart が正しく書けているかを確認し、Kubernetes へデプロイします。

### 4-1. テンプレートの検証

```bash
cd ~/dev/k8s-ubuntu-lightsail-kind-api-01-basic/container-nodejs-api-chart

# YAML出力を確認（--dry-run）
helm template . --values values.yaml
# => エラーがなければ k8sマニフェストが標準出力に表示される
```

### 4-2. インストール

```bash
# リリース名は自由に設定 (ここでは api)
helm install api . --values values.yaml

# または省略形
helm install api .
```

- `api` は Helm リリース名
- `.` はカレントディレクトリ(Chart の場所)
- これで Deployment / Service が作成されます

実行後、Kubernetes リソースが作成されたことを確認します。

```bash
kubectl get pods
kubectl get svc
helm list
# => NAME: api, STATUS: deployed
```

### 4-3. アップグレード

アプリのイメージタグや replicaCount を変えたい場合は、`values.yaml` を編集し、`helm upgrade` を行います。

```bash
# 例: replicas を 2 に増やしたい
vim values.yaml
# replicaCount: 2

helm upgrade api . --values values.yaml
kubectl get pods
# => 2つのPodが稼働している
```

> **ワンライナーで変更する場合**  
> `helm upgrade api . --set replicaCount=2` のように、コマンドライン引数でもオーバーライドできます。

### 4-4. アンインストール

削除時は `helm uninstall` を使います。

```bash
helm uninstall api
helm list
# => 何も表示されなければOK
```

---

## 5️⃣ port-forward で動作確認

- **外部公開しない**＆**NodePort 未使用**なので、`port-forward` でアクセス確認します。

```bash
# Service を 8080 でフォワード
kubectl port-forward service/api-container-nodejs-api 8080:8080
# ※ service 名は "リリース名-Chart名" となる。 helm install時に生成。

# もう1つのターミナルで
curl -v http://localhost:8080/
# => {"message":"Hello"} 等アプリの応答があればOK
```

Pod のログも Helm かどうかに関わらず同じ `kubectl logs` コマンドで確認できます。

```bash
kubectl logs -l app=container-nodejs-api
# => Express server running on port 8080 ...
```

---

# まとめ

- **Helm Chart** を用いることで、Deployment/Service などのマニフェストをまとめてパッケージ化し、`helm install` で一括デプロイできます。
- **Ingress, NodePort** 不要の構成を維持しながら、helm の利点(バージョン管理, values によるパラメータ注入)を利用できます。
- **アップグレード/ロールバック/アンインストール** が容易になるため、より複雑な構成を管理するときにも役立つでしょう。

このチュートリアルでは、**すでに動いていたマニフェストを Helm 化** しましたが、Helm Chart を最初から利用することで、開発/運用フローがよりスムーズになります。  
引き続き楽しい Kubernetes & Helm 学習をお楽しみください。