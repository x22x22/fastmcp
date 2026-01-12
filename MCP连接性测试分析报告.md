# FastMCP 连接性与可用性测试分析报告

## 1. 项目背景

FastMCP 是一个用于构建 MCP (Model Context Protocol) 服务器和客户端的 Python 框架。本报告分析如何基于本项目测试和调试 MCP 的连接性与可用性。

## 2. MCP 连接性测试的核心概念

### 2.1 MCP 协议基础

MCP (Model Context Protocol) 是一个标准化协议，用于向大语言模型提供上下文和工具。连接性测试的目标是验证：

1. **协议握手**：客户端和服务器能否成功建立连接
2. **能力交换**：双方能否正确交换支持的功能
3. **基础通信**：能否发送和接收基本消息
4. **功能可用性**：工具、资源和提示是否可以正常访问

### 2.2 FastMCP 的核心组件

```
服务器端 (FastMCP Server)
├── 工具 (Tools) - 可执行的函数
├── 资源 (Resources) - 数据源
├── 提示 (Prompts) - 模板
└── 上下文 (Context) - 请求上下文

客户端 (Client)
├── 传输层 (Transport) - 通信机制
├── 会话 (Session) - 连接状态
└── 初始化结果 (InitializeResult) - 握手信息
```

## 3. 测试 MCP 可用性的方法

### 3.1 方法一：使用 `ping()` 方法（最简单）

**描述**：发送一个 ping 请求，检查服务器是否响应。

**优点**：
- 最轻量级的测试方法
- 快速验证连接是否活跃
- 不涉及复杂的业务逻辑

**缺点**：
- 只能验证连接存在，无法验证功能是否正常
- 不能检测服务器的具体能力

**代码示例**：
```python
from fastmcp.client import Client
from fastmcp import FastMCP

# 创建服务器
server = FastMCP("TestServer")

@server.tool
def test_tool() -> str:
    return "OK"

# 测试连接
async def test_connectivity():
    async with Client(server) as client:
        is_alive = await client.ping()
        return is_alive  # True 表示连接正常
```

**适用场景**：
- 快速健康检查
- 监控脚本
- 连接保活检测

### 3.2 方法二：使用 `initialize()` 和检查 `initialize_result`（推荐用于握手验证）

**描述**：执行完整的 MCP 初始化握手，获取服务器信息和能力。

**优点**：
- 验证协议握手是否成功
- 获取服务器元信息（名称、版本）
- 了解服务器支持的能力（capabilities）
- 获取服务器指令（instructions）

**缺点**：
- 比 ping 稍重
- 需要完整的协议支持

**代码示例**：
```python
from fastmcp.client import Client

async def test_initialization():
    async with Client(server) as client:
        # 初始化结果在连接时自动获取
        result = client.initialize_result
        
        # 验证服务器信息
        assert result is not None
        assert result.serverInfo is not None
        
        print(f"服务器名称: {result.serverInfo.name}")
        print(f"服务器版本: {result.serverInfo.version}")
        print(f"协议版本: {result.protocolVersion}")
        
        # 检查能力
        if result.capabilities:
            print(f"支持的能力: {result.capabilities}")
        
        # 检查指令
        if result.instructions:
            print(f"服务器指令: {result.instructions}")
        
        return True
```

**适用场景**：
- 验证服务器配置
- 获取服务器元数据
- 检查协议兼容性

### 3.3 方法三：使用 `list_tools()` 检测工具可用性（推荐用于功能验证）

**描述**：列出服务器提供的所有工具，验证功能是否正常暴露。

**优点**：
- 验证服务器的核心功能（工具）是否可用
- 可以检查预期的工具是否都已注册
- 获取工具的详细信息（名称、描述、参数）
- 适合作为功能完整性测试

**缺点**：
- 不能测试工具的实际执行
- 需要服务器支持工具功能

**代码示例**：
```python
from fastmcp.client import Client

async def test_tools_availability():
    async with Client(server) as client:
        # 获取工具列表
        tools = await client.list_tools()
        
        print(f"可用工具数量: {len(tools)}")
        
        # 验证特定工具是否存在
        tool_names = {tool.name for tool in tools}
        expected_tools = {"greet", "add", "calculate"}
        
        missing_tools = expected_tools - tool_names
        if missing_tools:
            print(f"缺失的工具: {missing_tools}")
            return False
        
        # 打印工具详情
        for tool in tools:
            print(f"工具: {tool.name}")
            print(f"  描述: {tool.description}")
            print(f"  参数: {tool.inputSchema}")
        
        return len(tools) > 0
```

**适用场景**：
- 验证服务器功能完整性
- 检查工具注册是否正确
- 功能测试和集成测试

### 3.4 方法四：综合健康检查（最完整）

**描述**：结合多种检测方法，全面验证 MCP 服务器的可用性。

