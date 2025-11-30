# テストの重要性

IaCコードも通常のアプリケーションコードと同様に、**テストが不可欠**です。インフラの変更は本番環境に直接影響するため、事前にテストで品質を保証することが重要です。

## なぜIaCにテストが必要なのか

### 本番環境への影響を最小化

インフラの変更は、アプリケーションの可用性やセキュリティに直結します。

- **ダウンタイムの防止**: デプロイ前にエラーを検出
- **セキュリティリスクの回避**: 意図しない公開設定などを事前に発見
- **コスト最適化**: 不要なリソース作成を防ぐ

### リファクタリングの安全性

テストがあることで、安心してコードを改善できます。

- コードをリファクタリングしても、期待通りの動作を保証
- 新しいメンバーが変更を加える際の安全網

### ドキュメントとしての役割

テストコードは「期待される動作」を示すドキュメントになります。

- インフラの要件を明示的に記述
- チームメンバーが仕様を理解しやすくなる

## IaCテストの種類

IaCのテストは、テストの対象と範囲によって複数のレベルに分類されます。

### 1. 静的解析・Lint

**コードの品質をチェック**します。実際にリソースを作成せず、コード自体を検証します。

**目的**:

- コーディング規約違反の検出
- ベストプラクティスからの逸脱を発見
- セキュリティリスクの早期発見

**実行タイミング**: コード編集中、コミット前、CI/CD

=== "CDK (TypeScript)"
    ```bash
    # TypeScript型チェック
    npm run build

    # ESLintによるコード品質チェック
    npm run lint

    # CDK特有の問題をチェック
    cdk synth  # CloudFormationテンプレートを生成して構文エラーを検出
    ```

=== "Terraform (HCL)"
    ```bash
    # フォーマットチェック
    terraform fmt -check

    # 構文チェック
    terraform validate

    # Lintツール(tflint)
    tflint

    # セキュリティスキャン(tfsec)
    tfsec .
    ```

**例: tfsecによるセキュリティチェック**

