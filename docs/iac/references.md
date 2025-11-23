# 参考リンク集

このページでは、IaCをさらに深く学ぶための公式ドキュメント、ベストプラクティス、有用なリソースを紹介します。

## AWS CDK

### 公式ドキュメント

- **[AWS CDK 公式ドキュメント](https://docs.aws.amazon.com/cdk/)**
  AWS CDKの完全なドキュメント。APIリファレンス、ガイド、チュートリアルが含まれます。

- **[AWS CDK API Reference](https://docs.aws.amazon.com/cdk/api/v2/)**
  CDK v2のAPI詳細リファレンス。各Constructの使い方を確認できます。

- **[AWS CDK Workshop](https://cdkworkshop.com/)**
  ハンズオン形式でCDKを学べる公式ワークショップ。

### ベストプラクティス

- **[AWS CDK Best Practices](https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html)**
  AWS公式のCDKベストプラクティスガイド。

- **[CDK Patterns](https://cdkpatterns.com/)**
  サーバーレスアーキテクチャを中心としたCDKパターン集。実践的な例が豊富です。

### サンプルコード

- **[AWS CDK Examples](https://github.com/aws-samples/aws-cdk-examples)**
  AWS公式のCDKサンプルコードリポジトリ。TypeScript、Python、Java等の例があります。

- **[Awesome CDK](https://github.com/kolomied/awesome-cdk)**
  CDK関連のツール、ライブラリ、記事などをまとめたキュレーションリスト。

## Terraform

### 公式ドキュメント

- **[Terraform 公式ドキュメント](https://developer.hashicorp.com/terraform/docs)**
  HashiCorp公式のTerraformドキュメント。

- **[Terraform AWS Provider ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)**
  AWS Providerの全リソースとデータソースのリファレンス。

- **[Terraform Language Documentation](https://developer.hashicorp.com/terraform/language)**
  HCL（HashiCorp Configuration Language）の詳細な言語仕様。

### ベストプラクティス

- **[Terraform Best Practices](https://www.terraform-best-practices.com/)**
  コミュニティによるTerraformベストプラクティスガイド。実務で役立つ推奨事項が多数。

- **[Google Cloud Terraform Best Practices](https://cloud.google.com/docs/terraform/best-practices-for-terraform)**
  Google Cloudによるベストプラクティス。AWS環境でも適用できる原則が多く含まれます。

### サンプルコード

- **[Terraform AWS Modules](https://github.com/terraform-aws-modules)**
  再利用可能なAWS用Terraformモジュールの公式コレクション。VPC、ECS、RDS等。

- **[Gruntwork Infrastructure as Code Library](https://gruntwork.io/infrastructure-as-code-library/)**
  プロダクショングレードのTerraformモジュール集（一部有料）。

## 共通リソース

### IaC全般

- **[Infrastructure as Code (書籍)](https://www.oreilly.com/library/view/infrastructure-as-code/9781098114664/)**
  Kief Morris著。IaCの原則と実践を包括的に解説した定番書籍。

- **[Pulumi vs CDK vs Terraform 比較](https://www.pulumi.com/docs/concepts/vs/)**
  各IaCツールの特徴と違いを理解するための比較ガイド。

### テスト

- **[cdkassert (AWS CDK Testing)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.assertions-readme.html)**
  CDK公式のテストライブラリドキュメント。

- **[Terratest](https://terratest.gruntwork.io/)**
  Terraform、Packer等のインフラコードをテストするためのGoライブラリ。

- **[LocalStack](https://localstack.cloud/)**
  AWSサービスをローカル環境でエミュレートするツール。開発・テストに有用。

### CI/CD

- **[GitHub Actions for Terraform](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions)**
  GitHub ActionsでTerraformを自動化するチュートリアル。

### セキュリティ

- **[tfsec](https://github.com/aquasecurity/tfsec)**
  Terraformコードの静的セキュリティ分析ツール。

- **[Checkov](https://www.checkov.io/)**
  Terraform、CloudFormation、Kubernetes等の設定をスキャンするセキュリティツール。

- **[cdk-nag](https://github.com/cdklabs/cdk-nag)**
  CDKアプリケーションにセキュリティとコンプライアンスのルールを適用するツール。

## コミュニティ

### フォーラム・ディスカッション

- **[AWS CDK Slack](https://cdk.dev/)**
  CDKコミュニティのSlackワークスペース。

- **[Terraform Discuss](https://discuss.hashicorp.com/c/terraform-core/27)**
  HashiCorp公式のTerraformフォーラム。

### ブログ・記事

- **[AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/)**
  AWSアーキテクチャに関する公式ブログ。IaCのベストプラクティスも掲載。

- **[HashiCorp Blog](https://www.hashicorp.com/blog)**
  Terraform関連の最新情報やユースケースを紹介。

## 次のステップ

これらのリソースを活用して、実際のプロジェクトでIaCを実践してみましょう。

- 小規模なプロジェクトから始める
- テストを書く習慣をつける
- コミュニティに参加して知見を共有する
- 継続的に最新のベストプラクティスをキャッチアップする

[IaCトップページに戻る →](index.md){ .md-button }
