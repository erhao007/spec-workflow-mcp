# RESTful API 接口规范

## 1. 概述

本文档定义了 Spec Workflow MCP 系统的 RESTful API 接口规范。这些接口将支持系统的所有核心功能，包括用户认证、项目管理、规范管理、任务管理、审批工作流和实现日志等。设计遵循 RESTful 架构原则，提供清晰、一致和可扩展的 API 接口。

## 2. API 设计原则

- **资源导向**：API 围绕资源（用户、项目、规范等）设计
- **标准 HTTP 方法**：使用 GET、POST、PUT、DELETE 等标准方法
- **状态码**：使用标准 HTTP 状态码表示请求结果
- **认证授权**：基于 JWT 的认证和基于角色的授权
- **一致性**：保持 URL 格式、参数命名和响应格式的一致性
- **错误处理**：统一的错误响应格式
- **版本控制**：API 版本管理机制

## 3. 认证与授权

### 3.1. 认证机制

- **认证方式**：JWT (JSON Web Token)
- **Token 传递**：通过 HTTP 请求头 `Authorization: Bearer {token}` 传递
- **Token 有效期**：可配置，默认为 24 小时
- **刷新机制**：支持 Token 刷新

### 3.2. 授权机制

- **基于角色的访问控制 (RBAC)**
- **预定义角色**：
  - `admin`：系统管理员，拥有所有权限
  - `manager`：项目管理员，可管理项目和成员
  - `editor`：编辑者，可编辑规范和任务
  - `viewer`：查看者，仅可查看内容
- **项目级权限**：每个项目有独立的成员角色设置

## 4. API 版本控制

- **版本标识**：在 URL 路径中包含版本号
- **当前版本**：`v1`
- **示例**：`/api/v1/projects`

## 5. 统一响应格式

### 5.1. 成功响应

```json
{
  "success": true,
  "data": {...}, // 响应数据
  "pagination": {...}, // 分页信息（可选）
  "message": "操作成功" // 可选的消息
}
```

### 5.2. 错误响应

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE", // 错误代码
    "message": "错误消息", // 人类可读的错误消息
    "details": {...} // 详细错误信息（可选）
  }
}
```

## 6. 详细 API 规范

### 6.1. 认证相关 API

#### 6.1.1. 用户注册

- **URL**: `POST /api/v1/auth/register`
- **描述**: 创建新用户账户
- **请求体**:
  ```json
  {
    "username": "string", // 用户名
    "email": "string", // 邮箱地址
    "password": "string", // 密码
    "full_name": "string" // 全名（可选）
  }
  ```
- **成功响应** (201 Created):
  ```json
  {
    "success": true,
    "data": {
      "user": {
        "id": "uuid",
        "username": "string",
        "email": "string",
        "full_name": "string",
        "role": "user"
      },
      "token": "jwt_token",
      "expires_at": "timestamp"
    },
    "message": "用户注册成功"
  }
  ```

#### 6.1.2. 用户登录

- **URL**: `POST /api/v1/auth/login`
- **描述**: 用户登录并获取访问令牌
- **请求体**:
  ```json
  {
    "email": "string", // 邮箱地址
    "password": "string" // 密码
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "user": {
        "id": "uuid",
        "username": "string",
        "email": "string",
        "full_name": "string",
        "role": "string"
      },
      "token": "jwt_token",
      "expires_at": "timestamp"
    },
    "message": "登录成功"
  }
  ```

#### 6.1.3. 获取当前用户信息

- **URL**: `GET /api/v1/auth/me`
- **描述**: 获取当前登录用户的信息
- **认证**: 需要 JWT 令牌
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "username": "string",
      "email": "string",
      "full_name": "string",
      "role": "string",
      "last_login": "timestamp",
      "created_at": "timestamp"
    }
  }
  ```

#### 6.1.4. 用户登出

