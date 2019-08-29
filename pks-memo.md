# Jumpbox & PivNet
## 環境変数の設定
```bash
echo "GCP_PROJECT_ID=PA-shashitani" >> ~/.env
echo "PKS_PIVNET_UAA_REFRESH_TOKEN=8306aa7cec3e4d7bb5dd862449397e15-r" >> ~/.env
```
## Pivnetへのログイン
```bash
pivnet login --api-token=${PKS_PIVNET_UAA_REFRESH_TOKEN}
pivnet products
```
## SLUGベースでプロダクトを検索
```bash
PRODUCT_SLUG=pivotal-container-service
pivnet releases \
    --product-slug=${PRODUCT_SLUG}

RELEASE_VERSION=1.4.2
pivnet product-files \
  --product-slug=${PRODUCT_SLUG} \
  --release-version=${RELEASE_VERSION}
```
## CLIをダウンロード
```bash
pivnet download-product-files \
  --product-slug=${PRODUCT_SLUG} \
  --release-version=${RELEASE_VERSION} \
  --glob='pks-linux-*'

pivnet download-product-files \
  --product-slug=${PRODUCT_SLUG} \
  --release-version=${RELEASE_VERSION} \
  --glob='kubectl-linux-*'

# インストール
chmod +x pks-linux-*
chmod +x kubectl-linux-*
sudo mv pks-linux-* /usr/local/bin/pks
sudo mv kubectl-linux-* /usr/local/bin/kubectl
```

# Terraform
## Terraformテンプレートのダウンロード
```bash
TGCP_VERSION=0.94.0
wget -O ~/terraforming-gcp.tar.gz https://github.com/pivotal-cf/terraforming-gcp/releases/download/v${TGCP_VERSION}/terraforming-gcp-v${TGCP_VERSION}.tar.gz && \
  tar -zxvf ~/terraforming-gcp.tar.gz -C ~ && \
  rm ~/terraforming-gcp.tar.gz
```
## 設定
```bash
OPSMAN_IMAGE="https://storage.googleapis.com/ops-manager-us/pcf-gcp-2.6.4-build.166.tar.gz"

cd ~/terraforming/terraforming-pks
cat > terraform.tfvars <<-EOF
dns_suffix          = "${PKS_DOMAIN_NAME}"
env_name            = "${PKS_SUBDOMAIN_NAME}"
region              = "us-central1"
zones               = ["us-central1-b", "us-central1-a", "us-central1-c"]
project             = "$(gcloud config get-value core/project)"
opsman_image_url    = "${OPSMAN_IMAGE}"
opsman_vm           = 1
create_gcs_buckets  = "false"
external_database   = 0
isolation_segment   = 0
service_account_key = <<SERVICE_ACCOUNT_KEY
$(cat ~/workspace/gcp_credentials.json)
SERVICE_ACCOUNT_KEY
EOF
```
## 実行
```bash
terraform init
terraform plan

terraform apply --auto-approve
```

