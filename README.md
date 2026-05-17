# llama-swap 框架分析文档

> 本文档记录 llama-swap 框架的架构设计、功能支持、第三方依赖、代码结构和API路由定义，作为替换 yzma 框架的参考文档。

---

## 一、架构设计

### 1.1 整体架构

llama-swap 是一个用 Go 语言编写的高性能 AI 模型热切换代理服务器，采用反向代理架构实现多模型的按需切换。

```
┌─────────────────────────────────────────────────────────────────┐
│                        llama-swap                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    ProxyManager (核心)                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │  │
│  │  │ Gin Engine  │  │   Matrix    │  │  ProcessGroup   │   │  │
│  │  │  (HTTP路由) │  │ (并发模型) │  │  (进程管理)     │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │  │
│  │  │ PeerProxy   │  │ Config      │  │  Event System   │   │  │
│  │  │ (远程代理) │  │  (配置管理) │  │   (事件系统)    │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     监控与日志                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │  │
│  │  │ LogMonitor  │  │ PerfMonitor │  │ MetricsMonitor  │   │  │
│  │  │  (日志监控) │  │ (性能监控) │  │  (指标监控)     │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────────┐
        │           Upstream Inference Servers         │
        │  ┌─────────┐  ┌─────────┐  ┌─────────────┐  │
        │  │llama.cpp│  │  vllm   │  │ stable-diff │  │
        │  │ (LLM)   │  │ (LLM)   │  │   (图像)    │  │
        │  └─────────┘  └─────────┘  └─────────────┘  │
        └─────────────────────────────────────────────┘
```

### 1.2 核心组件说明

| 组件 | 文件位置 | 功能描述 |
|------|----------|----------|
| **ProxyManager** | `proxy/proxymanager.go` | 核心代理管理器，负责HTTP路由、进程切换、请求转发 |
| **Process** | `proxy/process.go` | 单个推理进程的生命周期管理 |
| **ProcessGroup** | `proxy/processgroup.go` | 进程组管理，支持互斥/持久化模式 |
| **Matrix** | `proxy/matrix.go` | DSL驱动的并发模型调度系统 |
| **PeerProxy** | `proxy/peerproxy.go` | 远程节点代理，支持分布式推理 |
| **Config** | `proxy/config/config.go` | YAML配置加载、宏替换、模型映射 |
| **Event** | `event/event.go` | 事件发布订阅系统，用于UI实时更新 |

### 1.3 工作流程

```
用户请求 → ProxyManager接收 → 提取model参数 → 查找模型配置
    ↓
检查当前运行进程 → 是否匹配?
    ↓                    ↓
  匹配: 直接转发      不匹配: 停止旧进程 → 启动新进程 → 等待就绪 → 转发
    ↓
请求转发到上游服务器 → 响应返回用户
```

---

## 二、功能支持

### 2.1 核心特性

| 特性 | 描述 | 状态 |
|------|------|------|
| **按需模型切换** | 根据请求中的 model 参数自动切换推理进程 | ✅ |
| **零依赖部署** | 单一二进制文件 + 单一配置文件，无外部依赖 | ✅ |
| **OpenAI API 兼容** | 支持 OpenAI API 格式的推理服务器 | ✅ |
| **并发模型运行** | 通过 Matrix DSL 支持多模型并发 | ✅ |
| **自动卸载** | TTL 超时后自动卸载闲置模型 | ✅ |
| **预加载** | 启动时预加载指定模型 | ✅ |
| **API Key 认证** | 支持 API Key 访问控制 | ✅ |
| **实时 Web UI** | Svelte 构建的实时监控界面 | ✅ |
| **日志流** | SSE 实时日志流输出 | ✅ |
| **Prometheus 指标** | 系统和 GPU 性能指标导出 | ✅ |
| **配置热重载** | 监听配置文件变化自动重载 | ✅ |
| **TLS 支持** | 支持 HTTPS 加密连接 | ✅ |
| **远程节点代理** | 支持代理到远程 llama-swap 节点 | ✅ |