- **URL**: `POST /api/v1/auth/logout`
- **描述**: 用户登出并使令牌失效
- **认证**: 需要 JWT 令牌
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "message": "登出成功"
  }
  ```

#### 6.1.5. 刷新令牌

- **URL**: `POST /api/v1/auth/refresh`
- **描述**: 刷新访问令牌
- **认证**: 需要 JWT 令牌
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "token": "new_jwt_token",
      "expires_at": "timestamp"
    },
    "message": "令牌刷新成功"
  }
  ```

### 6.2. 项目管理 API

#### 6.2.1. 获取项目列表

- **URL**: `GET /api/v1/projects`
- **描述**: 获取用户可访问的项目列表
- **认证**: 需要 JWT 令牌
- **查询参数**:
  - `page`: 页码（默认 1）
  - `limit`: 每页数量（默认 10）
  - `status`: 状态过滤（可选）
  - `search`: 搜索关键词（可选）
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "name": "string",
        "description": "string",
        "status": "string",
        "owner": {
          "id": "uuid",
          "username": "string"
        },
        "member_role": "string", // 当前用户在项目中的角色
        "created_at": "timestamp"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 50,
      "total_pages": 5
    }
  }
  ```

#### 6.2.2. 创建新项目

- **URL**: `POST /api/v1/projects`
- **描述**: 创建新项目
- **认证**: 需要 JWT 令牌
- **请求体**:
  ```json
  {
    "name": "string", // 项目名称
    "description": "string", // 项目描述
    "path": "string", // 项目路径（可选）
    "settings": {} // 项目设置（可选）
  }
  ```
- **成功响应** (201 Created):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "name": "string",
      "description": "string",
      "status": "active",
      "owner": {
        "id": "uuid",
        "username": "string"
      },
      "created_at": "timestamp",
      "updated_at": "timestamp"
    },
    "message": "项目创建成功"
  }
  ```

#### 6.2.3. 获取项目详情

- **URL**: `GET /api/v1/projects/:id`
- **描述**: 获取项目的详细信息
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `id`: 项目 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "name": "string",
      "description": "string",
      "path": "string",
      "status": "string",
      "settings": {},
      "owner": {
        "id": "uuid",
        "username": "string",
        "email": "string"
      },
      "member_role": "string",
      "member_count": 5,
      "spec_count": 10,
      "task_count": 50,
      "created_at": "timestamp",
      "updated_at": "timestamp"
    }
  }
  ```

#### 6.2.4. 更新项目

- **URL**: `PUT /api/v1/projects/:id`
- **描述**: 更新项目信息
- **认证**: 需要 JWT 令牌（仅项目所有者或管理员可操作）
- **路径参数**:
  - `id`: 项目 ID
- **请求体**:
  ```json
  {
    "name": "string", // 可选
    "description": "string", // 可选
    "status": "string", // 可选
    "settings": {} // 可选
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "name": "string",
      "description": "string",
      "status": "string",
      "updated_at": "timestamp"
    },
    "message": "项目更新成功"
  }
  ```

#### 6.2.5. 删除项目

- **URL**: `DELETE /api/v1/projects/:id`
- **描述**: 删除项目
- **认证**: 需要 JWT 令牌（仅项目所有者可操作）
- **路径参数**:
  - `id`: 项目 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "message": "项目删除成功"
  }
  ```

### 6.3. 项目成员管理 API

#### 6.3.1. 获取项目成员列表

