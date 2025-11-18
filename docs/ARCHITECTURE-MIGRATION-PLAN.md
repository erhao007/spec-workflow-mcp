# Go Gin + PostgreSQL 架构迁移方案

## 1. 项目概述

Spec Workflow MCP 是一个规范驱动的开发工作流管理系统，目前基于 Node.js 实现。本方案旨在将其重构为基于 Go Gin + PostgreSQL 的服务端应用，以支持远程访问和多用户管理能力。

## 2. 现有系统核心功能

通过代码分析，现有系统主要包含以下核心组件：

- **MCP 服务器**：提供规范和任务管理的指令系统
- **仪表盘服务器**：多项目的 Web 可视化界面
- **项目注册表**：全局项目管理和状态跟踪
- **规范解析器**：处理 Markdown 格式的规范文档
- **任务管理**：任务创建、跟踪和状态更新
- **审批工作流**：规范的审核和批准流程
- **实现日志**：记录功能实现进度和细节

## 3. 目标架构设计

### 3.1. 整体架构

```
┌────────────────────────────────────────────────────────────┐
│                     前端应用 (Web)                         │
└──────────────────────────────────┬─────────────────────────┘
                                   │
┌──────────────────────────────────▼─────────────────────────┐
│                      Go Gin 服务器                         │
├──────────┬──────────┬────────────┬────────────┬────────────┤
│ 认证模块  │ 项目管理 │ 规范管理    │ 任务管理    │ MCP 服务    │
└────┬─────┴────┬─────┴─────┬──────┴─────┬──────┴─────┬──────┘
     │          │           │           │           │
┌────▼──────────▼───────────▼───────────▼───────────▼──────┐
│                    PostgreSQL 数据库                      │
└──────────────────────────────────────────────────────────┘
```

### 3.2. 核心模块设计

#### 3.2.1. 认证与用户管理

- **用户认证**：JWT 令牌认证，支持多用户登录
- **权限控制**：基于角色的访问控制 (RBAC)
- **会话管理**：安全的会话处理和过期机制

#### 3.2.2. 项目管理模块

- **项目 CRUD**：创建、读取、更新、删除项目
- **项目注册**：项目全局注册和状态跟踪
- **项目访问控制**：控制用户对项目的访问权限

#### 3.2.3. 规范管理模块

- **规范 CRUD**：规范文档的完整生命周期管理
- **规范解析**：Markdown 解析和结构化数据转换
- **规范归档**：规范版本管理和归档功能

#### 3.2.4. 任务管理模块

- **任务解析**：从规范中解析任务列表
- **任务状态**：任务进度跟踪和状态更新
- **任务依赖**：管理任务间的依赖关系

#### 3.2.5. MCP 服务模块

- **指令处理**：处理客户端发来的 MCP 指令
- **工具注册**：动态注册和管理可用工具
- **实时通信**：WebSocket 支持实时更新

#### 3.2.6. 审批工作流模块

- **审批流程**：规范的提交、审核、批准流程
- **通知机制**：审批状态变更通知
- **历史记录**：审批过程的完整记录

### 3.3. 技术栈选择

- **后端框架**：Go Gin - 高性能的 Web 框架
- **数据库**：PostgreSQL - 强大的关系型数据库
- **ORM**：GORM - Go 语言的 ORM 库
- **认证**：JWT - 无状态认证
- **实时通信**：gorilla/websocket - WebSocket 支持
- **配置管理**：viper - 配置管理库
- **日志**：zap - 高性能日志库
- **缓存**：redis - 可选的缓存层

## 4. 数据库设计

### 4.1. 核心数据表

#### users 表
- id: UUID (主键)
- username: VARCHAR(255) (唯一)
- email: VARCHAR(255) (唯一)
- password_hash: VARCHAR(255)
- role: VARCHAR(50)
- created_at: TIMESTAMP
- updated_at: TIMESTAMP

#### projects 表
- id: UUID (主键)
- name: VARCHAR(255)
- description: TEXT
- path: VARCHAR(1024) (项目路径)
- owner_id: UUID (外键，关联 users)
- created_at: TIMESTAMP
- updated_at: TIMESTAMP

#### specs 表
- id: UUID (主键)
- project_id: UUID (外键，关联 projects)
- name: VARCHAR(255)
- content: TEXT (Markdown 内容)
- status: VARCHAR(50) (active, archived)
- created_by: UUID (外键，关联 users)
- created_at: TIMESTAMP
- updated_at: TIMESTAMP
- last_modified: TIMESTAMP

