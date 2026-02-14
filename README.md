# Fossology on AWS ECS - デプロイマニュアル

## 📋 目次

- [Fossology on AWS ECS - デプロイマニュアル](#fossology-on-aws-ecs---デプロイマニュアル)
  - [📋 目次](#-目次)
  - [前提条件](#前提条件)
    - [必要なツール](#必要なツール)
    - [AWSリソース](#awsリソース)
  - [mTLS証明書の作成](#mtls証明書の作成)
    - [ステップ1: 証明書ディレクトリの作成](#ステップ1-証明書ディレクトリの作成)
    - [ステップ2: ルートCA証明書の作成](#ステップ2-ルートca証明書の作成)
    - [ステップ3: クライアント証明書の作成](#ステップ3-クライアント証明書の作成)
    - [ステップ4: 証明書の検証](#ステップ4-証明書の検証)
    - [ステップ5: 生成されたファイルの確認](#ステップ5-生成されたファイルの確認)
    - [📁 証明書ファイルの用途](#-証明書ファイルの用途)
  - [S3バケットの準備](#s3バケットの準備)
    - [ステップ1: S3バケットの作成](#ステップ1-s3バケットの作成)
    - [ステップ2: ルートCA証明書のアップロード](#ステップ2-ルートca証明書のアップロード)
    - [ステップ3: S3オブジェクトの権限確認](#ステップ3-s3オブジェクトの権限確認)
  - [CloudFormationスタックのデプロイ](#cloudformationスタックのデプロイ)
    - [ステップ4: スタックの作成](#ステップ4-スタックの作成)
    - [ステップ5: デプロイ進捗の監視](#ステップ5-デプロイ進捗の監視)
    - [ステップ6: デプロイ完了の確認](#ステップ6-デプロイ完了の確認)

---

## 前提条件

### 必要なツール

```bash
# AWS CLI
aws --version
# aws-cli/2.x.x 以上

# OpenSSL
openssl version
# OpenSSL 1.1.1 以上

# jq (オプション、JSONパース用)
jq --version
```

### AWSリソース

以下が既に存在することを確認:

- ✅ Route53ホストゾーン
- ✅ ACM証明書

---

## mTLS証明書の作成

### ステップ1: 証明書ディレクトリの作成

```bash
# 作業ディレクトリを作成
mkdir -p ~/fossology-certs
cd ~/fossology-certs
```

### ステップ2: ルートCA証明書の作成

```bash
# 1. ルートCA秘密鍵の生成 (4096ビット RSA)
openssl genrsa -out rootCA.key 4096

# 2. ルートCA証明書の生成 (有効期限: 10年)
# -subjのパラメータは適宜変更のこと
openssl req -x509 -new -nodes \
  -key rootCA.key \
  -sha256 \
  -days 3650 \
  -out rootCA.pem \
  -subj "/C=JP/ST=Osaka/L=Osaka/O=Example/OU=Security/CN=Fossology Root CA"

# 3. ルートCA証明書の確認
openssl x509 -in rootCA.pem -text -noout
```

**重要**: `rootCA.key` は厳重に保管してください。このファイルが漏洩すると、mTLS認証の安全性が損なわれます。

### ステップ3: クライアント証明書の作成

```bash
# 1. クライアント秘密鍵の生成
openssl genrsa -out client.key 2048

# 2. クライアント証明書署名要求 (CSR) の作成
openssl req -new \
  -key client.key \
  -out client.csr \
  -subj "/C=JP/ST=Osaka/L=Osaka/O=Example/OU=Engineering/CN=fossology-client"

# 3. 拡張定義ファイルの作成(client_ext.cnf)
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth

# 4. クライアント証明書の署名 (有効期限: 2年)
openssl x509 -req \
  -in client.csr \
  -CA rootCA.pem \
  -CAkey rootCA.key \
  -CAcreateserial \
  -out client.crt \
  -days 730 \
  -sha256
  -extfile client_ext.cnf

# 5. クライアント証明書の確認
openssl x509 -in client.crt -text -noout

# 6. CSRファイルは不要なので削除
rm client.csr
```

### ステップ4: 証明書の検証

```bash
# クライアント証明書がルートCAで署名されていることを確認
openssl verify -CAfile rootCA.pem client.crt
# 出力: client.crt: OK
```

### ステップ5: 生成されたファイルの確認

```bash
ls -lh ~/fossology-certs/
# 出力例:
# -rw-r--r--  1 user  staff   1.2K  rootCA.pem      # ルートCA証明書 (S3にアップロード)
# -rw-------  1 user  staff   3.2K  rootCA.key      # ルートCA秘密鍵 (厳重保管)
# -rw-r--r--  1 user  staff   1.1K  client.crt      # クライアント証明書 (ブラウザ/curl使用)
# -rw-------  1 user  staff   1.7K  client.key      # クライアント秘密鍵 (ブラウザ/curl使用)
# -rw-r--r--  1 user  staff    17B  rootCA.srl      # シリアル番号ファイル
```

### 📁 証明書ファイルの用途

| ファイル | 用途 | 保管場所 |
|---------|------|---------|
| `rootCA.pem` | ALBのTrustStoreに登録 | S3バケット |
| `rootCA.key` | 新しいクライアント証明書の署名 | ローカル (厳重保管) |
| `client.crt` | クライアント認証用 | ブラウザ/アプリケーション |
| `client.key` | クライアント認証用 | ブラウザ/アプリケーション |

---

## S3バケットの準備

### ステップ1: S3バケットの作成

```bash
# S3バケット作成
aws s3 mb s3://fossology-rootca --region ap-northeast-1

# バケットのバージョニングを有効化 (推奨)
aws s3api put-bucket-versioning \
  --bucket fossology-rootca \
  --versioning-configuration Status=Enabled \
  --region ap-northeast-1

# パブリックアクセスブロックを有効化 (セキュリティ強化)
aws s3api put-public-access-block \
  --bucket fossology-rootca \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true" \
  --region ap-northeast-1
```

### ステップ2: ルートCA証明書のアップロード

```bash
# ルートCA証明書をS3にアップロード
cd ~/fossology-certs
aws s3 cp rootCA.pem s3://fossology-rootca/rootCA.pem \
  --region ap-northeast-1

# アップロードの確認
aws s3 ls s3://fossology-rootca/
# 出力: 2026-02-13 12:34:56       1234 rootCA.pem
```

### ステップ3: S3オブジェクトの権限確認

```bash
# オブジェクトのメタデータ確認
aws s3api head-object \
  --bucket fossology-rootca \
  --key rootCA.pem \
  --region ap-northeast-1
```

---

## CloudFormationスタックのデプロイ

### ステップ4: スタックの作成

```bash
# スタック作成
aws cloudformation create-stack \
  --stack-name Fossology \
  --template-body file://fossology-ecs-stack.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1 \
  --tags Key=Environment,Value=Production Key=Application,Value=Fossology

# スタックIDを取得
STACK_ID=$(aws cloudformation describe-stacks \
  --stack-name Fossology \
  --region ap-northeast-1 \
  --query 'Stacks[0].StackId' \
  --output text)

echo "Stack ID: $STACK_ID"
```

### ステップ5: デプロイ進捗の監視

```bash
# リアルタイムでイベントを監視
watch -n 10 'aws cloudformation describe-stack-events \
  --stack-name Fossology \
  --region ap-northeast-1 \
  --max-items 10 \
  --query "StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId]" \
  --output table'

# または、スタックステータスのみ確認
aws cloudformation describe-stacks \
  --stack-name Fossology \
  --region ap-northeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

**ステータスの意味**:
- `CREATE_IN_PROGRESS`: デプロイ中
- `CREATE_COMPLETE`: デプロイ成功 ✅
- `CREATE_FAILED`: デプロイ失敗 ❌
- `ROLLBACK_IN_PROGRESS`: ロールバック中

### ステップ6: デプロイ完了の確認

```bash
# スタック出力の取得
aws cloudformation describe-stacks \
  --stack-name Fossology \
  --region ap-northeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table

# JSONフォーマットで保存
aws cloudformation describe-stacks \
  --stack-name Fossology \
  --region ap-northeast-1 \
  --query 'Stacks[0].Outputs' \
  --output json > stack-outputs.json
```
