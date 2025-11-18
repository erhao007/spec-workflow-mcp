# Spec Workflow MCP 中文文档

## 概述

欢迎使用 Spec Workflow MCP 中文文档！本文档集合为 Spec Workflow MCP 项目提供了完整的中文技术文档，涵盖架构设计、使用指南、API参考等各个方面。

## 📚 文档导航

### 🏗️ 架构文档

| 文档名称 | 描述 | 适用读者 |
|---------|------|----------|
| [核心架构分析](./core-architecture.md) | 深度分析核心服务层架构、设计模式和实现细节 | 架构师、高级开发者 |
| [仪表板系统](./dashboard-system.md) | 仪表板后端和前端架构分析 | 前端开发者、全栈开发者 |
| [MCP工具系统](./mcp-tools-reference.md) | MCP工具完整参考文档 | AI开发者、集成开发者 |
| [MCP提示系统](./mcp-prompts-guide.md) | 提示系统使用指南和最佳实践 | AI开发者、提示工程师 |

### 🔧 开发文档

| 文档名称 | 描述 | 适用读者 |
|---------|------|----------|
| [开发环境配置](../DEVELOPMENT.md) | 开发环境搭建和配置指南 | 新手开发者 |
| [用户使用指南](../USER-GUIDE.md) | 完整的用户操作指南 | 所有用户 |
| [故障排除指南](../TROUBLESHOOTING.md) | 常见问题解决方案 | 所有用户 |

### 📋 参考文档

| 文档名称 | 描述 | 适用读者 |
|---------|------|----------|
| [配置参考](../CONFIGURATION.md) | 完整的配置选项说明 | 系统管理员 |
| [工具参考](../TOOLS-REFERENCE.md) | MCP工具快速参考 | 所有开发者 |
| [提示指南](../PROMPTING-GUIDE.md) | 提示工程最佳实践 | AI开发者 |

## 🚀 快速开始

### 环境要求
- Node.js 18+
- npm 或 yarn
- Git

### 安装和启动

```bash
# 克隆项目
git clone https://github.com/Pimzino/spec-workflow-mcp.git
cd spec-workflow-mcp

# 安装依赖
npm install

# 开发模式启动
npm run dev

# 启动仪表板
npm run dev:dashboard
```

### 第一个项目

1. **创建项目**
   ```bash
   mkdir my-project && cd my-project
   echo '# My Project' > README.md
   ```

2. **初始化工作空间**
   ```bash
   spec-workflow init
   ```

3. **打开仪表板**
   - 访问 http://localhost:3000
   - 查看工作流指导
   - 创建第一个规范

## 🏛️ 架构概览

Spec Workflow MCP 采用分层架构设计：

```
┌─────────────────────────────────────────┐
│              用户界面层                    │
│    Web仪表板 │ VSCode扩展 │ 命令行工具      │
├─────────────────────────────────────────┤
│              应用层                        │
│    MCP工具 │ MCP提示 │ 审批工作流           │
├─────────────────────────────────────────┤
│              服务层                        │
│  项目管理 │ 文件监控 │ 任务调度 │ 会话管理    │
├─────────────────────────────────────────┤
│              核心层                        │
│  路径工具 │ 解析器 │ 注册表 │ 存储服务       │
└─────────────────────────────────────────┘
```

### 核心组件

#### 1. 核心服务层 (`src/core/`)
- **PathUtils**: 安全的路径操作和验证
- **WorkspaceInitializer**: 工作空间初始化和模板管理
- **ProjectRegistry**: 全局项目注册和生命周期管理
- **SpecParser**: 规范文档解析和状态管理
- **TaskParser**: 任务解析和进度跟踪

#### 2. MCP工具系统 (`src/tools/`)
- **spec-workflow-guide**: 工作流指导
- **spec-status**: 状态查询
- **approvals**: 审批管理
- **log-implementation**: 实现日志记录

#### 3. MCP提示系统 (`src/prompts/`)
- **create-spec**: 创建规范文档
- **implement-task**: 任务实施指导
- **refresh-tasks**: 智能任务刷新

#### 4. 仪表板系统 (`src/dashboard/`)
- **后端服务**: Fastify + WebSocket 实时通信
- **前端应用**: React + TypeScript + Vite
- **多项目支持**: 同时管理多个开发项目

