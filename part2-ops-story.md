# Pivotal Container Serivce - First Contact 2019/08/30 - ハンズオン資料
## Ops Manager(online)にアクセス
Ops ManagerはPKSやPAS(Cloud Foundry)、その他関連パッケージを管理する為のオンラインシステムです。Ops Managerより新しいパッケージ（Tile）のインストール、設定変更及び反映を行うことが出来ます。
### Env 1 - arroyogrande
- password: yrxve6v1k773ysjb
- Ops Manager: https://pcf.arroyogrande.cf-app.com
### Env 2 - santiago
- password: hkw58onmnvd0j5xd
- Ops Manager: https://pcf.santiago.cf-app.com
### Env 3 - fowler
- password: hkw58onmnvd0j5xd
- Ops Manager: https://pcf.fowler.cf-app.com
### Env 4 - gardengrove
- password: gwhedm279p5rwn0c
- Ops Manager: https://pcf.gardengrove.cf-app.com

## Ops Manager、BOSH Directorにコマンドラインからアクセス
Ops ManagerはPKSのコンポーネントを管理する立場にあり、Ops Managerを経由すると各種コンポーネントへアクセス可能となります。\
Ops Managerにアクセス:
```bash
export TS_G_ENV=alisoviejo
touch ./${TS_G_ENV}.priv && chmod 600 ${TS_G_ENV}.priv && \
  echo "$(jq -r .ops_manager_private_key < ./config/${TS_G_ENV}.json)" \
  > ${TS_G_ENV}.priv

ssh -i ${TS_G_ENV}.priv ubuntu@pcf.${TS_G_ENV}.cf-app.com
```

## BOSH Directorへアクセス
BOSH DirectorにはDirector自体にログインしなくてもOps Managerからアクセスが可能です。アクセスに必要な設定はOps Manager(online)より取得する事が可能です
> BOSH Director for GCP -> Credentials -> Bosh Commandline Credeintials
取得した文字列をOps Managerにログインした状態で実行します。
```bash
export BOSH_CLIENT=ops_manager BOSH_CLIENT_SECRET=Ph0dkCPsv21dM610004tgjdq-biuAHk6 BOSH_CA_CERT=/var/tempest/workspaces/default/root_ca_certificate BOSH_ENVIRONMENT=10.0.0.5 bosh
```
以下のコマンドで正しくBOSH Directorにアクセス出来るかを確認します。
```
bosh deployments
```

BOSHからのヘルスチェック
```bash
# BOSH管理のVMインスタンスの一覧
bosh instances
# --vitals -d でDeploymentを指定
bosh instances --vitals -d pivotal-container-service-47401a601da02a2b127c
# ここから各プロセスのサマリも確認可能
bosh instances --vitals -d pivotal-container-service-47401a601da02a2b127c --ps
bosh instances --vitals -d service-instance_27f1d58c-04e7-4f32-8311-78ad68f8d6b5 --ps
# 単純に以下で全deploymentの全プロセスを確認可能
bosh instances --ps
```
BOSHからVMにsshしてmonitを確認
```bash
bosh instances

bosh instances --details

bosh ssh -d service-instance_27f1d58c-04e7-4f32-8311-78ad68f8d6b5 worker/404c7690-9eba-487b-9d66-34d28a022981

sudo su
monit summary
ps -aux | grep flanneld
```
プロセスを停止。再度monitを確認
```bash
kill 12163
monit summary
```
boshからVMを監視した状態でVMを外から停止
```bash
watch -n 5 bosh instances --details

watch -n 5 kubeclt get po -o wide
```
ログファイルを取得。対象となる全インスタンス/プロセスのログが一括して取得出来る。
```bash
# インスタンスグループごと抽出
bosh -e alisoviejo -d service-instance_27f1d58c-04e7-4f32-8311-78ad68f8d6b5 logs
# インスタンスを指定して抽出
bosh -e alisoviejo -d service-instance_27f1d58c-04e7-4f32-8311-78ad68f8d6b5 logs worker/1
# ログをtailで表示
bosh logs -d pivotal-container-service-47401a601da02a2b127c --follow
```

## Log Sink
Cluster Sinkを作成
```bash
pks create-sink cluster-1 --name papertrail-sink syslog-tls://logs2.papertrailapp.com:53822

# もしくはYAMLからCRDを生成
cat > papertrail-sink.yml << 'EOF'
---
apiVersion: apps.pivotal.io/v1beta1
kind: ClusterSink
metadata:
  name: papertrail-sink
spec:
  type: syslog
  host: logs2.papertrailapp.com
  port: :53822
  enable_tls: false
EOF
kubeclt apply -f papertrail-sink.yml
```

エラーが出た時のメモ
```
# Pod内のコンテナをリストアップ
kubectl get pods fluent-bit-2844b -n pks-system -o jsonpath='{.spec.containers[*].name}*'
kubectl logs fluent-bit-gszst -c fluent-bit
```