```hcl
# ❌ tfsecが検出する問題
resource "aws_s3_bucket" "bad" {
  bucket = "my-bucket"
  # 暗号化が有効化されていない → tfsecが警告
}

# ✅ セキュアな設定
resource "aws_s3_bucket" "good" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "good" {
  bucket = aws_s3_bucket.good.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

### 2. 単体テスト (Unit Test)

**個別のモジュールやコンストラクトの動作を検証**します。実際のクラウドリソースは作成せず、生成されるテンプレートをテストします。

**目的**:

- 期待通りのリソースが定義されているか
- 設定値が正しいか
- リソース間の関係が正しいか

**実行タイミング**: 開発中、コミット前、CI/CD

=== "CDK (TypeScript)"
    ```typescript
    // test/secure-bucket.test.ts
    import { Template, Match } from 'aws-cdk-lib/assertions';
    import { Stack } from 'aws-cdk-lib';
    import { SecureBucket } from '../lib/constructs/secure-bucket';

    describe('SecureBucket', () => {
      let stack: Stack;
      let template: Template;

      // 各テストの前に共通のセットアップを実行
      beforeEach(() => {
        stack = new Stack();
        new SecureBucket(stack, 'TestBucket');
        template = Template.fromStack(stack);
      });

      test('S3バケットが1つ作成される', () => {
        template.resourceCountIs('AWS::S3::Bucket', 1);
      });

      test('暗号化が有効化されている', () => {
        // 暗号化設定が含まれている
        template.hasResourceProperties('AWS::S3::Bucket', {
          BucketEncryption: {
            ServerSideEncryptionConfiguration: [{
              ServerSideEncryptionByDefault: {
                SSEAlgorithm: 'AES256'
              }
            }]
          }
        });
      });

      test('パブリックアクセスがブロックされている', () => {
        template.hasResourceProperties('AWS::S3::Bucket', {
          PublicAccessBlockConfiguration: {
            BlockPublicAcls: true,
            BlockPublicPolicy: true,
            IgnorePublicAcls: true,
            RestrictPublicBuckets: true
          }
        });
      });

      test('バージョニングが有効化されている', () => {
        template.hasResourceProperties('AWS::S3::Bucket', {
          VersioningConfiguration: {
            Status: 'Enabled'
          }
        });
      });

      test('SSL通信が強制されている', () => {
        // BucketPolicyでSSL通信を強制
        template.hasResourceProperties('AWS::S3::BucketPolicy', {
          PolicyDocument: Match.objectLike({
            Statement: Match.arrayWith([
              Match.objectLike({
                Effect: 'Deny',
                Condition: {
                  Bool: {
                    'aws:SecureTransport': 'false'
                  }
                }
              })
            ])
          })
        });
      });
    });
    ```

    ```bash
    # テスト実行
    npm test
    ```

    **使用するライブラリ:**

    ```json
    {
      "devDependencies": {
        "@types/jest": "^29.5.0",
        "@types/node": "^20.0.0",
        "aws-cdk-lib": "^2.100.0",
        "constructs": "^10.0.0",
        "jest": "^29.5.0",
        "ts-jest": "^29.1.0",
        "typescript": "^5.0.0"
      }
    }
    ```

=== "Terraform (HCL)"
    Terraformには複数のテスト方法があります。

    **1. `terraform test` (Terraform 1.6.0以降の公式機能)**

    最新のTerraformでは、公式のテスト機能が利用可能です。

    ```hcl
    # tests/s3_security.tftest.hcl

    # テスト設定
    variables {
      bucket_name = "test-secure-bucket"
      environment = "testing"
    }

    # S3バケットが1つ作成されることを確認
    run "check_bucket_count" {
      command = plan

      assert {
        condition     = length([aws_s3_bucket.secure_bucket]) == 1
        error_message = "S3バケットが1つ作成される必要があります"
      }
    }

    # 暗号化設定の検証
    run "check_encryption" {
      command = plan

      assert {
        condition = (
          aws_s3_bucket_server_side_encryption_configuration.secure_bucket.rule[0]
          .apply_server_side_encryption_by_default[0].sse_algorithm == "AES256"
        )
        error_message = "S3バケットはAES256で暗号化される必要があります"
      }
    }

    # パブリックアクセスブロックの検証
    run "check_public_access_block" {
      command = plan

      # すべてのパブリックアクセス設定を確認
      assert {
        condition = (
          aws_s3_bucket_public_access_block.secure_bucket.block_public_acls == true &&
          aws_s3_bucket_public_access_block.secure_bucket.block_public_policy == true &&
          aws_s3_bucket_public_access_block.secure_bucket.ignore_public_acls == true &&
          aws_s3_bucket_public_access_block.secure_bucket.restrict_public_buckets == true
        )
        error_message = "すべてのパブリックアクセスをブロックする必要があります"
      }
    }

    # バージョニング設定の検証
    run "check_versioning" {
      command = plan

      assert {
        condition = (
          aws_s3_bucket_versioning.secure_bucket.versioning_configuration[0].status == "Enabled"
        )
        error_message = "バケットのバージョニングを有効にする必要があります"
      }
    }

    # 無効な入力が失敗することを確認(ネガティブテスト)
    run "invalid_bucket_name_should_fail" {
      command = plan

      variables {
        bucket_name = "INVALID_BUCKET_NAME"  # 大文字は許可されない
      }

      expect_failures = [
        var.bucket_name  # バリデーションエラーが発生することを期待
      ]
    }
    ```

    ```bash
    # テスト実行
    terraform test

    # 特定のテストファイルのみ実行
    terraform test -filter=tests/s3_security.tftest.hcl
    ```

    **モック機能の活用 (Terraform 1.7.0以降)**

    外部サービスやデータソースをモック化して、実際のAWSリソースなしでテスト可能です。

    ```hcl
    # tests/with_mock.tftest.hcl

    # モックプロバイダーの設定
    mock_provider "aws" {
      # モックデータの定義
      mock_data "aws_caller_identity" {
        defaults = {
          account_id = "123456789012"
          arn        = "arn:aws:iam::123456789012:user/test"
          user_id    = "AIDACKCEVSQ6C2EXAMPLE"
        }
      }

      # 特定のリソースのモック
      mock_resource "aws_s3_bucket" {
        defaults = {
          id  = "test-bucket"
          arn = "arn:aws:s3:::test-bucket"
        }
      }
    }

    run "test_with_mocked_aws" {
      command = plan

      # モックを使用したテスト
      assert {
        condition     = aws_s3_bucket.secure_bucket.id != ""
        error_message = "バケットIDが設定されている必要があります"
      }
    }

    # データソースのモック例
    run "test_with_mocked_data_source" {
      command = plan

      # VPCデータソースをモック
      override_data {
        target = data.aws_vpc.existing
        values = {
          id         = "vpc-12345678"
          cidr_block = "10.0.0.0/16"
        }
      }

      assert {
        condition     = data.aws_vpc.existing.id == "vpc-12345678"
        error_message = "VPC IDが正しく取得できません"
      }
    }
    ```

    **`check`ブロックを使った継続的検証**

    `check`ブロックは、インフラの状態を継続的に検証するための機能です(Terraform 1.5.0以降)。

    ```hcl
    # main.tf

    resource "aws_s3_bucket" "secure_bucket" {
      bucket = var.bucket_name
    }

    resource "aws_s3_bucket_server_side_encryption_configuration" "secure_bucket" {
      bucket = aws_s3_bucket.secure_bucket.id

      rule {
        apply_server_side_encryption_by_default {
          sse_algorithm = "AES256"
        }
      }
    }

    # ランタイムチェック: apply後も常に検証
    check "bucket_encryption" {
      # データソースで実際の状態を取得
      data "aws_s3_bucket" "verify" {
        bucket = aws_s3_bucket.secure_bucket.id
      }

      # 暗号化が有効であることを確認
      assert {
        condition = (
          data.aws_s3_bucket.verify.server_side_encryption_configuration != null
        )
        error_message = "バケットの暗号化が設定されていません"
      }
    }

    check "bucket_versioning" {
      data "aws_s3_bucket_versioning" "verify" {
        bucket = aws_s3_bucket.secure_bucket.id
      }

      # バージョニングが有効であることを確認
      assert {
        condition     = data.aws_s3_bucket_versioning.verify.versioning_configuration[0].status == "Enabled"
        error_message = "バケットのバージョニングが無効です"
      }
    }

    # 複数のアサーションを含むチェック
    check "security_compliance" {
      data "aws_s3_bucket_public_access_block" "verify" {
        bucket = aws_s3_bucket.secure_bucket.id
      }

      assert {
        condition     = data.aws_s3_bucket_public_access_block.verify.block_public_acls == true
        error_message = "パブリックACLがブロックされていません"
      }

      assert {
        condition     = data.aws_s3_bucket_public_access_block.verify.block_public_policy == true
        error_message = "パブリックポリシーがブロックされていません"
      }
    }
    ```

    **`check`ブロックと`test`の違い:**

    | 項目 | `terraform test` | `check`ブロック |
    |------|-----------------|----------------|
    | **実行タイミング** | 明示的にテスト時のみ | plan/apply時に常に実行 |
    | **用途** | 開発時の単体・統合テスト | 本番運用時の継続的検証 |
    | **失敗時の挙動** | テスト失敗で停止 | 警告のみ(applyは継続) |
    | **配置場所** | `tests/`ディレクトリ | メインの`.tf`ファイル内 |
    | **モック対応** | 対応 | 非対応(実リソース必須) |

    ```bash
    # checkブロックの検証結果確認
    terraform plan  # チェック結果が表示される
    terraform apply # applyでもチェックが実行される

    # チェック結果の例:
    # Check results:
    # ✓ bucket_encryption passed
    # ✓ bucket_versioning passed
    # ✗ security_compliance failed
    #   - パブリックACLがブロックされていません
    ```

## IaCテストの責務と範囲

IaCテストでは、**「環境を正しく作ること」に集中すべき**です。実際のAWS動作検証は、アプリケーション側のE2E/統合テストで行います。

### IaCテストの責務

**✅ IaCが担当すべきテスト:**

- インフラ定義(CDK/Terraform)が正しいか
- リソースが期待通り定義されているか
- セキュリティ設定が適切か
- 命名規則や構成が組織のルールに準拠しているか

**❌ IaCで行うべきでないテスト:**

- S3にファイルがアップロードできるか → アプリのE2Eテスト
- Lambda関数が正しく動作するか → アプリのE2Eテスト
- API Gatewayが応答を返すか → アプリのE2Eテスト

### 責任の分離

```
┌──────────────────────────────────────────┐
│  IaCテスト (このドキュメントの範囲)      │
│  ──────────────────────────────────────  │
│  ✓ 静的解析 (Lint)                       │
│  ✓ 単体テスト (Template/Plan検証)       │
│  ✓ ポリシーテスト (コンプライアンス)     │
│  → インフラ定義が正しいことを保証        │
└──────────────────────────────────────────┘
                  ↓ デプロイ