## 🔄 工作流程

### 标准工作流 (Spec Workflow)

```mermaid
graph LR
    A[需求分析] --> B[设计阶段]
    B --> C[任务分解]
    C --> D[实施开发]
    D --> E[完成归档]

    A --> A1[创建需求文档]
    A1 --> A2[需求审批]
    A2 --> B

    B --> B1[创建设计文档]
    B1 --> B2[设计审批]
    B2 --> C

    C --> C1[创建任务列表]
    C1 --> C2[任务实施]
    C2 --> D

    D --> D1[记录实现日志]
    D1 --> D2[任务完成]
    D2 --> E
```

### 使用MCP工具

```bash
# 1. 获取工作流指导
工具: spec-workflow-guide

# 2. 创建需求文档
提示: create-spec (documentType: requirements)

# 3. 请求审批
工具: approvals (action: request)

# 4. 创建设计文档
提示: create-spec (documentType: design)

# 5. 创建任务列表
提示: create-spec (documentType: tasks)

# 6. 实施具体任务
提示: implement-task (taskId: "1.1")

# 7. 记录实现日志
工具: log-implementation
```

## 💡 核心特性

### 1. 规范驱动开发
- **四阶段工作流**: Requirements → Design → Tasks → Implementation
- **文档模板**: 标准化的文档结构和内容
- **版本控制**: 完整的文档版本和变更历史

### 2. 实时协作
- **WebSocket通信**: 实时的状态更新和事件通知
- **多项目支持**: 同时管理多个开发项目
- **审批工作流**: 完整的文档审批和反馈机制

### 3. AI辅助开发
- **MCP工具**: 标准化的AI工具接口
- **结构化提示**: 角色化、任务化的AI指导
- **实现日志**: 构建可搜索的知识库

### 4. 可视化管理
- **Web仪表板**: 直观的项目状态和进度展示
- **VSCode扩展**: 原生的编辑器集成
- **多语言支持**: 11种语言的国际化

## 🔧 配置选项

### 项目配置 (`.spec-workflow/config.toml`)

```toml
[project]
name = "我的项目"
description = "项目描述"
version = "1.0.0"

[dashboard]
port = 3000
host = "localhost"

[workflow]
auto_approve = false
require_reviews = true
notification_level = "all"

[features]
real_time_updates = true
auto_backup = true
export_formats = ["markdown", "pdf"]
```

### 环境变量

```bash
# 服务器配置
PORT=3000
HOST=localhost
NODE_ENV=development

# 数据库配置（可选）
DATABASE_URL=postgresql://user:password@localhost:5432/spec_workflow
REDIS_URL=redis://localhost:6379

# 安全配置
JWT_SECRET=your-secret-key
CORS_ORIGIN=http://localhost:3000

# 功能开关
ENABLE_ANALYTICS=false
ENABLE_BACKUP=true
```

## 🛠️ 开发指南

### 添加新的MCP工具

1. **创建工具文件**
   ```typescript
   // src/tools/my-tool.ts
   export const myTool: Tool = {
     name: 'my-tool',
     description: '工具描述',
     inputSchema: {
       type: 'object',
       properties: {
         param1: { type: 'string', required: true }
       }
     }
   };
   ```

2. **实现处理器**
   ```typescript
   export async function myToolHandler(args: any, context: ToolContext): Promise<ToolResponse> {
     // 实现工具逻辑
     return {
       success: true,
       message: '操作成功',
       data: result
     };
   }
   ```

3. **注册工具**
   ```typescript
   // src/tools/index.ts
   export function registerTools() {
     return [
       // ... 现有工具
       myTool
     ];
   }
   ```

### 添加新的提示

1. **创建提示文件**
   ```typescript
   // src/prompts/my-prompt.ts
   export const myPrompt: PromptDefinition = {
     name: 'my-prompt',
     description: '提示描述',
     arguments: [
       { name: 'param1', required: true, description: '参数描述' }
     ]
   };
   ```

2. **生成提示内容**
   ```typescript
   export function generateMyPrompt(args: any, context: ToolContext): string {
     return `
       # 提示内容
       参数: ${args.param1}
       项目: ${context.projectPath}
     `;
   }
   ```

