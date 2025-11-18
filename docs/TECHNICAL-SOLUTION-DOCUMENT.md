# Spec Workflow MCP 系统重构技术方案文档

## 1. 项目概述

### 1.1 项目背景

Spec Workflow MCP（Model-Centric Programming）系统是一个用于规范工作流管理的工具，旨在帮助团队创建、管理和执行规范文档，跟踪任务实现进度，并提供实时协作功能。当前系统基于Node.js技术栈开发，随着用户需求的增长，需要将系统重构为服务端应用，以支持远程管理功能并提高系统的可扩展性和性能。

### 1.2 重构目标

- **服务端运行**：将系统改造为基于Go Gin和PostgreSQL的服务端应用
- **远程管理**：支持多用户远程访问和管理规范工作流
- **权限控制**：实现基于角色的访问控制机制
- **数据持久化**：使用PostgreSQL存储所有项目、规范、任务和用户数据
- **性能优化**：提高系统响应速度和并发处理能力
- **可扩展性**：设计模块化架构，便于未来功能扩展

### 1.3 技术栈对比

| 方面 | 现有技术栈 | 目标技术栈 |
|------|------------|------------|
| 后端框架 | Node.js + Fastify | Go + Gin |
| 数据库 | 可能使用内存或文件存储 | PostgreSQL |
| 认证机制 | 可能使用简单认证 | JWT + RBAC |
| 部署方式 | 本地运行 | 服务端部署，支持容器化 |
| API设计 | 部分API支持 | RESTful API + WebSocket |

## 2. 系统架构设计

### 2.1 整体架构

![系统架构图](https://placeholder-for-architecture-diagram)

### 2.2 核心组件

#### 2.2.1 后端服务层

- **Web服务器**：基于Gin框架的HTTP服务器
- **API控制器**：处理所有API请求
- **业务逻辑层**：实现核心业务逻辑
- **数据访问层**：数据库操作封装
- **认证授权模块**：JWT认证和RBAC权限控制
- **WebSocket服务**：提供实时更新功能
- **工具集成模块**：集成各类MCP工具

#### 2.2.2 数据存储层

- **PostgreSQL数据库**：存储所有业务数据
- **Redis缓存**：用于缓存热点数据和会话管理

#### 2.2.3 前端层

- **Web应用**：基于React的单页应用
- **移动应用**：可选的移动客户端

### 2.3 模块关系图

```
┌─────────────────┐      ┌─────────────────────┐      ┌─────────────────┐
│   客户端层      │ ──── │     API网关层       │ ──── │  认证授权模块   │
└─────────────────┘      └─────────────────────┘      └────────┬────────┘
                                                               │
┌─────────────────┐      ┌─────────────────────┐      ┌────────▼────────┐
│  数据持久层     │ ◄─── │     服务层          │ ──── │  业务逻辑层     │
└─────────────────┘      └─────────────────────┘      └─────────────────┘
```

## 3. 数据库设计

### 3.1 数据库表结构

#### 3.1.1 用户表 (users)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100),
    role VARCHAR(20) NOT NULL DEFAULT 'user',
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.1.2 项目表 (projects)

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    path VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    owner_id UUID NOT NULL REFERENCES users(id),
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.1.3 项目成员表 (project_members)

```sql
CREATE TABLE project_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'viewer',
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, user_id)
);
```

#### 3.1.4 规范表 (specifications)

```sql
CREATE TABLE specifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    type VARCHAR(20) NOT NULL DEFAULT 'standard',
    version VARCHAR(50),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_by UUID NOT NULL REFERENCES users(id),
    last_modified_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.1.5 任务表 (tasks)

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spec_id UUID NOT NULL REFERENCES specifications(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    priority INTEGER DEFAULT 0,
    order_index INTEGER DEFAULT 0,
    assignee_id UUID REFERENCES users(id),
    created_by UUID NOT NULL REFERENCES users(id),
    due_date TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.1.6 审批表 (approvals)

```sql
CREATE TABLE approvals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spec_id UUID NOT NULL REFERENCES specifications(id) ON DELETE CASCADE,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    requested_by UUID NOT NULL REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    comments TEXT,
    requested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    responded_at TIMESTAMP,
    expires_at TIMESTAMP
);
```

#### 3.1.7 实现日志表 (implementation_logs)

```sql
CREATE TABLE implementation_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    spec_id UUID REFERENCES specifications(id) ON DELETE CASCADE,
    task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    artifacts JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.2 索引设计

