# Spec Workflow MCP 工具系统参考文档

## 概述

本文档详细介绍了 Spec Workflow MCP 中的所有工具，包括它们的功能、使用方法、参数说明和最佳实践。这些工具为AI辅助的规范驱动开发提供了完整的工作流支持。

## 工具系统架构

### 文件结构

```
src/tools/
├── index.ts                    # 工具注册和分发中心
├── __tests__/projectPath.test.ts # 项目路径验证测试
├── spec-workflow-guide.ts      # 规范工作流指导工具
├── steering-guide.ts           # 指导文档创建工具
├── spec-status.ts              # 规范状态查询工具
├── approvals.ts                # 审批工作流管理工具
└── log-implementation.ts        # 实现日志记录工具
```

### 核心设计模式

#### 注册模式
所有工具通过 `registerTools()` 函数集中注册：

```typescript
export function registerTools() {
  return [
    specWorkflowGuideTool,
    steeringGuideTool,
    specStatusTool,
    approvalsTool,
    logImplementationTool
  ];
}
```

#### 工厂模式
通过 `handleToolCall()` 根据工具名称动态选择处理器：

```typescript
export async function handleToolCall(
  name: string,
  args: any,
  context: ToolContext
): Promise<ToolResponse> {
  let response: ToolResponse;
  let isError = false;

  try {
    switch (name) {
      case 'spec-workflow-guide':
        response = await specWorkflowGuideHandler(args, context);
        break;
      // ... 其他工具处理
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
    isError = !response.success;
  } catch (error) {
    // 统一错误处理
    const errorMessage = error instanceof Error ? error.message : String(error);
    response = {
      success: false,
      message: `Tool execution failed: ${errorMessage}`
    };
    isError = true;
  }

  return toMCPResponse(response, isError);
}
```

#### 统一响应格式
所有工具都返回 `ToolResponse` 类型，通过 `toMCPResponse` 转换为 MCP 格式：

```typescript
interface ToolResponse {
  success: boolean;
  message: string;
  data?: any;
  nextSteps?: string[];
}
```

## 工具详细说明

### 1. spec-workflow-guide - 规范工作流指导

#### 功能描述
为整个规范驱动开发提供完整的工作流指导，包括四阶段开发流程、最佳实践和可视化流程图。

#### 工具定义
```typescript
export const specWorkflowGuideTool: Tool = {
  name: 'spec-workflow-guide',
  description: `Load essential spec workflow instructions to guide feature development from idea to implementation.
This provides the complete workflow sequence (Requirements → Design → Tasks → Implementation) that must be followed.
Always load before any other spec tools when users request spec creation or feature development.`,
  inputSchema: {
    type: 'object',
    properties: {},
    additionalProperties: false
  }
};
```

#### 使用场景
- 开始新的规范开发项目
- 需要了解完整的开发流程
- 团队成员培训和工作流指导
- 回顾开发最佳实践

#### 返回内容
- **四阶段工作流**：Requirements → Design → Tasks → Implementation
- **Mermaid流程图**：可视化的工作流程图
- **最佳实践指导**：当前年份的开发最佳实践
- **模板化操作指导**：如何使用模板创建文档

#### 使用示例
```bash
# 调用工具获取工作流指导
工具名称: spec-workflow-guide
参数: 无（空对象）
```

#### 输出示例
```markdown
# Spec Workflow Guide

## Phase 1: Requirements
- 创建 requirements.md 文档
- 定义用户故事和验收标准
- 获取审批后进入下一阶段

## Phase 2: Design
- 创建 design.md 文档
- 定义架构和技术方案
- 获取审批后进入下一阶段

## Phase 3: Tasks
- 创建 tasks.md 文档
- 分解具体实现任务
- 开始实施阶段

## Phase 4: Implementation
- 按任务列表实施功能
- 记录实现日志
- 完成后归档规范
```

---

### 2. steering-guide - 指导文档创建

