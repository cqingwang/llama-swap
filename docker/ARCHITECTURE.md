# llama-swap 服务代理架构详解

> 本文档详细说明 llama-swap 如何代理并转发推理服务请求，以及如何实现底层服务的无缝衔接。

---

## 一、整体架构概览

### 1.1 核心设计理念

llama-swap 采用 **反向代理 + 进程管理** 的架构模式，核心思想是：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           llama-swap (代理层)                            │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        ProxyManager (核心)                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐    │  │
│  │  │ Gin Engine  │  │   Process   │  │    httputil.ReverseProxy│    │  │
│  │  │  (HTTP路由) │  │  (进程管理) │  │      (请求转发)         │    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                  │                                       │
│                    ┌─────────────┴─────────────┐                        │
│                    │        配置系统           │                        │
│                    │  config.yaml + 宏替换     │                        │
│                    └───────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼ HTTP 反向代理
         ┌─────────────────────────────────────────────────────┐
         │              Upstream Inference Servers              │
         │  ┌─────────────────┐  ┌─────────────────┐            │
         │  │  llama-server   │  │    sd-server    │  ...       │
         │  │  (LLM 推理)     │  │   (图像生成)    │            │
         │  │  端口: ${PORT}  │  │  端口: ${PORT}  │            │
         │  └─────────────────┘  └─────────────────┘            │
         └─────────────────────────────────────────────────────┘
```

**关键点**：
1. **代理层不处理推理**：llama-swap 只负责路由、进程管理和请求转发
2. **真正的处理在底层服务**：llama-server、sd-server 等进程负责实际推理
3. **无缝衔接**：通过 HTTP 反向代理实现透明的请求转发

---

## 二、请求转发机制详解

### 2.1 请求处理流程

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. Gin Engine 接收请求                                                  │
│    - 路由匹配 (如 /v1/chat/completions)                                 │
│    - API Key 认证                                                       │
│    - 请求体解析                                                         │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 提取模型名称                                                         │
│    - 从 JSON body 中提取 "model" 字段                                   │
│    - 查找模型配置 (支持别名)                                            │
│    - 确定目标 ProcessGroup                                              │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 进程状态检查与切换                                                   │
│    - 检查当前运行的进程是否匹配                                         │
│    - 不匹配则：停止旧进程 → 启动新进程 → 等待就绪                       │
│    - 匹配则：直接使用现有进程                                           │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. 请求参数处理                                                         │
│    - useModelName: 替换模型名称                                         │
│    - stripParams: 删除指定参数                                          │
│    - setParams: 设置/覆盖参数                                           │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. httputil.ReverseProxy 转发                                          │
│    - 将请求转发到上游服务 (如 http://127.0.0.1:9999)                    │
│    - 流式响应自动处理 (SSE)                                             │
│    - 响应头自动处理 (X-Accel-Buffering: no)                             │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
上游服务处理请求并返回响应
```

### 2.2 核心代码路径

**proxymanager.go** 中的关键函数：

```go
// mkProxyJSONHandler 创建 JSON 请求的代理处理器
func (pm *ProxyManager) mkProxyJSONHandler(cf captureFields) func(*gin.Context) {
    return func(c *gin.Context) {
        // 1. 读取请求体
        bodyBytes, _ := io.ReadAll(c.Request.Body)
        
        // 2. 提取模型名称
        requestedModel := gjson.GetBytes(bodyBytes, "model").String()
        
        // 3. 查找模型配置
        modelID, found := pm.config.RealModelName(requestedModel)
        
        // 4. 获取或启动进程
        if pm.matrix != nil {
            nextHandler = pm.matrix.ProxyRequest
        } else {
            processGroup, _ := pm.swapProcessGroup(modelID)
            nextHandler = processGroup.ProxyRequest
        }
        
        // 5. 处理参数过滤
        // - useModelName 替换
        // - stripParams 删除
        // - setParams 设置
        
        // 6. 调用 Process.ProxyRequest 转发请求
        nextHandler(modelID, c.Writer, c.Request)
    }
}
```

