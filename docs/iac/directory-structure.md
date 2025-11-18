# ディレクトリ構成

IaCプロジェクトでは、適切なディレクトリ構成が保守性と拡張性を大きく左右します。このセクションでは、CDKとTerraformそれぞれの推奨ディレクトリ構成を紹介します。

## なぜディレクトリ構成が重要なのか

### コードの見通しが良くなる

- **役割の明確化**: ファイルの配置場所から役割が理解できる
- **変更箇所の特定**: 修正が必要なファイルをすぐに見つけられる
- **チーム開発**: メンバー全員が同じ構造を理解できる

### スケーラビリティ

- **段階的な成長**: プロジェクトの拡大に対応できる
- **モジュール化**: 再利用可能なコンポーネントを整理
- **環境分離**: 開発・ステージング・本番を明確に分離

### 運用の安全性

- **責任範囲の明確化**: どのファイルがどのリソースを管理しているか明確
- **変更の影響範囲**: 修正が他の環境に影響しないことを保証
- **レビューしやすさ**: Pull Requestでの変更範囲が理解しやすい

## CDKのディレクトリ構成

### 基本構成 (学習・概念理解用)

!!! info "実務での位置づけ"
    この構成は**学習用やCDKの概念理解**に適しています。実務では、ほぼ必ず複数環境(dev/staging/prod)が必要になるため、**マルチ環境対応構成**から始めることを強く推奨します。

```
my-cdk-project/
├── bin/
│   └── app.ts                    # CDKアプリのエントリーポイント
├── lib/
│   ├── stacks/
│   │   ├── network-stack.ts      # VPC、サブネットなどのネットワーク
│   │   ├── database-stack.ts     # RDS、DynamoDBなど
│   │   └── application-stack.ts  # Lambda、API Gatewayなど
│   └── constructs/
│       ├── secure-bucket.ts      # 再利用可能なS3バケット
│       └── monitoring.ts         # CloudWatch、アラーム設定
├── test/
│   ├── stacks/
│   │   └── network-stack.test.ts
│   └── constructs/
│       └── secure-bucket.test.ts
├── cdk.json                      # CDK設定ファイル
├── package.json
├── tsconfig.json
└── README.md
```

**ディレクトリの役割:**

- **`bin/`**: CDKアプリケーションのエントリーポイント。環境変数の読み込みやスタックのインスタンス化を行う
- **`lib/stacks/`**: CloudFormationスタック定義。責務ごとに分割
- **`lib/constructs/`**: 再利用可能なコンポーネント(L2/L3 Construct)
- **`test/`**: 単体テスト。ソースコードと同じ構造で配置

### マルチ環境対応構成 (実務推奨)

**実務で最も一般的に採用される構成**です。環境ごとの設定を`lib/config/`ディレクトリで管理します。

!!! tip "実務での推奨"
    実務では、ほぼ必ず複数環境(dev/staging/prod)が必要になります。この構成を習得すれば、実プロジェクトですぐに活用できます。

```
my-cdk-project/
├── bin/
│   └── app.ts
├── lib/
│   ├── stacks/
│   │   ├── network-stack.ts
│   │   ├── database-stack.ts
│   │   └── application-stack.ts
│   ├── constructs/
│   │   ├── secure-bucket.ts
│   │   └── monitoring.ts
│   └── config/
│       ├── types.ts              # 環境設定の型定義
│       ├── dev.ts                # 開発環境設定
│       ├── staging.ts            # ステージング環境設定
│       └── prod.ts               # 本番環境設定
├── test/
│   ├── stacks/
│   └── constructs/
├── cdk.json
├── package.json
├── tsconfig.json
└── README.md
```

**環境設定の例:**

=== "lib/config/types.ts"
    ```typescript
    export interface EnvironmentConfig {
      env: {
        account: string;
        region: string;
      };
      database: {
        instanceType: string;
        multiAz: boolean;
      };
      monitoring: {
        enableDetailedMonitoring: boolean;
        alarmEmail: string;
      };
    }
    ```

=== "lib/config/prod.ts"
    ```typescript
    import { EnvironmentConfig } from './types';

    export const prodConfig: EnvironmentConfig = {
      env: {
        account: process.env.CDK_PROD_ACCOUNT!,
        region: 'ap-northeast-1',
      },
      database: {
        instanceType: 't3.medium',
        multiAz: true,
      },
      monitoring: {
        enableDetailedMonitoring: true,
        alarmEmail: 'prod-alerts@example.com',
      },
    };
    ```

