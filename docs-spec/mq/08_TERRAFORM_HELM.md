# 08. IaC 구조 설계 (Terraform + Helm)

```
Document ID    : MQ-SPEC-001-08
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. IaC 전체 구조

### 1.1 저장소 구조

```
platform-infrastructure/
├── terraform/
│   ├── modules/
│   │   ├── rabbitmq-cluster/           # RabbitMQ 클러스터 (Helm)
│   │   ├── rabbitmq-vhost/             # vhost + User + Permission
│   │   ├── rabbitmq-exchange-queue/    # Exchange + Queue + Binding
│   │   ├── rabbitmq-policy/            # Policy 관리
│   │   └── vault-secrets/              # Vault Secret 관리
│   │
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── backend.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── projects/
│   │   │       ├── eventflow.tf
│   │   │       └── _common.tf
│   │   │
│   │   ├── staging/
│   │   │   └── (동일 구조)
│   │   │
│   │   └── prod/
│   │       └── (동일 구조)
│   │
│   └── shared/
│       ├── providers.tf
│       ├── versions.tf
│       └── data.tf
│
├── helm/
│   ├── charts/
│   │   └── platform-rabbitmq/          # Custom RabbitMQ Chart
│   │       ├── Chart.yaml
│   │       ├── values.yaml
│   │       ├── values-dev.yaml
│   │       ├── values-staging.yaml
│   │       ├── values-prod.yaml
│   │       └── templates/
│   │           ├── _helpers.tpl
│   │           ├── statefulset.yaml
│   │           ├── service.yaml
│   │           ├── configmap.yaml
│   │           ├── secret.yaml
│   │           ├── pdb.yaml
│   │           ├── networkpolicy.yaml
│   │           └── servicemonitor.yaml
│   │
│   └── releases/
│       ├── dev/
│       │   └── rabbitmq.yaml           # Helmfile 또는 ArgoCD App
│       ├── staging/
│       │   └── rabbitmq.yaml
│       └── prod/
│           └── rabbitmq.yaml
│
└── docs/
    ├── terraform-usage.md
    └── helm-usage.md
```

### 1.2 배포 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        IaC Deployment Flow                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Phase 1: Infrastructure (Kubernetes + Vault)                    │    │
│  │                                                                  │    │
│  │  [Terraform]                                                     │    │
│  │  - Kubernetes Cluster                                            │    │
│  │  - Vault Server                                                  │    │
│  │  - cert-manager                                                  │    │
│  │  - Prometheus Stack                                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Phase 2: RabbitMQ Cluster (Helm)                                │    │
│  │                                                                  │    │
│  │  [Helm / ArgoCD]                                                 │    │
│  │  - StatefulSet                                                   │    │
│  │  - Services                                                      │    │
│  │  - ConfigMap                                                     │    │
│  │  - NetworkPolicy                                                 │    │
│  │  - ServiceMonitor                                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Phase 3: RabbitMQ Resources (Terraform)                         │    │
│  │                                                                  │    │
│  │  [Terraform - rabbitmq provider]                                 │    │
│  │  - vhosts                                                        │    │
│  │  - Users                                                         │    │
│  │  - Permissions                                                   │    │
│  │  - Policies                                                      │    │
│  │  - Exchanges                                                     │    │
│  │  - Queues                                                        │    │
│  │  - Bindings                                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Phase 4: Application Deployment                                 │    │
│  │                                                                  │    │
│  │  [ArgoCD / Helm]                                                 │    │
│  │  - eventflow Application                                         │    │
│  │  - eventflow System                                              │    │
│  │  - 기타 프로젝트                                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Terraform 모듈 상세

### 2.1 rabbitmq-cluster 모듈 (Helm 배포)

```hcl
# modules/rabbitmq-cluster/main.tf

terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

resource "kubernetes_namespace" "rabbitmq" {
  metadata {
    name = var.namespace

    labels = {
      name                         = var.namespace
      "app.kubernetes.io/name"     = "rabbitmq"
      "app.kubernetes.io/instance" = var.release_name
    }
  }
}