#### 功能描述
为项目级别的指导文档创建提供指导，包括产品愿景文档（product.md）、技术标准文档（tech.md）和项目结构文档（structure.md）。

#### 工具定义
```typescript
export const steeringGuideTool: Tool = {
  name: 'steering-guide',
  description: `Load guide for creating project steering documents.
Call ONLY when user explicitly requests steering document creation or asks about project architecture docs.
Not part of standard spec workflow.`,
  inputSchema: {
    type: 'object',
    properties: {},
    additionalProperties: false
  }
};
```

#### 使用限制
- **仅在明确请求时使用**：不是标准工作流的一部分
- **适用于已建立的代码库**：需要对现有项目有基本了解
- **与标准工作流分离**：独立的指导文档创建流程

#### 三文档工作流
1. **Product Document** → 2. **Tech Document** → 3. **Structure Document**

#### 使用示例
```bash
# 创建产品愿景文档
提示词: "帮我创建项目的产品愿景文档"

# 创建技术标准文档
提示词: "我需要写技术规范文档"

# 创建项目结构文档
提示词: "如何定义项目的目录结构"
```

---

### 3. spec-status - 规范状态查询

#### 功能描述
查询规范文档的详细状态，包括各阶段的完成情况、任务进度统计和下一步建议。

#### 工具定义
```typescript
export const specStatusTool: Tool = {
  name: 'spec-status',
  description: `Display comprehensive specification progress overview and completion status.
Use when resuming work or checking overall completion status.
Shows current development phase and task progress.`,
  inputSchema: {
    type: 'object',
    properties: {
      projectPath: {
        type: 'string',
        description: 'Path to the project directory (overrides context projectPath)'
      },
      specName: {
        type: 'string',
        description: 'Name of the specification to check (required if not using detailed mode for all specs)',
        required: false
      },
      detailed: {
        type: 'string',
        description: 'Show detailed view vs summary overview',
        enum: ['true', 'false'],
        required: false
      }
    },
    required: ['specName']
  }
};
```

#### 参数说明

| 参数名 | 类型 | 必需 | 默认值 | 描述 |
|--------|------|------|--------|------|
| `projectPath` | string | 否 | 来自上下文 | 项目目录路径 |
| `specName` | string | 是 | - | 规范名称（kebab-case格式） |
| `detailed` | string | 否 | 'false' | 是否显示详细视图 |

#### 状态分类
- `not-started` - 未开始
- `requirements-needed` - 需要需求文档
- `design-needed` - 需要设计文档
- `tasks-needed` - 需要任务文档
- `implementing` - 实施中
- `completed` - 已完成

#### 路径验证机制
```typescript
// 优先使用参数，回退到上下文
const projectPath = args.projectPath || context.projectPath;

if (!projectPath) {
  return {
    success: false,
    message: 'Project path is required but not provided in args or context'
  };
}
```

#### 使用示例
```bash
# 查询单个规范状态
工具名称: spec-status
参数: {
  "specName": "user-authentication",
  "detailed": "true"
}

# 查询所有规范概要（如果支持）
工具名称: spec-status
参数: {
  "detailed": "false"
}
```

#### 返回示例
```json
{
  "success": true,
  "data": {
    "specName": "user-authentication",
    "status": "implementing",
    "phases": {
      "requirements": { "status": "completed", "fileExists": true },
      "design": { "status": "completed", "fileExists": true },
      "tasks": { "status": "completed", "fileExists": true },
      "implementation": { "status": "in-progress", "progress": 60 }
    },
    "taskProgress": {
      "total": 12,
      "completed": 7,
      "pending": 5,
      "inProgress": 0
    },
    "nextStep": "Continue with remaining tasks"
  }
}
```

---

### 4. approvals - 审批工作流管理

#### 功能描述
管理审批请求的完整生命周期，包括创建审批、查询状态和清理已完成的审批请求。