### 2.2 支持的推理服务器

| 服务器类型 | 用途 | 备注 |
|------------|------|------|
| llama.cpp / llama-server | LLM 推理 | 主要支持，最佳兼容 |
| vllm | LLM 推理 | 推荐 Docker/Podman 运行 |
| tabbyAPI | LLM 推理 | Python 实现 |
| stable-diffusion.cpp | 图像生成 | 支持 SDAPI |
| whisper.cpp | 语音识别 | 支持 audio API |
| ik-llama-server | LLM 推理 | 优化版本 |

---

## 三、第三方 SDK 引用

### 3.1 主要依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `github.com/gin-gonic/gin` | v1.10.0 | HTTP Web 框架，路由和中间件 |
| `github.com/tidwall/gjson` | v1.18.0 | 高性能 JSON 解析 |
| `github.com/tidwall/sjson` | v1.2.5 | JSON 动态修改 |
| `gopkg.in/yaml.v3` | v3.0.1 | YAML 配置解析 |
| `github.com/shirou/gopsutil/v4` | v4.26.4 | 系统性能监控（CPU/内存/GPU） |
| `github.com/klauspost/compress` | v1.18.5 | HTTP 响应压缩（gzip/brotli） |
| `github.com/billziss-gh/golib` | v0.2.0 | Shell 命令解析（shlex） |
| `github.com/fxamacker/cbor/v2` | v2.9.1 | CBOR 编码支持 |
| `github.com/stretchr/testify` | v1.11.1 | 单元测试框架 |

### 3.2 间接依赖

| 包名 | 用途 |
|------|------|
| `github.com/bytedance/sonic` | 高性能 JSON 编解码 |
| `github.com/go-playground/validator/v10` |请求验证 |
| `golang.org/x/net` | 网络库扩展 |
| `google.golang.org/protobuf` | Protobuf 支持 |

---

## 四、代码结构

### 4.1 目录树

```
llama-swap/
├── llama-swap.go              # 主入口文件，服务器启动和信号处理
├── go.mod                     # Go 模块定义
├── go.sum                     # 依赖版本锁定
├── Makefile                   # 构建脚本
├── config.example.yaml        # 配置示例
├── config-schema.json         # 配置 JSON Schema
├── .goreleaser.yaml           # Goreleaser 发布配置
│
├── proxy/                     # 核心代理模块
│   ├── proxymanager.go        # 代理管理器（核心）
│   ├── proxymanager_api.go    # API 处理器
│   ├── proxymanager_loghandlers.go  # 日志处理器
│   ├── process.go             # 进程管理
│   ├── process_unix.go        # Unix 进程实现
│   ├── process_windows.go     # Windows 进程实现
│   ├── processgroup.go        # 进程组管理
│   ├── matrix.go              # Matrix DSL 并发调度
│   ├── peerproxy.go           # 远程节点代理
│   ├── metrics_monitor.go     # 指标监控
│   ├── sanitize_cors.go       # CORS 安全处理
│   ├── ui_compress.go         # UI 压缩服务
│   ├── ui_embed.go            # UI 嵌入文件系统
│   ├── events.go              # 事件定义
│   ├── discardWriter.go       # 丢弃写入器
│   ├── helpers_test.go        # 测试辅助
│   │
│   ├── config/                # 配置子模块
│   │   ├── config.go          # 配置加载和验证
│   │   └── matrix.go          # Matrix 配置解析
│   │
│   ├── configwatcher/         # 配置监听
│   │   └── watcher.go         # 文件变化监听
│   │
│   └── cache/                 # 缓存模块
│
├── event/                     # 事件系统
│   ├── event.go               # 事件发布订阅
│   ├── default.go             # 默认事件处理
│   └ README.md                # 模块说明
│
├── internal/                  # 内部模块
│   ├── logmon/                # 日志监控
│   │   ├── monitor.go         # Ring Buffer 日志
│   │
│   ├── perf/                  # 性能监控
│   │   ├── monitor.go         # 系统/GPU 性能采集
│   │
│   └── ring/                  # Ring Buffer 实现
│       ├── ring.go            # 高效环形缓冲
│
├── ui-svelte/                 # Web UI（Svelte）
│   ├── src/                   # 源代码
│   ├── public/                # 静态资源
│   ├── package.json           # NPM 配置
│   ├── vite.config.ts         # Vite 构建配置
│   └── svelte.config.js       # Svelte 配置
│
├── cmd/                       # 辅助命令
│   ├── simple-responder/      # 简单响应测试
│   ├── monitor-test/          # 监控测试
│   ├── wol-proxy/             # Wake-on-LAN 代理
│   └ misc/                    # 其他工具
│
├── docker/                    # Docker 构建
│   ├── build-image.sh         # 构建脚本
│   ├── build-container.sh     # 容器构建
│   ├── llama-swap.Containerfile  # 容器定义
│   └ unified/                 # 统一容器构建
│
├── docs/                      # 文档
│   ├── configuration.md       # 配置文档
│   ├── container-security.md  # 容器安全
│   ├── assets/                # 文档资源
│   ├── examples/              # 示例
│   └ grafana/                 # Grafana 仪表盘
│
├── scripts/                   # 安装脚本
│   ├── install.sh             # 安装脚本
│   └ uninstall.sh             # 卸载脚本
│
└── ai-plans/                  # AI 开发计划
    ├── 2025-12-14-efficient-ring-buffer.md
    ├── improve-tests-655.md
    └ issue-264-add-metadata.md
    └ issue-336-macro-in-macro.md
```