resource "helm_release" "rabbitmq" {
  name       = var.release_name
  namespace  = kubernetes_namespace.rabbitmq.metadata[0].name
  repository = var.chart_repository
  chart      = var.chart_name
  version    = var.chart_version

  values = [
    templatefile("${path.module}/values.yaml.tpl", {
      replicas           = var.replicas
      image_tag          = var.image_tag
      storage_class      = var.storage_class
      storage_size       = var.storage_size
      memory_limit       = var.memory_limit
      cpu_limit          = var.cpu_limit
      memory_watermark   = var.memory_watermark
      disk_free_limit    = var.disk_free_limit
      plugins            = join(",", var.plugins)
      tls_secret_name    = var.tls_secret_name
      admin_secret_name  = var.admin_secret_name
    })
  ]

  # 환경별 values 파일 오버라이드
  dynamic "set" {
    for_each = var.additional_values
    content {
      name  = set.key
      value = set.value
    }
  }

  depends_on = [kubernetes_namespace.rabbitmq]
}

# modules/rabbitmq-cluster/variables.tf
variable "namespace" {
  description = "Kubernetes namespace for RabbitMQ"
  type        = string
  default     = "platform-mq"
}

variable "release_name" {
  description = "Helm release name"
  type        = string
  default     = "rabbitmq"
}

variable "chart_repository" {
  description = "Helm chart repository"
  type        = string
  default     = "oci://registry-1.docker.io/bitnamicharts"
}

variable "chart_name" {
  description = "Helm chart name"
  type        = string
  default     = "rabbitmq"
}

variable "chart_version" {
  description = "Helm chart version"
  type        = string
  default     = "12.15.0"
}

variable "replicas" {
  description = "Number of RabbitMQ replicas"
  type        = number
  default     = 3
}

variable "image_tag" {
  description = "RabbitMQ image tag"
  type        = string
  default     = "3.13.1-debian-12-r0"
}

variable "storage_class" {
  description = "Storage class for PVC"
  type        = string
  default     = "ssd-retain"
}

variable "storage_size" {
  description = "PVC size"
  type        = string
  default     = "500Gi"
}

variable "memory_limit" {
  description = "Memory limit for pods"
  type        = string
  default     = "8Gi"
}

variable "cpu_limit" {
  description = "CPU limit for pods"
  type        = string
  default     = "4"
}

variable "memory_watermark" {
  description = "Memory high watermark (relative)"
  type        = number
  default     = 0.6
}

variable "disk_free_limit" {
  description = "Disk free limit"
  type        = string
  default     = "5GB"
}

variable "plugins" {
  description = "List of RabbitMQ plugins to enable"
  type        = list(string)
  default = [
    "rabbitmq_management",
    "rabbitmq_prometheus",
    "rabbitmq_shovel",
    "rabbitmq_shovel_management",
    "rabbitmq_peer_discovery_k8s"
  ]
}

variable "tls_secret_name" {
  description = "TLS certificate secret name"
  type        = string
}

variable "admin_secret_name" {
  description = "Admin credentials secret name"
  type        = string
}

variable "additional_values" {
  description = "Additional Helm values to set"
  type        = map(string)
  default     = {}
}

# modules/rabbitmq-cluster/outputs.tf
output "namespace" {
  value = kubernetes_namespace.rabbitmq.metadata[0].name
}

output "service_name" {
  value = "${var.release_name}-rabbitmq"
}

output "management_url" {
  value = "https://${var.release_name}-rabbitmq.${var.namespace}.svc.cluster.local:15672"
}

output "amqp_url" {
  value = "amqps://${var.release_name}-rabbitmq.${var.namespace}.svc.cluster.local:5671"
}
```

### 2.2 rabbitmq-vhost 모듈

```hcl
# modules/rabbitmq-vhost/main.tf

terraform {
  required_providers {
    rabbitmq = {
      source  = "cyrilgdn/rabbitmq"
      version = "~> 1.8"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.25"
    }
  }
}

# vhost 생성
resource "rabbitmq_vhost" "project" {
  name = var.vhost_name
}