=== "bin/app.ts (CDK Context使用)"
    ```typescript
    #!/usr/bin/env node
    import 'source-map-support/register';
    import * as cdk from 'aws-cdk-lib';
    import { NetworkStack } from '../lib/stacks/network-stack';
    import { DatabaseStack } from '../lib/stacks/database-stack';
    import { devConfig } from '../lib/config/dev';
    import { prodConfig } from '../lib/config/prod';

    const app = new cdk.App();

    // CDK Contextから環境を取得
    const environment = app.node.tryGetContext('environment') || 'dev';
    const config = environment === 'prod' ? prodConfig : devConfig;

    // スタックのデプロイ
    const networkStack = new NetworkStack(app, `${environment}-NetworkStack`, {
      env: config.env,
    });

    new DatabaseStack(app, `${environment}-DatabaseStack`, {
      env: config.env,
      vpc: networkStack.vpc,
      config: config.database,
    });
    ```

=== "bin/app.ts (環境変数使用)"
    ```typescript
    #!/usr/bin/env node
    import 'source-map-support/register';
    import * as cdk from 'aws-cdk-lib';
    import { NetworkStack } from '../lib/stacks/network-stack';
    import { DatabaseStack } from '../lib/stacks/database-stack';
    import { devConfig } from '../lib/config/dev';
    import { prodConfig } from '../lib/config/prod';

    const app = new cdk.App();

    // 環境変数から環境を取得
    const environment = process.env.ENV || 'dev';
    const config = environment === 'prod' ? prodConfig : devConfig;

    // スタックのデプロイ
    const networkStack = new NetworkStack(app, `${environment}-NetworkStack`, {
      env: config.env,
    });

    new DatabaseStack(app, `${environment}-DatabaseStack`, {
      env: config.env,
      vpc: networkStack.vpc,
      config: config.database,
    });
    ```

=== "package.json (npm scripts)"
    ```json
    {
      "name": "my-cdk-project",
      "scripts": {
        "deploy:dev": "ENV=dev cdk deploy --all",
        "deploy:staging": "ENV=staging cdk deploy --all",
        "deploy:prod": "ENV=prod cdk deploy --all",
        "diff:dev": "ENV=dev cdk diff",
        "diff:prod": "ENV=prod cdk diff"
      }
    }
    ```

**デプロイコマンド:**

=== "CDK Context"
    ```bash
    # 開発環境
    cdk deploy --context environment=dev --all

    # 本番環境
    cdk deploy --context environment=prod --all
    ```

=== "環境変数"
    ```bash
    # 開発環境
    ENV=dev cdk deploy --all

    # 本番環境
    ENV=prod cdk deploy --all
    ```

=== "npm scripts (推奨)"
    ```bash
    # 開発環境
    npm run deploy:dev

    # 本番環境
    npm run deploy:prod

    # 差分確認
    npm run diff:prod
    ```

### 大規模プロジェクト構成

複数のサービスやマイクロサービスを管理する場合の構成です。

```
my-cdk-monorepo/
├── packages/
│   ├── common/                   # 共通ライブラリ
│   │   ├── lib/
│   │   │   ├── constructs/
│   │   │   │   ├── secure-bucket.ts
│   │   │   │   └── monitoring.ts
│   │   │   └── utils/
│   │   │       └── naming.ts
│   │   ├── test/
│   │   └── package.json
│   │
│   ├── infrastructure/           # 基盤インフラ
│   │   ├── bin/
│   │   │   └── app.ts
│   │   ├── lib/
│   │   │   └── stacks/
│   │   │       ├── network-stack.ts
│   │   │       └── security-stack.ts
│   │   ├── test/
│   │   ├── cdk.json
│   │   └── package.json
│   │
│   ├── service-a/                # サービスA
│   │   ├── bin/
│   │   ├── lib/
│   │   │   └── stacks/
│   │   │       ├── api-stack.ts
│   │   │       └── database-stack.ts
│   │   ├── test/
│   │   ├── cdk.json
│   │   └── package.json
│   │
│   └── service-b/                # サービスB
│       ├── bin/
│       ├── lib/
│       ├── test/
│       ├── cdk.json
│       └── package.json
├── package.json                  # ルートpackage.json (workspace設定)
└── README.md
```

