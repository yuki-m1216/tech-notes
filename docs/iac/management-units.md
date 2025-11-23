# 管理単位のベストプラクティス

IaCでは、全てのリソースを1つのスタックやstateファイルで管理するのではなく、**適切な単位に分割**することが重要です。このセクションでは、CDKのスタック分割とTerraformのstate分割について学びます。

## なぜ分割が必要なのか

### 単一管理の問題点

全てのリソースを1つの管理単位にまとめると、以下の問題が発生します:

**問題1: 変更の影響範囲が大きい**

```
単一のstateファイルやスタック
├── VPCの設定変更
├── データベースの変更
└── アプリケーションの変更
    ↓
全てのリソースが影響を受けるリスク
```

- ネットワークの小さな変更でも、全リソースの管理状態を操作
- 意図しない変更が他のレイヤーに波及する可能性

**問題2: 並行作業が困難**

- チームAがネットワーク変更、チームBがアプリケーション変更を同時にできない
- ロック機構(Terraformのstate lock、CDKのCloudFormationスタック操作)により、片方が待たされる

**問題3: デプロイ時間が長い**

- 1つの小さな変更でも、全リソースの差分確認・デプロイが実行される
- CI/CDのフィードバックループが遅くなる

**問題4: 障害時の影響範囲が大きい**

- 管理状態の破損時(Terraformのstate破損、CDKのスタックエラー)の影響が全リソースに及ぶ
- ロールバックが困難

### 分割のメリット

適切に分割することで:

- ✅ **変更の影響範囲を限定**: あるレイヤーの変更が他に波及しない
- ✅ **並行作業が可能**: 複数チームが同時に異なるレイヤーを変更できる
- ✅ **デプロイ時間の短縮**: 変更のあるレイヤーのみデプロイ
- ✅ **リスクの分離**: 問題発生時の影響範囲を限定
- ✅ **責任の明確化**: レイヤーごとに担当チームを割り当て可能

## 分割の軸

管理単位を分割する際の主な軸は以下の4つです。実務では、**これらの軸を組み合わせて**分割することが一般的です。

### 1. 機能単位 (責務)

リソースの機能や役割に応じて分割します。

**分割例:**

- **ネットワーク**: VPC、サブネット、ルートテーブル、NAT Gateway
- **セキュリティ**: IAM、KMS、Security Hub、WAF
- **データベース**: RDS、DynamoDB、ElastiCache
- **アプリケーション**: Lambda、ECS、API Gateway、ALB

!!! tip "機能単位の利点"
    機能ごとに分割することで、変更の影響範囲が明確になり、専門性を持ったチームが担当しやすくなります。

!!! note "監視リソースの配置"
    CloudWatch、X-Ray、アラーム設定などの監視リソースは、通常は**各機能の中に含める**ことが推奨されます。

    **各機能に含める例:**

    - RDSのアラーム → `database`スタック/state
    - Lambdaのメトリクス・ログ → `application`スタック/state
    - VPCフローログ → `network`スタック/state

    **独立させる例:**

    - 複数レイヤーをまたぐ統合ダッシュボード
    - 全体的なログ集約・分析基盤
    - 横断的なアラート管理システム

    監視対象とアラーム設定を同じスタック/stateで管理することで、リソースと監視が一緒にデプロイ/削除され、依存関係もシンプルになります。

### 2. ライフサイクル (変更頻度)

リソースの変更頻度に応じて分割します。

```
変更頻度: 低 → 高
┌─────────────┬──────────────┬───────────────┐
│ Foundation  │ Data Layer   │ Application   │
├─────────────┼──────────────┼───────────────┤
│ VPC         │ RDS          │ Lambda        │
│ Subnets     │ DynamoDB     │ ECS           │
│ Route Table │ S3 (data)    │ API Gateway   │
│ IAM Roles   │              │               │
├─────────────┼──────────────┼───────────────┤
│ 低頻度      │ 中頻度       │ 高頻度        │
└─────────────┴──────────────┴───────────────┘
```

**分割例:**

- **Foundation (基盤)**: VPC、サブネット、セキュリティグループ等 - 変更頻度が低い
- **Data Layer (データ)**: RDS、DynamoDB、S3等のデータストア - 変更頻度が中程度
- **Application (アプリケーション)**: Lambda、ECS、API Gateway等 - 変更頻度が高い

### 3. 所有者/責任範囲

**チーム構造による分割:**