┌──────────────────────────────────────────┐
│  アプリケーションE2E/統合テスト           │
│  ──────────────────────────────────────  │
│  ✓ デプロイされた環境での動作検証        │
│  ✓ サービス間連携の確認                  │
│  ✓ 実際のAWS APIの動作確認               │
│  → インフラが期待通り動作することを保証  │
└──────────────────────────────────────────┘
```

**IaCは「土台を正しく作ること」、アプリは「土台が正しく動くこと」に集中する**ことで、責任が明確になり保守しやすくなります。

### 3. ポリシーテスト (Policy as Code)

**ポリシーテストは、組織のセキュリティ・コンプライアンスポリシーに準拠しているかを自動検証する手法です。**

!!! info "実務での位置づけ"
    ポリシーテストは主に**大企業や厳格なコンプライアンス要件がある組織**で導入されている高度な手法です。一般的なプロジェクトでは、静的解析ツール(tflint、tfsec等)で十分なケースが多いです。

**主なツール**:

- **Open Policy Agent (OPA)**: Regoという言語でポリシーを記述する汎用的なポリシーエンジン
- **Sentinel** (Terraform Cloud/Enterprise): Terraformに特化したポリシーエンジン

**適用例**:

- すべてのS3バケットは必ず暗号化する
- 本番環境のRDSインスタンスはマルチAZ構成にする
- 特定のリージョン以外にリソースを作成しない
- タグ付けルール(環境、プロジェクト名など)を強制する

**実行タイミング**: CI/CD、Plan/Synth後、デプロイ前

興味がある方は、[Open Policy Agent公式ドキュメント](https://www.openpolicyagent.org/)を参照してください。

## テスト戦略

### IaCテストピラミッド

IaCテストは、**インフラ定義の正しさ**を保証することに集中します。

```
           △
          ╱ ╲
         ╱   ╲        少ない
        ╱ ポリ ╲      (組織ルール)
       ╱ シー  ╲
      ╱  テスト ╲
     ╱───────────╲
    ╱             ╲   中程度
   ╱   単体テスト   ╲  (モジュール検証)
  ╱                 ╲
 ╱───────────────────╲