**ルートpackage.jsonの例:**

```json
{
  "name": "my-cdk-monorepo",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  }
}
```

## Terraformのディレクトリ構成

### 基本構成 (学習・概念理解用)

!!! info "実務での位置づけ"
    この構成は**学習用やTerraformの概念理解**に適しています。実務では、ほぼ必ず複数環境(dev/staging/prod)が必要になるため、**マルチ環境対応構成**(ディレクトリ分離またはWorkspaces)から始めることを強く推奨します。

```
my-terraform-project/
├── main.tf                       # メインの構成定義
├── variables.tf                  # 入力変数の定義
├── outputs.tf                    # 出力値の定義
├── providers.tf                  # プロバイダー設定
├── terraform.tfvars              # 変数の値(Gitignore推奨)
├── versions.tf                   # Terraformとプロバイダーのバージョン制約
├── modules/
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── tests/
│   ├── network.tftest.hcl
│   └── database.tftest.hcl
└── README.md
```

**ファイルの役割:**

- **`main.tf`**: メインのリソース定義
- **`variables.tf`**: 入力変数の宣言
- **`outputs.tf`**: 他のモジュールや外部から参照する値
- **`providers.tf`**: AWSプロバイダーなどの設定
- **`versions.tf`**: Terraformとプロバイダーのバージョン指定
- **`modules/`**: 再利用可能なモジュール

### マルチ環境対応構成 (実務推奨)

**実務で最も一般的に採用される構成**です。環境ごとにディレクトリを分離し、さらにレイヤー(network/database/application)ごとにstateを分割します。

!!! info "stateの分割"
    この例では、レイヤーごとにstateファイルを分割しています。これにより、あるレイヤーの変更が他のレイヤーのstateに影響を与えず、並行作業やリスク分離が可能になります。詳細は[管理単位のベストプラクティス](management-units.md)を参照してください。

```
my-terraform-project/
├── modules/                      # 共通モジュール
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── application/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/
│   ├── dev/
│   │   ├── network/              # ネットワーク層のstate
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── providers.tf
│   │   │   └── backend.tf
│   │   ├── database/             # データベース層のstate
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── providers.tf
│   │   │   └── backend.tf
│   │   └── application/          # アプリケーション層のstate
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── providers.tf
│   │       └── backend.tf
│   │
│   └── prod/
│       ├── network/
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   ├── outputs.tf
│       │   ├── providers.tf
│       │   └── backend.tf
│       ├── database/
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   ├── outputs.tf
│       │   ├── providers.tf
│       │   └── backend.tf
│       └── application/
│           ├── main.tf
│           ├── variables.tf
│           ├── outputs.tf
│           ├── providers.tf
│           └── backend.tf
│
├── tests/
│   └── modules/
│       ├── network.tftest.hcl
│       └── database.tftest.hcl
└── README.md
```

**レイヤーごとのmain.tfの例:**

=== "environments/prod/network/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"
    }

    module "network" {
      source = "../../../modules/network"

      vpc_cidr          = var.vpc_cidr
      environment       = "prod"
      enable_nat        = true
      nat_gateway_count = 3  # 本番は3AZ
    }
    ```

=== "environments/prod/database/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"
    }

    # ネットワーク層のoutputを参照
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

      vpc_id         = data.terraform_remote_state.network.outputs.vpc_id
      subnet_ids     = data.terraform_remote_state.network.outputs.database_subnet_ids
      instance_class = "db.t3.medium"
      multi_az       = true
      environment    = "prod"
    }
    ```

=== "environments/prod/application/main.tf"
    ```hcl
    terraform {
      required_version = ">= 1.6.0"
    }

    # 他のレイヤーのoutputを参照
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

      vpc_id              = data.terraform_remote_state.network.outputs.vpc_id
      subnet_ids          = data.terraform_remote_state.network.outputs.app_subnet_ids
      database_endpoint   = data.terraform_remote_state.database.outputs.endpoint
      environment         = "prod"
    }
    ```

**backend.tfの例:**

```hcl
# environments/prod/network/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-prod"
    key            = "prod/network/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock-prod"
  }
}
```

