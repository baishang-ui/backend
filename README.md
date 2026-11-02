# 智能桌面盒子 · Node.js 后端

为智能桌面盒子提供待办 REST API、SQLite 持久化、WebSocket 推送及阿里云 IoT 同步。小程序负责用户交互，后端负责待办状态和同步，STM32 固件负责设备侧输入与显示。

| 组件 | 仓库 |
| --- | --- |
| 微信小程序 | [wechat_code](https://github.com/baishang-ui/wechat_code) |
| 后端（本仓库） | [backend](https://github.com/baishang-ui/backend) |
| STM32 固件 | [end_stm32](https://github.com/baishang-ui/end_stm32) |

> **公开部署前必读**：[安全说明](SECURITY.md)。当前仓库历史包含运行配置和数据库文件，需要单独完成凭据轮换及历史暴露评估；本文更新不代表这些问题已处理。不要复用历史凭据、打开已有业务数据库或直接启动生产连接。

## 技术栈与职责

- Node.js、Express、内置 `node:sqlite`：待办 CRUD、软删除、版本与同步事件。
- `ws`：连接后发送快照，随后推送待办变更。
- 阿里云 SDK：通过服务端 Pub API 向设备下发消息。
- `rhea`：使用 AMQP 订阅设备上行数据。
- 后端不以设备三元组重复登录 MQTT，避免与硬件争用同一设备身份。

```text
微信小程序 <-- REST / WebSocket --> Node.js + SQLite
                                      |
                             Pub API / AMQP
                                      |
                                  阿里云 IoT
                                      |
                                 STM32 设备
```

## 运行环境

推荐 **Node.js 22.13+ 的受支持补丁版本**。本项目使用 `require('node:sqlite').DatabaseSync`，不能按 Node.js 18 项目运行。`node:sqlite` 自 Node.js 22.5 提供；22.5—22.12 需要实验开关，本仓库默认启动脚本没有附加该开关，因此不应把 22.5 当作直接运行默认脚本的最低版本。

```sh
node --version
node -e "require('node:sqlite'); console.log('node:sqlite available')"
```

出现实验性 API 提示不等于执行失败，但仍需对所用 Node.js 版本进行兼容验证。

## 安全的本地联调

在克隆出的开发副本中安装依赖。仓库曾跟踪依赖目录，不能把其现有内容视为可信的安装结果；使用自己的依赖安装流程，并审查锁文件。

```sh
npm ci --ignore-scripts
```

不要直接复制仓库中的历史 `.env`。先阅读 `src/config.js`，以新建的测试配置运行。以下 PowerShell 示例把服务限制在本机，生成临时鉴权值并使用独立数据库，显式关闭云端同步：

```powershell
$env:HOST = '127.0.0.1'
$env:PORT = '3000'
$env:PUBLIC_BASE_URL = 'http://127.0.0.1:3000'
$env:DB_FILE = Join-Path $env:TEMP ('desktop-box-' + [guid]::NewGuid() + '.db')
$env:MINIAPP_API_TOKEN = [guid]::NewGuid().ToString()
$env:DEVICE_API_TOKEN = [guid]::NewGuid().ToString()
$env:ALIYUN_IOT_ENABLED = 'false'
npm start
```

`dotenv` 默认读取启动目录的 `.env`；显式环境变量通常优先，但上述设置不应被理解为允许保留或传播历史凭据。更严格的隔离测试应从不含 `.env` 的临时工作目录启动 `src/server.js` 的绝对路径。开发时可用 `npm run dev` 启用文件监听。

## 配置项

配置的唯一代码依据是 `src/config.js`。部署前阅读源码，不从文档复制真实设备身份。

| 字段 | 用途与行为 |
| --- | --- |
| `HOST` / `PORT` | 绑定地址与端口；本地测试显式设为回环地址 |
| `PUBLIC_BASE_URL` | 对外服务地址；真机需使用其可访问的地址 |
| `DB_FILE` | SQLite 路径；相对路径以项目根目录解析 |
| `MINIAPP_API_TOKEN` | REST 与小程序 WebSocket 鉴权；空值会关闭相应校验 |
| `DEVICE_API_TOKEN` | 设备 HTTP 消息入口鉴权；空值也会关闭校验 |
| `ALIYUN_IOT_ENABLED` | 代码默认 `false`；本地验证保持关闭 |
| `ALIYUN_REGION_ID` | 云端地域 |
| `ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET` | 服务端云凭据；仅在受保护环境配置 |
| `ALIYUN_PRODUCT_KEY` / `ALIYUN_DEVICE_NAME` | 自有测试设备身份 |
| `ALIYUN_IOT_INSTANCE_ID` | 适用场景的 IoT 实例 |
| `ALIYUN_AMQP_HOST` / `ALIYUN_AMQP_CONSUMER_GROUP_ID` | 上行订阅服务与消费组 |
| `ALIYUN_DEVICE_COMMAND_TOPIC` | 下行命令 Topic |
| `ALIYUN_DEVICE_REPORT_TOPIC(S)` | 单个/多个上行同步 Topic |
| `ALIYUN_DEVICE_TELEMETRY_TOPIC(S)` | 单个/多个遥测 Topic |

云端联调需要有效的测试账号、最小权限凭据与正确 Topic。源码已有的硬编码回退 Topic 也必须审查，不能直接用于无关设备。

## API

认证优先读取 `Authorization: Bearer <token>`，也支持 `x-api-key`。WebSocket 还支持查询参数，但 URL 可能进入日志，推荐避免把长期令牌放入 URL。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/health` | 健康检查与 `iotEnabled`，不要求令牌 |
| GET | `/api/todos` | 待办列表；可选 `includeDeleted=true` |
| GET | `/api/todos/:id` | 单条待办 |
| POST | `/api/todos` | 创建 `{ title }` |
| PATCH | `/api/todos/:id` | 更新 `{ title?, completed? }` |
| DELETE | `/api/todos/:id` | 软删除 |
| GET | `/api/events` | 同步事件；可选 `limit` |
| POST | `/api/device/messages` | 设备消息入口，使用设备令牌 |
| WebSocket | `/ws/todos` | `todo.snapshot` / `todo.upsert` / `todo.delete` |

未删除待办上限为 5 条。对象包含 `id`、`title`、`completed`、`deleted`、`source`、毫秒时间戳与 `version`。云端关闭时本地操作仍可执行，但不能据此声称设备同步成功。

## 代码导航

```text
src/server.js                   HTTP 路由、错误处理与服务启动
src/config.js                   环境变量解析与配置
src/db.js                       SQLite 初始化、索引与事务
src/services/todoService.js      待办及同步消息领域逻辑
src/services/todoPushService.js  WebSocket 快照与广播
src/services/iotService.js       阿里云 Pub 下发
src/services/amqpService.js      AMQP 上行订阅
```

## 验证与限制

本次整理对源码进行了只读核对。与当前版本一致的本地代码已在独立数据库、回环地址、关闭云连接条件下验证健康检查、鉴权拒绝和待办增删改查；没有验证真实 IoT 账户、AMQP 网络或硬件端到端同步。

仓库目前没有 `npm test` 脚本，不要把“服务能启动”当作完整测试通过。生产使用还需要完善认证、凭据管理、访问控制、跨域策略、备份和发布验证。