### 4.2 关键文件说明

| 文件 | 行数 | 功能 |
|------|------|------|
| `proxy/proxymanager.go` | ~1232 | 核心代理管理器，HTTP路由定义，请求处理 |
| `proxy/config/config.go` | ~827 | 配置加载，宏替换，模型别名处理 |
| `llama-swap.go` | ~249 | 主入口，信号处理，配置重载 |
| `proxy/process.go` | - | 进程启动/停止/健康检查 |
| `proxy/matrix.go` | - | DSL 解析和并发调度 |

---

## 五、API 路由定义

### 5.1 OpenAI 兼容 API

| 方法 | 路径 | 功能 | 备注 |
|------|------|------|------|
| POST | `/v1/chat/completions` | 聊天补全 | 主要接口 |
| POST | `/v1/completions` | 文本补全 | 传统接口 |
| POST | `/v1/responses` | 响应生成 | 新接口 |
| POST | `/v1/embeddings` | 文本嵌入 | 向量化 |
| POST | `/v1/audio/speech` | 语音合成 | TTS |
| POST | `/v1/audio/transcriptions` | 语音转录 | ASR |
| GET/POST | `/v1/audio/voices` | 语音列表 | 可用语音 |
| POST | `/v1/images/generations` | 图像生成 | DALL-E 格式 |
| POST | `/v1/images/edits` | 图像编辑 | 图像修改 |
| GET | `/v1/models` | 模型列表 | 返回可用模型 |

### 5.2 Anthropic API

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/v1/messages` | Anthropic Messages API |
| POST | `/v1/messages/count_tokens` | Token 计数 |

### 5.3 llama-server 专用 API

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/rerank` | 重排序 |
| POST | `/reranking` | 重排序（别名） |
| POST | `/v1/rerank` | 重排序（版本化） |
| POST | `/v1/reranking` | 重排序（版本化） |
| POST | `/infill` | 代码填充 |
| POST | `/completion` | 补全（无版本） |

