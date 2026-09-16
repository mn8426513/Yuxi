# 动态 Agent 沙箱预装 .NET SDK

状态：implemented
类型：feature
Owner：docker/sandbox/Dockerfile

## 问题

`minimax-docx` 在 Agent 动态沙箱内执行，需要 .NET 8 SDK。`sandbox-provisioner` 镜像虽然安装了 .NET，但 provisioner 只创建和管理动态沙箱，Agent 的文件与命令操作发生在独立的 `all-in-one-sandbox` 容器中。基础沙箱镜像没有 `dotnet`，provisioner 的安装目录也不会进入动态沙箱，因此真实 Skill 执行无法使用该工具链。

## 决策

动态沙箱使用由 `docker/sandbox/Dockerfile` 构建的派生镜像。该镜像从 AIO Sandbox `1.11.0` 开始，显式安装 .NET 在 Ubuntu 22.04 上需要的系统库，安装固定版本的 .NET 8 SDK 到 `/opt/dotnet`，并把它加入 `PATH`；同时向默认 `python` 环境安装固定版本的 `python-docx` 与 `matplotlib`。Compose 中的 `sandbox-image` 服务只负责构建和验证该镜像，使用 `scale: 0` 避免创建常驻容器；`sandbox-provisioner` 的 `SANDBOX_IMAGE` 默认指向这一本地位带 SDK 的镜像。

开发和生产 Compose 都保留 `SANDBOX_BASE_IMAGE`、`SANDBOX_DOTNET_VERSION`、`SANDBOX_PYTHON_DOCX_VERSION` 与 `SANDBOX_MATPLOTLIB_VERSION` 构建参数。初始化脚本构建镜像并执行一次 `dotnet --version` 探针；system-tests 先验证镜像，再运行通过 provisioner 创建真实动态沙箱的集成测试。

## 替代方案

| 替代方案 | 取舍 |
| --- | --- |
| 继续只在 provisioner 镜像安装 .NET | provisioner 有 SDK，但 Agent 命令仍在无 .NET 的动态沙箱内执行 |
| 每次创建沙箱时在线安装 .NET | 创建延迟、网络依赖和失败面进入每次 Agent 操作，镜像内容随外部源漂移 |
| 把 provisioner 的 `/opt/dotnet` 作为共享卷挂载到每个沙箱 | 增加 provisioner 与运行容器的生命周期耦合，跨平台和 Kubernetes 路径扩展更复杂 |
| 改用自包含发布产物绕过 SDK | 改变 `minimax-docx` 当前使用方式，并保留与用户明确选择的 .NET 工具链不一致的维护面 |

## 后果

派生镜像比基础镜像增加 .NET SDK 和 Python 包层，首次构建需要下载依赖，基础镜像、SDK 或 Python 包版本变化时也必须重建。已经创建并仍在运行的动态沙箱继续使用启动时的旧镜像，新镜像只在重新创建后生效。当前固定 `8.0.425`、`python-docx 1.2.0` 与 `matplotlib 3.10.9`，升级依赖需要显式修改对应构建参数并重跑镜像与真实沙箱探针。

## 验证

`docker compose build sandbox-image` 已构建最终镜像 `yuxi-sandbox:0.7.3`。`docker compose run --rm --no-deps sandbox-image` 输出 `8.0.425`，镜像环境包含 `/opt/dotnet` 与 `DOTNET_ROOT`。默认 `python` 可以导入 `python-docx 1.2.0` 与 `matplotlib 3.10.9`。

`backend/test/integration/services/test_project_workdir_provisioner.py::test_dynamic_agent_sandbox_exposes_dotnet_sdk` 通过 provisioner 创建真实动态沙箱，执行 `dotnet --version` 并断言返回 .NET 8 版本。`test_dynamic_agent_sandbox_exposes_python_document_toolchain` 使用默认 `python` 导入两个包并断言版本。完整同文件 `7 passed`，覆盖 UserWorkspace、runtime 重建、Skill 投影和权限。恢复为未安装工具的原始镜像时，对应用例会因 `dotnet: not found` 或 `No module named 'docx'` 失败。

相关沙箱后端 unit 集合 `158 passed`。工程信任检查通过，并确认 system-tests 保留构建与镜像探针命令。完整后端 unit 套件在无关用例中长时间未结束，已停止；本次改动未以该未完成结果作为通过证据。
