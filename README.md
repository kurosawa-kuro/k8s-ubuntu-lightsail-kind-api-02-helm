# Helm による最小構成 API デプロイ

以下では、先ほどの **Deployment/Service (ClusterIP) 構成** を Helm で管理する方法をご紹介します。  
マニフェストを Helm Chart 化し、`helm install` / `helm upgrade` / `helm uninstall` の流れを学べる **最小構成チュートリアル** となります。

---
# 目次

1. **Chart の構造・ファイル内容**  
2. **Helm install / upgrade / uninstall**  
3. **port-forward で動作確認**  

> - 引き続き **Ingress 不要・禁止**  
> - **NodePort も使わない**  
> - **サーバ内部でのみアクセス**  
> - イメージ: `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8080:latest`  
> - コンテナポート: **8080**  

---

## 1️⃣ Chart の構造・ファイル内容

### 1-1. Chart.yaml

```yaml
apiVersion: v2
name: container-nodejs-api
description: A Helm chart for container-nodejs-api
version: 0.1.0
appVersion: "1.0"
```

### 1-2. values.yaml

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

# ServiceAccount設定
serviceAccount:
  create: true
  name: "container-nodejs-api"
  annotations: {}

# Ingress設定（無効化）
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []
  tls: []

# オートスケーリング設定（無効化）
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80

# リソース制限
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### 1-3. templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "container-nodejs-api-chart.fullname" . }}
  labels:
    {{- include "container-nodejs-api-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "container-nodejs-api-chart.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 1-4. templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "container-nodejs-api-chart.fullname" . }}
  labels:
    {{- include "container-nodejs-api-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "container-nodejs-api-chart.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
      protocol: TCP
      name: http
```

---

## 2️⃣ Helm install / upgrade / uninstall

### 2-1. テンプレートの検証

```bash
cd ~/dev/k8s-ubuntu-lightsail-kind-api-02-helm/container-nodejs-api-chart

# YAML出力を確認
helm template . --values values.yaml
```

### 2-2. インストール

```bash
# リリース名 api でインストール
helm install api . --values values.yaml
```

実行後、Kubernetes リソースが作成されたことを確認します。

```bash
kubectl get pods
kubectl get svc
helm list
```

### 2-3. アップグレード

```bash
# 例: replicas を 2 に増やす場合
helm upgrade api . --set replicaCount=2
```

### 2-4. アンインストール

```bash
helm uninstall api
```

---

## 3️⃣ port-forward で動作確認

```bash
# Service を 8080 でフォワード
kubectl port-forward service/api-container-nodejs-api 8080:8080

# 別ターミナルで確認
curl -v http://localhost:8080/
# => {"status":"ok","timestamp":"2025-04-14T03:40:12.137Z"}
```

---

# まとめ

- **Helm Chart** を用いることで、Kubernetes リソースをまとめてパッケージ化し、一括デプロイできます。
- **Ingress, NodePort** 不要の構成を維持しながら、helm の利点を活用できます。
- ServiceAccount や HPA など、将来の拡張に備えた設定も含まれています。

引き続き楽しい Kubernetes & Helm 学習をお楽しみください。