### 扩展仪表板功能

1. **添加API端点**
   ```typescript
   // src/dashboard/my-feature.ts
   app.get('/api/my-feature', async (request, reply) => {
     const result = await myFeatureService.getData();
     return result;
   });
   ```

2. **创建前端组件**
   ```typescript
   // src/dashboard_frontend/src/components/MyFeature.tsx
   export const MyFeature: React.FC = () => {
     const [data, setData] = useState([]);

     useEffect(() => {
       fetch('/api/my-feature')
         .then(res => res.json())
         .then(setData);
     }, []);

     return <div>{/* 组件内容 */}</div>;
   };
   ```

## 🧪 测试

### 运行测试

```bash
# 运行所有测试
npm run test

# 运行特定测试文件
npm run test src/tools/__tests__/projectPath.test.ts

# 运行测试并生成覆盖率报告
npm run test:coverage

# 监听模式运行测试
npm run test:watch
```

### 编写测试

```typescript
// src/tools/__tests__/my-tool.test.ts
import { describe, it, expect } from 'vitest';
import { myToolHandler } from '../my-tool';

describe('MyTool', () => {
  it('should handle valid input', async () => {
    const args = { param1: 'test' };
    const context = { projectPath: '/test' };

    const result = await myToolHandler(args, context);

    expect(result.success).toBe(true);
    expect(result.message).toContain('成功');
  });
});
```

## 📊 性能优化

### 后端优化
- **缓存策略**: 使用内存缓存减少文件系统访问
- **批量操作**: 合并多个小操作减少IO开销
- **连接池**: 复用数据库连接提高性能
- **压缩传输**: 启用gzip压缩减少网络传输

### 前端优化
- **代码分割**: 使用React.lazy进行路由级别的代码分割
- **虚拟滚动**: 大列表使用虚拟滚动提高渲染性能
- **数据预取**: 预先获取可能需要的数据
- **缓存策略**: 使用React Query进行数据缓存

## 🔒 安全最佳实践

### 输入验证
- 所有用户输入都进行严格验证和清理
- 防止SQL注入、XSS攻击等常见安全漏洞
- 使用参数化查询和ORM防止注入攻击

### 访问控制
- 实施适当的认证和授权机制
- 使用HTTPS加密传输敏感数据
- 定期更新依赖包修复安全漏洞

### 文件安全
- 限制文件上传类型和大小
- 验证文件路径防止目录遍历攻击
- 定期备份重要数据

## 🤝 贡献指南

### 贡献流程
1. Fork项目仓库
2. 创建功能分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建Pull Request

### 代码规范
- 使用TypeScript严格模式
- 遵循ESLint和Prettier配置
- 编写完整的单元测试
- 添加适当的文档和注释

### 提交规范
```
feat: 新功能
fix: 修复bug
docs: 文档更新
style: 代码格式调整
refactor: 代码重构
test: 测试相关
chore: 构建工具或辅助工具的变动
```

## 🆘 获取帮助

### 社区支持
- [GitHub Issues](https://github.com/Pimzino/spec-workflow-mcp/issues) - 报告bug和功能请求
- [GitHub Discussions](https://github.com/Pimzino/spec-workflow-mcp/discussions) - 社区讨论和问答
- [Wiki文档](https://github.com/Pimzino/spec-workflow-mcp/wiki) - 社区维护的文档

### 常见问题
- 查看 [故障排除指南](../TROUBLESHOOTING.md) 解决常见问题
- 搜索现有的GitHub Issues
- 查看Wiki文档中的FAQ

### 联系方式
- 项目主页: https://github.com/Pimzino/spec-workflow-mcp
- 文档站点: https://spec-workflow-mcp.docs.dev
- 发布页面: https://www.npmjs.com/package/@pimzino/spec-workflow-mcp

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](https://github.com/Pimzino/spec-workflow-mcp/blob/main/LICENSE) 文件了解详情。

## 🙏 致谢

感谢所有为 Spec Workflow MCP 项目做出贡献的开发者！

---

**开始您的规范驱动开发之旅吧！** 🚀

如果这些文档对您有帮助，请给项目一个⭐️支持！