### 2.3 进程启动与健康检查

**process.go** 中的进程管理：

```go
// start 启动上游进程
func (p *Process) start() error {
    // 1. 状态检查：必须是 StateStopped 才能启动
    if curState, err := p.swapState(StateStopped, StateStarting); err != nil {
        return err
    }
    
    // 2. 执行启动命令
    args, _ := p.config.SanitizedCommand()
    p.cmd = exec.CommandContext(cmdContext, args[0], args[1:]...)
    p.cmd.Start()
    
    // 3. 健康检查循环
    for {
        if err := p.checkHealthEndpoint(healthURL); err == nil {
            break  // 健康检查通过
        }
        <-time.After(5 * time.Second)  // 等待后重试
    }
    
    // 4. 状态更新为 Ready
    p.swapState(StateStarting, StateReady)
}
```

---

## 三、响应处理机制

### 3.1 流式响应 (SSE) 处理

llama-swap 对流式响应进行透明转发，关键配置在 `process.go`：

```go
// 创建反向代理
reverseProxy := httputil.NewSingleHostReverseProxy(proxyURL)

// 修改响应：禁用 nginx 缓冲
reverseProxy.ModifyResponse = func(resp *http.Response) error {
    if strings.Contains(resp.Header.Get("Content-Type"), "text/event-stream") {
        resp.Header.Set("X-Accel-Buffering", "no")  // 禁用 nginx 缓冲
    }
    return nil
}
```

### 3.2 加载状态反馈

当模型需要切换时，llama-swap 可以向客户端发送加载状态：

```go
// statusResponseWriter 实现 SSE 加载状态反馈
func (s *statusResponseWriter) statusUpdates(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            if s.process.CurrentState() == StateReady {
                return
            }
            s.sendData(".")  // 发送进度点
        }
    }
}
```

---

## 四、底层服务承载实现

### 4.1 支持的推理服务类型

| 服务类型 | 可执行文件 | 用途 | 健康检查端点 |
|----------|------------|------|-------------|
| llama-server | `llama-server` | LLM 推理 | `/health` |
| sd-server | `sd-server` | 图像生成 | `/` |
| whisper-server | `whisper-server` | 语音识别 | `/` |
| ik-llama-server | `ik-llama-server` | 优化版 LLM | `/health` |

### 4.2 配置示例解析

```yaml
models:
  "qwen2.5":
    proxy: "http://127.0.0.1:9999"       # 上游服务地址
    cmd: >                                # 启动命令
      /app/llama-server
      -hf bartowski/Qwen2.5-0.5B-Instruct-GGUF:Q4_K_M
      --port 9999                         # 必须与 proxy 端口一致
    checkEndpoint: /health                # 健康检查路径
    ttl: 300                              # 闲置 300 秒后自动卸载
    aliases: [gpt-4o-mini, gpt-3.5-turbo] # 模型别名
```

**关键配置说明**：

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `cmd` | 启动命令，`${PORT}` 宏自动替换 | 必填 |
| `proxy` | 上游服务地址，`${PORT}` 宏自动替换 | 必填 |
| `checkEndpoint` | 健康检查路径 | `/health` |
| `ttl` | 闲置超时（秒），超时自动卸载 | 不卸载 |
| `aliases` | 模型别名列表 | 无 |
| `useModelName` | 发送到上游的实际模型名 | 配置的模型名 |

### 4.3 实现自定义底层服务

要实现自己的底层推理服务，需要满足以下条件：

#### 条件 1：HTTP 服务接口

底层服务必须是 HTTP 服务，监听指定端口：

```bash
# 示例：自定义推理服务
my-inference-server --port 9999 --model /path/to/model
```

#### 条件 2：OpenAI API 兼容

底层服务应实现 OpenAI API 格式的端点：

