# n8n Python Runner 配置踩坑指南：从 Internal Mode 到 External Mode 完整部署

> 解决 "Python runner unavailable" 错误，成功在 Docker 环境中运行 Python Code 节点的完整实战经验

## 前言

最近在使用 n8n 进行工作流自动化时，遇到了一个常见但棘手的问题：Python Code 节点无法运行，提示 "Python runner unavailable"。经过一番调研和折腾，终于成功解决了这个问题。本文将完整记录从问题发现到解决的整个过程，帮助其他开发者避免踩同样的坑。

## 问题背景

在本地 Docker 环境中部署的 n8n (版本 2.6.3)，尝试在 Code 节点中使用 Python 时报错：

```
Python runner unavailable: Python 3 is missing from this system 
Internal mode is intended only for debugging. 
For production, deploy in external mode: 
https://docs.n8n.io/hosting/configuration/task-runners/#setting-up-external-mode
```

简单的 Python 测试代码都无法执行：

```python
# 定义一个带return的简单函数，计算两数之和
def add_numbers(a, b):
    # 计算a和b的和
    result = a + b
    # 返回计算结果
    return result

return add_numbers(5, 3)
```

## 问题分析

### 什么是 Internal Mode 和 External Mode？

n8n 的 Code 节点支持多种语言执行模式：

- **Internal Mode**: 在 n8n 主进程内直接执行代码，仅用于调试，不适合生产环境
- **External Mode**: 使用独立的 runner 容器来执行代码，更安全、更稳定

### 为什么需要 External Mode？

1. **安全性**: 隔离执行环境，防止恶意代码影响主系统
2. **资源管理**: 独立的资源限制和监控
3. **扩展性**: 可以添加额外的依赖库
4. **稳定性**: 代码执行不会影响 n8n 主服务

## 解决方案

### 1. Docker Compose 配置

最初的问题配置：
```yaml
# 错误配置 - 缺少必要的环境变量
n8n:
  image: docker.n8n.io/n8nio/n8n:2.6.3
  environment:
    - N8N_RUNNERS_MODE=external
    - N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
    - N8N_RUNNERS_AUTH_TOKEN=n8n-secret-token
    - N8N_NATIVE_PYTHON_RUNNER=true
```

正确的配置：
```yaml
version: "3.9"

services:
  # n8n 主服务
  n8n:
    image: docker.n8n.io/n8nio/n8n:2.6.3
    container_name: n8n-main
    ports:
      - "5678:5678"
    environment:
      - TZ=Asia/Shanghai
      - GENERIC_TIMEZONE=Asia/Shanghai
      - N8N_RUNNERS_ENABLED=true  # 关键配置：启用 task runners
      - N8N_RUNNERS_MODE=external
      - N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
      - N8N_RUNNERS_AUTH_TOKEN=n8n-secret-token
      - N8N_NATIVE_PYTHON_RUNNER=true
      - N8N_LOG_LEVEL=debug  # 调试时启用
    volumes:
      - n8n_data:/home/node/.n8n

  # Task Runners
  n8n-runners:
    image: n8nio/runners:2.6.3  # 版本必须与 n8n 匹配
    container_name: n8n-runners
    environment:
      - N8N_RUNNERS_AUTH_TOKEN=n8n-secret-token
      - N8N_RUNNERS_TASK_BROKER_URI=http://n8n-main:5679
      - N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT=0  # 防止空闲超时
    depends_on:
      - n8n

volumes:
  n8n_data:
    external: true
```

### 2. 关键配置说明

#### 必须添加的环境变量
- `N8N_RUNNERS_ENABLED=true`: 启用 task runners 功能
- `N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT=0`: 防止 runner 在空闲时自动关闭

#### 网络配置
- n8n 主服务监听 5678 端口 (Web UI)
- task broker 监听 5679 端口 (内部通信)
- 版本匹配：n8n 和 runners 必须使用相同版本号

### 3. 部署步骤

```bash
# 停止现有服务
docker-compose down

# 启动服务
docker-compose up -d

# 检查服务状态
docker-compose ps

# 查看日志确认 runner 注册成功
docker-compose logs n8n | grep "Registered runner"
```

期望的日志输出：
```
Registered runner "launcher-python" (fe21626a16259df7) 
Registered runner "launcher-javascript" (0034d50f39a1f555) 
```

## 踩坑记录

### 坑1：缺少 N8N_RUNNERS_ENABLED
**症状**: runner 服务正常启动，但 n8n 无法连接
**解决**: 添加 `N8N_RUNNERS_ENABLED=true`

### 坑2：版本不匹配
**症状**: runner 无法注册到 n8n
**解决**: 确保 n8n 和 runners 使用相同版本号

### 坑3：空闲超时
**症状**: runner 初始连接正常，但使用时消失
**解决**: 设置 `N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT=0`

### 坑4：执行节点无限转圈
**原因**: 单独执行 Code 节点，没有触发器
**解决**: 添加 Manual Trigger 或使用完整 workflow

## Python Code 节点最佳实践

### 返回格式要求

n8n 的 Python Code 节点需要返回字典数组格式：

```python
# ❌ 错误格式
def add_numbers(a, b):
    return a + b

return add_numbers(5, 3)

# ✅ 正确格式
def add_numbers(a, b):
    return a + b

return [{
    "result": add_numbers(5, 3),
    "timestamp": "2024-01-01",
    "status": "success"
}]
```

### 调试技巧

1. **启用调试日志**: 设置 `N8N_LOG_LEVEL=debug`
2. **监控 runner 日志**: `docker-compose logs -f n8n-runners`
3. **使用简单测试**: 从最基础的函数开始测试

### 扩展依赖

如果需要额外的 Python 包，可以创建自定义 runners 镜像：

```dockerfile
FROM n8nio/runners:2.6.3

USER root
RUN cd /opt/runners/task-runner-python && pip install numpy pandas requests
USER runner
```

## 验证测试

### 测试环境
- n8n 版本: 2.6.3
- Docker 环境: macOS Docker Desktop
- 测试代码: 简单的数学函数

### 测试步骤
1. 访问 http://localhost:5678
2. 创建新 workflow
3. 添加 Manual Trigger 节点
4. 添加 Code 节点，选择 Python 语言
5. 使用正确的返回格式
6. 执行 workflow

### 预期结果
```json
[
  {
    "result": 8
  }
]
```

## 总结

通过这次踩坑经验，总结出几个关键要点：

1. **版本一致性**: n8n 和 runners 必须使用相同版本
2. **配置完整性**: 确保 `N8N_RUNNERS_ENABLED=true` 和正确的网络配置
3. **超时设置**: 避免空闲超时导致 runner 离线
4. **返回格式**: Python Code 必须返回字典数组格式
5. **调试方法**: 善用日志来排查连接问题

External Mode 虽然配置复杂一些，但提供了更好的安全性和稳定性，是生产环境的推荐选择。

## 参考资源

- [n8n 官方文档 - Task Runners](https://docs.n8n.io/hosting/configuration/task-runners/)
- [n8n GitHub Issues](https://github.com/n8n-io/n8n/issues)
- [n8n Community Forum](https://community.n8n.io/)

希望这篇文章能帮助您顺利配置 n8n Python 环境，避免重复踩坑！