**优点**：
- 最全面的测试覆盖
- 可以诊断具体问题
- 适合生产环境监控

**代码示例**：
```python
from fastmcp.client import Client
import asyncio
from typing import Dict, Any

async def comprehensive_health_check(server) -> Dict[str, Any]:
    """全面的 MCP 健康检查"""
    
    results = {
        "connectivity": False,
        "initialization": False,
        "tools": False,
        "resources": False,
        "prompts": False,
        "errors": []
    }
    
    try:
        async with Client(server) as client:
            # 1. 测试连接性
            try:
                results["connectivity"] = await client.ping()
            except Exception as e:
                results["errors"].append(f"Ping 失败: {e}")
            
            # 2. 测试初始化
            try:
                init_result = client.initialize_result
                if init_result and init_result.serverInfo:
                    results["initialization"] = True
                    results["server_name"] = init_result.serverInfo.name
                    results["server_version"] = init_result.serverInfo.version
            except Exception as e:
                results["errors"].append(f"初始化检查失败: {e}")
            
            # 3. 测试工具列表
            try:
                tools = await client.list_tools()
                results["tools"] = len(tools) > 0
                results["tool_count"] = len(tools)
                results["tool_names"] = [tool.name for tool in tools]
            except Exception as e:
                results["errors"].append(f"工具列表获取失败: {e}")
            
            # 4. 测试资源列表
            try:
                resources = await client.list_resources()
                results["resources"] = len(resources) > 0
                results["resource_count"] = len(resources)
            except Exception as e:
                results["errors"].append(f"资源列表获取失败: {e}")
            
            # 5. 测试提示列表
            try:
                prompts = await client.list_prompts()
                results["prompts"] = len(prompts) > 0
                results["prompt_count"] = len(prompts)
            except Exception as e:
                results["errors"].append(f"提示列表获取失败: {e}")
    
    except Exception as e:
        results["errors"].append(f"连接失败: {e}")
    
    # 计算总体健康状态
    results["overall_health"] = all([
        results["connectivity"],
        results["initialization"],
        results["tools"] or results["resources"] or results["prompts"]
    ])
    
    return results

# 使用示例
async def main():
    from fastmcp import FastMCP
    
    # 创建测试服务器
    server = FastMCP("HealthCheckServer")
    
    @server.tool
    def test_tool() -> str:
        return "OK"
    
    # 执行健康检查
    results = await comprehensive_health_check(server)
    
    print("=" * 50)
    print("MCP 健康检查报告")
    print("=" * 50)
    print(f"总体状态: {'✅ 健康' if results['overall_health'] else '❌ 异常'}")
    print(f"连接性: {'✅' if results['connectivity'] else '❌'}")
    print(f"初始化: {'✅' if results['initialization'] else '❌'}")
    print(f"工具: {'✅' if results['tools'] else '❌'} ({results.get('tool_count', 0)} 个)")
    print(f"资源: {'✅' if results['resources'] else '❌'} ({results.get('resource_count', 0)} 个)")
    print(f"提示: {'✅' if results['prompts'] else '❌'} ({results.get('prompt_count', 0)} 个)")
    
    if results['errors']:
        print("\n错误信息:")
        for error in results['errors']:
            print(f"  - {error}")
    
    if results.get('server_name'):
        print(f"\n服务器信息:")
        print(f"  名称: {results['server_name']}")
        print(f"  版本: {results['server_version']}")
    
    if results.get('tool_names'):
        print(f"\n可用工具:")
        for name in results['tool_names']:
            print(f"  - {name}")

if __name__ == "__main__":
    asyncio.run(main())
```

## 4. 各种传输方式的测试

FastMCP 支持多种传输方式，每种都需要不同的测试方法：

### 4.1 直接内存传输 (FastMCPTransport)

**用途**：测试和开发
**优点**：最快，无网络开销
**示例**：
```python
from fastmcp.client import Client

server = FastMCP("TestServer")

async with Client(server) as client:  # 自动使用 FastMCPTransport
    is_alive = await client.ping()
```

### 4.2 HTTP 传输 (StreamableHttpTransport)

**用途**：生产环境
**优点**：标准 HTTP 协议，易于部署
**示例**：
```python
from fastmcp.client import Client

async with Client("http://localhost:8000") as client:
    is_alive = await client.ping()
```

### 4.3 标准输入输出传输 (StdioTransport)

**用途**：命令行工具
**优点**：简单，无需网络
**示例**：
```python
from fastmcp.client import Client
from pathlib import Path

# Python 服务器
async with Client(Path("server.py")) as client:
    tools = await client.list_tools()

# Node.js 服务器
async with Client("npx @modelcontextprotocol/server-example") as client:
    tools = await client.list_tools()
```

## 5. 推荐的测试策略

### 5.1 快速健康检查