| 端点 | 功能 | 请求格式 |
|------|------|---------|
| `POST /v1/chat/completions` | 聊天补全 | OpenAI ChatCompletion |
| `POST /v1/completions` | 文本补全 | OpenAI Completion |
| `POST /v1/embeddings` | 向量嵌入 | OpenAI Embedding |
| `GET /health` | 健康检查 | 返回 200 OK |

#### 条件 3：健康检查端点

底层服务需要提供健康检查端点，返回 HTTP 200：

```bash
# 健康检查示例
curl http://localhost:9999/health
# 响应: 200 OK
```

#### 条件 4：流式响应支持（可选）

如果支持流式输出，需要实现 SSE (Server-Sent Events)：

```
data: {"choices":[{"delta":{"content":"Hello"}}]}

data: {"choices":[{"delta":{"content":" world"}}]}

data: [DONE]
```

---

## 五、Docker 部署架构

### 5.1 统一容器镜像 (unified/Dockerfile)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    llama-swap Unified Container                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     /usr/local/bin/                              │   │
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐          │   │
│  │  │ llama-server  │ │  sd-server    │ │whisper-server │          │   │
│  │  └───────────────┘ └───────────────┘ └───────────────┘          │   │
│  │  ┌───────────────┐ ┌───────────────┐                             │   │
│  │  │  llama-cli   │ │   sd-cli      │  ...                        │   │
│  │  └───────────────┘ └───────────────┘                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     /usr/local/bin/                              │   │
│  │  ┌───────────────────────────────────────────────────────────┐  │   │
│  │  │                    llama-swap (主进程)                     │  │   │
│  │  └───────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        /models/                                  │   │
│  │  模型文件存储目录 (通过 Volume 挂载)                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  /etc/llama-swap/config/                         │   │
│  │  config.yaml - 配置文件                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 容器启动流程

```bash
# 容器入口点
ENTRYPOINT ["llama-swap"]
CMD ["-config", "/etc/llama-swap/config/config.yaml", "-listen", "0.0.0.0:8080"]
```

启动流程：
1. **llama-swap 主进程启动**：监听 8080 端口
2. **等待请求到达**：初始无推理进程运行
3. **请求到达时**：根据配置启动对应的推理服务
4. **推理服务处理**：实际的模型加载和推理
5. **响应返回**：通过 llama-swap 代理返回给客户端

### 5.3 构建自定义容器

```bash
# CUDA 版本
docker buildx build --build-arg BACKEND=cuda -t llama-swap:cuda .

# Vulkan 版本
docker buildx build --build-arg BACKEND=vulkan -t llama-swap:vulkan .

# 指定 CUDA 架构
docker buildx build --build-arg BACKEND=cuda \
    --build-arg CMAKE_CUDA_ARCHITECTURES="86;89" \
    -t llama-swap:cuda-custom .
```

---

## 六、进程组与模型切换

### 6.1 进程组 (ProcessGroup) 概念

进程组用于管理一组模型的切换行为：

```yaml
groups:
  "default":
    swap: true        # 是否允许切换
    exclusive: true   # 是否互斥（同一时间只运行一个）
    persistent: false # 是否持久运行（不自动卸载）
    members: ["model1", "model2"]
```

### 6.2 模型切换流程

```
请求 modelB (当前运行 modelA)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ ProcessGroup.ProxyRequest("modelB")                                     │
│                                                                         │
│  1. 检查 swap=true                                                      │
│  2. 检查 lastUsedProcess != "modelB"                                    │
│  3. 等待 in-flight 请求完成                                             │
│  4. 停止 modelA 进程                                                    │
│     └─ Process.Stop()                                                   │
│        └─ 等待 in-flight 请求                                           │
│        └─ 发送 SIGTERM                                                  │
│        └─ 等待进程退出                                                  │
│  5. 调用 modelB.ProxyRequest()                                          │
│     └─ Process.start()                                                  │
│        └─ 执行 cmd 命令                                                 │
│        └─ 健康检查循环                                                  │
│        └─ 状态更新为 Ready                                              │
│     └─ httputil.ReverseProxy.ServeHTTP()                                │
│  6. 更新 lastUsedProcess = "modelB"                                     │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
请求转发到 modelB 的上游服务
```

