# 🔒 Storage Lock

**一个存储后端无关的分布式锁框架** — 基于 Go 语言实现，通过抽象 Storage 接口，支持任何可分布式访问的存储介质来实现分布式锁。

---

## 📖 项目简介

Storage Lock 抽象了一套分布式锁的模型定义和算法，将锁逻辑与存储后端完全解耦。只要存储介质可以被分布式访问，就可以作为锁的存储后端——无论是关系型数据库、NoSQL、对象存储、分布式文件系统还是内存存储。

核心思路：

```
┌─────────────────────────────────────────────┐
│              应用层 (Application)             │
│   HTTP API / gRPC API / CLI / UI / SDK      │
├─────────────────────────────────────────────┤
│           锁核心 (go-storage-lock)           │
│   可重入锁 / 乐观锁 / 看门狗续期 / 事件通知    │
├─────────────────────────────────────────────┤
│          存储抽象 (go-storage)               │
│   Storage 接口 / LockInformation 模型        │
├─────────────────────────────────────────────┤
│              存储实现 (Storage Impl)          │
│  MySQL / Redis / MongoDB / S3 / ... 50+ 种  │
└─────────────────────────────────────────────┘
```

---

## ✨ 核心特性

- 🔄 **存储后端无关** — 同一套锁逻辑可运行在 50+ 种不同的存储后端上
- 🔁 **可重入锁** — 支持同一线程/协程多次获取同一把锁，按加锁次数释放
- 🔒 **乐观锁（CAS）** — 基于版本号的乐观锁机制，避免 ABA 问题
- 🐕 **看门狗（Watch Dog）** — 自动续期机制，防止业务未完成时锁过期
- 📡 **事件通知** — 锁的获取、释放等事件可通过事件监听器处理
- 🏭 **工厂模式** — 通过 Factory 便捷创建锁实例
- 📊 **监控集成** — 内置 Prometheus 指标支持

---

## 🧠 理论基础与正确性保障

分布式锁的正确性建立在两个**必要条件**之上，本框架在代码层面强制校验：

| 必要条件 | 含义 | 不满足的后果 |
|---------|------|------------|
| **CAS 原子性** | CreateWithVersion/UpdateWithVersion/DeleteWithVersion 必须是原子的"检查+操作" | 互斥性被破坏：多个节点可能同时持有同一把锁 |
| **可靠时间源** | GetTime 返回单调递增、节点间一致的时间 | 时钟回拨会导致锁提前释放；时钟漂移破坏互斥性 |

- `Storage` 接口新增 `Capabilities()` 方法，实现方声明自身支持的能力
- `NewStorageLockWithOptions` 在创建锁时校验存储是否满足必要条件，不满足直接报错（而非静默产生不安全行为）
- 不满足条件的存储（如文件系统多进程场景）可通过 `SkipCapabilityCheck` 显式跳过，并自行承担风险

### 测试框架