- **URL**: `GET /api/v1/projects/:id/members`
- **描述**: 获取项目成员列表
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `id`: 项目 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "user": {
          "id": "uuid",
          "username": "string",
          "email": "string",
          "full_name": "string"
        },
        "role": "string",
        "joined_at": "timestamp"
      }
    ]
  }
  ```

#### 6.3.2. 添加项目成员

- **URL**: `POST /api/v1/projects/:id/members`
- **描述**: 添加用户到项目
- **认证**: 需要 JWT 令牌（仅项目所有者或管理员可操作）
- **路径参数**:
  - `id`: 项目 ID
- **请求体**:
  ```json
  {
    "email": "string", // 用户邮箱
    "role": "string" // 角色：editor, viewer
  }
  ```
- **成功响应** (201 Created):
  ```json
  {
    "success": true,
    "data": {
      "user": {
        "id": "uuid",
        "username": "string",
        "email": "string"
      },
      "role": "string",
      "joined_at": "timestamp"
    },
    "message": "成员添加成功"
  }
  ```

#### 6.3.3. 更新成员角色

- **URL**: `PUT /api/v1/projects/:id/members/:userId`
- **描述**: 更新项目成员角色
- **认证**: 需要 JWT 令牌（仅项目所有者可操作）
- **路径参数**:
  - `id`: 项目 ID
  - `userId`: 用户 ID
- **请求体**:
  ```json
  {
    "role": "string" // 新角色
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "user_id": "uuid",
      "role": "string"
    },
    "message": "成员角色更新成功"
  }
  ```

#### 6.3.4. 移除项目成员

- **URL**: `DELETE /api/v1/projects/:id/members/:userId`
- **描述**: 从项目中移除成员
- **认证**: 需要 JWT 令牌（仅项目所有者可操作）
- **路径参数**:
  - `id`: 项目 ID
  - `userId`: 用户 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "message": "成员移除成功"
  }
  ```

### 6.4. 规范管理 API

#### 6.4.1. 获取规范列表

- **URL**: `GET /api/v1/projects/:projectId/specs`
- **描述**: 获取项目的规范列表
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `projectId`: 项目 ID
- **查询参数**:
  - `page`: 页码（默认 1）
  - `limit`: 每页数量（默认 10）
  - `status`: 状态过滤（active, archived, draft）
  - `type`: 类型过滤（standard, technical, requirement）
  - `search`: 搜索关键词
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "name": "string",
        "title": "string",
        "type": "string",
        "status": "string",
        "version": "string",
        "created_by": {
          "id": "uuid",
          "username": "string"
        },
        "last_modified": "timestamp",
        "task_count": 5
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 25,
      "total_pages": 3
    }
  }
  ```

#### 6.4.2. 创建新规范

- **URL**: `POST /api/v1/projects/:projectId/specs`
- **描述**: 创建新规范
- **认证**: 需要 JWT 令牌（editor 及以上角色）
- **路径参数**:
  - `projectId`: 项目 ID
- **请求体**:
  ```json
  {
    "name": "string", // 规范名称
    "title": "string", // 规范标题
    "content": "string", // Markdown 内容
    "type": "string", // 类型
    "version": "string" // 版本（可选）
  }
  ```
- **成功响应** (201 Created):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "project_id": "uuid",
      "name": "string",
      "title": "string",
      "content": "string",
      "status": "active",
      "type": "string",
      "version": "string",
      "created_by": {
        "id": "uuid",
        "username": "string"
      },
      "last_modified": "timestamp",
      "created_at": "timestamp"
    },
    "message": "规范创建成功"
  }
  ```

#### 6.4.3. 获取规范详情

- **URL**: `GET /api/v1/specs/:id`
- **描述**: 获取规范的详细内容
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `id`: 规范 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "project_id": "uuid",
      "name": "string",
      "title": "string",
      "content": "string",
      "status": "string",
      "type": "string",
      "version": "string",
      "created_by": {
        "id": "uuid",
        "username": "string"
      },
      "last_modified_by": {
        "id": "uuid",
        "username": "string"
      },
      "last_modified": "timestamp",
      "created_at": "timestamp",
      "approval_status": "string" // 审批状态
    }
  }
  ```

#### 6.4.4. 更新规范

- **URL**: `PUT /api/v1/specs/:id`
- **描述**: 更新规范内容
- **认证**: 需要 JWT 令牌（editor 及以上角色）
- **路径参数**:
  - `id`: 规范 ID
- **请求体**:
  ```json
  {
    "title": "string", // 可选
    "content": "string", // 可选
    "version": "string" // 可选，用于版本更新
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "title": "string",
      "content": "string",
      "version": "string",
      "last_modified_by": {
        "id": "uuid",
        "username": "string"
      },
      "last_modified": "timestamp"
    },
    "message": "规范更新成功"
  }
  ```

#### 6.4.5. 删除规范

- **URL**: `DELETE /api/v1/specs/:id`
- **描述**: 删除规范
- **认证**: 需要 JWT 令牌（项目管理员或所有者）
- **路径参数**:
  - `id`: 规范 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "message": "规范删除成功"
  }
  ```