```sql
-- 为常用查询创建索引
CREATE INDEX idx_projects_owner_id ON projects(owner_id);
CREATE INDEX idx_project_members_project_id ON project_members(project_id);
CREATE INDEX idx_project_members_user_id ON project_members(user_id);
CREATE INDEX idx_specifications_project_id ON specifications(project_id);
CREATE INDEX idx_tasks_spec_id ON tasks(spec_id);
CREATE INDEX idx_tasks_assignee_id ON tasks(assignee_id);
CREATE INDEX idx_approvals_spec_id ON approvals(spec_id);
CREATE INDEX idx_implementation_logs_project_id ON implementation_logs(project_id);
CREATE INDEX idx_implementation_logs_user_id ON implementation_logs(user_id);
```

### 3.3 数据关系图

```
用户 (users) ──┐
              │
              ▼
项目成员 (project_members) ◄─── 项目 (projects)
              │
              ▼
规范 (specifications) ────► 任务 (tasks)
     │
     ▼
审批 (approvals)
```

## 4. API 设计

### 4.1 RESTful API 规范

完整的API规范详见 `REST-API-SPECIFICATION.md` 文件，主要包括以下几类接口：

- **认证相关API**：注册、登录、获取用户信息、登出、刷新令牌
- **项目管理API**：项目CRUD、项目成员管理
- **规范管理API**：规范CRUD、版本管理、归档
- **任务管理API**：任务CRUD、批量操作
- **审批工作流API**：提交审批、审批操作、查看审批历史
- **实现日志API**：日志CRUD
- **MCP服务API**：工具调用、提示管理

### 4.2 WebSocket API

- **连接端点**：`/api/v1/ws`
- **支持的事件**：
  - 任务状态更新
  - 规范内容变更
  - 审批状态变更
  - 实时消息通知

## 5. 核心功能实现

### 5.1 用户认证与授权

- **JWT实现**：使用Go的JWT库生成和验证令牌
- **密码加密**：使用bcrypt进行密码哈希
- **RBAC实现**：基于角色的访问控制
- **会话管理**：结合Redis进行会话存储和管理

### 5.2 项目管理

- **项目CRUD**：完整的项目创建、读取、更新、删除功能
- **项目成员管理**：邀请成员、管理角色、移除成员
- **权限检查**：确保用户只能访问有权限的项目

### 5.3 规范管理

- **规范解析**：Markdown内容解析
- **任务提取**：从规范中提取任务
- **版本控制**：支持规范的版本历史和回滚
- **内容比较**：不同版本间的内容比较

### 5.4 任务管理

- **任务状态流转**：待办、进行中、已完成等状态管理
- **任务分配**：指派任务给团队成员
- **优先级管理**：任务优先级设置
- **截止日期提醒**：任务截止日期管理

### 5.5 审批工作流

- **审批流程**：提交流程、审批/拒绝操作
- **通知机制**：邮件或系统内通知提醒审批人
- **审批历史**：完整的审批记录追踪

### 5.6 MCP工具集成

- **工具注册**：支持动态注册和管理MCP工具
- **工具调用**：安全地调用各类MCP工具
- **提示模板**：管理和应用提示模板

## 6. 迁移策略

### 6.1 数据迁移

1. **数据导出**：从现有系统导出数据
2. **数据转换**：将数据转换为PostgreSQL兼容格式
3. **数据导入**：导入到新的PostgreSQL数据库
4. **数据验证**：确保数据完整性和一致性

### 6.2 代码迁移

1. **增量迁移**：按模块逐步迁移代码
2. **接口兼容**：保持API接口兼容性
3. **并行运行**：支持新旧系统并行运行
4. **平滑切换**：实现平滑的系统切换

### 6.3 部署策略

1. **环境准备**：设置开发、测试、生产环境
2. **容器化**：使用Docker进行应用容器化
3. **CI/CD**：建立持续集成和持续部署流程
4. **监控告警**：设置系统监控和告警机制