# Concourse
## Concourse用ストレージを作成
```bash
gsutil mb -c regional -l us-central1 gs://${PKS_SUBDOMAIN_NAME}-concourse-resources
gsutil versioning set on gs://${PKS_SUBDOMAIN_NAME}-concourse-resources
```
## Control Towerを使ってConcourseをデプロイ
```bash
GOOGLE_APPLICATION_CREDENTIALS=~/workspace/gcp_credentials.json \
  control-tower deploy \
    --region us-central1 \
    --iaas gcp \
    --workers 3 \
    --worker-size medium \
    ${PKS_SUBDOMAIN_NAME}
```
## Control TowerからCredHubの設定を抽出
```bash
# CredHubのバージョンを確認。Serverは空になってるはず。
credhub --version

INFO=$(GOOGLE_APPLICATION_CREDENTIALS=~/workspace/gcp_credentials.json \
  control-tower info \
    --region us-central1 \
    --iaas gcp \
    --json \
    ${PKS_SUBDOMAIN_NAME}
)

# 設定を.envに書き出し
echo "CC_ADMIN_PASSWD=$(echo ${INFO} | jq --raw-output .config.concourse_password)" >> ~/.env
echo "export CREDHUB_CA_CERT='$(echo ${INFO} | jq --raw-output .config.credhub_ca_cert)'" >> ~/.env
echo "export CREDHUB_CLIENT=credhub_admin" >> ~/.env
echo "export CREDHUB_SECRET=$(echo ${INFO} | jq --raw-output .config.credhub_admin_client_secret)" >> ~/.env
echo "export CREDHUB_SERVER=$(echo ${INFO} | jq --raw-output .config.credhub_url)" >> ~/.env

echo "export OM_TARGET=https://pcf.\${PKS_SUBDOMAIN_NAME}.\${PKS_DOMAIN_NAME}" >> ~/.env
echo "export OM_USERNAME=admin" >> ~/.env
echo "export OM_PASSWORD=$(uuidgen)" >> ~/.env
echo "export OM_DECRYPTION_PASSPHRASE=\${OM_PASSWORD}" >> ~/.env
echo "export OM_SKIP_SSL_VALIDATION=true" >> ~/.env
source ~/.env

# aliacing tower
fly targets
cat ~/.flyrc | sed s/control-tower-${PKS_SUBDOMAIN_NAME}/cc/ > ~/.flyrc.new
mv ~/.flyrc.new ~/.flyrc
# synching clinet cli and server
sudo fly -t cc sync
```
## Firewallルール設定
```bash
gcloud compute firewall-rules create control-tower-${PKS_SUBDOMAIN_NAME}-private-gcp-git \
  --network=control-tower-${PKS_SUBDOMAIN_NAME} \
  --action=ALLOW \
  --rules=tcp:2022 \
  --source-ranges=10.0.1.0/24 \
  --target-tags=worker,internal,external
```
## パイプラインの構築 (ver. 1)
```bash
fly -t cc set-pipeline -n -p pivotal-container-service \
  -c ~/workspace/git-resources/ci/pivotal-container-service/pipeline-01.yml \
  -l ~/workspace/git-resources/ci/pivotal-container-service/vars.yml
fly -t cc unpause-pipeline -p pivotal-container-service
```
## 環境変数ファイルの作成
```bash
cat > vars.yml << EOF
---
config-uri: ${CONFIG_URL}
config-dir: config
gcs-bucket: ${PKS_SUBDOMAIN_NAME}-concourse-resources
credhub-ca-cert: |
$(echo $CREDHUB_CA_CERT | sed 's/- /-\n/g; s/ -/\n-/g' | sed '/CERTIFICATE/! s/ /\n/g' | sed 's/^/  /')
credhub-client: ${CREDHUB_CLIENT}
credhub-secret: ${CREDHUB_SECRET}
credhub-server: ${CREDHUB_SERVER}
EOF
```
## CredHubに認証情報を登録
Ops ManagerのデプロイにPublic IPが必要になるので事前に確保の上pcf.${PKS_SUBDOMAIN_NAME}.${PKS_DOMAIN_NAME}としてDNS登録しておく必要あり
```bash
credhub set -n /concourse/main/pivnet-api-token -t value -v ${PKS_PIVNET_UAA_REFRESH_TOKEN}
credhub set -n /concourse/main/gcp-credentials -t value -v "$(cat ~/workspace/gcp_credentials.json)"
credhub set -n /concourse/main/config-private-key -t value -v "$(cat ~/.ssh/id_rsa)"
credhub set -n /concourse/main/subdomain-name -t value -v "${PKS_SUBDOMAIN_NAME}"
credhub set -n /concourse/main/domain-name -t value -v "${PKS_DOMAIN_NAME}"
credhub set -n /concourse/main/gcp-project-id -t value -v "$(gcloud config get-value core/project)"
credhub set -n /concourse/main/opsman-public-ip -t value -v "$(dig +short pcf.${PKS_SUBDOMAIN_NAME}.${PKS_DOMAIN_NAME})"
credhub set -n /concourse/main/om-target -t value -v "${OM_TARGET}"
credhub set -n /concourse/main/om-username -t value -v "${OM_USERNAME}"
credhub set -n /concourse/main/om-password -t value -v "${OM_PASSWORD}"
credhub set -n /concourse/main/om-decryption-passphrase -t value -v "${OM_DECRYPTION_PASSPHRASE}"
credhub set -n /concourse/main/om-skip-ssl-validation -t json -v ${OM_SKIP_SSL_VALIDATION}
```
## PKSのパイプライン作成 + 実行
```bash
fly -t cc set-pipeline -n -p pivotal-container-service \
  -c ~/workspace/git-resources/ci/pivotal-container-service/pipeline-01.yml \
  -l ~/workspace/git-resources/ci/pivotal-container-service/vars.yml
fly -t cc unpause-pipeline -p pivotal-container-service

# ログの確認
fly -t cc watch -j pivotal-container-service/fetch-opsman
```
## Ops ManagerとPKSのデプロイ
パイプラインの更新
```bash
fly -t cc set-pipeline -n -p pivotal-container-service \
  -c ~/workspace/git-resources/ci/pivotal-container-service/pipeline-02.yml \
  -l ~/workspace/git-resources/ci/pivotal-container-service/vars.yml
```
空のstate.ymlファイルをストレージにほぞすることによりjobが起動する
```bash
echo "---" > ~/state.yml
gsutil cp ~/state.yml gs://${PKS_SUBDOMAIN_NAME}-concourse-resources/
```
## PKS証明書を発行
credhub generate --name /concourse/main/pks-cert \
  --type certificate \
  --common-name "${PKS_SUBDOMAIN_NAME}.${PKS_DOMAIN_NAME}" \
  --alternative-name "*.${PKS_SUBDOMAIN_NAME}.${PKS_DOMAIN_NAME}" \
  --alternative-name "*.pks.${PKS_SUBDOMAIN_NAME}.${PKS_DOMAIN_NAME}" \
  --self-sign