---

## 七、实现指南：如何添加自定义推理服务

### 7.1 步骤概览

1. **开发推理服务**：实现 OpenAI API 兼容的 HTTP 服务
2. **配置模型**：在 config.yaml 中添加模型配置
3. **部署运行**：启动 llama-swap 并测试

### 7.2 示例：添加自定义 Python 推理服务

**推理服务代码** (`my_server.py`)：

```python
from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn

app = FastAPI()

class ChatRequest(BaseModel):
    model: str
    messages: list
    stream: bool = False

@app.post("/v1/chat/completions")
async def chat_completions(request: ChatRequest):
    # 实现推理逻辑
    return {
        "id": "chatcmpl-123",
        "object": "chat.completion",
        "choices": [{
            "message": {"role": "assistant", "content": "Hello!"},
            "finish_reason": "stop"
        }]
    }

@app.get("/health")
async def health():
    return {"status": "ok"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=9999)
```

**配置文件** (`config.yaml`)：

```yaml
models:
  "my-model":
    cmd: "python /app/my_server.py"
    proxy: "http://127.0.0.1:9999"
    checkEndpoint: /health
    aliases: ["custom-llm"]
```

### 7.3 测试验证

```bash
# 启动 llama-swap
llama-swap -config config.yaml

# 测试请求
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "my-model", "messages": [{"role": "user", "content": "Hi"}]}'
```

---

## 八、监控与调试

### 8.1 日志级别

```yaml
logLevel: "debug"  # debug, info, warn, error
logToStdout: "both"  # none, proxy, upstream, both
```

### 8.2 请求监控

```bash
# 查看运行中的模型
curl http://localhost:8080/running

# 查看可用模型
curl http://localhost:8080/v1/models

# 卸载所有模型
curl http://localhost:8080/unload
```

### 8.3 性能指标

```bash
# Prometheus 指标
curl http://localhost:8080/metrics
```

---

## 九、常见问题

### Q1: 为什么选择进程模式而不是嵌入式 FFI？

| 方面 | 进程模式 | 嵌入式 FFI |
|------|---------|-----------|
| 稳定性 | 进程隔离，崩溃不影响主进程 | 崩溃可能导致主进程崩溃 |
| 内存管理 | 进程退出即释放 | 需要手动管理，易泄漏 |
| 多语言支持 | 任意语言实现推理服务 | 需要绑定 |
| 跨平台编译 | 无 CGO 依赖，易于编译 | CGO 依赖，编译复杂 |
| 模型切换 | 进程级切换，彻底释放内存 | 需要卸载/重新加载 |

### Q2: 如何实现零停机切换？

1. **预热模式**：使用 Matrix DSL 实现并发模型运行
2. **负载均衡**：使用 PeerProxy 代理到多个 llama-swap 实例
3. **持久化进程**：设置 `persistent: true` 保持进程运行

### Q3: 如何调试上游服务问题？

1. 查看上游日志：`/logs/stream/upstream`
2. 直接访问上游：`/upstream/{model_id}/`
3. 检查健康状态：`/running`

---

## 十、Docker 容器二进制打包详解

### 10.1 多阶段构建策略

