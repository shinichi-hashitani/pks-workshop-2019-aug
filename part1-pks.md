# Pivotal Container Serivce - First Contact 2019/08/30 - ハンズオン資料
## PKSへアクセス
参加者皆さんに別の環境を用意しており、それぞれに名前が付いています。
### Env 1 - arroyogrande
### Env 2 - santiago
### Env 3 - fowler
### Env 4 - gardengrove

パスワードの抽出
```bash
export TS_G_ENV=alisoviejo
export UAA_ADMIN_PASSWORD=$(cat ./config/${TS_G_ENV}.json | jq -r .pks_api.uaa_admin_password)
echo $UAA_ADMIN_PASSWORD
```
ログイン
```bash
pks login -a api.pks.${TS_G_ENV}.cf-app.com -u admin -p ${UAA_ADMIN_PASSWORD} -k
```

## Kubernetesクラスタの作成
Kubernetesのクラスタはコマンドで作成できます。
```bash
export CLUSTER_NAME=cluster-1

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

** SKIP ** フロントエンドサービスのExternal IPとNodePortサービスポートを取得します。
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