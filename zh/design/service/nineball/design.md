# Nineball (Init Manager) 设计文档

## 1. 简介
Nineball 是 Glenda 的 **系统初始化管理器 (Second-stage Init)**。它被 `Warren` 启动，并负责根据配置文件协调整个系统服务的生命周期。

## 2. 核心机制

### 2.1 配置文件 (Manifest)
Nineball 从 `initrd` 中读取 `init.json` 配置文件。
*   **服务定义**：名称、二进制路径、参数。
*   **依赖管理**：通过 `dependencies` 字段定义启动顺序。
*   **自动启动**：标记 `auto_start` 的服务会被 Nineball 自动列入引导序列。

### 2.2 启动序列
1.  **初始化**：Nineball 启动后，向 Warren 请求自己的 `ENDPOINT`。
2.  **配置读取**：通过 `ResourceClient::get_config` 从 Warren 获取 `init.json`。
3.  **Bootstrap**：在 `bootstrap()` 循环中，检查已运行的服务及其依赖，逐步 `start_service`。
4.  **服务探测**：Nineball 充当 `INIT_ENDPOINT` 的监听者，服务启动后通常会向 Nineball `REPORT` 其就绪状态。

## 3. 接口协议 (INIT_PROTO)

Nineball 暴露了以下 RPC 方法：
*   **START / STOP / RESTART**：手动管理服务。
*   **QUERY / LIST**：查询服务状态和已注册服务列表。
*   **REPORT**：服务向管理器上报其当前的 `ServiceState` (Starting, Running, Stopped, Failed)。

## 4. 与 Warren 的协作
*   **创建进程**：Nineball 并不直接管理内存，而是通过 `ProcessClient` 发送 `SPAWN` 消息给 Warren。
*   **能力注册**：Nineball 帮助新服务寻找其他服务的端点。例如，`fossil` 启动后将其端点注册到 Nineball，随后启动的应用通过 Nineball 查询 `fossil` 的能力。

## 5. 关键服务编排
Nineball 通常按以下顺序拉起核心服务：
1. `unicorn` (Virtio/驱动代理)
2. `fossil` (文件系统)
3. `gopher` (网络)
4. `prism` (显示) / `login` (登录)