llama-swap 采用 **多阶段构建 (Multi-stage Build)** 策略，将编译环境和运行环境分离：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         构建阶段 (builder-base)                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  nvidia/cuda:12.9.1-devel-ubuntu24.04 (CUDA)                     │  │
│  │  或 ubuntu:24.04 (Vulkan)                                        │  │
│  │                                                                   │  │
│  │  安装：build-essential, cmake, git, ccache, python3              │  │
│  │                                                                   │  │
│  │  执行：install-llama.sh → /install/bin/llama-server              │  │
│  │        install-sd.sh → /install/bin/sd-server                    │  │
│  │        install-whisper.sh → /install/bin/whisper-server          │  │
│  │        install-llama-swap.sh → /install/bin/llama-swap           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ COPY --from=build-stage
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         运行阶段 (runtime)                               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  nvidia/cuda:12.9.1-runtime-ubuntu24.04 (CUDA)                   │  │
│  │  或 ubuntu:24.04 (Vulkan)                                        │  │
│  │                                                                   │  │
│  │  /usr/local/bin/llama-server                                     │  │
│  │  /usr/local/bin/sd-server                                        │  │
│  │  /usr/local/bin/whisper-server                                   │  │
│  │  /usr/local/bin/llama-swap  ← 主进程 (ENTRYPOINT)                │  │
│  │                                                                   │  │
│  │  /etc/llama-swap/config/config.yaml                              │  │
│  │  /models/  ← 模型文件挂载目录                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 安装脚本分析

**install-llama.sh** - 编译 llama.cpp：

```bash
# 1. 克隆源码
git clone https://github.com/ggml-org/llama.cpp.git

# 2. CMake 编译 (CUDA 后端)
cmake -B build \
    -DGGML_CUDA=ON \
    -DGGML_NATIVE=OFF \
    -DCMAKE_BUILD_TYPE=Release

# 3. 构建目标二进制
cmake --build build --target llama-cli llama-server

# 4. 复制到安装目录
cp build/bin/llama-server /install/bin/
```

**install-llama-swap.sh** - 下载预编译二进制：

```bash
# 从 GitHub Release 下载
VERSION=$(curl -s https://api.github.com/repos/mostlygeek/llama-swap/releases/latest | jq -r .tag_name)
URL="https://github.com/mostlygeek/llama-swap/releases/download/${VERSION}/llama-swap_${VERSION}_linux_amd64.tar.gz"
curl -fSL -o /tmp/llama-swap.tar.gz "$URL"
tar -xzf /tmp/llama-swap.tar.gz -C /install/bin/
```

### 10.3 最终镜像内容

| 路径 | 内容 | 来源 |
|------|------|------|
| `/usr/local/bin/llama-swap` | 代理主程序 | GitHub Release |
| `/usr/local/bin/llama-server` | LLM 推理服务 | 源码编译 |
| `/usr/local/bin/sd-server` | 图像生成服务 | 源码编译 |
| `/usr/local/bin/whisper-server` | 语音识别服务 | 源码编译 |
| `/etc/llama-swap/config/config.yaml` | 配置文件 | 复制 |
| `/models/` | 模型文件目录 | Volume 挂载 |

---

## 十一、实现类似功能的完整指南

### 11.1 准备工作清单

#### 阶段一：推理服务准备

| 步骤 | 内容 | 说明 |
|------|------|------|
| 1.1 | 选择或开发推理服务 | llama-server、sd-server、自定义服务 |
| 1.2 | 确保支持 HTTP API | 监听端口，处理请求 |
| 1.3 | 实现健康检查端点 | `/health` 返回 200 OK |
| 1.4 | 支持 OpenAI API 格式 | `/v1/chat/completions` 等 |
| 1.5 | 编译或下载二进制 | 静态链接优先，减少依赖 |

#### 阶段二：配置系统设计

| 步骤 | 内容 | 说明 |
|------|------|------|
| 2.1 | 设计 YAML 配置格式 | models、groups、macros |
| 2.2 | 实现宏替换系统 | `${PORT}`, `${MODEL_ID}` |
| 2.3 | 实现模型别名 | aliases 映射 |
| 2.4 | 实现参数过滤 | stripParams, setParams |

#### 阶段三：进程管理实现

| 步骤 | 内容 | 说明 |
|------|------|------|
| 3.1 | 实现进程状态机 | stopped → starting → ready → stopping |
| 3.2 | 实现健康检查循环 | HTTP GET 直到返回 200 |
| 3.3 | 实现进程停止策略 | SIGTERM 优雅停止，超时 SIGKILL |
| 3.4 | 实现 TTL 自动卸载 | 闲置超时自动停止 |

