# AWS CLI コマンド集

## 前提

```bash
# ログイン（1時間ごとに必要）
saml2aws login --profile dev

# 使用プロファイル
--profile smv-admin   # アカウント: 558631582515 / ロール: smv_admin
--profile dev         # アカウント: 524889507893 / ロール: smv_console_admin

# デフォルトリージョン
--region ap-northeast-1
```

---

## 認証・ロール確認

```bash
# 現在の認証情報確認
aws sts get-caller-identity --profile smv-admin

# ロールのスイッチ（assume-role）
aws sts assume-role \
  --role-arn arn:aws:iam::558631582515:role/smv_admin \
  --role-session-name switch-role-session \
  --profile dev
```

---

## EC2

```bash
# インスタンス一覧（ID・状態・タイプ・名前）
aws ec2 describe-instances \
  --profile smv-admin \
  --region ap-northeast-1 \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# 起動中のみ
aws ec2 describe-instances \
  --profile smv-admin \
  --region ap-northeast-1 \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

---

## RDS

```bash
# DB インスタンス一覧
aws rds describe-db-instances \
  --profile smv-admin \
  --region ap-northeast-1 \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Engine,DBInstanceClass]' \
  --output table
```

---

## ElastiCache

```bash
# クラスター一覧
aws elasticache describe-cache-clusters \
  --profile smv-admin \
  --region ap-northeast-1 \
  --output table

# レプリケーショングループ
aws elasticache describe-replication-groups \
  --profile smv-admin \
  --region ap-northeast-1 \
  --output table
```

---

## ELB（ロードバランサー）

```bash
# ALB / NLB 一覧
aws elbv2 describe-load-balancers \
  --profile smv-admin \
  --region ap-northeast-1 \
  --query 'LoadBalancers[*].[LoadBalancerName,State.Code,Type]' \
  --output table

# Classic ELB 一覧
aws elb describe-load-balancers \
  --profile smv-admin \
  --region ap-northeast-1 \
  --output table
```

---

## S3

```bash
# バケット一覧
aws s3 ls --profile smv-admin

# バケットのサイズ確認
aws s3 ls s3://<バケット名> --recursive --human-readable --summarize --profile smv-admin
```

---

## Lambda

```bash
# 関数一覧
aws lambda list-functions \
  --profile smv-admin \
  --region ap-northeast-1 \
  --query 'Functions[*].[FunctionName,Runtime,LastModified]' \
  --output table
```

---

## CloudWatch

```bash
# アラーム一覧
aws cloudwatch describe-alarms \
  --profile smv-admin \
  --region ap-northeast-1 \
  --query 'MetricAlarms[*].[AlarmName,StateValue]' \
  --output table
```

---

## 全リソース一覧（Resource Explorer）

```bash
# 全リソースをサービス別に集計
aws resource-explorer-2 search \
  --query-string "" \
  --profile smv-admin \
  --region ap-northeast-1 \
  --output json | python3 -c "
import json, sys
data = json.load(sys.stdin)
resources = data.get('Resources', [])
from collections import defaultdict
services = defaultdict(list)
for r in resources:
    rtype = r.get('ResourceType', 'unknown')
    services[rtype].append(rtype)
for rtype, items in sorted(services.items()):
    print(f'{len(items):3d}  {rtype}')
print(f'\n合計: {len(resources)}')
"
```