#### 工具定义
```typescript
export const approvalsTool: Tool = {
  name: 'approvals',
  description: `Manage approval workflow for documents and actions in the spec development process.
Supports requesting approval, checking status, and managing completed approvals.`,
  inputSchema: {
    type: 'object',
    properties: {
      action: {
        type: 'string',
        enum: ['request', 'status', 'delete'],
        description: 'Approval operation to perform'
      },
      projectPath: {
        type: 'string',
        description: 'Path to the project directory'
      },
      title: {
        type: 'string',
        description: 'Brief title for the approval request (required for request action)'
      },
      filePath: {
        type: 'string',
        description: 'Path to the file that needs approval (required for request action)'
      },
      type: {
        type: 'string',
        enum: ['document', 'action'],
        description: 'Type of approval request (required for request action)'
      },
      categoryName: {
        type: 'string',
        description: 'Name of the spec or "steering" for steering documents (required for request action)'
      },
      approvalId: {
        type: 'string',
        description: 'ID of the approval request (required for status and delete actions)'
      }
    },
    required: ['action']
  }
};
```

#### 三种操作模式

##### 1. request - 创建审批请求
**必需参数**：
- `title`: 审批请求标题
- `filePath`: 文件路径
- `type`: 审批类型（'document' | 'action'）
- `categoryName`: 规范名称或 "steering"

**使用场景**：
- 文档创建完成后请求审批
- 重要操作前请求授权
- 阶段性成果确认

##### 2. status - 查询审批状态
**必需参数**：
- `approvalId`: 审批请求ID

**返回信息**：
- 审批状态（pending/approved/rejected/needs-revision）
- 创建和响应时间
- 审批意见和评论
- 修订历史

##### 3. delete - 清理已完成的审批
**必需参数**：
- `approvalId`: 审批请求ID

**安全机制**：
- 只有已批准的请求才能被删除
- 阻塞状态下的删除请求会被拒绝

#### 状态处理逻辑
```typescript
switch (action) {
  case 'request':
    // 创建新的审批请求
    return await requestApproval(args, context);
  case 'status':
    // 查询现有审批状态
    return await getApprovalStatus(args, context);
  case 'delete':
    // 删除已完成的审批
    return await deleteApproval(args, context);
}
```

#### 安全机制
```typescript
// 只有已批准的请求才能被删除
if (approval.status !== 'approved') {
  return {
    success: false,
    message: `BLOCKED: Cannot proceed - status is "${approval.status}"`,
    data: { blockProgress: true, canProceed: false }
  };
}
```

#### 使用示例
```bash
# 创建文档审批
工具名称: approvals
参数: {
  "action": "request",
  "title": "用户认证需求文档",
  "filePath": ".spec-workflow/specs/user-authentication/requirements.md",
  "type": "document",
  "categoryName": "user-authentication"
}

# 查询审批状态
工具名称: approvals
参数: {
  "action": "status",
  "approvalId": "req_20250118_143022_a1b2c3d4"
}

# 删除已完成的审批
工具名称: approvals
参数: {
  "action": "delete",
  "approvalId": "req_20250118_143022_a1b2c3d4"
}
```

---

### 5. log-implementation - 实现日志记录

#### 功能描述
记录已完成任务的实现详情，构建可搜索的知识库，防止代码重复和技术债务积累。

#### 工具定义
```typescript
export const logImplementationTool: Tool = {
  name: 'log-implementation',
  description: `Record comprehensive implementation details for completed tasks to build a searchable knowledge base.
