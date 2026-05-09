# 中文 Syslog 实时日志审计系统

一个可直接通过 Docker Compose 部署的微服务 SIEM 平台，包含：

- FastAPI 后端 API
- Vue3 + Element Plus 中文前端
- Elasticsearch 日志检索
- Redis 消息队列 / 缓存 / 发布订阅
- Syslog UDP/TCP 接收服务
- 实时 WebSocket 日志流
- 资产、规则、告警、系统设置与示例初始化数据

## 服务结构

- `frontend`：中文 Web 界面
- `backend`：认证、仪表盘、资产、规则、告警、日志查询、WebSocket
- `collector`：Syslog UDP/TCP/HTTP 接入
- `processor`：日志清洗、规则匹配、告警生成、写入 Elasticsearch
- `elasticsearch`：日志存储与全文检索
- `redis`：消息队列、缓存、实时消息总线
- `kibana`：可选的 Elasticsearch 数据浏览
- `seed-data`：启动后自动注入示例日志

## 一键启动

```bash
docker-compose up -d --build
```

当前仓库已经内置：

- `backend/vendor`、`collector/vendor`、`processor/vendor`：Python 离线运行依赖
- `frontend/dist`：前端预构建静态资源

因此自定义服务镜像在 Docker 构建阶段不再访问 PyPI 或 npm registry。

注意：

- 仍需保证 Docker 能获取基础镜像和第三方镜像，或提前执行离线打包
- 若目标环境完全离线，请使用下文“完全离线部署”

启动完成后访问：

- 前端：`http://localhost:8080`
- 后端 OpenAPI：`http://localhost:8000/docs`
- Kibana：`http://localhost:5601`

## 项目目录

```text
backend/     FastAPI API 服务
collector/   Syslog 接收服务
processor/   日志处理与告警引擎
frontend/    Vue3 中文前端
examples/    rsyslog / filebeat 示例配置
scripts/     辅助脚本
data/        SQLite 与本地持久化目录
```

## 功能说明

### 1. 仪表盘

- 最近 24 小时日志总数
- 每秒日志数（EPS）
- 资产数量
- Elasticsearch 存储使用量
- CPU / 内存 / 磁盘使用率
- 当前告警数量
- 日志趋势、来源 TOP10、服务类型 TOP10 图表

### 2. 日志检索

- 实时日志流
- 按 IP、资产名、级别、服务、时间范围筛选
- Elasticsearch 全文搜索
- WebSocket 自动刷新

### 3. 资产管理

- 新增 / 编辑 / 删除资产
- 支持资产分组

### 4. 规则与告警

- 关键词匹配规则
- 阈值规则
- 告警列表与处理状态更新

### 5. 日志接入

- Syslog UDP 514
- Syslog TCP 514
- HTTP 日志接入接口
- `examples/` 中提供 `rsyslog` 和 `filebeat` 示例
- `collector` 默认使用宿主机网络模式，避免 Docker 网桥把设备真实源 IP 改写成容器网关地址

## 初始化数据

- API 启动时自动初始化管理员、演示资产、演示规则、基础设置
- `seed-data` 服务在系统启动后自动写入演示日志

## 启动步骤

推荐先把数据目录放到大容量磁盘，例如你的 `/home` 盘：

```bash
mkdir -p /home/syslog-data/elasticsearch /home/syslog-data/redis /home/syslog-data/app
chown -R 1000:0 /home/syslog-data/elasticsearch
chmod -R 775 /home/syslog-data/elasticsearch /home/syslog-data/redis /home/syslog-data/app
export ES_DATA_DIR=/home/syslog-data/elasticsearch
export REDIS_DATA_DIR=/home/syslog-data/redis
export APP_DATA_DIR=/home/syslog-data/app
export HOST_STORAGE_ROOT=/home/syslog-data
```

1. 构建并启动服务

```bash
export INITIAL_ADMIN_PASSWORD='请自行设置强密码'
export STREAM_BATCH_SIZE=1000
export FILEBEAT_BATCH_SIZE=300
docker-compose up -d --build
```

2. 查看服务状态

```bash
docker-compose ps
```

3. 查看日志

```bash
docker-compose logs -f backend
docker-compose logs -f collector
docker-compose logs -f processor
```

4. 如果初始化日志未自动注入，可手动执行

```bash
docker-compose run --rm seed-data
```

## 受限网络部署

如果当前机器只能访问 Docker 镜像仓库，但容器构建阶段无法访问 PyPI / npm：

```bash
export INITIAL_ADMIN_PASSWORD='请自行设置强密码'
export STREAM_BATCH_SIZE=1000
export FILEBEAT_BATCH_SIZE=300
docker compose build backend collector processor frontend
docker compose up -d
```

这四个自定义镜像会直接使用仓库内的离线依赖与静态资源完成构建。

## 完全离线部署

### 在可联网机器上准备离线包

```bash
./scripts/prepare-offline-package.sh
```

该脚本会：

- 重新生成 Python `vendor` 目录
- 重新构建前端 `dist`
- 构建自定义镜像
- 拉取 Elasticsearch / Kibana / Redis 镜像
- 导出 `offline/syslog-offline-images.tar`

### 在离线机器上导入并启动

1. 复制整个项目目录和 `offline/syslog-offline-images.tar` 到离线机器

2. 导入镜像

```bash
docker load -i offline/syslog-offline-images.tar
```

3. 启动

```bash
export INITIAL_ADMIN_PASSWORD='请自行设置强密码'
export HOST_STORAGE_ROOT=/home/syslog-data
export STREAM_BATCH_SIZE=1000
export FILEBEAT_BATCH_SIZE=300
docker compose -f docker-compose.yml -f docker-compose.offline.yml up -d
```

## 示例接入

### 通过 HTTP 写入日志

```bash
curl -X POST http://localhost:1514/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "hostname": "db-server-01",
    "ip": "10.10.1.21",
    "service": "mysql",
    "level": "error",
    "message": "SQL注入攻击特征命中 union select"
  }'
```

### 通过 Syslog UDP 发送

```bash
logger -n 127.0.0.1 -P 514 -d "Failed password for root from 10.10.1.99"
```

## 配置文件

- `examples/rsyslog/rsyslog.conf`
- `examples/filebeat/filebeat.yml`

## 说明

- SQLite 仅用于演示资产、规则、告警、用户等业务数据持久化
- 日志检索与聚合统计均使用 Elasticsearch
- Redis 同时承担队列、缓存和发布订阅通道
- 默认支持通过 `ES_DATA_DIR`、`REDIS_DATA_DIR`、`APP_DATA_DIR` 把数据目录挂载到大容量磁盘
- `HOST_STORAGE_ROOT` 用于让系统设置页直接探测宿主机存储目录容量，无需额外修复脚本
- `STREAM_BATCH_SIZE`、`FILEBEAT_BATCH_SIZE` 用于提升高日志量场景下的处理吞吐
- 管理员密码不会写入仓库；首次启动时必须通过 `INITIAL_ADMIN_PASSWORD` 环境变量传入
- 本版本已内建日志时间修正、日志检索排序/设备直选/模糊搜索，以及宿主机存储容量探测
- 若修改了 Python 依赖或前端源码，请重新执行 `./scripts/prepare-offline-package.sh`

## 后续建议

- 生产环境建议启用 HTTPS、RBAC、多租户与专用消息队列
- 若日志量更大，可将 Redis 替换为 Kafka，并增加 processor 副本数
