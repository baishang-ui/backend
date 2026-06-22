<!-- 标题 -->
# Todo IoT Backend

基于 **Node.js + Express + SQLite** 的待办事项后端服务，通过阿里云 IoT 平台与硬件设备通信。支持小程序 HTTP REST 接口管理待办，并通过阿里云 `Pub` API 将变更实时同步下发至物联网设备。

---

## 系统架构

```
┌──────────┐     REST / HTTP     ┌───────────────┐     Pub API     ┌──────────────┐
│  小程序  │ ◄──────────────────► │  Todo Backend  │ ──────────────► │  阿里云 IoT   │
└──────────┘                      │   (Express)    │                 └──────┬───────┘
                                  │   + SQLite     │                        │
                                  └───────┬───────┘              ┌─────────▼──────┐
                                          │                      │   IoT 硬件设备   │
                                          │ AMQP (订阅上行 Topic) │                 │
                                          └─────────────────────►│ 订阅下行 Topic  │
                                                                 └────────────────┘
```

- **小程序 → 后端 REST → SQLite**：待办增删改查
- **后端 → 阿里云 IoT `Pub` 接口 → 设备下行 Topic**：变更同步下发
- **设备上行数据**：通过阿里云 AMQP 订阅接收

后端不再使用设备三元组登录 MQTT，避免与硬件设备争抢同一设备证书导致互踢下线。

---

## 目录结构

```
backend/
├── src/
│   ├── server.js                 # HTTP 服务入口（路由、中间件、启动）
│   ├── config.js                 # 环境变量与配置解析
│   ├── db.js                     # SQLite 数据库初始化与迁移
│   └── services/
│       ├── todoService.js        # 待办 CRUD 业务逻辑与同步消息处理
│       ├── todoPushService.js    # WebSocket 推送服务（小程序实时同步）
│       ├── iotService.js         # 阿里云 IoT Pub 下行下发服务
│       └── amqpService.js        # 阿里云 IoT AMQP 上行消息订阅服务
├── data/                         # SQLite 数据库文件目录
├── .env.example                  # 环境变量模板
├── package.json
├── start-backend.bat             # Windows 一键启动脚本
└── start-backend.ps1             # PowerShell 自动安装 & 启动脚本
```

---

## 快速启动

### 1. 环境准备

- **Node.js** >= 18
- **npm** >= 9

### 2. 安装与配置

```bash
cd backend
cp .env.example .env        # 拷贝并编辑 .env 填入真实凭证
npm install
```

### 3. 启动服务

```bash
npm run dev                 # 开发模式（--watch 热重载）
```

或直接双击 `start-backend.bat`（自动检测 .env / node_modules 并启动）。

---

## 环境变量

| 变量名 | 说明 | 默认值 |
|---|---|---|
| `PORT` | HTTP 监听端口 | `3000` |
| `HOST` | 监听地址 | `0.0.0.0` |
| `PUBLIC_BASE_URL` | 公网访问地址 | `http://10.232.28.55:3000` |
| `DB_FILE` | SQLite 数据库文件路径 | `./data/todos.db` |
| `MINIAPP_API_TOKEN` | 小程序 API 访问 Token | （必填） |
| `DEVICE_API_TOKEN` | 设备 HTTP 接口 Token | （选填） |
| `ALIYUN_IOT_ENABLED` | 是否启用阿里云 IoT | `true` |
| `ALIYUN_REGION_ID` | 阿里云区域 | `cn-shanghai` |
| `ALIYUN_ACCESS_KEY_ID` | 阿里云 AccessKeyId | （必填） |
| `ALIYUN_ACCESS_KEY_SECRET` | 阿里云 AccessKeySecret | （必填） |
| `ALIYUN_IOT_INSTANCE_ID` | IoT 实例 ID（企业版必填） | （空） |
| `ALIYUN_AMQP_HOST` | AMQP 服务端地址 | （必填） |
| `ALIYUN_AMQP_CONSUMER_GROUP_ID` | AMQP 消费组 ID | `DEFAULT_GROUP` |
| `ALIYUN_AMQP_CLIENT_ID` | AMQP 客户端 ID | `todo-backend-amqp` |
| `ALIYUN_PRODUCT_KEY` | IoT 产品 Key | `k1e98mYkQSF` |
| `ALIYUN_DEVICE_NAME` | 设备名称 | `device` |
| `ALIYUN_USE_TLS` | 是否启用 TLS（保留字段） | `false` |
| `ALIYUN_DEVICE_COMMAND_TOPIC` | 下行下发 Topic | `/k1e98mYkQSF/device/user/recv_data` |
| `ALIYUN_DEVICE_REPORT_TOPIC` | 上行上报 Topic | `/k1e98mYkQSF/device/user/report_data` |