**使用场景**：监控、快速验证
**方法**：`ping()`
**频率**：每 10-30 秒

```python
async def quick_health_check():
    try:
        async with Client(server, timeout=5) as client:
            return await client.ping()
    except Exception:
        return False
```

### 5.2 功能完整性检查

**使用场景**：部署后验证、集成测试
**方法**：`list_tools()` + `initialize_result`
**频率**：每次部署后、每小时

```python
async def functionality_check():
    async with Client(server) as client:
        # 检查初始化
        if not client.initialize_result:
            return False
        
        # 检查工具
        tools = await client.list_tools()
        return len(tools) > 0
```

### 5.3 端到端测试

**使用场景**：完整的功能测试
**方法**：实际调用工具
**频率**：每次代码变更后

```python
async def e2e_test():
    async with Client(server) as client:
        # 调用实际工具
        result = await client.call_tool("test_tool", {})
        return result.is_error is False
```

## 6. 最佳实践建议

### 6.1 分层测试策略

```
第一层：连接性测试 (ping)
    ↓ 失败则停止
第二层：协议握手测试 (initialize)
    ↓ 失败则停止
第三层：功能可用性测试 (list_tools/resources/prompts)
    ↓ 失败则停止
第四层：功能正确性测试 (call_tool)
```

### 6.2 超时设置

```python
# 快速健康检查：5 秒超时
client = Client(server, timeout=5)

# 功能测试：30 秒超时
client = Client(server, timeout=30)

# 初始化超时单独设置
client = Client(server, init_timeout=10)
```

### 6.3 错误处理

```python
from mcp import McpError

async def safe_health_check():
    try:
        async with Client(server, timeout=5) as client:
            return await client.ping()
    except McpError as e:
        print(f"MCP 协议错误: {e}")
        return False
    except TimeoutError:
        print("连接超时")
        return False
    except Exception as e:
        print(f"未知错误: {e}")
        return False
```

## 7. 结论与建议

### 7.1 回答原问题

**"是不是可以用获取工具列表的函数就可以判断 MCP 是否正常？"**

**答案**：**可以，但不够全面。**

`list_tools()` 可以有效判断 MCP 的核心功能是否正常，因为它：
1. ✅ 验证了连接性（必须先连接才能列出工具）
2. ✅ 验证了协议握手（必须先初始化才能列出工具）
3. ✅ 验证了服务器的核心功能（工具是 MCP 的主要功能）

**但建议采用分层测试策略**：
- **快速检查**：使用 `ping()`（最轻量）
- **基础验证**：检查 `initialize_result`（验证握手）
- **功能验证**：使用 `list_tools()`（验证功能）
- **完整测试**：使用综合健康检查（生产环境）

### 7.2 推荐的最小可行测试

如果只能选择一个测试方法，推荐使用 **`list_tools()` + 检查是否返回预期的工具列表**：

```python
async def minimal_health_check(expected_tools: set[str]) -> bool:
    """最小可行的健康检查"""
    try:
        async with Client(server, timeout=10) as client:
            tools = await client.list_tools()
            tool_names = {tool.name for tool in tools}
            return expected_tools.issubset(tool_names)
    except Exception:
        return False
```

这个方法：
- ✅ 简单易用
- ✅ 验证连接性
- ✅ 验证功能可用性
- ✅ 验证预期工具存在
- ✅ 适合大多数场景

### 7.3 生产环境监控建议

```python
import asyncio
from datetime import datetime

async def production_monitor():
    """生产环境监控示例"""
    
    while True:
        timestamp = datetime.now().isoformat()
        
        # 快速健康检查
        is_healthy = await quick_health_check()
        
        if is_healthy:
            print(f"[{timestamp}] ✅ 服务正常")
        else:
            print(f"[{timestamp}] ❌ 服务异常，执行详细检查...")
            
            # 详细检查
            details = await comprehensive_health_check(server)
            print(f"详细状态: {details}")
            
            # 发送告警（示例）
            # send_alert(details)
        
        # 每 30 秒检查一次
        await asyncio.sleep(30)
```

## 8. 附录：完整测试脚本示例

以下是一个完整的、可以直接运行的测试脚本：

