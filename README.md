# はじめに

これはmyblogアプリのマニフェストファイル用リポジトリです。


# 前提条件
k8sクラスタは構築済みです。
（GKE等のマネージドサービスは使わずにkubeadm等で構築した環境を想定）



# 初期セットアップ

## リポジトリクローン（masterブランチ）
git clone https://github.com/ultimania/myblog-config.git

cd ./myblog-config


## 名前空間の作成
kubectl apply -f ./system/ns-system.yaml

kubectl apply -f ./namespaces/prod.yaml

|No |名前空間|用途|
|:--|:--|:--|
|1|ingress-nginx|ingress用|
|2|myblog-system|myblogアプリのシステム関連|
|3|myblog-prod|myblogアプリのPROD環境|


## イメージリポジトリの作成
```
kubectl apply -f ./system/pv-registry.yaml
kubectl apply -f ./system/dp-registry.yaml
kubectl apply -f ./system/svc-registry.yaml
```


## Helmの導入
```
curl https://storage.googleapis.com/kubernetes-helm/helm-v2.12.3-linux-amd64.tar.gz | tar zx linux-amd64/helm
mv linux-amd64/helm /usr/local/bin/; rm -rf linux-amd64
kubectl apply -f ./system/helm.yaml
helm init --service-account tiller
```

## DashBoardの導入
```
kubectl apply -f ./system/dashboard.yaml
```

## Redmineの導入
```
helm install -f helm-redmine.yaml -name myredmine stable/redmine
```


## Prometheusの導入
### exporter
```
docker run -d --name node_exporter -p 9100:9100 prom/node-exporter
※全ノードで実行すること
```

### Prometheus Server
```
helm install -f helm-prometheus.yaml --name myprometheus stable/prometheus
```

## Grafanaの導入
```
helm install -f helm-grafana.yaml --name mygrafana stable/grafana
```

## ArgoCDの導入
### ArgoCD の導入
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v0.10.6/manifests/install.yaml
```

### ArgoCD CLI
```
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

## Jenkinsの導入
```
helm install -f jenkins.yaml --name jenkins stable/jenkins
```