---

## API 接口

所有小程序接口需携带 `Authorization: Bearer <MINIAPP_API_TOKEN>` 请求头（也可用 `x-api-key` 头）。

### 健康检查

```http
GET /api/health
```

响应示例：

```json
{
  "ok": true,
  "service": "todo-iot-backend",
  "time": 1713250010000,
  "iotEnabled": true
}
```

### 获取待办列表

```http
GET /api/todos
GET /api/todos?includeDeleted=true
Authorization: Bearer <token>
```

### 获取单条待办

```http
GET /api/todos/:id
Authorization: Bearer <token>
```

### 新增待办

```http
POST /api/todos
Authorization: Bearer <token>
Content-Type: application/json

{ "title": "买电池" }
```

### 更新待办

```http
PATCH /api/todos/:id
Authorization: Bearer <token>
Content-Type: application/json

{ "title": "买 5 号电池", "completed": true }
```

### 删除待办（软删除）

```http
DELETE /api/todos/:id
Authorization: Bearer <token>
```

### 获取同步事件

```http
GET /api/events?limit=50
Authorization: Bearer <token>
```

### 设备上行消息（HTTP 通道）

```http
POST /api/device/messages
Authorization: Bearer <device_token>    # 或 x-api-key
Content-Type: application/json

{
  "type": "todo.sync",
  "todo": { "id": "xxx", "completed": true }
}
```

---

## 消息下发格式

每次待办变更后，后端自动通过阿里云 `Pub` 接口将完整快照下发到下行 Topic。

**下发 Topic**：`/k1e98mYkQSF/device/user/recv_data`

```json
{
  "type": "todo.snapshot",
  "reason": "miniapp.create",
  "timestamp": 1713250010000,
  "todos": [
    {
      "id": "todo-1713250000000-1234",
      "title": "买电池",
      "completed": false,
      "deleted": false,
      "createdAt": 1713250000000,
      "updatedAt": 1713250000000,
      "source": "miniapp",
      "version": 1
    }
  ]
}
```

### 触发场景

| 场景 | `reason` 值 |
|---|---|
| 小程序新增 | `miniapp.create` |
| 小程序修改 | `miniapp.update` |
| 小程序删除 | `miniapp.delete` |
| 设备同步 | `device.pull` |
| AMQP 上行同步 | `sync.*` |
| WebSocket 推送 | `device.pull` |

---

## 数据库

使用 SQLite（WAL 模式），主要表结构：

- **todos**：待办事项表，支持软删除（`deleted` 字段）、乐观锁（`version` 字段）
- **sync_events**：同步事件日志，记录所有变更来源与负载

---

## 关键设计说明

- 后端不持有设备三元组，不建立设备 MQTT 长连接 —— 避免与硬件互踢
- 下行通过阿里云 `Pub` API 发布，上行通过 AMQP 消费组订阅
- WebSocket 推送支持小程序实时获取变更
- 所有待办操作均在事务中执行，保证数据一致性
- 支持多 Topic 订阅上报数据（`ALIYUN_DEVICE_REPORT_TOPICS` 逗号分隔）

---

## 技术栈

| 技术 | 用途 |
|---|---|
| [Express](https://expressjs.com/) | HTTP 框架 |
| [SQLite (node:sqlite)](https://nodejs.org/api/sqlite.html) | 本地数据库 |
| [@alicloud/iot20180120](https://www.npmjs.com/package/@alicloud/iot20180120) | 阿里云 IoT SDK（Pub 下发） |
| [rhea](https://www.npmjs.com/package/rhea) | AMQP 客户端（上行订阅） |
| [ws](https://www.npmjs.com/package/ws) | WebSocket 推送 |
| [mqtt](https://www.npmjs.com/package/mqtt) | MQTT 客户端（保留依赖） |
| [dotenv](https://www.npmjs.com/package/dotenv) | 环境变量管理 |
| [cors](https://www.npmjs.com/package/cors) | 跨域中间件 |