#### 6.4.6. 归档规范

- **URL**: `POST /api/v1/specs/:id/archive`
- **描述**: 归档规范
- **认证**: 需要 JWT 令牌（editor 及以上角色）
- **路径参数**:
  - `id`: 规范 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "status": "archived"
    },
    "message": "规范归档成功"
  }
  ```

#### 6.4.7. 取消归档规范

- **URL**: `POST /api/v1/specs/:id/unarchive`
- **描述**: 将归档的规范恢复为活跃状态
- **认证**: 需要 JWT 令牌（editor 及以上角色）
- **路径参数**:
  - `id`: 规范 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "status": "active"
    },
    "message": "规范已恢复为活跃状态"
  }
  ```

#### 6.4.8. 获取规范版本历史

- **URL**: `GET /api/v1/specs/:id/versions`
- **描述**: 获取规范的所有版本历史
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `id`: 规范 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "version": "string",
        "created_by": {
          "id": "uuid",
          "username": "string"
        },
        "created_at": "timestamp"
      }
    ]
  }
  ```

### 6.5. 任务管理 API

#### 6.5.1. 获取规范任务列表

- **URL**: `GET /api/v1/specs/:specId/tasks`
- **描述**: 获取规范中的任务列表
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `specId`: 规范 ID
- **查询参数**:
  - `status`: 状态过滤
  - `assignee`: 负责人过滤
  - `priority`: 优先级过滤
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "spec_id": "uuid",
        "title": "string",
        "description": "string",
        "status": "string",
        "priority": 0,
        "order_index": 0,
        "assignee": {
          "id": "uuid",
          "username": "string"
        },
        "created_by": {
          "id": "uuid",
          "username": "string"
        },
        "completed_at": "timestamp",
        "due_date": "timestamp",
        "created_at": "timestamp",
        "updated_at": "timestamp"
      }
    ]
  }
  ```

#### 6.5.2. 获取项目任务列表

- **URL**: `GET /api/v1/projects/:projectId/tasks`
- **描述**: 获取项目中的所有任务
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `projectId`: 项目 ID
- **查询参数**:
  - `page`: 页码
  - `limit`: 每页数量
  - `status`: 状态过滤
  - `assignee`: 负责人过滤
  - `spec_id`: 规范 ID 过滤
  - `priority`: 优先级过滤
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "spec_id": "uuid",
        "spec_name": "string",
        "title": "string",
        "status": "string",
        "priority": 0,
        "assignee": {
          "id": "uuid",
          "username": "string"
        },
        "due_date": "timestamp",
        "completed_at": "timestamp"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "total_pages": 5
    }
  }
  ```

#### 6.5.3. 更新任务

- **URL**: `PUT /api/v1/tasks/:id`
- **描述**: 更新任务信息
- **认证**: 需要 JWT 令牌（editor 及以上角色或任务负责人）
- **路径参数**:
  - `id`: 任务 ID
- **请求体**:
  ```json
  {
    "title": "string", // 可选
    "description": "string", // 可选
    "status": "string", // 可选
    "priority": 0, // 可选
    "assignee_id": "uuid", // 可选
    "due_date": "timestamp" // 可选
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "title": "string",
      "status": "string",
      "assignee": {
        "id": "uuid",
        "username": "string"
      },
      "updated_at": "timestamp"
    },
    "message": "任务更新成功"
  }
  ```

#### 6.5.4. 批量更新任务状态

- **URL**: `PUT /api/v1/tasks/batch/status`
- **描述**: 批量更新多个任务的状态
- **认证**: 需要 JWT 令牌
- **请求体**:
  ```json
  {
    "task_ids": ["uuid1", "uuid2"],
    "status": "string"
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "updated_count": 2
    },
    "message": "批量更新成功"
  }
  ```

### 6.6. 审批工作流 API

#### 6.6.1. 提交规范审批

- **URL**: `POST /api/v1/specs/:specId/approvals`
- **描述**: 提交规范进行审批
- **认证**: 需要 JWT 令牌（editor 及以上角色）
- **路径参数**:
  - `specId`: 规范 ID
- **请求体**:
  ```json
  {
    "comments": "string", // 提交备注
    "expires_at": "timestamp" // 过期时间（可选）
  }
  ```
- **成功响应** (201 Created):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "spec_id": "uuid",
      "status": "pending",
      "requested_by": {
        "id": "uuid",
        "username": "string"
      },
      "requested_at": "timestamp",
      "expires_at": "timestamp"
    },
    "message": "审批请求已提交"
  }
  ```

