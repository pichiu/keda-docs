# KEDA 繁體中文文件

[簡介](./_index.md)

---

# 入門指南

- [部署 KEDA](./deploy.md)
- [版本遷移](./migration.md)

---

# 核心概念

- [概念總覽](./concepts/_index.md)
  - [擴縮 Deployments](./concepts/scaling-deployments.md)
  - [擴縮 Jobs](./concepts/scaling-jobs.md)
  - [驗證機制](./concepts/authentication.md)
  - [外部擴縮器](./concepts/external-scalers.md)
  - [Admission Webhooks](./concepts/admission-webhooks.md)
  - [疑難排解](./concepts/troubleshooting.md)

---

# 驗證提供者

- [驗證提供者總覽](./authentication-providers/_index.md)
  - [Kubernetes Secret](./authentication-providers/secret.md)
  - [環境變數](./authentication-providers/environment-variable.md)
  - [ConfigMap](./authentication-providers/configmap.md)
  - [AWS Pod Identity](./authentication-providers/aws.md)
  - [AWS EKS Pod Identity](./authentication-providers/aws-eks.md)
  - [AWS Secret Manager](./authentication-providers/aws-secret-manager.md)
  - [Azure AD Workload Identity](./authentication-providers/azure-ad-workload-identity.md)
  - [Azure Key Vault](./authentication-providers/azure-key-vault.md)
  - [GCP Workload Identity](./authentication-providers/gcp-workload-identity.md)
  - [GCP Secret Manager](./authentication-providers/gcp-secret-manager.md)
  - [HashiCorp Vault](./authentication-providers/hashicorp-vault.md)
  - [繫結服務帳戶令牌](./authentication-providers/bound-service-account-token.md)

---

# 擴縮器

- [擴縮器總覽](./scalers/_index.md)
  - [ActiveMQ](./scalers/activemq.md)
  - [Apache Kafka](./scalers/apache-kafka.md)
  - [AWS CloudWatch](./scalers/aws-cloudwatch.md)
  - [AWS DynamoDB](./scalers/aws-dynamodb.md)
  - [AWS Kinesis](./scalers/aws-kinesis.md)
  - [AWS SQS](./scalers/aws-sqs.md)
  - [Azure Event Hub](./scalers/azure-event-hub.md)
  - [Azure Log Analytics](./scalers/azure-log-analytics.md)
  - [Azure Monitor](./scalers/azure-monitor.md)
  - [Azure Pipelines](./scalers/azure-pipelines.md)
  - [Azure Service Bus](./scalers/azure-service-bus.md)
  - [Azure Storage Queue](./scalers/azure-storage-queue.md)
  - [Cassandra](./scalers/cassandra.md)
  - [CPU](./scalers/cpu.md)
  - [Cron](./scalers/cron.md)
  - [Datadog](./scalers/datadog.md)
  - [Elasticsearch](./scalers/elasticsearch.md)
  - [External](./scalers/external.md)
  - [GCP Pub/Sub](./scalers/gcp-pub-sub.md)
  - [GitHub Runner](./scalers/github-runner.md)
  - [Graphite](./scalers/graphite.md)
  - [IBM MQ](./scalers/ibm-mq.md)
  - [InfluxDB](./scalers/influxdb.md)
  - [Kubernetes Workload](./scalers/kubernetes-workload.md)
  - [Loki](./scalers/loki.md)
  - [Memory](./scalers/memory.md)
  - [Metrics API](./scalers/metrics-api.md)
  - [MongoDB](./scalers/mongodb.md)
  - [MSSQL](./scalers/mssql.md)
  - [MySQL](./scalers/mysql.md)
  - [NATS JetStream](./scalers/nats-jetstream.md)
  - [New Relic](./scalers/new-relic.md)
  - [PostgreSQL](./scalers/postgresql.md)
  - [Prometheus](./scalers/prometheus.md)
  - [Pulsar](./scalers/pulsar.md)
  - [RabbitMQ](./scalers/rabbitmq-queue.md)
  - [Redis Lists](./scalers/redis-lists.md)
  - [Redis Streams](./scalers/redis-streams.md)
  - [Selenium Grid](./scalers/selenium-grid-scaler.md)
  - [Solace PubSub+](./scalers/solace-pub-sub.md)

---

# 維運指南

- [維運總覽](./operate/_index.md)
  - [叢集需求](./operate/cluster.md)
  - [安全性設定](./operate/security.md)
  - [Metrics Server](./operate/metrics-server.md)

---

# 參考資料

- [參考資料總覽](./reference/_index.md)
  - [ScaledObject 規格](./reference/scaledobject-spec.md)
  - [ScaledJob 規格](./reference/scaledjob-spec.md)
  - [事件參考](./reference/events.md)
  - [常見問題](./reference/faq.md)
  - [術語表](./reference/glossary.md)