# vhost Limits 설정
resource "rabbitmq_vhost_limits" "limits" {
  vhost = rabbitmq_vhost.project.name

  max_connections = var.max_connections
  max_queues      = var.max_queues
}

# Vault에서 비밀번호 생성 및 저장
resource "random_password" "producer" {
  length  = 32
  special = false
}

resource "random_password" "consumer" {
  length  = 32
  special = false
}

resource "random_password" "admin" {
  length  = 32
  special = false
}

# Producer User
resource "rabbitmq_user" "producer" {
  name     = "${var.project_id}-producer-${var.service_name}"
  password = random_password.producer.result
  tags     = ["producer"]
}

resource "rabbitmq_permissions" "producer" {
  user  = rabbitmq_user.producer.name
  vhost = rabbitmq_vhost.project.name

  permissions {
    configure = ""
    write     = "^${var.project_id}\\.exchange\\..*"
    read      = ""
  }
}

# Consumer User
resource "rabbitmq_user" "consumer" {
  name     = "${var.project_id}-consumer-${var.service_name}"
  password = random_password.consumer.result
  tags     = ["consumer"]
}

resource "rabbitmq_permissions" "consumer" {
  user  = rabbitmq_user.consumer.name
  vhost = rabbitmq_vhost.project.name

  permissions {
    configure = ""
    write     = ""
    read      = "^${var.project_id}\\.queue\\..*"
  }
}

# Admin User
resource "rabbitmq_user" "admin" {
  name     = "${var.project_id}-admin"
  password = random_password.admin.result
  tags     = ["administrator", "management"]
}

resource "rabbitmq_permissions" "admin" {
  user  = rabbitmq_user.admin.name
  vhost = rabbitmq_vhost.project.name

  permissions {
    configure = "^${var.project_id}\\..*"
    write     = "^${var.project_id}\\..*"
    read      = "^${var.project_id}\\..*"
  }
}

# Vault에 Credentials 저장
resource "vault_kv_secret_v2" "producer" {
  mount = var.vault_mount
  name  = "projects/${var.project_name}/mq/producer/${var.service_name}"

  data_json = jsonencode({
    username   = rabbitmq_user.producer.name
    password   = random_password.producer.result
    vhost      = rabbitmq_vhost.project.name
    host       = var.rabbitmq_host
    port       = var.rabbitmq_port
    created_at = timestamp()
  })

  lifecycle {
    ignore_changes = [data_json]
  }
}

resource "vault_kv_secret_v2" "consumer" {
  mount = var.vault_mount
  name  = "projects/${var.project_name}/mq/consumer/${var.service_name}"

  data_json = jsonencode({
    username   = rabbitmq_user.consumer.name
    password   = random_password.consumer.result
    vhost      = rabbitmq_vhost.project.name
    host       = var.rabbitmq_host
    port       = var.rabbitmq_port
    created_at = timestamp()
  })

  lifecycle {
    ignore_changes = [data_json]
  }
}

resource "vault_kv_secret_v2" "admin" {
  mount = var.vault_mount
  name  = "projects/${var.project_name}/mq/admin"

  data_json = jsonencode({
    username   = rabbitmq_user.admin.name
    password   = random_password.admin.result
    vhost      = rabbitmq_vhost.project.name
    host       = var.rabbitmq_host
    port       = var.rabbitmq_port
    created_at = timestamp()
  })

  lifecycle {
    ignore_changes = [data_json]
  }
}

# modules/rabbitmq-vhost/variables.tf
variable "vhost_name" {
  description = "vhost name (e.g., /eventflow)"
  type        = string
}

variable "project_id" {
  description = "Project identifier (e.g., ef)"
  type        = string
}

variable "project_name" {
  description = "Project name (e.g., eventflow)"
  type        = string
}

variable "service_name" {
  description = "Service name (e.g., application, system)"
  type        = string
  default     = "application"
}

variable "max_connections" {
  description = "Maximum connections for vhost"
  type        = number
  default     = 500
}