組織のチーム構造に応じて分割します。

```
┌──────────────────┬─────────────────┬──────────────────┐
│ Platform Team    │ Data Team       │ Service Team     │
├──────────────────┼─────────────────┼──────────────────┤
│ 共通インフラ     │ データ基盤      │ サービス固有     │
│ - Network        │ - Data Lake     │ - Service A      │
│ - Security       │ - Analytics     │ - Service B      │
│ - Monitoring     │ - ETL Pipeline  │ - Service C      │
└──────────────────┴─────────────────┴──────────────────┘
```

**機能責任による分割:**

チーム構造に関わらず、機能責任で分割することもあります。

- **インフラ基盤チーム**: ネットワーク、セキュリティ、IAM
- **データ基盤チーム**: データベース、データレイク、分析基盤
- **アプリケーションチーム**: 各サービスのインフラ

### 4. 環境 (dev/staging/prod)

環境ごとに完全に分離します。

```
environments/
├── dev/        # 開発環境
├── staging/    # ステージング環境
└── prod/       # 本番環境
```

!!! warning "環境間の分離は必須"
    環境ごとの分離は**必須**です。開発環境の変更が本番環境に影響することは絶対に避けなければなりません。

### 軸の組み合わせ

実務では、複数の軸を組み合わせて分割します。

**例1: 環境 × 機能**

```
environments/
└── prod/
    ├── network/      # 機能: ネットワーク
    ├── security/     # 機能: セキュリティ
    ├── database/     # 機能: データベース
    └── application/  # 機能: アプリケーション
```

**例2: 環境 × チーム × ライフサイクル**

```
environments/
└── prod/
    ├── platform-team/
    │   ├── foundation/     # 基盤 (変更頻度: 低)
    │   └── monitoring/     # 監視 (変更頻度: 中)
    │
    ├── data-team/
    │   ├── database/       # DB (変更頻度: 中)
    │   └── analytics/      # 分析 (変更頻度: 中)
    │
    └── service-team-a/
        ├── database/       # サービス固有DB (変更頻度: 中)
        └── application/    # アプリ (変更頻度: 高)
```

**例3: 環境 × 機能 × サービス**

```
environments/
└── prod/
    ├── shared/           # 共通インフラ
    │   ├── network/
    │   └── security/
    │
    ├── service-a/        # サービスA
    │   ├── database/
    │   └── application/
    │
    └── service-b/        # サービスB
        ├── database/
        └── application/
```

!!! info "分割の判断基準"
    どの軸を優先するかは、組織の規模、チーム構成、変更頻度のパターンによって異なります。小規模なプロジェクトでは「環境 × 機能」、大規模なプロジェクトでは「環境 × チーム × 機能」のように組み合わせることが多いです。

## CDKのスタック分割

CDKでは、`Stack`クラスを使って管理単位を分割します。

### 基本的な分割パターン

=== "bin/app.ts"
    ```typescript
    #!/usr/bin/env node
    import 'source-map-support/register';
    import * as cdk from 'aws-cdk-lib';
    import { NetworkStack } from '../lib/stacks/network-stack';
    import { DatabaseStack } from '../lib/stacks/database-stack';
    import { ApplicationStack } from '../lib/stacks/application-stack';
    import { prodConfig } from '../lib/config/prod';

    const app = new cdk.App();
    const environment = process.env.ENV || 'dev';

    // 1. 基盤レイヤー (変更頻度: 低)
    const networkStack = new NetworkStack(app, `${environment}-NetworkStack`, {
      env: prodConfig.env,
    });

    // 2. データレイヤー (変更頻度: 中)
    const databaseStack = new DatabaseStack(app, `${environment}-DatabaseStack`, {
      env: prodConfig.env,
      vpc: networkStack.vpc,  // ネットワークスタックに依存
    });

    // 3. アプリケーションレイヤー (変更頻度: 高)
    const applicationStack = new ApplicationStack(app, `${environment}-ApplicationStack`, {
      env: prodConfig.env,
      vpc: networkStack.vpc,
      database: databaseStack.database,  // データベーススタックに依存
    });
    ```