```python
#!/usr/bin/env python3
"""
MCP 连接性和可用性测试脚本

使用方法:
    python test_mcp_connectivity.py
"""

import asyncio
from datetime import datetime
from typing import Dict, Any
from fastmcp import FastMCP
from fastmcp.client import Client


def create_test_server() -> FastMCP:
    """创建一个测试服务器"""
    server = FastMCP("TestServer", version="1.0.0")
    
    @server.tool
    def hello(name: str) -> str:
        """打招呼"""
        return f"你好, {name}!"
    
    @server.tool
    def add(a: int, b: int) -> int:
        """加法运算"""
        return a + b
    
    @server.resource(uri="data://users")
    async def get_users() -> str:
        import json
        return json.dumps(["Alice", "Bob", "Charlie"])
    
    return server


async def test_connectivity(server: FastMCP) -> bool:
    """测试 1: 连接性测试"""
    print("\n" + "=" * 50)
    print("测试 1: 连接性测试 (ping)")
    print("=" * 50)
    
    try:
        async with Client(server, timeout=5) as client:
            result = await client.ping()
            print(f"✅ Ping 成功: {result}")
            return result
    except Exception as e:
        print(f"❌ Ping 失败: {e}")
        return False


async def test_initialization(server: FastMCP) -> bool:
    """测试 2: 初始化测试"""
    print("\n" + "=" * 50)
    print("测试 2: 初始化测试")
    print("=" * 50)
    
    try:
        async with Client(server) as client:
            result = client.initialize_result
            
            if result and result.serverInfo:
                print(f"✅ 初始化成功")
                print(f"   服务器名称: {result.serverInfo.name}")
                print(f"   服务器版本: {result.serverInfo.version}")
                print(f"   协议版本: {result.protocolVersion}")
                return True
            else:
                print("❌ 初始化失败: 无法获取服务器信息")
                return False
    except Exception as e:
        print(f"❌ 初始化失败: {e}")
        return False


async def test_tools(server: FastMCP) -> bool:
    """测试 3: 工具列表测试"""
    print("\n" + "=" * 50)
    print("测试 3: 工具列表测试")
    print("=" * 50)
    
    try:
        async with Client(server) as client:
            tools = await client.list_tools()
            
            print(f"✅ 获取工具列表成功")
            print(f"   工具数量: {len(tools)}")
            
            for tool in tools:
                print(f"   - {tool.name}: {tool.description}")
            
            return len(tools) > 0
    except Exception as e:
        print(f"❌ 获取工具列表失败: {e}")
        return False


async def test_resources(server: FastMCP) -> bool:
    """测试 4: 资源列表测试"""
    print("\n" + "=" * 50)
    print("测试 4: 资源列表测试")
    print("=" * 50)
    
    try:
        async with Client(server) as client:
            resources = await client.list_resources()
            
            print(f"✅ 获取资源列表成功")
            print(f"   资源数量: {len(resources)}")
            
            for resource in resources:
                print(f"   - {resource.uri}: {resource.name}")
            
            return True  # 资源可以为空
    except Exception as e:
        print(f"❌ 获取资源列表失败: {e}")
        return False


async def test_tool_execution(server: FastMCP) -> bool:
    """测试 5: 工具执行测试"""
    print("\n" + "=" * 50)
    print("测试 5: 工具执行测试")
    print("=" * 50)
    
    try:
        async with Client(server) as client:
            # 测试 hello 工具
            result1 = await client.call_tool("hello", {"name": "测试者"})
            print(f"✅ 工具 'hello' 执行成功: {result1.data}")
            
            # 测试 add 工具
            result2 = await client.call_tool("add", {"a": 1, "b": 2})
            print(f"✅ 工具 'add' 执行成功: {result2.data}")
            
            return not (result1.is_error or result2.is_error)
    except Exception as e:
        print(f"❌ 工具执行失败: {e}")
        return False


async def comprehensive_test(server: FastMCP) -> Dict[str, Any]:
    """综合测试"""
    print("\n" + "=" * 60)
    print("MCP 连接性和可用性综合测试")
    print("=" * 60)
    print(f"测试时间: {datetime.now().isoformat()}")
    
    results = {
        "connectivity": await test_connectivity(server),
        "initialization": await test_initialization(server),
        "tools": await test_tools(server),
        "resources": await test_resources(server),
        "execution": await test_tool_execution(server),
    }
    
    # 总结
    print("\n" + "=" * 60)
    print("测试结果总结")
    print("=" * 60)
    
    all_passed = all(results.values())
    
    for test_name, passed in results.items():
        status = "✅ 通过" if passed else "❌ 失败"
        print(f"{test_name:20s}: {status}")
    
    print("\n" + "=" * 60)
    if all_passed:
        print("🎉 所有测试通过！MCP 服务器运行正常。")
    else:
        print("⚠️  部分测试失败，请检查服务器配置。")
    print("=" * 60)
    
    return results


async def main():
    """主函数"""
    # 创建测试服务器
    server = create_test_server()
    
    # 运行综合测试
    results = await comprehensive_test(server)
    
    # 返回测试结果
    return results


if __name__ == "__main__":
    results = asyncio.run(main())
    
    # 根据测试结果设置退出码
    exit_code = 0 if all(results.values()) else 1
    exit(exit_code)
```

---

**报告生成时间**: {datetime.now().isoformat()}  
**FastMCP 版本**: v2.0  
**作者**: FastMCP 项目分析