variable "max_queues" {
  description = "Maximum queues for vhost"
  type        = number
  default     = 200
}

variable "vault_mount" {
  description = "Vault KV mount path"
  type        = string
  default     = "secret"
}

variable "rabbitmq_host" {
  description = "RabbitMQ host"
  type        = string
}

variable "rabbitmq_port" {
  description = "RabbitMQ port"
  type        = number
  default     = 5671
}

# modules/rabbitmq-vhost/outputs.tf
output "vhost_name" {
  value = rabbitmq_vhost.project.name
}

output "producer_user" {
  value = rabbitmq_user.producer.name
}

output "consumer_user" {
  value = rabbitmq_user.consumer.name
}

output "admin_user" {
  value = rabbitmq_user.admin.name
}

output "vault_producer_path" {
  value = vault_kv_secret_v2.producer.name
}

output "vault_consumer_path" {
  value = vault_kv_secret_v2.consumer.name
}
```

### 2.3 rabbitmq-exchange-queue 모듈

```hcl
# modules/rabbitmq-exchange-queue/main.tf

terraform {
  required_providers {
    rabbitmq = {
      source  = "cyrilgdn/rabbitmq"
      version = "~> 1.8"
    }
  }
}

# Exchanges
resource "rabbitmq_exchange" "exchanges" {
  for_each = var.exchanges

  name  = each.key
  vhost = var.vhost

  settings {
    type        = each.value.type
    durable     = lookup(each.value, "durable", true)
    auto_delete = lookup(each.value, "auto_delete", false)
  }
}

# Queues
resource "rabbitmq_queue" "queues" {
  for_each = var.queues

  name  = each.key
  vhost = var.vhost

  settings {
    durable = lookup(each.value, "durable", true)

    arguments = merge(
      {
        "x-queue-type" = lookup(each.value, "type", "quorum")
      },
      lookup(each.value, "dlx", null) != null ? {
        "x-dead-letter-exchange" = each.value.dlx
      } : {},
      lookup(each.value, "dlx_routing_key", null) != null ? {
        "x-dead-letter-routing-key" = each.value.dlx_routing_key
      } : {},
      lookup(each.value, "max_length", null) != null ? {
        "x-max-length" = each.value.max_length
      } : {},
      lookup(each.value, "max_length_bytes", null) != null ? {
        "x-max-length-bytes" = each.value.max_length_bytes
      } : {},
      lookup(each.value, "message_ttl", null) != null ? {
        "x-message-ttl" = each.value.message_ttl
      } : {},
      lookup(each.value, "overflow", null) != null ? {
        "x-overflow" = each.value.overflow
      } : {}
    )
  }

  depends_on = [rabbitmq_exchange.exchanges]
}

# Bindings
resource "rabbitmq_binding" "bindings" {
  for_each = var.bindings

  source           = each.value.exchange
  vhost            = var.vhost
  destination      = each.key
  destination_type = "queue"
  routing_key      = each.value.routing_key

  depends_on = [
    rabbitmq_exchange.exchanges,
    rabbitmq_queue.queues
  ]
}

# modules/rabbitmq-exchange-queue/variables.tf
variable "vhost" {
  description = "vhost name"
  type        = string
}

variable "exchanges" {
  description = "Map of exchanges to create"
  type = map(object({
    type        = string
    durable     = optional(bool, true)
    auto_delete = optional(bool, false)
  }))
  default = {}
}

variable "queues" {
  description = "Map of queues to create"
  type = map(object({
    type             = optional(string, "quorum")
    durable          = optional(bool, true)
    dlx              = optional(string)
    dlx_routing_key  = optional(string)
    max_length       = optional(number)
    max_length_bytes = optional(number)
    message_ttl      = optional(number)
    overflow         = optional(string, "reject-publish")
  }))
  default = {}
}

variable "bindings" {
  description = "Map of bindings (queue_name -> {exchange, routing_key})"
  type = map(object({
    exchange    = string
    routing_key = string
  }))
  default = {}
}