=== "lib/stacks/network-stack.ts"
    ```typescript
    import * as cdk from 'aws-cdk-lib';
    import * as ec2 from 'aws-cdk-lib/aws-ec2';
    import { Construct } from 'constructs';

    export class NetworkStack extends cdk.Stack {
      public readonly vpc: ec2.Vpc;

      constructor(scope: Construct, id: string, props?: cdk.StackProps) {
        super(scope, id, props);

        // VPCの作成
        this.vpc = new ec2.Vpc(this, 'MainVpc', {
          maxAzs: 3,
          natGateways: 1,
          subnetConfiguration: [
            {
              name: 'Public',
              subnetType: ec2.SubnetType.PUBLIC,
            },
            {
              name: 'Private',
              subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
            },
            {
              name: 'Isolated',
              subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
            },
          ],
        });

        // Outputとしてエクスポート
        new cdk.CfnOutput(this, 'VpcId', {
          value: this.vpc.vpcId,
          exportName: `${this.stackName}-VpcId`,
        });
      }
    }
    ```

=== "lib/stacks/database-stack.ts"
    ```typescript
    import * as cdk from 'aws-cdk-lib';
    import * as rds from 'aws-cdk-lib/aws-rds';
    import * as ec2 from 'aws-cdk-lib/aws-ec2';
    import { Construct } from 'constructs';

    export interface DatabaseStackProps extends cdk.StackProps {
      vpc: ec2.IVpc;
    }

    export class DatabaseStack extends cdk.Stack {
      public readonly database: rds.DatabaseInstance;

      constructor(scope: Construct, id: string, props: DatabaseStackProps) {
        super(scope, id, props);

        // RDSインスタンスの作成
        this.database = new rds.DatabaseInstance(this, 'Database', {
          engine: rds.DatabaseInstanceEngine.postgres({
            version: rds.PostgresEngineVersion.VER_15,
          }),
          vpc: props.vpc,
          vpcSubnets: {
            subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
          },
          instanceType: ec2.InstanceType.of(
            ec2.InstanceClass.T3,
            ec2.InstanceSize.MEDIUM
          ),
        });

        // Outputとしてエクスポート
        new cdk.CfnOutput(this, 'DatabaseEndpoint', {
          value: this.database.dbInstanceEndpointAddress,
          exportName: `${this.stackName}-Endpoint`,
        });
      }
    }
    ```

### スタック間の依存関係

CDKでは、スタック間の依存関係を明示的に定義します。

**方法1: プロパティで直接渡す (推奨)**

```typescript
// ネットワークスタックのVPCをデータベーススタックに渡す
const databaseStack = new DatabaseStack(app, 'DatabaseStack', {
  vpc: networkStack.vpc,  // 直接参照
});
```

**メリット:**

- 型安全
- 依存関係が明確
- IDEの補完が効く

**方法2: CloudFormation Exportsを使用**

```typescript
// ネットワークスタック側
new cdk.CfnOutput(this, 'VpcId', {
  value: this.vpc.vpcId,
  exportName: 'MyVpcId',
});

// データベーススタック側
const vpcId = cdk.Fn.importValue('MyVpcId');
```

**メリット:**

- Export名を明示的に制御できる
- リージョン内で一意な名前で値を共有可能

**デメリット:**

- CloudFormation Export削除制限あり
- 手動でExport名を管理する必要がある

**方法3: SSM Parameter Storeを経由 (疎結合)**

```typescript
// ネットワークスタック側: Parameter Storeに保存
import * as ssm from 'aws-cdk-lib/aws-ssm';

new ssm.StringParameter(this, 'VpcIdParameter', {
  parameterName: '/myapp/network/vpc-id',
  stringValue: this.vpc.vpcId,
});

// データベーススタック側: Parameter Storeから読み取り
const vpcId = ssm.StringParameter.valueFromLookup(this, '/myapp/network/vpc-id');
// または実行時に取得する場合
const vpcIdParam = ssm.StringParameter.fromStringParameterName(
  this, 'VpcId', '/myapp/network/vpc-id'
);
const vpc = ec2.Vpc.fromVpcAttributes(this, 'Vpc', {
  vpcId: vpcIdParam.stringValue,
  availabilityZones: ['ap-northeast-1a', 'ap-northeast-1c'],
});
```

**メリット:**

- スタック間の依存関係が緩やかになる（疎結合）
- 異なるデプロイサイクルのスタック間で値を共有しやすい
- CloudFormation Exportの削除制限を回避できる

**デメリット:**

- 型安全性が低下する
- 値の存在を実行時まで検証できない（`valueFromLookup`使用時は合成時）
- Parameter Storeへの読み書きが必要（わずかなコスト）