## 7. 安全考虑

### 7.1 认证与授权安全

- **令牌安全**：JWT令牌过期时间设置
- **密码策略**：强密码要求和定期修改
- **角色隔离**：严格的权限边界控制

### 7.2 数据安全

- **数据加密**：敏感数据加密存储
- **备份恢复**：定期数据备份和恢复演练
- **传输安全**：全站HTTPS加密

### 7.3 应用安全

- **输入验证**：严格的输入数据验证
- **防止注入**：防止SQL注入、XSS攻击
- **请求限流**：防止暴力攻击

## 8. 性能优化

### 8.1 数据库优化

- **索引优化**：为常用查询创建适当的索引
- **查询优化**：优化SQL查询性能
- **连接池**：使用连接池管理数据库连接

### 8.2 应用层优化

- **缓存策略**：使用Redis缓存热点数据
- **异步处理**：使用Go的goroutine处理并发请求
- **资源限制**：合理设置资源使用限制

### 8.3 负载均衡

- **水平扩展**：支持多实例部署
- **负载分发**：合理的负载均衡策略

## 9. 开发计划

### 9.1 阶段一：基础架构搭建

- **时间**：4周
- **任务**：
  - 搭建Go Gin基础框架
  - 配置PostgreSQL数据库
  - 实现认证授权模块
  - 搭建CI/CD流水线

### 9.2 阶段二：核心功能开发

- **时间**：8周
- **任务**：
  - 实现用户管理功能
  - 实现项目管理功能
  - 实现规范管理功能
  - 实现任务管理功能

### 9.3 阶段三：高级功能开发

- **时间**：6周
- **任务**：
  - 实现审批工作流
  - 实现实现日志功能
  - 集成MCP工具
  - 实现WebSocket实时更新

### 9.4 阶段四：测试与优化

- **时间**：4周
- **任务**：
  - 系统功能测试
  - 性能测试和优化
  - 安全测试
  - Bug修复

### 9.5 阶段五：部署与迁移

- **时间**：2周
- **任务**：
  - 数据迁移
  - 系统部署
  - 用户培训
  - 上线准备

## 10. 风险评估与缓解策略

### 10.1 技术风险

| 风险 | 影响 | 缓解策略 |
|------|------|----------|
| Go语言学习曲线 | 开发效率降低 | 提前进行技术培训，使用成熟的Go库 |
| 数据库迁移复杂 | 数据丢失风险 | 详细的数据迁移计划，多次演练，保留备份 |
| 性能瓶颈 | 用户体验下降 | 早期进行性能测试，优化数据库和应用代码 |

### 10.2 进度风险

| 风险 | 影响 | 缓解策略 |
|------|------|----------|
| 需求变更 | 开发延期 | 敏捷开发方法，合理的变更管理流程 |
| 资源不足 | 进度延误 | 合理规划资源，必要时调整范围 |
| 技术难点 | 开发阻碍 | 提前识别技术难点，寻求专家支持 |

### 10.3 运维风险

| 风险 | 影响 | 缓解策略 |
|------|------|----------|
| 系统稳定性 | 服务中断 | 完善的监控告警，自动恢复机制 |
| 安全漏洞 | 数据泄露 | 定期安全审计，及时更新补丁 |
| 扩展性问题 | 无法满足用户增长 | 设计时考虑扩展性，预留扩展空间 |

## 11. 结论与建议

### 11.1 可行性总结

基于Go Gin和PostgreSQL的重构方案具有很高的可行性。Go语言的高性能和强并发特性非常适合构建服务端应用，PostgreSQL作为功能强大的关系型数据库，能够很好地支持系统的数据存储需求。重构后的系统将具有更好的性能、可扩展性和安全性，能够满足用户的远程管理需求。

### 11.2 建议

1. **分阶段实施**：按照开发计划分阶段实施重构
2. **保持兼容性**：在重构过程中保持API接口的兼容性，降低对现有用户的影响
3. **注重用户体验**：在开发过程中始终关注用户体验
4. **定期评估进度**：定期评估项目进度，及时调整计划
5. **全面测试**：确保系统稳定性和数据一致性

---

*文档创建日期：2024年*
*版本：1.0.0*