## 認証処理を追加
パイプラインを更新
```bash
fly -t cc set-pipeline -p pivotal-container-service \
  -c ~/workspace/git-resources/ci/pivotal-container-service/pipeline-03.yml \
  -l ~/workspace/git-resources/ci/pivotal-container-service/vars.yml
```
再実行
```bash
fly -t cc trigger-job \
  -j pivotal-container-service/install-opsman
```
## jumpboxからCCとPKSのサブネットワークを繋げる
ネットワークピアリングの設定
```bash
# 疎通確認（サブネットワーク間の通信が出来ないのでタイムアウトエラーとなる）
bosh env

gcloud compute networks peerings create default-to-${PKS_SUBDOMAIN_NAME}-pcf-network \
  --network=default \
  --peer-network=${PKS_SUBDOMAIN_NAME}-pcf-network \
  --auto-create-routes

gcloud compute networks peerings create ${PKS_SUBDOMAIN_NAME}-pcf-network-to-default \
  --network=${PKS_SUBDOMAIN_NAME}-pcf-network \
  --peer-network=default \
  --auto-create-routes
```
Firewallルールの設定
```bash
gcloud compute firewall-rules create bosh \
  --direction=INGRESS \
  --priority=1000 \
  --network=${PKS_SUBDOMAIN_NAME}-pcf-network \
  --action=ALLOW \
  --rules=tcp:25555,tcp:8443,tcp:22 \
  --source-ranges=10.0.0.0/8 \
  --source-tags=jbox-pks \
  --target-tags=${PKS_SUBDOMAIN_NAME}
# 疎通確認（サブネットワーク間の通信が出来ないのでタイムアウトエラーとなる）
bosh env
```
BOSHへのアクセス情報を抽出
```bash
(eval "$(om bosh-env)" && \
  echo "export BOSH_CLIENT=${BOSH_CLIENT}" && \
  echo "export BOSH_CLIENT_SECRET=${BOSH_CLIENT_SECRET}" && \
  echo "export BOSH_ENVIRONMENT=${BOSH_ENVIRONMENT}" && \
  echo "export BOSH_CA_CERT='${BOSH_CA_CERT}'") >> ~/.env

# 疎通確認（サブネットワーク間の通信が出来ないのでタイムアウトエラーとなる）
bosh env
```