!!! warning "スタック間参照の削除制限"
    **方法1と方法2では、内部的にCloudFormation ExportsとImportValueが使用されます**。そのため、他のスタックから参照されている間は、参照元のスタックを削除することも、Export値を変更することもできません。

    **削除時の手順（方法1・2の場合）:**

    1. 消費スタックから参照を削除してデプロイ
    2. 生成スタックからExportを削除またはスタックを削除

    **方法3（SSM Parameter Store）では、この削除制限がありません**。Parameter Storeを削除しても、他のスタックには影響しません（ただし、値を参照できなくなります）。

!!! tip "使い分けの推奨"
    - **同一CDKアプリ内**: 方法1（プロパティ渡し）- CDKによる自動管理、型安全性
    - **異なるStage/アプリ間**: 方法3（SSM Parameter Store）- 疎結合、削除制限なし
    - **Export名を明示的に管理したい場合**: 方法2（Export）- リージョン内で一意な名前で値を共有

### デプロイ順序

依存関係のあるスタックは、順番にデプロイする必要があります。

```bash
# 1. 基盤レイヤー
cdk deploy prod-NetworkStack

# 2. データレイヤー (ネットワークに依存)
cdk deploy prod-DatabaseStack

# 3. アプリケーションレイヤー (ネットワークとデータベースに依存)
cdk deploy prod-ApplicationStack

# または、全てを一度にデプロイ (CDKが依存順序を解決)
cdk deploy --all
```

## Terraformのstate分割

Terraformでは、ディレクトリ構造とbackend設定でstateを分割します。

### 基本的な分割パターン

[directory-structure.md](directory-structure.md#マルチ環境対応構成-実務推奨)で紹介した構造を詳しく見ていきます。

```
environments/
└── prod/
    ├── network/      # ネットワーク層のstate
    ├── database/     # データベース層のstate
    └── application/  # アプリケーション層のstate
```

### レイヤー間のデータ参照

Terraformでは、`terraform_remote_state`を使って他のレイヤーのoutputを参照します。

=== "environments/prod/network/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"

      backend "s3" {
        bucket         = "my-company-terraform-state-prod"
        key            = "prod/network/terraform.tfstate"
        region         = "ap-northeast-1"
        encrypt        = true
        dynamodb_table = "terraform-state-lock-prod"
      }
    }

    module "network" {
      source = "../../../modules/network"

      vpc_cidr    = "10.0.0.0/16"
      environment = "prod"
    }

    # 他のレイヤーから参照できるようにoutput
    output "vpc_id" {
      value = module.network.vpc_id
    }

    output "database_subnet_ids" {
      value = module.network.database_subnet_ids
    }

    output "app_subnet_ids" {
      value = module.network.app_subnet_ids
    }
    ```

=== "environments/prod/database/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"

      backend "s3" {
        bucket         = "my-company-terraform-state-prod"
        key            = "prod/database/terraform.tfstate"
        region         = "ap-northeast-1"
        encrypt        = true
        dynamodb_table = "terraform-state-lock-prod"
      }
    }

    # ネットワーク層のstateからデータを取得
    data "terraform_remote_state" "network" {
      backend = "s3"
      config = {
        bucket = "my-company-terraform-state-prod"
        key    = "prod/network/terraform.tfstate"
        region = "ap-northeast-1"
      }
    }

    module "database" {
      source = "../../../modules/database"

      # ネットワーク層のoutputを使用
      vpc_id         = data.terraform_remote_state.network.outputs.vpc_id
      subnet_ids     = data.terraform_remote_state.network.outputs.database_subnet_ids
      instance_class = "db.t3.medium"
      multi_az       = true
      environment    = "prod"
    }

    # アプリケーション層から参照できるようにoutput
    output "database_endpoint" {
      value = module.database.endpoint
    }

    output "database_port" {
      value = module.database.port
    }
    ```