# modules/rabbitmq-exchange-queue/outputs.tf
output "exchanges" {
  value = [for e in rabbitmq_exchange.exchanges : e.name]
}

output "queues" {
  value = [for q in rabbitmq_queue.queues : q.name]
}
```

---

## 3. 환경별 Terraform 설정

### 3.1 Production 환경

```hcl
# environments/prod/main.tf

terraform {
  required_version = ">= 1.7.0"

  backend "s3" {
    bucket         = "platform-terraform-state"
    key            = "prod/rabbitmq/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Provider 설정
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "prod-cluster"
}

provider "helm" {
  kubernetes {
    config_path    = "~/.kube/config"
    config_context = "prod-cluster"
  }
}

provider "rabbitmq" {
  endpoint = "https://rabbitmq.platform-mq.svc.cluster.local:15672"
  username = data.vault_kv_secret_v2.admin.data["username"]
  password = data.vault_kv_secret_v2.admin.data["password"]
}

provider "vault" {
  address = "https://vault.internal:8200"
}

# Vault에서 Admin 자격증명 조회
data "vault_kv_secret_v2" "admin" {
  mount = "secret"
  name  = "platform/mq/admin"
}

# RabbitMQ Cluster 배포
module "rabbitmq_cluster" {
  source = "../../modules/rabbitmq-cluster"

  namespace     = "platform-mq"
  release_name  = "rabbitmq"
  replicas      = 3
  storage_class = "ssd-retain"
  storage_size  = "500Gi"
  memory_limit  = "16Gi"
  cpu_limit     = "8"

  memory_watermark = 0.6
  disk_free_limit  = "5GB"

  tls_secret_name   = "rabbitmq-tls"
  admin_secret_name = "rabbitmq-admin"

  additional_values = {
    "metrics.enabled"              = "true"
    "metrics.serviceMonitor.enabled" = "true"
  }
}

# environments/prod/projects/eventflow.tf

# eventflow vhost 및 User
module "eventflow_vhost" {
  source = "../../../modules/rabbitmq-vhost"

  vhost_name      = "/eventflow"
  project_id      = "ef"
  project_name    = "eventflow"
  service_name    = "application"
  max_connections = 500
  max_queues      = 200

  vault_mount   = "secret"
  rabbitmq_host = module.rabbitmq_cluster.service_name
  rabbitmq_port = 5671

  depends_on = [module.rabbitmq_cluster]
}

# Consumer 계정 추가 (System 서비스용)
module "eventflow_consumer_system" {
  source = "../../../modules/rabbitmq-vhost"

  vhost_name   = "/eventflow"
  project_id   = "ef"
  project_name = "eventflow"
  service_name = "system"

  # vhost는 이미 생성됨, User만 추가
  create_vhost = false

  vault_mount   = "secret"
  rabbitmq_host = module.rabbitmq_cluster.service_name
  rabbitmq_port = 5671

  depends_on = [module.eventflow_vhost]
}

# eventflow Exchange/Queue/Binding
module "eventflow_resources" {
  source = "../../../modules/rabbitmq-exchange-queue"

  vhost = module.eventflow_vhost.vhost_name

  exchanges = {
    "ef.exchange.order" = {
      type    = "topic"
      durable = true
    }
    "ef.exchange.order.dlx" = {
      type    = "fanout"
      durable = true
    }
    "ef.exchange.user" = {
      type    = "topic"
      durable = true
    }
    "ef.exchange.user.dlx" = {
      type    = "fanout"
      durable = true
    }
    "ef.exchange.notification" = {
      type    = "topic"
      durable = true
    }
    "ef.exchange.notification.dlx" = {
      type    = "fanout"
      durable = true
    }
  }

  queues = {
    # Order domain
    "ef.queue.order.created" = {
      type        = "quorum"
      dlx         = "ef.exchange.order.dlx"
      max_length  = 100000
      message_ttl = 86400000
    }
    "ef.queue.order.updated" = {
      type        = "quorum"
      dlx         = "ef.exchange.order.dlx"
      max_length  = 100000
      message_ttl = 86400000
    }
    "ef.queue.order.dlq" = {
      type        = "quorum"
      max_length  = 50000
      message_ttl = 604800000
    }

    # User domain
    "ef.queue.user.registered" = {
      type        = "quorum"
      dlx         = "ef.exchange.user.dlx"
      max_length  = 50000
      message_ttl = 86400000
    }
    "ef.queue.user.updated" = {
      type        = "quorum"
      dlx         = "ef.exchange.user.dlx"
      max_length  = 50000
      message_ttl = 86400000
    }
    "ef.queue.user.dlq" = {
      type        = "quorum"
      max_length  = 10000
      message_ttl = 604800000
    }

    # Notification domain
    "ef.queue.notification.email" = {
      type        = "quorum"
      dlx         = "ef.exchange.notification.dlx"
      max_length  = 200000
      message_ttl = 86400000
    }
    "ef.queue.notification.sms" = {
      type        = "quorum"
      dlx         = "ef.exchange.notification.dlx"
      max_length  = 100000
      message_ttl = 86400000
    }
    "ef.queue.notification.push" = {
      type        = "quorum"
      dlx         = "ef.exchange.notification.dlx"
      max_length  = 200000
      message_ttl = 86400000
    }
    "ef.queue.notification.dlq" = {
      type        = "quorum"
      max_length  = 20000
      message_ttl = 604800000
    }
  }

  bindings = {
    "ef.queue.order.created" = {
      exchange    = "ef.exchange.order"
      routing_key = "order.created"
    }
    "ef.queue.order.updated" = {
      exchange    = "ef.exchange.order"
      routing_key = "order.updated"
    }
    "ef.queue.order.dlq" = {
      exchange    = "ef.exchange.order.dlx"
      routing_key = "#"
    }
    "ef.queue.user.registered" = {
      exchange    = "ef.exchange.user"
      routing_key = "user.registered"
    }
    "ef.queue.user.updated" = {
      exchange    = "ef.exchange.user"
      routing_key = "user.updated"
    }
    "ef.queue.user.dlq" = {
      exchange    = "ef.exchange.user.dlx"
      routing_key = "#"
    }
    "ef.queue.notification.email" = {
      exchange    = "ef.exchange.notification"
      routing_key = "notification.email"
    }
    "ef.queue.notification.sms" = {
      exchange    = "ef.exchange.notification"
      routing_key = "notification.sms"
    }
    "ef.queue.notification.push" = {
      exchange    = "ef.exchange.notification"
      routing_key = "notification.push"
    }
    "ef.queue.notification.dlq" = {
      exchange    = "ef.exchange.notification.dlx"
      routing_key = "#"
    }
  }

  depends_on = [module.eventflow_vhost]
}
```

---

## 4. Helm Chart 구조

### 4.1 Custom values.yaml

```yaml
# helm/charts/platform-rabbitmq/values.yaml

global:
  imageRegistry: "harbor.platform.internal"
  imagePullSecrets:
    - name: harbor-secret

image:
  repository: rabbitmq
  tag: "3.13.1-management-alpine"
  pullPolicy: IfNotPresent

replicaCount: 3

auth:
  existingPasswordSecret: rabbitmq-admin-secret
  existingPasswordSecretKey: password
  existingErlangCookieSecret: rabbitmq-erlang-cookie
  existingErlangCookieSecretKey: cookie

clustering:
  enabled: true
  rebalance: false
  forceBoot: false

persistence:
  enabled: true
  storageClass: "ssd-retain"
  size: 500Gi
  accessModes:
    - ReadWriteOnce

resources:
  limits:
    cpu: "8"
    memory: "16Gi"
  requests:
    cpu: "4"
    memory: "8Gi"

# RabbitMQ Configuration
configuration: |
  ## Cluster formation
  cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
  cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
  cluster_formation.k8s.address_type = hostname
  cluster_formation.node_cleanup.interval = 30
  cluster_formation.node_cleanup.only_log_warning = true
  cluster_partition_handling = pause_minority
  queue_master_locator = min-masters

  ## Default queue type
  default_queue_type = quorum

  ## Memory
  vm_memory_high_watermark.relative = 0.6
  vm_memory_high_watermark_paging_ratio = 0.7

  ## Disk
  disk_free_limit.absolute = 5GB

  ## Connection
  heartbeat = 60
  channel_max = 128

  ## Logging
  log.file.level = info
  log.console = true
  log.console.level = info
  log.console.formatter = json

extraPlugins:
  - rabbitmq_management
  - rabbitmq_prometheus
  - rabbitmq_shovel
  - rabbitmq_shovel_management
  - rabbitmq_peer_discovery_k8s

# TLS Configuration
tls:
  enabled: true
  existingSecret: rabbitmq-tls-secret
  failIfNoPeerCert: true
  sslOptionsVerify: verify_peer

# Services
service:
  type: ClusterIP
  ports:
    amqp: 5672
    amqpTls: 5671
    epmd: 4369
    dist: 25672
    manager: 15672
    prometheus: 15692

# Pod Disruption Budget
pdb:
  create: true
  minAvailable: 2

# Pod Affinity
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: rabbitmq
        topologyKey: kubernetes.io/hostname
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-role
              operator: In
              values:
                - mq

# Network Policy
networkPolicy:
  enabled: true
  allowedNamespaces:
    - eventflow
    - platform-mq-monitoring
  additionalRules: []

# ServiceMonitor for Prometheus
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: platform-mq-monitoring
    interval: 15s
    scrapeTimeout: 10s
```

### 4.2 환경별 values 오버라이드

```yaml
# helm/charts/platform-rabbitmq/values-prod.yaml

replicaCount: 3

resources:
  limits:
    cpu: "8"
    memory: "16Gi"
  requests:
    cpu: "4"
    memory: "8Gi"

persistence:
  size: 500Gi
  storageClass: "ssd-retain"

# Production 전용 설정
extraConfiguration: |
  ## Production tuning
  vm_memory_high_watermark.relative = 0.6
  tcp_listen_options.backlog = 256
  tcp_listen_options.nodelay = true

pdb:
  minAvailable: 2

metrics:
  serviceMonitor:
    enabled: true
```

```yaml
# helm/charts/platform-rabbitmq/values-dev.yaml

replicaCount: 1

resources:
  limits:
    cpu: "2"
    memory: "4Gi"
  requests:
    cpu: "1"
    memory: "2Gi"

persistence:
  size: 50Gi
  storageClass: "standard"

tls:
  enabled: false  # Dev는 TLS 없이

pdb:
  create: false

metrics:
  serviceMonitor:
    enabled: false
```

---

## 5. CI/CD 파이프라인

### 5.1 GitLab CI/CD

```yaml
# .gitlab-ci.yml

stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "terraform/environments"

.terraform_template:
  image: hashicorp/terraform:1.7
  before_script:
    - cd ${TF_ROOT}/${ENV}
    - terraform init -backend=true

validate:
  extends: .terraform_template
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check -recursive
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

plan:dev:
  extends: .terraform_template
  stage: plan
  variables:
    ENV: dev
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/dev/plan.tfplan
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

plan:staging:
  extends: .terraform_template
  stage: plan
  variables:
    ENV: staging
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/staging/plan.tfplan
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

plan:prod:
  extends: .terraform_template
  stage: plan
  variables:
    ENV: prod
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/prod/plan.tfplan
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/

apply:dev:
  extends: .terraform_template
  stage: apply
  variables:
    ENV: dev
  script:
    - terraform apply -auto-approve plan.tfplan
  dependencies:
    - plan:dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
  when: manual

apply:staging:
  extends: .terraform_template
  stage: apply
  variables:
    ENV: staging
  script:
    - terraform apply -auto-approve plan.tfplan
  dependencies:
    - plan:staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual

apply:prod:
  extends: .terraform_template
  stage: apply
  variables:
    ENV: prod
  script:
    - terraform apply -auto-approve plan.tfplan
  dependencies:
    - plan:prod
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
  when: manual
  environment:
    name: production
```

---

END OF DOCUMENT