#### 6.6.2. 获取规范审批状态

- **URL**: `GET /api/v1/specs/:specId/approvals`
- **描述**: 获取规范的审批历史和当前状态
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `specId`: 规范 ID
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "current": {
        "id": "uuid",
        "status": "string",
        "requested_by": {
          "id": "uuid",
          "username": "string"
        },
        "requested_at": "timestamp",
        "expires_at": "timestamp"
      },
      "history": [
        {
          "id": "uuid",
          "status": "string",
          "approved_by": {
            "id": "uuid",
            "username": "string"
          },
          "comments": "string",
          "responded_at": "timestamp"
        }
      ]
    }
  }
  ```

#### 6.6.3. 审批操作

- **URL**: `PUT /api/v1/approvals/:id`
- **描述**: 批准或拒绝审批请求
- **认证**: 需要 JWT 令牌（项目管理员或所有者）
- **路径参数**:
  - `id`: 审批 ID
- **请求体**:
  ```json
  {
    "status": "string", // approved 或 rejected
    "comments": "string" // 审批意见
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "status": "string",
      "approved_by": {
        "id": "uuid",
        "username": "string"
      },
      "comments": "string",
      "responded_at": "timestamp"
    },
    "message": "审批操作成功"
  }
  ```

### 6.7. 实现日志 API

#### 6.7.1. 创建实现日志

- **URL**: `POST /api/v1/implementation-logs`
- **描述**: 创建实现日志记录
- **认证**: 需要 JWT 令牌
- **请求体**:
  ```json
  {
    "spec_id": "uuid", // 规范 ID（可选）
    "task_id": "uuid", // 任务 ID（可选）
    "project_id": "uuid", // 项目 ID（必填）
    "content": "string", // 日志内容
    "artifacts": { // 实现的产物
      "api_endpoints": ["string"],
      "components": ["string"],
      "functions": ["string"],
      "files": ["string"]
    }
  }
  ```
- **成功响应** (201 Created):
  ```json
  {
    "success": true,
    "data": {
      "id": "uuid",
      "spec_id": "uuid",
      "task_id": "uuid",
      "project_id": "uuid",
      "user": {
        "id": "uuid",
        "username": "string"
      },
      "content": "string",
      "artifacts": {},
      "created_at": "timestamp"
    },
    "message": "实现日志创建成功"
  }
  ```

#### 6.7.2. 获取实现日志列表

- **URL**: `GET /api/v1/projects/:projectId/implementation-logs`
- **描述**: 获取项目的实现日志
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `projectId`: 项目 ID
- **查询参数**:
  - `page`: 页码
  - `limit`: 每页数量
  - `spec_id`: 规范 ID 过滤
  - `task_id`: 任务 ID 过滤
  - `user_id`: 用户 ID 过滤
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "uuid",
        "spec_id": "uuid",
        "spec_name": "string",
        "task_id": "uuid",
        "task_title": "string",
        "user": {
          "id": "uuid",
          "username": "string"
        },
        "content": "string",
        "artifacts": {},
        "created_at": "timestamp"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 50,
      "total_pages": 3
    }
  }
  ```

### 6.8. MCP 服务 API

#### 6.8.1. 获取可用工具列表

