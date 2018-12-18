# CronJob

## 概要

cronのように、設定されたスケジュールに従って `Job` を生成する機能。

CronJobはスケジュールされた時間にジョブを作成する責任を負い、生成されたジョブは処理の正常な終了を保証する。

- [CronJob - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

## ユースケース

- 定期的な処理の実行（バッチ処理など）
