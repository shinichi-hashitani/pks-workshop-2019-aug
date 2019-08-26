# Pivotal Container Serivce - First Contact 2019/08/30 - ハンズオン資料
## Kubernetesクラスタの作成
Kubernetesのクラスタはコマンドで作成できます。
```bash
export CLUSTER_NAME=cluster-test

pks clusters
pks plans
# クラスタの作成
pks create-cluster ${CLUSTER_NAME} --external-hostname ${CLUSTER_NAME}.${TS_G_ENV}.cf-app.com --plan small
```
クラスタの生成には30分程度かかります。下記を実行して状況を逐次確認します。
```bash
watch -n 5 pks cluster $CLUSTER_NAME
```

## クラスタへの認証情報を取得
```bash
pks get-credentials "${CLUSTER_NAME}"
# 環境の切り替え
kubectl config use-context ${CLUSTER_NAME}
kubectl cluster-info
```

## アプリケーションのデプロイ
KubernetesのGuestbook サンプル(ストレージはRedis)をデプロイします。

Redis マスタ
```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml

kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml
```
Redis スレーブ
```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-deployment.yaml

kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-service.yaml
```
Redisのデプロイ確認
```
kubeclt get po
kubectl get deploy
kubectl get svc
```
フロントエンドアプリのデプロイ
```
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml

kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml
```

フロントエンドサービスのExternal IPとNodePortサービスポートを取得します。
```bash
# namespaceをdefaultに戻します（他に切り替えている場合）
kubectl config set-context --current --namespace=default

kubectl get po -o wide
kubectl get no -o wide
kubectl get svc -o wide

# サービスポートの番号をjqで抜きます
SERVICE_PORT=$(kubectl get services --output=json | \
  jq '.items[] | select(.metadata.name=="frontend") | .spec.ports[0].nodePort')
echo "Service port = ${SERVICE_PORT}"
```
NODE_IP:SERVICE_PORTでアプリにアクセス可能となります。NODE_IPを切りかえるとそれぞれのPodにアクセス可能です。

## ロードバランサを設定
```bash
cat > frontend-load-balanced.yml << 'EOF'
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
EOF
```
LoadBalancerサービスのデプロイ
```
kubectl apply -f frontend-load-balanced.yml

kubectl get svc -o wide
```
この後IaaS側でLoadBalancerとMasterのVPを手作業で紐づける作業が必要です。