- [go-storage-test-helper](https://github.com/storage-lock/go-storage-test-helper) — Storage 层 CRUD + CAS 正确性测试
- [go-storage-lock-test](https://github.com/storage-lock/go-storage-lock-test) — **分布式锁语义测试框架**，验证互斥性、可重入、租约过期、看门狗续租、并发竞争、ABA 防护等核心语义。存储实现方只需实现 `StorageLockTestFactory` 接口即可接入完整测试套件

---

## 🚀 快速开始

### 安装

```bash
go get -u github.com/storage-lock/go-storage-lock
```

### 使用示例（以 MySQL 为例）

```go
package main

import (
    "context"
    "log"

    mysql_storage "github.com/storage-lock/go-mysql-storage"
    mysql_locks "github.com/storage-lock/go-mysql-locks"
    storage_lock "github.com/storage-lock/go-storage-lock"
)

func main() {
    // 1. 创建 Storage
    storage, err := mysql_storage.NewMysqlStorage(mysql_storage.NewMysqlStorageOptions{
        Host:     "127.0.0.1",
        Port:     3306,
        Username: "root",
        Password: "password",
        Database: "test",
    })
    if err != nil {
        log.Fatal(err)
    }

    // 2. 创建 Lock
    lock, err := mysql_locks.NewMysqlLocks(mysql_locks.NewMysqlLocksOptions{
        Storage: storage,
    })
    if err != nil {
        log.Fatal(err)
    }

    // 3. 加锁
    err = lock.Lock(context.Background(), "my-lock-key")
    if err != nil {
        log.Fatal(err)
    }
    defer lock.Unlock(context.Background(), "my-lock-key")

    // 4. 执行业务逻辑...
}
```

---

## 🗄️ 支持的存储后端

### 关系型数据库

| 存储 | Storage 包 | Locks 包 |
|------|-----------|----------|
| MySQL | [go-mysql-storage](https://github.com/storage-lock/go-mysql-storage) | [go-mysql-locks](https://github.com/storage-lock/go-mysql-locks) |
| PostgreSQL | [go-postgresql-storage](https://github.com/storage-lock/go-postgresql-storage) | [go-postgresql-locks](https://github.com/storage-lock/go-postgresql-locks) |
| MariaDB | [go-mariadb-storage](https://github.com/storage-lock/go-mariadb-storage) | [go-mariadb-locks](https://github.com/storage-lock/go-mariadb-locks) |
| SQL Server | [go-sqlserver-storage](https://github.com/storage-lock/go-sqlserver-storage) | [go-sqlserver-locks](https://github.com/storage-lock/go-sqlserver-locks) |
| TiDB | [go-tidb-storage](https://github.com/storage-lock/go-tidb-storage) | [go-tidb-locks](https://github.com/storage-lock/go-tidb-locks) |
| Oracle | [go-oracle-storage](https://github.com/storage-lock/go-oracle-storage) | — |
| SQLite3 | [go-sqlite3-storage](https://github.com/storage-lock/go-sqlite3-storage) | — |
| OceanBase | [go-oceanbase-storage](https://github.com/storage-lock/go-oceanbase-storage) | — |
| 达梦 (Dameng) | [go-dameng-storage](https://github.com/storage-lock/go-dameng-storage) | — |
| ClickHouse | [go-clickhouse-storage](https://github.com/storage-lock/go-clickhouse-storage) | — |
| Percona | [go-percona-storage](https://github.com/storage-lock/go-percona-storage) | — |
| Snowflake | [go-snowflake-storage](https://github.com/storage-lock/go-snowflake-storage) | — |
| DB2 | [go-db2-storage](https://github.com/storage-lock/go-db2-storage) | — |
| Sybase | [go-sybase-storage](https://github.com/storage-lock/go-sybase-storage) | — |
| Teradata | [go-teradata-storage](https://github.com/storage-lock/go-teradata-storage) | — |
| Firebird | [go-firebird-storage](https://github.com/storage-lock/go-firebird-storage) | — |
| Access | [go-access-storage](https://github.com/storage-lock/go-access-storage) | — |

### ORM 框架

| 框架 | Locks 包 |
|------|----------|
| GORM | [go-gorm-locks](https://github.com/storage-lock/go-gorm-locks) |
| sqlx | [go-sqlx-locks](https://github.com/storage-lock/go-sqlx-locks) |
| xorm | [go-xorm-locks](https://github.com/storage-lock/go-xorm-locks) |
| gorp | [go-gorp-locks](https://github.com/storage-lock/go-gorp-locks) |
| beego | [go-beego-locks](https://github.com/storage-lock/go-beego-locks) |
| *sql.DB 通用 | [go-sqldb-locks](https://github.com/storage-lock/go-sqldb-locks) |

### NoSQL

| 存储 | Storage 包 | Locks 包 |
|------|-----------|----------|
| MongoDB | [go-mongodb-storage](https://github.com/storage-lock/go-mongodb-storage) | [go-mongodb-locks](https://github.com/storage-lock/go-mongodb-locks) |
| Redis | [go-redis-storage](https://github.com/storage-lock/go-redis-storage) | — |
| HBase | [go-hbase-storage](https://github.com/storage-lock/go-hbase-storage) | — |
| Cassandra | [go-cassandra-storage](https://github.com/storage-lock/go-cassandra-storage) | — |
| Memcache | [go-memcache-storage](https://github.com/storage-lock/go-memcache-storage) | — |
| DynamoDB | [go-dynamodb-storage](https://github.com/storage-lock/go-dynamodb-storage) | — |
| InfluxDB | [go-influxdb-storage](https://github.com/storage-lock/go-influxdb-storage) | — |

### 对象存储

| 存储 | Storage 包 |
|------|-----------|
| S3 | [go-s3-storage](https://github.com/storage-lock/go-s3-storage) |
| MinIO | [go-minio-storage](https://github.com/storage-lock/go-minio-storage) |
| Ceph | [go-ceph-storage](https://github.com/storage-lock/go-ceph-storage) |
| 阿里云 OSS | [go-oss-storage](https://github.com/storage-lock/go-oss-storage) |
| 腾讯云 COS | [go-cos-storage](https://github.com/storage-lock/go-cos-storage) |
| 百度云 BOS | [go-bos-storage](https://github.com/storage-lock/go-bos-storage) |
| 七牛云 Kodo | [go-kodo-storage](https://github.com/storage-lock/go-kodo-storage) |
| 华为云 OBS | [go-obs-storage](https://github.com/storage-lock/go-obs-storage) |
| 金山云 KS3 | [go-ks3-storage](https://github.com/storage-lock/go-ks3-storage) |
| 又拍云 USS | [go-uss-storage](https://github.com/storage-lock/go-uss-storage) |
| 青云 QingStor | [go-qingcloud-object-storage](https://github.com/storage-lock/go-qingcloud-object-storage) |
| US3 | [go-us3-storage](https://github.com/storage-lock/go-us3-storage) |
| Zenko | [go-zenko-storage](https://github.com/storage-lock/go-zenko-storage) |

### 嵌入式 / KV 存储

| 存储 | Storage 包 |
|------|-----------|
| LevelDB | [go-leveldb-storage](https://github.com/storage-lock/go-leveldb-storage) |
| RocksDB | [go-rocksdb-storage](https://github.com/storage-lock/go-rocksdb-storage) |
| BerkeleyDB | [go-berkeleydb-storage](https://github.com/storage-lock/go-berkeleydb-storage) |
| LedisDB | [go-ledisdb-storage](https://github.com/storage-lock/go-ledisdb-storage) |
| 内存 | [go-memory-storage](https://github.com/storage-lock/go-memory-storage) | [go-memory-locks](https://github.com/storage-lock/go-memory-locks) |

### 分布式文件系统

| 存储 | Storage 包 |
|------|-----------|
| HDFS | [go-hdfs-storage](https://github.com/storage-lock/go-hdfs-storage) |
| GlusterFS | [go-glusterfs-storage](https://github.com/storage-lock/go-glusterfs-storage) |
| JuiceFS | [go-juicefs-storage](https://github.com/storage-lock/go-juicefs-storage) |
| FastDFS | [go-fastdfs-storage](https://github.com/storage-lock/go-fastdfs-storage) |
| TFS | [go-tfs-storage](https://github.com/storage-lock/go-tfs-storage) |
| MooseFS | [go-moosefs-storage](https://github.com/storage-lock/go-moosefs-storage) |
| FTP | [go-ftp-storage](https://github.com/storage-lock/go-ftp-storage) |

### 其他存储

| 存储 | Storage 包 |
|------|-----------|
| Jackrabbit | [go-jackrabbit-storage](https://github.com/storage-lock/go-jackrabbit-storage) |
| ModeShape | [go-mode-shape-storage](https://github.com/storage-lock/go-mode-shape-storage) |
| CAS | [go-cas-storage](https://github.com/storage-lock/go-cas-storage) |
| Triton | [go-triton-storage](https://github.com/storage-lock/go-triton-storage) |
| LeoFS | [go-leofs-storage](https://github.com/storage-lock/go-leofs-storage) |
| Riak S2 | [go-riak-s2-storage](https://github.com/storage-lock/go-riak-s2-storage) |
| KV 通用 | [go-kv-storage](https://github.com/storage-lock/go-kv-storage) | [go-kv-locks](https://github.com/storage-lock/go-kv-locks) |
| HTTP | [go-http-storage](https://github.com/storage-lock/go-http-storage) | [go-http-locks](https://github.com/storage-lock/go-http-locks) |

---

## 🌐 生态项目

| 项目 | 说明 |
|------|------|
| [go-storage-lock](https://github.com/storage-lock/go-storage-lock) | 🔒 核心库 — 分布式锁模型定义与算法实现 |
| [go-storage](https://github.com/storage-lock/go-storage) | 🗄️ 存储抽象接口定义 |
| [storage-lock-http-api](https://github.com/storage-lock/storage-lock-http-api) | 🌐 HTTP API 服务 |
| [storage-lock-grpc-api](https://github.com/storage-lock/storage-lock-grpc-api) | ⚡ gRPC API 服务 |
| [storage-lock-cli](https://github.com/storage-lock/storage-lock-cli) | 💻 命令行工具 |
| [storage-lock-ui](https://github.com/storage-lock/storage-lock-ui) | 🖥️ Web UI 管理界面 |
| [java-storage-lock](https://github.com/storage-lock/java-storage-lock) | ☕ Java SDK |
| [python-storage-lock](https://github.com/storage-lock/python-storage-lock) | 🐍 Python SDK |
| [go-storage-lock-factory](https://github.com/storage-lock/go-storage-lock-factory) | 🏭 工厂模式创建锁 |
| [go-storage-lock-metric](https://github.com/storage-lock/go-storage-lock-metric) | 📊 监控指标 |
| [go-storage-lock-prometheus](https://github.com/storage-lock/go-storage-lock-prometheus) | 📈 Prometheus 集成 |
| [go-ntp-time-provider](https://github.com/storage-lock/go-ntp-time-provider) | 🕐 NTP 时间提供者 |
| [go-events](https://github.com/storage-lock/go-events) | 📡 事件系统 |
| [go-storage-events](https://github.com/storage-lock/go-storage-events) | 📡 存储事件 |
| [go-event-listener-stdout](https://github.com/storage-lock/go-event-listener-stdout) | 📋 标准输出事件监听器 |
| [go-zap-logger-event-listener](https://github.com/storage-lock/go-zap-logger-event-listener) | 📋 Zap 日志事件监听器 |
| [go-utils](https://github.com/storage-lock/go-utils) | 🔧 通用工具库 |

---

## 📄 License

[MIT License](https://github.com/storage-lock/.github/blob/main/LICENSE)

Copyright (c) 2023 Storage Lock