### 5.4 SDAPI（stable-diffusion.cpp）

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/sdapi/v1/txt2img` | 文本生成图像 |
| POST | `/sdapi/v1/img2img` | 图像到图像 |
| GET | `/sdapi/v1/loras` | LoRA 模型列表 |

### 5.5 llama-swap 管理 API

| 方法 | 路径 | 功能 | 认证 |
|------|------|------|------|
| GET | `/ui` | Web UI | 无 |
| GET | `/ui/*filepath` | UI 静态文件 | 无 |
| GET | `/upstream/:model_id` | 直接访问上游 | API Key |
| ANY | `/upstream/:model_id/*path` | 上游代理 | API Key |
| GET | `/running` | 运行模型列表 | API Key |
| GET | `/unload` | 卸载所有模型 | API Key |
| POST | `/api/models/unload` | 卸载所有模型 | API Key |
| POST | `/api/models/unload/:model_id` | 卸载指定模型 | API Key |
| GET | `/api/events` | SSE 事件流 | API Key |
| GET | `/api/metrics` | JSON 指标 | API Key |
| GET | `/api/performance` | 性能数据 | API Key |
| GET | `/api/version` | 版本信息 | API Key |
| GET | `/api/captures/:id` | 请求捕获数据 | API Key |

### 5.6 日志 API

| 方法 | 路径 | 功能 | 备注 |
|------|------|------|------|
| GET | `/logs` | 缓冲日志 | 文本格式 |
| GET | `/logs/stream` | 日志流 | SSE |
| GET | `/logs/stream/proxy` | 代理日志流 | SSE |
| GET | `/logs/stream/upstream` | 上游日志流 | SSE |
| GET | `/logs/stream/:model_id` | 指定模型日志 | SSE |

### 5.7 健康检查 API

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/health` | 健康检查（返回 "OK"） |
| GET | `/wol-health` | Wake-on-LAN 健康检查 |
| GET | `/metrics` | Prometheus 指标 |

### 5.8 版本化 API（无 v1 前缀）

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/v/chat/completions` | 聊天补全 |
| POST | `/v/completions` | 文本补全 |
| POST | `/v/responses` | 响应生成 |
| POST | `/v/messages` | Anthropic Messages |
| POST | `/v/embeddings` | 文本嵌入 |
| POST | `/v/rerank` | 重排序 |
| POST | `/v/reranking` | 重排序 |

---

## 六、配置结构

### 6.1 配置示例

```yaml
# 最小配置
models:
  model1:
    cmd: llama-server --port ${PORT} --model /path/to/model.gguf

# 完整配置结构
models:
  model-id:
    cmd: "启动命令"
    cmdStop: "停止命令"       # Docker/Podman 使用
    proxy: "http://localhost:${PORT}"
    checkEndpoint: "/health"  # 健康检查路径
    ttl: 300                  # 自动卸载超时（秒）
    aliases: ["gpt-4o-mini"]  # 模型别名
    useModelName: "real-name" # 发送到上游的模型名
    unlisted: false           # 是否隐藏
    name: "显示名称"
    description: "模型描述"
    env: ["VAR=value"]        # 环境变量
    filters:
      stripParams: "top_p,temperature"
      setParams: {"temperature": 0.7}
      setParamsByID: {"alias-name": {"temperature": 0.5}}
    metadata: {}              # 自定义元数据
    macros: []                # 模型级宏

groups:
  group-id:
    swap: true                # 是否切换
    exclusive: true           # 是否互斥
    persistent: false         # 是否持久
    members: ["model1", "model2"]

matrix:                       # 并发模型 DSL
  sets:
    - members: ["modelA", "modelB"]
      concurrent: true

macros:                       # 全局宏
  MODEL_PATH: "/models"

hooks:
  on_startup:
    preload: ["model1"]       # 预加载模型

apiKeys: ["secret-key"]       # API Key 列表
peers:                        # 远程节点
  peer1:
    url: "http://remote:8080"
    apiKey: "key"
    models: ["remote-model"]
    filters:
      stripParams: ""
      setParams: {}

globalTTL: 0                  # 全局 TTL
startPort: 5800               # 端口起始
healthCheckTimeout: 120       # 健康检查超时
logLevel: "info"              # 日志级别
logToStdout: "proxy"          # 日志输出
performance:
  every: 5s                   # 性能采集间隔
  disabled: false
```

### 6.2 宏系统

| 宏 | 描述 |
|----|----|
| `${PORT}` | 自动分配的端口 |
| `${MODEL_ID}` | 模型 ID |
| `${PID}` | 进程 ID（运行时替换） |
| `${env.VAR_NAME}` | 环境变量 |
| 自定义宏 | 用户定义 |

---

## 七、事件系统

### 7.1 事件类型

| 事件 | 触发时机 |
|------|----------|
| `ProcessStateChangeEvent` | 进程状态变化 |
| `ConfigFileChangedEvent` | 配置文件变化 |
| `ActivityLogEvent` | 活动日志记录 |
| `InFlightRequestsEvent` | 请求计数变化 |
| `ModelPreloadedEvent` | 模型预加载完成 |

---

## 八、与 yzma 对比

### 8.1 llama-swap 优势

| 特性 | llama-swap | yzma |
|------|------------|------|
| **部署复杂度** | 单二进制 | 需要多组件 |
| **模型切换** | 自动热切换 | 手动/有限 |
| **并发模型** | Matrix DSL | 单模型 |
| **Web UI** | 内置 Svelte | 无/外部 |
| **日志流** | SSE 实时 | 文件日志 |
| **远程代理** | PeerProxy | 无 |
| **配置重载** | 热重载 | 重启 |
| **第三方依赖** | 零依赖 | 多依赖 |

---

## 九、Docker 代理架构详解

### 9.1 核心设计理念

llama-swap 采用 **反向代理 + 进程管理** 的架构模式，实现推理请求的透明转发：

```
用户请求 → llama-swap (代理层) → 上游推理服务 (llama-server/sd-server等)
                ↓
         1. 解析请求中的 model 参数
         2. 查找对应的模型配置
         3. 检查/启动/切换进程
         4. 通过 httputil.ReverseProxy 转发请求
         5. 响应透明返回给用户
```

**关键点**：
- **代理层不处理推理**：llama-swap 只负责路由、进程管理和请求转发
- **真正的处理在底层服务**：llama-server、sd-server 等进程负责实际推理
- **无缝衔接**：通过 HTTP 反向代理实现透明的请求转发

### 9.2 Docker 配置详解

| 文件 | 功能 |
|------|------|
| `docker/config.example.yaml` | 模型配置示例，定义 cmd、proxy、checkEndpoint |
| `docker/llama-swap.Containerfile` | 基础容器定义，包含 llama-swap + llama-server |
| `docker/llama-swap-sd.Containerfile` | 扩展容器，添加 stable-diffusion.cpp |
| `docker/build-container.sh` | 多架构构建脚本 (cuda/rocm/cpu/vulkan等) |
| `docker/unified/Dockerfile` | 统一容器，集成 LLM/图像/语音服务 |
| `docker/unified/config.example.yaml` | 统一容器配置示例 |

### 9.3 实现底层推理服务的条件

要实现自己的底层推理服务，需要满足：

1. **HTTP 服务接口**：监听指定端口
2. **OpenAI API 兼容**：实现 `/v1/chat/completions` 等端点
3. **健康检查端点**：`GET /health` 返回 HTTP 200
4. **流式响应支持**（可选）：SSE 格式输出

详细架构文档请参阅：**`docker/ARCHITECTURE.md`**

---

## 十、参考资料

- 项目地址: https://github.com/mostlygeek/llama-swap
- 配置文档: `docs/configuration.md`
- Docker 构建: `docker/`
- 容器安全: `docs/container-security.md`
- 代理架构详解: `docker/ARCHITECTURE.md`