╱                     ╲ 多い
      静的解析          (全コード)
─────────────────────────
```

**推奨バランス**:

- **静的解析**: 全てのコードに適用、コミット前に必ず実行(必須)
- **単体テスト**: 重要なモジュール/コンストラクトに適用(推奨)
- **ポリシーテスト**: 厳格なコンプライアンス要件がある場合のみ(オプション)

### CI/CDへの組み込み

テストは自動化して、CI/CDパイプラインに組み込むことが重要です。

=== "CDK (TypeScript)"
    ```yaml
    # .github/workflows/cdk-test.yml (GitHub Actions例)
    name: CDK Test

    on:
      pull_request:
        paths:
          - 'infrastructure/**'

    jobs:
      static-analysis:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4

          - name: Setup Node.js
            uses: actions/setup-node@v4
            with:
              node-version: '20'

          - name: Install dependencies
            run: npm ci

          # 静的解析
          - name: ESLint
            run: npm run lint

          - name: CDK Synth Check
            run: npx cdk synth

      unit-test:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4

          - name: Setup Node.js
            uses: actions/setup-node@v4
            with:
              node-version: '20'

          - name: Install dependencies
            run: npm ci

          # 単体テスト
          - name: Run unit tests
            run: npm test

          - name: Upload coverage
            uses: codecov/codecov-action@v5
            with:
              files: ./coverage/lcov.info

      # ポリシーテスト (オプション: 厳格なコンプライアンス要件がある場合)
      # policy-test:
      #   runs-on: ubuntu-latest
      #   steps:
      #     - uses: actions/checkout@v4
      #     - name: Setup Node.js
      #       uses: actions/setup-node@v4
      #       with:
      #         node-version: '20'
      #     - name: Install dependencies
      #       run: npm ci
      #     - name: CDK Synth
      #       run: npx cdk synth -o cdk.out
      #     - name: Setup OPA
      #       uses: open-policy-agent/setup-opa@v2
      #     - name: Run Policy Tests
      #       run: opa eval --data policy/ --input cdk.out/manifest.json "data.cdk.deny"
    ```

=== "Terraform (HCL)"
    ```yaml
    # .github/workflows/terraform-test.yml (GitHub Actions例)
    name: Terraform Test

    on:
      pull_request:
        paths:
          - 'infrastructure/**'

    jobs:
      static-analysis:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4

          - name: Setup Terraform
            uses: hashicorp/setup-terraform@v3

          # 静的解析
          - name: Terraform Format Check
            run: terraform fmt -check -recursive

          - name: TFLint
            uses: terraform-linters/setup-tflint@v4
          - run: tflint --init
          - run: tflint --recursive

          - name: tfsec
            uses: aquasecurity/tfsec-action@v1.0.0

      unit-test:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4

          - name: Setup Terraform
            uses: hashicorp/setup-terraform@v3

          # 単体テスト
          - name: Terraform Test
            run: terraform test

      # ポリシーテスト (オプション: 厳格なコンプライアンス要件がある場合)
      # policy-test:
      #   runs-on: ubuntu-latest
      #   steps:
      #     - uses: actions/checkout@v4
      #     - name: Setup Terraform
      #       uses: hashicorp/setup-terraform@v3
      #     - name: Terraform Plan
      #       run: |
      #         terraform init -backend=false
      #         terraform plan -out=plan.out
      #         terraform show -json plan.out > plan.json
      #     - name: Setup OPA
      #       uses: open-policy-agent/setup-opa@v2
      #     - name: Run Policy Tests
      #       run: opa eval --data policy/ --input plan.json "data.terraform.deny"
    ```

## テストのベストプラクティス

### 1. 早い段階でテストを書く

- コードと同時にテストを作成(TDD推奨)
- テストがないコードはマージしない

### 2. テストは独立させる

- テスト間で状態を共有しない
- 並列実行可能にする
- テストの実行順序に依存しない

### 3. 失敗時のデバッグ情報を残す

- エラーメッセージを詳細に
- テスト失敗時のテンプレート/Plan出力を保存
- どのアサーションが失敗したか明確にする

### 4. IaCの責務を明確にする

- **IaCテスト**: インフラ定義が正しいことを検証
- **アプリテスト**: デプロイ後の動作検証はアプリケーション側で実施
- 責任を分離することで、テストが保守しやすくなる

## まとめ

IaCのテストは、**インフラ定義の品質と安全性**を保証するために不可欠です:

- **静的解析(必須)**: コーディング規約とセキュリティの基本チェック
- **単体テスト(推奨)**: インフラ定義テンプレート/Planの検証
- **ポリシーテスト(オプション)**: 厳格なコンプライアンス要件がある場合のみ

**重要**: 実際のAWS動作検証(S3へのアップロード、Lambda実行など)は、アプリケーション側のE2E/統合テストで行います。IaCは「環境を正しく作ること」に集中しましょう。

まずは静的解析から始めて、必要に応じて単体テストを追加していくのが現実的なアプローチです。適切なテスト戦略を立て、CI/CDに組み込むことで、安心してインフラの変更を行えるようになります。

次のセクションでは、実際のプロジェクトでの推奨ディレクトリ構成について学びます。

[ディレクトリ構成 →](directory-structure.md){ .md-button .md-button--primary }
