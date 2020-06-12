# はじめに

これはmyblogアプリのマニフェストファイル用リポジトリです。


# 前提条件
k8sクラスタは構築済みです。
（GKE等のマネージドサービスは使わずにkubeadm等で構築した環境を想定）



# 初期セットアップ

## リポジトリクローン（masterブランチ）
```
git clone https://github.com/ultimania/myblog-config.git
cd ./myblog-config
```

## 名前空間の作成
```
kubectl apply -f ./systems/ns-system.yaml
kubectl apply -f ./namespaces/prod.yaml
kubectl get ns
export NS=myblog-system
```

|No |名前空間|用途|
|:--|:--|:--|
|1|ingress-nginx|ingress用|
|2|myblog-system|myblogアプリのシステム関連|
|3|myblog-prod|myblogアプリのPROD環境|



## イメージリポジトリの作成
```
kubectl apply -f ./systems/pv-registry.yaml
kubectl get pv
kubectl apply -f ./systems/dp-registry.yaml
kubectl get po -n $NS
kubectl apply -f ./systems/svc-registry.yaml
kubectl get svc -n $NS
kubectl port-forward svc/registry-svc 5000 --address 0.0.0.0 -n $NS
```


## Helmの導入
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash
helm init
kubectl get deployment -n kube-system
helm version
kubectl create serviceaccount --namespace kube-system tiller
serviceaccount "tiller" created
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
clusterrolebinding "tiller-cluster-rule" created
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
deployment "tiller-deploy" patched
helm init --service-account tiller
```

## Ingressの導入

```
helm search nginx-ingress
helm upgrade --install nginx-ingress stable/nginx-ingress --namespace $NS
kubectl apply -f ./systems/ig-system.yaml
kubectl port-forward svc/kubernetes-dashboard 8000:443 --address 0.0.0.0 -n kube-system
```


## DashBoardの導入
```
kubectl apply -f ./systems/dashboard.yaml
kubectl port-forward svc/kubernetes-dashboard 8000:443 --address 0.0.0.0 -n kubernetes-dashboard &
export SECRET=`kubectl get secret -n kubernetes-dashboard | grep admin | awk '{print $1}'`
kubectl describe secret -n kubernetes-dashboard ${SECRET}
```

## Redmineの導入
```
kubectl apply -f ./systems/pv-redmine.yaml
helm install -f ./systems/helm-redmine.yaml --name myredmine stable/redmine --namespace $NS
kubectl port-forward svc/myredmine 8001:80 --address 0.0.0.0 -n $NS &
```


## Prometheusの導入
### exporter
```
docker run -d --name node_exporter -p 9100:9100 prom/node-exporter
※全ノードで実行すること
```

### Prometheus Server
```
kubectl apply -f ./systems/pv-prometheus.yaml
helm install -f ./systems/helm-prometheus.yaml --name myprometheus --namespace $NS stable/prometheus
kubectl port-forward svc/myprometheus-server -n $NS 8002:80 --address 0.0.0.0 -n $NS  &
```

## Grafanaの導入
```
helm install -f ./systems/helm-grafana.yaml --name mygrafana --namespace $NS stable/grafana
kubectl port-forward svc/mygrafana -n $NS 8003:80 --address 0.0.0.0 &
```

## ArgoCDの導入
### ArgoCD の導入
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v0.10.6/manifests/install.yaml --namespace $NS
kubectl port-forward svc/argocd-server -n $NS 8004:80 --address 0.0.0.0 & 
```

> 初回ログインはadmin/<argocd-serverのPOD名>

### ArgoCD CLI
```
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

## Jenkinsの導入
```
kubectl apply -f ./systems/pv-jenkins.yaml
helm install -f ./systems/helm-jenkins.yaml --name myjenkins --namespace $NS stable/jenkins
kubectl port-forward svc/myjenkins -n $NS 8005:8080 --address 0.0.0.0 &
printf $(kubectl get secret --namespace myblog-system myjenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```


# 掃除（全削除）

## パッケージ削除
```
helm delete --purge nginx-ingress
kubectl delete -f ./systems/dashboard.yaml
helm delete --purge myredmine 
helm delete --purge myprometheus 
helm delete --purge mygrafana
kubectl delete -f https://raw.githubusercontent.com/argoproj/argo-cd/v0.10.6/manifests/install.yaml
helm delete --purge myjenkins
docker rm -f --name node_exporter
```

## PersistentVolume削除
```
kubectl delete -f ./systems/pv-redmine.yaml
kubectl delete -f ./systems/pv-prometheus.yaml
kubectl delete -f ./systems/pv-jenkins.yaml
```