#### 阶段四：反向代理实现

| 步骤 | 内容 | 说明 |
|------|------|------|
| 4.1 | 使用 httputil.ReverseProxy | 标准库反向代理 |
| 4.2 | 处理流式响应 | SSE 支持，禁用缓冲 |
| 4.3 | 实现请求修改 | useModelName, 参数过滤 |
| 4.4 | 实现并发控制 | 信号量限制并发数 |

### 11.2 核心代码示例

#### 进程启动 (pkg/proxy/process.go)

```go
func (p *Process) start() error {
    // 1. 状态检查
    if _, err := p.swapState(StateStopped, StateStarting); err != nil {
        return err
    }
    
    // 2. 解析启动命令
    args, _ := p.config.SanitizedCommand()  // 如: ["llama-server", "--port", "9999"]
    
    // 3. 创建进程上下文
    cmdContext, cancel := context.WithCancel(context.Background())
    p.cmd = exec.CommandContext(cmdContext, args[0], args[1:]...)
    p.cmd.Stdout = p.processLogger  // 捕获标准输出
    p.cmd.Stderr = p.processLogger  // 捕获标准错误
    p.cmd.Env = append(p.cmd.Environ(), p.config.Env...)
    
    // 4. 启动进程
    if err := p.cmd.Start(); err != nil {
        return fmt.Errorf("start failed: %v", err)
    }
    
    // 5. 健康检查循环
    healthURL := fmt.Sprintf("%s%s", p.config.Proxy, p.config.CheckEndpoint)
    for {
        if err := p.checkHealthEndpoint(healthURL); err == nil {
            break  // 健康检查通过
        }
        <-time.After(5 * time.Second)
    }
    
    // 6. 状态更新
    p.swapState(StateStarting, StateReady)
    return nil
}
```

#### 请求代理 (pkg/proxy/process.go)

```go
func (p *Process) ProxyRequest(w http.ResponseWriter, r *http.Request) {
    // 1. 自动启动进程
    if p.CurrentState() != StateReady {
        if err := p.start(); err != nil {
            http.Error(w, err.Error(), http.StatusBadGateway)
            return
        }
    }
    
    // 2. 并发控制
    select {
    case p.concurrencyLimitSemaphore <- struct{}{}:
        defer func() { <-p.concurrencyLimitSemaphore }()
    default:
        http.Error(w, "Too many requests", http.StatusTooManyRequests)
        return
    }
    
    // 3. 转发请求
    p.reverseProxy.ServeHTTP(w, r)  // httputil.ReverseProxy
}
```

#### 反向代理创建 (pkg/proxy/process.go)

```go
func NewProcess(config ModelConfig) *Process {
    // 解析代理目标 URL
    proxyURL, _ := url.Parse(config.Proxy)  // 如: http://127.0.0.1:9999
    
    // 创建反向代理
    reverseProxy := httputil.NewSingleHostReverseProxy(proxyURL)
    
    // 自定义传输配置
    reverseProxy.Transport = &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   30 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        ResponseHeaderTimeout: 60 * time.Second,
    }
    
    // 修改响应：禁用 nginx 缓冲 (支持 SSE)
    reverseProxy.ModifyResponse = func(resp *http.Response) error {
        if strings.Contains(resp.Header.Get("Content-Type"), "text/event-stream") {
            resp.Header.Set("X-Accel-Buffering", "no")
        }
        return nil
    }
    
    return &Process{
        reverseProxy: reverseProxy,
        config:       config,
        state:        StateStopped,
    }
}
```

### 11.3 配置示例