- **URL**: `GET /api/v1/tools`
- **描述**: 获取所有可用的 MCP 工具
- **认证**: 需要 JWT 令牌
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "name": "string",
        "description": "string",
        "parameters": {},
        "category": "string"
      }
    ]
  }
  ```

#### 6.8.2. 调用工具

- **URL**: `POST /api/v1/tools/call`
- **描述**: 调用指定的 MCP 工具
- **认证**: 需要 JWT 令牌
- **请求体**:
  ```json
  {
    "name": "string", // 工具名称
    "arguments": {}
  }
  ```
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "result": {}
    },
    "message": "工具调用成功"
  }
  ```

#### 6.8.3. 获取提示列表

- **URL**: `GET /api/v1/prompts`
- **描述**: 获取所有可用的提示模板
- **认证**: 需要 JWT 令牌
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": [
      {
        "name": "string",
        "description": "string",
        "category": "string",
        "is_system": true
      }
    ]
  }
  ```

#### 6.8.4. 获取提示详情

- **URL**: `GET /api/v1/prompts/:name`
- **描述**: 获取特定提示模板的详细内容
- **认证**: 需要 JWT 令牌
- **路径参数**:
  - `name`: 提示名称
- **成功响应** (200 OK):
  ```json
  {
    "success": true,
    "data": {
      "name": "string",
      "content": "string",
      "description": "string",
      "category": "string",
      "is_system": true
    }
  }
  ```

## 7. WebSocket API

### 7.1. 实时更新连接

- **URL**: `ws://api.example.com/api/v1/ws`
- **认证**: WebSocket 握手时需要 JWT 令牌
- **参数**:
  - `projectId`: 可选，项目 ID 过滤
- **消息格式**:

  客户端发送:
  ```json
  {
    "type": "subscribe",
    "channels": ["project:uuid", "spec:uuid", "task:uuid"]
  }
  ```

  服务端推送:
  ```json
  {
    "type": "update",
    "channel": "project:uuid",
    "data": {
      "event": "task_status_changed",
      "task_id": "uuid",
      "status": "completed"
    },
    "timestamp": "timestamp"
  }
  ```

## 8. 错误处理

### 8.1. 常见错误码

| 错误码 | HTTP 状态码 | 描述 |
|--------|-------------|------|
| `AUTH_REQUIRED` | 401 | 需要认证 |
| `INVALID_TOKEN` | 401 | 无效的认证令牌 |
| `PERMISSION_DENIED` | 403 | 权限不足 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `VALIDATION_ERROR` | 400 | 请求数据验证失败 |
| `DUPLICATE_RESOURCE` | 409 | 资源已存在 |
| `SERVER_ERROR` | 500 | 服务器内部错误 |
| `DATABASE_ERROR` | 500 | 数据库错误 |

### 8.2. 错误响应示例

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "请求数据验证失败",
    "details": {
      "email": "邮箱地址格式不正确"
    }
  }
}
```

## 9. API 安全性

### 9.1. 安全最佳实践

- 所有 API 请求必须使用 HTTPS
- 敏感操作需要二次确认
- 实现请求限流防止暴力攻击
- 输入数据验证和净化
- 防止 SQL 注入和 XSS 攻击
- 敏感数据加密存储
- 定期安全审计

### 9.2. CORS 配置

```json
{
  "origins": ["*"], // 生产环境中应该设置为具体域名
  "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  "headers": ["Origin", "Content-Type", "Authorization", "Accept"],
  "credentials": true
}
```

## 10. 文档和测试

### 10.1. API 文档

- 使用 Swagger/OpenAPI 进行 API 文档自动生成
- 提供交互式 API 测试界面
- 文档 URL: `/api/v1/docs`

### 10.2. 测试要求

- 单元测试覆盖率 > 80%
- 集成测试覆盖所有关键 API 端点
- 性能测试确保 API 在负载下正常工作

## 11. 总结

本 API 规范提供了 Spec Workflow MCP 系统完整的 RESTful API 设计。这些 API 覆盖了系统的所有核心功能，包括用户管理、项目管理、规范管理、任务管理、审批工作流和实现日志等。设计遵循 RESTful 原则，提供了清晰、一致和可扩展的接口，为系统的前后端分离架构提供了坚实的基础。

---

*文档创建日期：2024年*
*版本：1.0.0*