```hcl
# environments/prod/database/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-prod"
    key            = "prod/database/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock-prod"
  }
}
```

**デプロイコマンド:**

```bash
# 本番環境のネットワーク層をデプロイ
cd environments/prod/network
terraform init
terraform plan
terraform apply

# ネットワーク層に依存するデータベース層をデプロイ
cd ../database
terraform init
terraform plan
terraform apply

# 両方に依存するアプリケーション層をデプロイ
cd ../application
terraform init
terraform plan
terraform apply
```

### ワークスペース利用構成

Terraform Workspacesを使用する場合の構成です。

```
my-terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── backend.tf                    # 共通のバックエンド設定
├── configs/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
├── modules/
│   ├── network/
│   └── database/
└── tests/
    └── *.tftest.hcl
```

**環境別変数ファイルの例:**

=== "configs/prod.tfvars"
    ```hcl
    environment       = "prod"
    vpc_cidr          = "10.0.0.0/16"
    instance_class    = "db.t3.medium"
    multi_az          = true
    enable_monitoring = true
    ```

=== "configs/dev.tfvars"
    ```hcl
    environment       = "dev"
    vpc_cidr          = "10.10.0.0/16"
    instance_class    = "db.t3.micro"
    multi_az          = false
    enable_monitoring = false
    ```

**ワークスペースの使用方法:**

```bash
# 開発環境用ワークスペース
terraform workspace new dev
terraform workspace select dev
terraform apply -var-file="configs/dev.tfvars"

# 本番環境用ワークスペース
terraform workspace new prod
terraform workspace select prod
terraform apply -var-file="configs/prod.tfvars"
```

## ベストプラクティス

### 1. 環境分離の選択

=== "CDK"
    **推奨**: 環境設定ファイルでの分離

    - 環境ごとの設定を明示的に管理
    - スタック名にプレフィックスを付与(`dev-`, `prod-`)
    - CDK ContextまたはnpmScripts＋環境変数で環境を切り替え

    **環境切り替え方法の比較**:

    | 方法 | メリット | デメリット |
    |------|---------|-----------|
    | **CDK Context** | CDK公式の推奨方法、`cdk.json`に保存可能 | コマンドが長くなりがち |
    | **環境変数** | シンプル、CI/CDとの統合が容易 | typoに気づきにくい |
    | **npm scripts (推奨)** | わかりやすい、typoを防げる | package.jsonの設定が必要 |

=== "Terraform"
    **推奨**: ディレクトリ分離 or Workspaces

    **ディレクトリ分離**:

    - ✅ 環境ごとのstateが完全に分離
    - ✅ 環境ごとに異なるバックエンド設定が可能
    - ❌ コードの重複が発生しやすい

    **Workspaces**:

    - ✅ コードの重複を削減
    - ✅ 環境の切り替えが容易
    - ❌ 同じバックエンドを共有(誤操作のリスク)

### 2. モジュール/Constructの粒度

**適切な粒度**:

- 単一責任の原則に従う
- 独立して再利用可能
- テストしやすい単位

!!! tip "関連セクション"
    モジュール化の詳細とDRY原則については、[コーディング原則](coding-principles.md#1-dry原則-dont-repeat-yourself)も参照してください。

**例:**

```
✅ 良い例: secure-bucket (セキュアなS3バケットの設定一式)
✅ 良い例: monitoring (CloudWatchアラームとダッシュボード)

❌ 悪い例: aws-resources (様々なリソースの詰め合わせ)
❌ 悪い例: utils (関連性のない汎用ユーティリティ)
```

## まとめ

適切なディレクトリ構成は、IaCプロジェクトの成功に不可欠です:

- **CDK**: スタックとConstructsで責務を分離、環境設定ファイル(`lib/config/`)で環境を管理
- **Terraform**: モジュールで再利用性を高め、環境ごとにディレクトリ(`environments/`)またはWorkspacesで分離
- **共通**: 適切なモジュール粒度で保守性を高める

プロジェクトの規模に応じて、基本構成から始めて段階的に拡張していくアプローチが推奨されます。

次のセクションでは、スタックやstateの適切な分割方法について学びます。

[管理単位のベストプラクティス →](management-units.md){ .md-button .md-button--primary }
