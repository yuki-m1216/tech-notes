# セキュアな開発プロセス

**📝 このページは現在編集中です**

このセクションでは、開発段階からセキュリティを組み込む考え方について学びます。

## 予定している内容

### Shift Left の考え方

- なぜ開発段階でセキュリティを考えるのか
- 後工程での修正コストの増大
- セキュリティを開発プロセスに組み込む利点

### セキュアコーディング

- 代表的な脆弱性とその対策
- OWASP Top 10の理解
- 入力検証の原則
- 出力エスケープの重要性

### セキュリティテストの自動化

- SAST (Static Application Security Testing) とは
- DAST (Dynamic Application Security Testing) とは
- SASTとDASTの使い分け
- CI/CDパイプラインへの統合戦略

### シークレット管理

- なぜシークレットをコードに含めてはいけないか
- シークレットのライフサイクル管理
- シークレットローテーション戦略
- 環境変数と専用サービスの使い分け

### AWS環境での実装

- AWS Secrets ManagerとParameter Storeの違い
- CodePipelineでのセキュリティゲート
- よくある失敗パターン

[脆弱性管理と重大度評価を学ぶ →](vulnerability-management.md){ .md-button }