CRITICAL: Artifacts are REQUIRED to prevent code duplication and technical debt.
Future AI agents use implementation logs to discover existing code before implementing new tasks.`,
  inputSchema: {
    type: 'object',
    properties: {
      projectPath: { type: 'string' },
      specName: { type: 'string' },
      taskId: { type: 'string' },
      summary: {
        type: 'string',
        description: 'Brief summary of what was implemented'
      },
      filesModified: {
        type: 'array',
        items: { type: 'string' },
        description: 'List of files that were modified'
      },
      filesCreated: {
        type: 'array',
        items: { type: 'string' },
        description: 'List of files that were created'
      },
      statistics: {
        type: 'object',
        properties: {
          linesAdded: { type: 'number' },
          linesRemoved: { type: 'number' }
        }
      },
      artifacts: {
        type: 'object',
        description: 'REQUIRED: Structured data about implemented artifacts',
        properties: {
          apiEndpoints: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                method: { type: 'string' },
                path: { type: 'string' },
                purpose: { type: 'string' },
                location: { type: 'string' }
              }
            }
          },
          components: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                name: { type: 'string' },
                type: { type: 'string' },
                purpose: { type: 'string' },
                location: { type: 'string' },
                props: { type: 'string' },
                exports: { type: 'array', items: { type: 'string' } }
              }
            }
          },
          functions: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                name: { type: 'string' },
                purpose: { type: 'string' },
                location: { type: 'string' },
                signature: { type: 'string' },
                isExported: { type: 'boolean' }
              }
            }
          },
          classes: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                name: { type: 'string' },
                purpose: { type: 'string' },
                location: { type: 'string' },
                methods: { type: 'array', items: { type: 'string' } },
                isExported: { type: 'boolean' }
              }
            }
          },
          integrations: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                description: { type: 'string' },
                frontendComponent: { type: 'string' },
                backendEndpoint: { type: 'string' },
                dataFlow: { type: 'string' }
              }
            }
          }
        }
      }
    },
    required: ['projectPath', 'specName', 'taskId', 'summary', 'filesModified', 'filesCreated', 'statistics', 'artifacts']
  }
};
```

#### 工件类型说明

##### API Endpoints（API端点）
记录创建或修改的API端点：
```json
{
  "method": "GET",
  "path": "/api/specs/:name/implementation-log",
  "purpose": "Retrieve implementation logs with optional filtering",
  "requestFormat": "Query params: taskId (string, optional), search (string, optional)",
  "responseFormat": "{ entries: ImplementationLogEntry[] }",
  "location": "src/dashboard/server.ts:245"
}
```

##### Components（组件）
记录创建的UI组件：
```json
{
  "name": "LogsPage",
  "type": "React",
  "purpose": "Main dashboard page for viewing implementation logs with search and filtering",
  "location": "src/modules/pages/LogsPage.tsx",
  "props": "{ specs: any[], selectedSpec: string, onSelect: (value: string) => void }",
  "exports": ["LogsPage (default)"]
}
```

##### Functions（函数）
记录创建的工具函数：
```json
{
  "name": "searchLogs",
  "purpose": "Search implementation logs by keyword",
  "location": "src/dashboard/implementation-log-manager.ts:156",
  "signature": "(searchTerm: string) => Promise<ImplementationLogEntry[]>",
  "isExported": true
}
```

##### Classes（类）
记录创建的类：
```json
{
  "name": "ImplementationLogManager",
  "purpose": "Manages CRUD operations for implementation logs",
  "location": "src/dashboard/implementation-log-manager.ts",
  "methods": ["loadLog", "addLogEntry", "getAllLogs", "searchLogs", "getTaskStats"],
  "isExported": true
}
```

##### Integrations（集成）
记录前后端集成：
```json
{
  "description": "LogsPage fetches logs via REST API and subscribes to WebSocket for real-time updates",
  "frontendComponent": "LogsPage",
  "backendEndpoint": "GET /api/specs/:name/implementation-log",
  "dataFlow": "Component mount → API fetch → Display logs → WebSocket subscription → Real-time updates on new entries"
}
```

#### 任务验证机制
```typescript
// 验证任务是否存在
const taskExists = parseResult.tasks.some(t => t.id === taskId);
if (!taskExists) {
  return {
    success: false,
    message: `Task '${taskId}' not found in specification '${specName}'`
  };
}
```