```yaml
# 最小配置示例
healthCheckTimeout: 300  # 健康检查超时（秒）

models:
  "llama-7b":
    # 启动命令：${PORT} 宏自动替换为分配的端口
    cmd: >
      llama-server
      -m /models/llama-7b.gguf
      --port ${PORT}
      --ctx-size 4096
    
    # 代理目标地址
    proxy: "http://127.0.0.1:${PORT}"
    
    # 健康检查端点
    checkEndpoint: /health
    
    # 闲置 5 分钟后自动卸载
    ttl: 300
    
    # 模型别名
    aliases: [gpt-4o-mini, gpt-3.5-turbo]

# 进程组配置
groups:
  "default":
    swap: true      # 允许模型切换
    exclusive: true # 同一时间只运行一个模型
    members: ["llama-7b"]
```

### 11.4 Docker 部署示例

```dockerfile
# Dockerfile
FROM nvidia/cuda:12.9.1-runtime-ubuntu24.04

# 安装运行时依赖
RUN apt-get update && apt-get install -y curl ca-certificates

# 复制推理服务二进制
COPY llama-server /usr/local/bin/
COPY llama-swap /usr/local/bin/

# 复制配置文件
COPY config.yaml /etc/llama-swap/config.yaml

# 创建模型目录
RUN mkdir -p /models

# 设置入口点
WORKDIR /models
ENTRYPOINT ["llama-swap"]
CMD ["-config", "/etc/llama-swap/config.yaml", "-listen", "0.0.0.0:8080"]
```

```bash
# 构建镜像
docker build -t my-llama-swap .

# 运行容器
docker run -d \
    --gpus all \
    -p 8080:8080 \
    -v /path/to/models:/models \
    my-llama-swap

# 测试请求
curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "llama-7b", "messages": [{"role": "user", "content": "Hello"}]}'
```

已完成 llama-swap Docker 代理架构的深度分析，并创建了详细的架构文档。

## 分析结论

### Docker 容器二进制打包方式

**是的，llama-swap 把所有推理服务的二进制都预打包进了容器中**：

| 安装脚本 | 编译内容 | 安装目标 |
|----------|---------|---------|
| `install-llama.sh` | 从源码编译 llama.cpp → `llama-server`, `llama-cli` | `/install/bin/` |
| `install-sd.sh` | 从源码编译 stable-diffusion.cpp → `sd-server`, `sd-cli` | `/install/bin/` |
| `install-whisper.sh` | 从源码编译 whisper.cpp → `whisper-server`, `whisper-cli` | `/install/bin/` |
| `install-llama-swap.sh` | 从 GitHub Release 下载预编译 `llama-swap` | `/install/bin/` |

最终在统一 Dockerfile（`unified/Dockerfile`）中通过多阶段构建，把所有二进制 `COPY --from=xxx /install/bin/` 到运行时镜像的 `/usr/local/bin/`。

### 推理服务启动的 Go 代码核心机制

`pkg/proxy/process.go` 中的关键代码：

1. **进程启动**：`exec.CommandContext()` 执行配置中的 `cmd` 命令
2. **健康检查**：循环 HTTP GET 请求 `proxy + checkEndpoint`，直到返回 200
3. **反向代理**：`httputil.NewSingleHostReverseProxy(proxyURL)` 转发请求
4. **流式支持**：`ModifyResponse` 中对 SSE 响应设置 `X-Accel-Buffering: no`
5. **进程状态机**：`stopped → starting → ready → stopping → shutdown`

### 实现类似功能需要做的准备

详见 `doc/lswap/docker/ARCHITECTURE.md` 第十、十一章，核心要点：

1. **推理服务需实现**：HTTP 服务 + 健康检查端点 + OpenAI API 兼容
2. **容器打包**：多阶段构建，编译推理二进制 + 下载 llama-swap 二进制
3. **配置关联**：`cmd` 中的 `--port ${PORT}` 必须与 `proxy` 中的端口一致
4. **核心代码**：基于 Go 的 `exec.CommandContext` + `httputil.ReverseProxy` 实现
---

*文档版本: v1.1*
*最后更新: 2026-05-17*