=== "environments/prod/application/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"

      backend "s3" {
        bucket         = "my-company-terraform-state-prod"
        key            = "prod/application/terraform.tfstate"
        region         = "ap-northeast-1"
        encrypt        = true
        dynamodb_table = "terraform-state-lock-prod"
      }
    }

    # ネットワーク層とデータベース層のstateからデータを取得
    data "terraform_remote_state" "network" {
      backend = "s3"
      config = {
        bucket = "my-company-terraform-state-prod"
        key    = "prod/network/terraform.tfstate"
        region = "ap-northeast-1"
      }
    }

    data "terraform_remote_state" "database" {
      backend = "s3"
      config = {
        bucket = "my-company-terraform-state-prod"
        key    = "prod/database/terraform.tfstate"
        region = "ap-northeast-1"
      }
    }

    module "application" {
      source = "../../../modules/application"

      # 他のレイヤーのoutputを使用
      vpc_id            = data.terraform_remote_state.network.outputs.vpc_id
      subnet_ids        = data.terraform_remote_state.network.outputs.app_subnet_ids
      database_endpoint = data.terraform_remote_state.database.outputs.database_endpoint
      database_port     = data.terraform_remote_state.database.outputs.database_port
      environment       = "prod"
    }
    ```

### SSM Parameter Storeを使った疎結合な参照

`terraform_remote_state`の代わりに、SSM Parameter Storeを経由して値を共有することもできます。

=== "environments/prod/network/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"

      backend "s3" {
        bucket         = "my-company-terraform-state-prod"
        key            = "prod/network/terraform.tfstate"
        region         = "ap-northeast-1"
        encrypt        = true
        dynamodb_table = "terraform-state-lock-prod"
      }
    }

    module "network" {
      source = "../../../modules/network"

      vpc_cidr    = "10.0.0.0/16"
      environment = "prod"
    }

    # SSM Parameter Storeに値を保存
    resource "aws_ssm_parameter" "vpc_id" {
      name  = "/myapp/prod/network/vpc-id"
      type  = "String"
      value = module.network.vpc_id
    }

    resource "aws_ssm_parameter" "database_subnet_ids" {
      name  = "/myapp/prod/network/database-subnet-ids"
      type  = "StringList"
      value = join(",", module.network.database_subnet_ids)
    }

    resource "aws_ssm_parameter" "app_subnet_ids" {
      name  = "/myapp/prod/network/app-subnet-ids"
      type  = "StringList"
      value = join(",", module.network.app_subnet_ids)
    }
    ```

=== "environments/prod/database/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"

      backend "s3" {
        bucket         = "my-company-terraform-state-prod"
        key            = "prod/database/terraform.tfstate"
        region         = "ap-northeast-1"
        encrypt        = true
        dynamodb_table = "terraform-state-lock-prod"
      }
    }

    # SSM Parameter Storeから値を取得
    data "aws_ssm_parameter" "vpc_id" {
      name = "/myapp/prod/network/vpc-id"
    }

    data "aws_ssm_parameter" "database_subnet_ids" {
      name = "/myapp/prod/network/database-subnet-ids"
    }

    module "database" {
      source = "../../../modules/database"

      vpc_id         = data.aws_ssm_parameter.vpc_id.value
      subnet_ids     = split(",", data.aws_ssm_parameter.database_subnet_ids.value)
      instance_class = "db.t3.medium"
      multi_az       = true
      environment    = "prod"
    }
    ```

**メリット:**

- レイヤー間の依存関係が疎結合になる
- stateファイルへの直接アクセスが不要
- 異なるデプロイサイクルのレイヤー間で値を共有しやすい

**デメリット:**

- Parameter Storeへの読み書きが必要（わずかなコスト）
- リスト型の値は`StringList`として保存し、`split()`で変換が必要
- Parameter Storeが存在しない場合のエラーハンドリングが必要
- Terraformの`output`定義のような値の説明（description）が分離される

### デプロイ順序

```bash
# 1. ネットワーク層 (依存なし)
cd environments/prod/network
terraform init
terraform apply

# 2. データベース層 (ネットワークに依存)
cd ../database
terraform init
terraform apply

# 3. アプリケーション層 (ネットワークとデータベースに依存)
cd ../application
terraform init
terraform apply
```

## ベストプラクティス

### 1. 機能単位での分割判断

**✅ 適切な分割:**

- 関連するリソースをまとめる
- 機能的な凝集性が高い

```
✅ 良い例: 機能ごとにグループ化
├── network/        # VPC、サブネット、ルートテーブル、NAT Gateway
├── database/       # RDS、DynamoDB、ElastiCache
└── application/    # Lambda、ECS、API Gateway、ALB
```

**❌ 避けるべき分割:**

- リソースタイプごとに過度に細分化
- 機能的な関連性がない

```
❌ 悪い例: リソースタイプごとに分割
├── vpc/
├── subnets/
├── route-tables/
├── security-groups/
└── ... (管理が複雑化)
```

### 2. ライフサイクルでの分割判断

**✅ 適切な分割:**

- 変更頻度が大きく異なるリソースは分離

```
✅ 良い例: 変更頻度で分離
├── foundation/     # 変更頻度が低い（VPC、セキュリティグループ等）
├── data/           # 変更頻度が中程度（RDS、DynamoDB等）
└── application/    # 変更頻度が高い（Lambda、ECS等）
```

**❌ 避けるべき分割:**

- 同じ変更頻度のリソースを過度に分割

### 3. 所有者/責任範囲での分割判断

**✅ 適切な分割:**

- チームの責任範囲に沿って分割
- 各チームが独立してデプロイ可能

```
✅ 良い例: チームごとに分割
├── platform-team/
│   ├── foundation/
│   └── monitoring/
├── data-team/
│   └── database/
└── service-team-a/
    └── application/