#### tasks 表
- id: UUID (主键)
- spec_id: UUID (外键，关联 specs)
- title: VARCHAR(255)
- description: TEXT
- status: VARCHAR(50) (pending, in_progress, completed)
- priority: INTEGER
- assignee_id: UUID (外键，关联 users，可选)
- created_at: TIMESTAMP
- updated_at: TIMESTAMP
- completed_at: TIMESTAMP

#### approvals 表
- id: UUID (主键)
- spec_id: UUID (外键，关联 specs)
- approver_id: UUID (外键，关联 users)
- status: VARCHAR(50) (pending, approved, rejected)
- comments: TEXT
- created_at: TIMESTAMP
- updated_at: TIMESTAMP

#### implementation_logs 表
- id: UUID (主键)
- spec_id: UUID (外键，关联 specs)
- task_id: UUID (外键，关联 tasks，可选)
- user_id: UUID (外键，关联 users)
- content: TEXT
- artifacts: JSONB (API端点、组件、函数等)
- created_at: TIMESTAMP

## 5. API 设计

### 5.1. 认证 API
- POST /api/auth/login - 用户登录
- POST /api/auth/register - 用户注册
- GET /api/auth/me - 获取当前用户信息
- POST /api/auth/logout - 用户登出

### 5.2. 项目 API
- GET /api/projects - 获取项目列表
- POST /api/projects - 创建新项目
- GET /api/projects/:id - 获取项目详情
- PUT /api/projects/:id - 更新项目
- DELETE /api/projects/:id - 删除项目

### 5.3. 规范 API
- GET /api/projects/:projectId/specs - 获取规范列表
- POST /api/projects/:projectId/specs - 创建新规范
- GET /api/specs/:id - 获取规范详情
- PUT /api/specs/:id - 更新规范
- DELETE /api/specs/:id - 删除规范
- POST /api/specs/:id/archive - 归档规范
- POST /api/specs/:id/unarchive - 取消归档

### 5.4. 任务 API
- GET /api/specs/:specId/tasks - 获取任务列表
- PUT /api/tasks/:id - 更新任务状态
- GET /api/projects/:projectId/tasks - 获取项目所有任务

### 5.5. 审批 API
- POST /api/specs/:specId/approvals - 提交审批
- GET /api/specs/:specId/approvals - 获取审批状态
- PUT /api/approvals/:id - 审批操作 (approve/reject)

### 5.6. MCP 服务 API
- GET /api/tools - 获取可用工具列表
- POST /api/tools/call - 调用工具
- GET /api/prompts - 获取提示列表
- GET /api/prompts/:id - 获取提示详情

## 6. 迁移策略

### 6.1. 数据迁移

1. **导出现有数据**：从文件系统导出项目、规范、任务等数据
2. **数据转换**：将数据转换为 PostgreSQL 兼容格式
3. **导入新系统**：使用数据导入脚本将数据导入 PostgreSQL

### 6.2. 代码迁移

1. **模块化迁移**：按功能模块逐步迁移
2. **并行运行**：初期支持双系统并行运行，逐步切换
3. **兼容性层**：提供兼容性接口，确保现有客户端可以使用新系统

### 6.3. 部署策略

1. **容器化**：使用 Docker 和 Docker Compose 容器化部署
2. **CI/CD**：建立持续集成和部署流程
3. **监控**：集成监控和日志系统

## 7. 安全考虑

### 7.1. 数据安全
- 密码加密存储
- 敏感数据传输加密
- 数据库访问控制

### 7.2. 应用安全
- 输入验证和消毒
- SQL 注入防护
- XSS 和 CSRF 防护
- 速率限制

### 7.3. 操作安全
- 细粒度的权限控制
- 操作审计日志
- 定期安全更新

## 8. 性能优化

### 8.1. 数据库优化
- 索引优化
- 查询优化
- 连接池管理

### 8.2. 应用优化
- 缓存策略
- 并发处理
- 资源限制

## 9. 总结与下一步

本架构方案提供了将 Spec Workflow MCP 从 Node.js 迁移到 Go Gin + PostgreSQL 的完整路线图。通过这一迁移，系统将获得更好的性能、可靠性和可扩展性，同时支持远程访问和多用户管理功能。

下一步工作将包括：
1. 详细的数据模型设计
2. RESTful API 接口规范编写
3. 技术方案文档的最终确认
4. 开发计划制定和资源分配

---

*文档创建日期：2024年*
*版本：1.0.0*