#### 使用示例
```bash
# 记录API端点实现
工具名称: log-implementation
参数: {
  "projectPath": "/path/to/project",
  "specName": "user-authentication",
  "taskId": "2.1",
  "summary": "实现了用户登录API端点",
  "filesModified": ["src/api/auth.ts"],
  "filesCreated": ["src/middleware/auth.ts"],
  "statistics": {
    "linesAdded": 120,
    "linesRemoved": 5
  },
  "artifacts": {
    "apiEndpoints": [
      {
        "method": "POST",
        "path": "/api/auth/login",
        "purpose": "用户身份验证",
        "requestFormat": "JSON body: { email, password }",
        "responseFormat": "JSON: { token, user }",
        "location": "src/api/auth.ts:45"
      }
    ],
    "functions": [
      {
        "name": "authenticateUser",
        "purpose": "验证用户凭据",
        "location": "src/services/auth.ts:23",
        "signature": "(email: string, password: string) => Promise<User | null>",
        "isExported": true
      }
    ]
  }
}
```

## 工具使用最佳实践

### 1. 工作流顺序
```
1. spec-workflow-guide     # 获取工作流指导
2. create-spec             # 创建规范文档
3. approvals               # 请求审批
4. implement-task          # 实施任务
5. log-implementation      # 记录实现日志
```

### 2. 错误处理策略
- **参数验证**：所有工具都进行严格的参数验证
- **路径安全**：防止目录遍历攻击
- **权限检查**：确保文件操作权限
- **回滚机制**：支持操作失败时的回滚

### 3. 性能优化
- **延迟加载**：按需初始化服务
- **缓存机制**：缓存频繁查询的结果
- **批量操作**：合并多个小操作
- **资源清理**：及时释放文件句柄和连接

### 4. 安全考虑
- **输入清理**：所有用户输入都经过验证和清理
- **路径验证**：严格的文件路径安全检查
- **权限控制**：基于文件系统的权限验证
- **审计日志**：记录所有关键操作

## 集成示例

### 完整的规范开发流程
```typescript
// 1. 获取工作流指导
await handleToolCall('spec-workflow-guide', {}, context);

// 2. 创建需求文档
await handleToolCall('create-spec', {
  specName: 'user-authentication',
  documentType: 'requirements',
  description: '用户身份验证系统'
}, context);

// 3. 请求审批
await handleToolCall('approvals', {
  action: 'request',
  title: '用户认证需求文档',
  filePath: '.spec-workflow/specs/user-authentication/requirements.md',
  type: 'document',
  categoryName: 'user-authentication'
}, context);

// 4. 查询状态
await handleToolCall('spec-status', {
  specName: 'user-authentication',
  detailed: 'true'
}, context);

// 5. 实施任务并记录日志
await handleToolCall('log-implementation', {
  projectPath: '/project/path',
  specName: 'user-authentication',
  taskId: '1.1',
  summary: '实现了用户登录功能',
  filesModified: ['src/auth/login.ts'],
  filesCreated: ['src/auth/middleware.ts'],
  statistics: { linesAdded: 85, linesRemoved: 3 },
  artifacts: { /* 详细工件信息 */ }
}, context);
```

## 总结

Spec Workflow MCP 的工具系统为AI辅助开发提供了完整的规范驱动开发支持：

### 核心优势
- **完整性**：覆盖从需求分析到实现记录的完整生命周期
- **一致性**：统一的接口设计和错误处理机制
- **安全性**：全面的输入验证和路径保护
- **可扩展性**：模块化设计便于添加新工具
- **可维护性**：清晰的代码结构和完整的文档

### 创新特性
- **实现日志知识库**：防止代码重复，构建可搜索的实现历史
- **智能状态管理**：自动跟踪项目状态和任务进度
- **审批工作流集成**：完整的文档审批和版本控制
- **多项目支持**：支持同时管理多个开发项目

这套工具系统为规范驱动的软件开发提供了强大的基础设施，显著提高了开发效率和代码质量。