```

**❌ 避けるべき分割:**

- 複数チームが1つのスタック/stateを共有
- チームの境界を無視した分割

### 4. 全てを1つにまとめない

**❌ 避けるべき:**

```
❌ 悪い例: 全てが1つ
└── all-resources/  # 影響範囲が大きく、並行作業不可
```

### 5. 依存関係の管理

**原則:**

- ✅ **単方向の依存**: Foundation → Data → Application
- ✅ **明示的な依存**: プロパティやremote stateで明示
- ❌ **循環依存の禁止**: A → B → A のような循環は避ける（CDK、Terraformなど多くのIaCツールでエラーが発生します）

**依存の方向:**

```
Foundation (基盤)
    ↓
Data (データ)
    ↓
Application (アプリケーション)
```

**依存関係の実装方法:**

| 方法 | メリット | デメリット |
|-----|---------|-----------|
| **CDK: プロパティ渡し** | 型安全、自動管理 | CloudFormation Export制限あり |
| **CDK: Export** | Export名を明示制御 | CloudFormation Export制限あり |
| **CDK: SSM Parameter Store** | 疎結合、削除制限なし | 型安全性低下 |
| **Terraform: remote_state** | 型情報あり、標準的 | stateファイルへのアクセス必要 |
| **Terraform: SSM Parameter Store** | 疎結合、state不要 | 型情報の説明が分離 |

!!! tip "疎結合な設計の推奨"
    大規模プロジェクトや異なるチームが管理するレイヤー間では、SSM Parameter Storeを使った疎結合な設計が推奨されます。削除制限がなく、デプロイサイクルの独立性が高まります。

### 6. 環境ごとの分離

環境間でstateやスタックは**完全に分離**します。

**CDK:**

```typescript
// 環境ごとに異なるスタック名
const networkStack = new NetworkStack(app, `${env}-NetworkStack`, {
  env: { account: '123456789012', region: 'ap-northeast-1' },
});
```

**Terraform:**

```
environments/
├── dev/
│   ├── network/
│   ├── database/
│   └── application/
└── prod/
    ├── network/
    ├── database/
    └── application/
```

### 7. ドキュメント化

各レイヤーのREADME.mdに以下を記載:

```markdown
# Network Layer

## 責任範囲

- VPC
- Subnets
- Route Tables
- NAT Gateways

## 依存関係

なし (最下層)

## このレイヤーに依存するレイヤー

- database
- application

## デプロイ頻度
低頻度
```

## まとめ

適切な管理単位への分割は、IaCプロジェクトの成功に不可欠です:

**管理単位の分割:**

- **CDK**: Stackクラスで分割
- **Terraform**: ディレクトリとbackendで分割
- **共通**: ライフサイクル、責任範囲、チーム構造に応じて適切に分割

**依存関係の管理:**

- **CDK**: プロパティ渡し（同一アプリ内）またはSSM Parameter Store（異なるアプリ/Stage間）
- **Terraform**: terraform_remote_state（同一プロジェクト内）またはSSM Parameter Store（異なるプロジェクト間）
- **注意**: CloudFormation Exportsは削除制限があるため、疎結合が必要な場合はSSM Parameter Storeを推奨

**分割のメリット:**

- ✅ 変更の影響範囲を限定
- ✅ 並行作業を可能にする
- ✅ リスクを分離
- ✅ デプロイ時間の短縮
- ✅ 責任の明確化

さらに学びを深めるには、公式ドキュメントやコミュニティのベストプラクティスを参照してください。

[参考リンク集を見る →](references.md){ .md-button .md-button--primary }
