# モニタリング・アラート

システムの安定運用に欠かせないモニタリングとアラートの設計について学びます。

## はじめに

システムを安定して運用するためには、適切なモニタリングとアラート設計が不可欠です。このセクションでは、実践的なモニタリング・アラート設計の考え方を学びます。

## 学習内容

### 1. アラート設計

効果的なアラート設計の基礎を学びます。

- [ユーザー視点の分類](alert-classification.md) - アラートを誰のために設計するか
- [アラートレベルの決め方](alert-levels.md) - 重要度の適切な設定方法
- [よくある失敗例](common-mistakes.md) - 失敗パターンと対策

### 2. 多層モニタリング

システムを多層的に監視する重要性と実践方法を学びます。

- [多層モニタリングとは](multi-layer-monitoring.md) - 多層監視の必要性
- [実践例](multi-layer-examples.md) - 具体的な実装例

### 3. ダッシュボード設計

効果的なダッシュボードの設計方法を学びます。

- [ダッシュボード設計のベストプラクティス](dashboard-design.md) - 目的・メトリクス選定・レイアウト

### 4. クラウド環境での考え方

クラウド環境特有のモニタリング設計を学びます。

- [クラウド特有の設計](cloud-specific.md) - クラウドならではの考慮点
- [リソース監視](resource-monitoring.md) - クラウドリソースの効果的な監視

## 参考資料

### 書籍

- [入門 監視 ―モダンなモニタリングのためのデザインパターン](https://amzn.asia/d/7zJNoub) - Mike Julian 著。モニタリングの基本概念から実践的なデザインパターンまでを解説
- [システム運用アンチパターン](https://amzn.asia/d/dgvUPvS) - Jeffery D. Smith 著。DevOpsの観点から組織・自動化・コミュニケーションの問題を解決する方法を解説

### オンラインリソース

- [Monitoring 101: Collecting the Right Data（Datadog）](https://www.datadoghq.com/blog/monitoring-101-collecting-data/)（[日本語訳](https://www.datadoghq.com/ja/blog/%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%AA%E3%83%B3%E3%82%B0%E3%81%AE%E3%83%99%E3%82%B9%E3%83%88%E3%83%97%E3%83%A9%E3%82%AF%E3%83%86%E3%82%A3%E3%82%B9/)） - Datadog CTOによるモニタリング戦略の解説。ワークメトリクスとリソースメトリクスの分類、アクション可能なアラートの重要性について
- [Monitoring Distributed Systems（Google SRE Book）](https://sre.google/sre-book/monitoring-distributed-systems/) - Googleのモニタリング原則。アラート疲れの問題と「人を呼び出すことは高コスト」という考え方を解説
- [Alerting Principles（PagerDuty）](https://response.pagerduty.com/oncall/alerting_principles/) - 「アラートは人間のアクションを必要とするもの」という原則を解説
- [AWS Observability Best Practices](https://aws-observability.github.io/observability-best-practices/) - AWSにおけるオブザーバビリティのベストプラクティス集。CloudWatch、X-Ray、Syntheticsなどの活用方法
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/) - Grafana公式のダッシュボード設計ガイド。ダッシュボードスプロール（乱立）の防止、テンプレート変数の活用について
- [Crafting an Actionable Observability Dashboard Experience（Chronosphere）](https://chronosphere.io/learn/observability-dashboard-experience/) - ダッシュボードを「コックピット」として設計する考え方。インシデント対応での活用、視覚的階層について
