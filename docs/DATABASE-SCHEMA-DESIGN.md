# PostgreSQL 数据库结构设计

## 1. 概述

本文档详细定义了将 Spec Workflow MCP 迁移到 PostgreSQL 数据库的数据模型设计。设计基于对现有系统功能的分析，确保数据结构能够支持所有核心功能，同时提供更好的性能、可扩展性和多用户支持。

## 2. 数据模型设计原则

- **完整性**：确保所有核心功能的数据需求得到满足
- **一致性**：保持数据模型的一致性和可理解性
- **性能**：优化表结构和索引以支持高效查询
- **可扩展性**：设计灵活的数据结构以适应未来需求变化
- **安全性**：包含必要的安全机制如权限控制和数据隔离

## 3. 详细数据库模式

### 3.1. 用户管理相关表

#### `users` 表

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'user' NOT NULL, -- admin, manager, user
    avatar_url TEXT,
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE(username, email)
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type User struct {
    ID           uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    Username     string     `gorm:"type:varchar(50);uniqueIndex;not null" json:"username"`
    Email        string     `gorm:"type:varchar(255);uniqueIndex;not null" json:"email"`
    PasswordHash string     `gorm:"type:varchar(255);not null" json:"-"`
    FullName     string     `gorm:"type:varchar(255)" json:"full_name"`
    Role         string     `gorm:"type:varchar(50);default:'user';not null;index" json:"role"`
    AvatarURL    string     `gorm:"type:text" json:"avatar_url"`
    LastLogin    *time.Time `gorm:"type:timestamp" json:"last_login"`
    IsActive     bool       `gorm:"default:true;not null" json:"is_active"`
    CreatedAt    time.Time  `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt    time.Time  `gorm:"autoUpdateTime" json:"updated_at"`
}
```

#### `user_sessions` 表

```sql
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(512) NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_user_sessions_token ON user_sessions(token);
CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions(expires_at);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type UserSession struct {
    ID           uuid.UUID `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    UserID       uuid.UUID `gorm:"type:uuid;not null;index" json:"user_id"`
    User         User      `gorm:"foreignKey:UserID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    Token        string    `gorm:"type:varchar(512);not null;index" json:"token"`
    IPAddress    string    `gorm:"type:varchar(45)" json:"ip_address"`
    UserAgent    string    `gorm:"type:text" json:"user_agent"`
    ExpiresAt    time.Time `gorm:"type:timestamp;not null;index" json:"expires_at"`
    CreatedAt    time.Time `gorm:"autoCreateTime" json:"created_at"`
    LastActivity time.Time `gorm:"autoUpdateTime" json:"last_activity"`
}
```

### 3.2. 项目管理相关表

#### `projects` 表

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    path VARCHAR(1024),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(50) DEFAULT 'active' NOT NULL,
    settings JSONB DEFAULT '{}' NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_projects_name ON projects(name);
CREATE INDEX idx_projects_owner_id ON projects(owner_id);
CREATE INDEX idx_projects_status ON projects(status);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
    "gorm.io/gorm"
)

type Project struct {
    ID          uuid.UUID      `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    Name        string         `gorm:"type:varchar(255);not null;index" json:"name"`
    Description string         `gorm:"type:text" json:"description"`
    Path        string         `gorm:"type:varchar(1024)" json:"path"`
    OwnerID     uuid.UUID      `gorm:"type:uuid;not null;index" json:"owner_id"`
    Owner       User           `gorm:"foreignKey:OwnerID;references:ID;constraint:OnDelete:CASCADE" json:"owner"`
    Status      string         `gorm:"type:varchar(50);default:'active';not null;index" json:"status"`
    Settings    map[string]any `gorm:"type:jsonb;default:'{}';not null" json:"settings"`
    CreatedAt   time.Time      `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt   time.Time      `gorm:"autoUpdateTime" json:"updated_at"`
    DeletedAt   gorm.DeletedAt `gorm:"index" json:"-"`
}
```

#### `project_members` 表

```sql
CREATE TABLE project_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) DEFAULT 'viewer' NOT NULL, -- owner, manager, editor, viewer
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE(project_id, user_id)
);

CREATE INDEX idx_project_members_project_id ON project_members(project_id);
CREATE INDEX idx_project_members_user_id ON project_members(user_id);
CREATE INDEX idx_project_members_role ON project_members(role);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type ProjectMember struct {
    ID        uuid.UUID `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    ProjectID uuid.UUID `gorm:"type:uuid;not null;index" json:"project_id"`
    Project   Project   `gorm:"foreignKey:ProjectID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    UserID    uuid.UUID `gorm:"type:uuid;not null;index" json:"user_id"`
    User      User      `gorm:"foreignKey:UserID;references:ID;constraint:OnDelete:CASCADE" json:"user"`
    Role      string    `gorm:"type:varchar(50);default:'viewer';not null;index" json:"role"`
    JoinedAt  time.Time `gorm:"autoCreateTime" json:"joined_at"`
}

// 确保每个用户在一个项目中只有一个成员记录
func (ProjectMember) TableName() string {
    return "project_members"
}
```

### 3.3. 规范管理相关表

#### `specs` 表

```sql
CREATE TABLE specs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    title VARCHAR(255),
    content TEXT,
    status VARCHAR(50) DEFAULT 'active' NOT NULL, -- active, archived, draft
    type VARCHAR(50) DEFAULT 'standard' NOT NULL, -- standard, technical, requirement
    version VARCHAR(50) DEFAULT '1.0.0' NOT NULL,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    last_modified_by UUID REFERENCES users(id) ON DELETE SET NULL,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE(project_id, name)
);

CREATE INDEX idx_specs_project_id ON specs(project_id);
CREATE INDEX idx_specs_status ON specs(status);
CREATE INDEX idx_specs_type ON specs(type);
CREATE INDEX idx_specs_created_by ON specs(created_by);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type Spec struct {
    ID              uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    ProjectID       uuid.UUID  `gorm:"type:uuid;not null;index" json:"project_id"`
    Project         Project    `gorm:"foreignKey:ProjectID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    Name            string     `gorm:"type:varchar(255);not null" json:"name"`
    Title           string     `gorm:"type:varchar(255)" json:"title"`
    Content         string     `gorm:"type:text" json:"content"`
    Status          string     `gorm:"type:varchar(50);default:'active';not null;index" json:"status"`
    Type            string     `gorm:"type:varchar(50);default:'standard';not null;index" json:"type"`
    Version         string     `gorm:"type:varchar(50);default:'1.0.0';not null" json:"version"`
    CreatedBy       uuid.UUID  `gorm:"type:uuid;not null;index" json:"created_by"`
    CreatedByUser   User       `gorm:"foreignKey:CreatedBy;references:ID;constraint:OnDelete:SET NULL" json:"created_by_user"`
    LastModifiedBy  *uuid.UUID `gorm:"type:uuid" json:"last_modified_by"`
    ModifiedByUser  *User      `gorm:"foreignKey:LastModifiedBy;references:ID;constraint:OnDelete:SET NULL" json:"modified_by_user"`
    LastModified    time.Time  `gorm:"autoUpdateTime;not null" json:"last_modified"`
    CreatedAt       time.Time  `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt       time.Time  `gorm:"autoUpdateTime" json:"updated_at"`
}

// 确保每个项目中的规范名称唯一
func (Spec) TableName() string {
    return "specs"
}
```

#### `spec_versions` 表

```sql
CREATE TABLE spec_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    version VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    created_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_spec_versions_spec_id ON spec_versions(spec_id);
CREATE INDEX idx_spec_versions_version ON spec_versions(version);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type SpecVersion struct {
    ID        uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    SpecID    uuid.UUID  `gorm:"type:uuid;not null;index" json:"spec_id"`
    Spec      Spec       `gorm:"foreignKey:SpecID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    Version   string     `gorm:"type:varchar(50);not null;index" json:"version"`
    Content   string     `gorm:"type:text;not null" json:"content"`
    CreatedBy *uuid.UUID `gorm:"type:uuid" json:"created_by"`
    Creator   *User      `gorm:"foreignKey:CreatedBy;references:ID;constraint:OnDelete:SET NULL" json:"creator"`
    CreatedAt time.Time  `gorm:"autoCreateTime" json:"created_at"`
}
```

### 3.4. 任务管理相关表

#### `tasks` 表

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(50) DEFAULT 'pending' NOT NULL, -- pending, in_progress, completed
    priority INTEGER DEFAULT 0 NOT NULL,
    order_index INTEGER DEFAULT 0 NOT NULL,
    assignee_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    completed_at TIMESTAMP,
    due_date TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_tasks_spec_id ON tasks(spec_id);
CREATE INDEX idx_tasks_project_id ON tasks(project_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_assignee_id ON tasks(assignee_id);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_order_index ON tasks(order_index);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type Task struct {
    ID          uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    SpecID      uuid.UUID  `gorm:"type:uuid;not null;index" json:"spec_id"`
    Spec        Spec       `gorm:"foreignKey:SpecID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    ProjectID   uuid.UUID  `gorm:"type:uuid;not null;index" json:"project_id"`
    Project     Project    `gorm:"foreignKey:ProjectID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    Title       string     `gorm:"type:varchar(255);not null" json:"title"`
    Description string     `gorm:"type:text" json:"description"`
    Status      string     `gorm:"type:varchar(50);default:'pending';not null;index" json:"status"`
    Priority    int        `gorm:"default:0;not null;index" json:"priority"`
    OrderIndex  int        `gorm:"default:0;not null;index" json:"order_index"`
    AssigneeID  *uuid.UUID `gorm:"type:uuid;index" json:"assignee_id"`
    Assignee    *User      `gorm:"foreignKey:AssigneeID;references:ID;constraint:OnDelete:SET NULL" json:"assignee"`
    CreatedBy   uuid.UUID  `gorm:"type:uuid;not null" json:"created_by"`
    Creator     User       `gorm:"foreignKey:CreatedBy;references:ID;constraint:OnDelete:SET NULL" json:"creator"`
    CompletedAt *time.Time `gorm:"type:timestamp" json:"completed_at"`
    DueDate     *time.Time `gorm:"type:timestamp" json:"due_date"`
    CreatedAt   time.Time  `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt   time.Time  `gorm:"autoUpdateTime" json:"updated_at"`
}
```

#### `task_dependencies` 表

```sql
CREATE TABLE task_dependencies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    depends_on_task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE(task_id, depends_on_task_id)
);

CREATE INDEX idx_task_dependencies_task_id ON task_dependencies(task_id);
CREATE INDEX idx_task_dependencies_depends_on ON task_dependencies(depends_on_task_id);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type TaskDependency struct {
    ID                uuid.UUID `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    TaskID            uuid.UUID `gorm:"type:uuid;not null;index" json:"task_id"`
    Task              Task      `gorm:"foreignKey:TaskID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    DependsOnTaskID   uuid.UUID `gorm:"type:uuid;not null;index" json:"depends_on_task_id"`
    DependsOnTask     Task      `gorm:"foreignKey:DependsOnTaskID;references:ID;constraint:OnDelete:CASCADE" json:"depends_on_task"`
    CreatedAt         time.Time `gorm:"autoCreateTime" json:"created_at"`
}

// 确保依赖关系唯一
func (TaskDependency) TableName() string {
    return "task_dependencies"
}
```

### 3.5. 审批工作流相关表

#### `approvals` 表

```sql
CREATE TABLE approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status VARCHAR(50) DEFAULT 'pending' NOT NULL, -- pending, approved, rejected
    requested_by UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    approved_by UUID REFERENCES users(id) ON DELETE SET NULL,
    comments TEXT,
    requested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    responded_at TIMESTAMP,
    expires_at TIMESTAMP
);

CREATE INDEX idx_approvals_spec_id ON approvals(spec_id);
CREATE INDEX idx_approvals_project_id ON approvals(project_id);
CREATE INDEX idx_approvals_status ON approvals(status);
CREATE INDEX idx_approvals_requested_by ON approvals(requested_by);
CREATE INDEX idx_approvals_expires_at ON approvals(expires_at);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type Approval struct {
    ID          uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    SpecID      uuid.UUID  `gorm:"type:uuid;not null;index" json:"spec_id"`
    Spec        Spec       `gorm:"foreignKey:SpecID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    ProjectID   uuid.UUID  `gorm:"type:uuid;not null;index" json:"project_id"`
    Project     Project    `gorm:"foreignKey:ProjectID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    Status      string     `gorm:"type:varchar(50);default:'pending';not null;index" json:"status"`
    RequestedBy uuid.UUID  `gorm:"type:uuid;not null;index" json:"requested_by"`
    Requester   User       `gorm:"foreignKey:RequestedBy;references:ID;constraint:OnDelete:SET NULL" json:"requester"`
    ApprovedBy  *uuid.UUID `gorm:"type:uuid" json:"approved_by"`
    Approver    *User      `gorm:"foreignKey:ApprovedBy;references:ID;constraint:OnDelete:SET NULL" json:"approver"`
    Comments    string     `gorm:"type:text" json:"comments"`
    RequestedAt time.Time  `gorm:"autoCreateTime" json:"requested_at"`
    RespondedAt *time.Time `gorm:"type:timestamp" json:"responded_at"`
    ExpiresAt   *time.Time `gorm:"type:timestamp;index" json:"expires_at"`
}
```

#### `approval_history` 表

```sql
CREATE TABLE approval_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    approval_id UUID NOT NULL REFERENCES approvals(id) ON DELETE CASCADE,
    status VARCHAR(50) NOT NULL,
    changed_by UUID REFERENCES users(id) ON DELETE SET NULL,
    comments TEXT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_approval_history_approval_id ON approval_history(approval_id);
CREATE INDEX idx_approval_history_status ON approval_history(status);
CREATE INDEX idx_approval_history_changed_at ON approval_history(changed_at);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type ApprovalHistory struct {
    ID         uuid.UUID  `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    ApprovalID uuid.UUID  `gorm:"type:uuid;not null;index" json:"approval_id"`
    Approval   Approval   `gorm:"foreignKey:ApprovalID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    Status     string     `gorm:"type:varchar(50);not null;index" json:"status"`
    ChangedBy  *uuid.UUID `gorm:"type:uuid" json:"changed_by"`
    Changer    *User      `gorm:"foreignKey:ChangedBy;references:ID;constraint:OnDelete:SET NULL" json:"changer"`
    Comments   string     `gorm:"type:text" json:"comments"`
    ChangedAt  time.Time  `gorm:"autoCreateTime" json:"changed_at"`
}
```

### 3.6. 实现日志相关表

#### `implementation_logs` 表

```sql
CREATE TABLE implementation_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id UUID REFERENCES specs(id) ON DELETE CASCADE,
    task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    content TEXT NOT NULL,
    artifacts JSONB DEFAULT '{}' NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_implementation_logs_spec_id ON implementation_logs(spec_id);
CREATE INDEX idx_implementation_logs_task_id ON implementation_logs(task_id);
CREATE INDEX idx_implementation_logs_project_id ON implementation_logs(project_id);
CREATE INDEX idx_implementation_logs_user_id ON implementation_logs(user_id);
CREATE INDEX idx_implementation_logs_created_at ON implementation_logs(created_at);
```

对应的 Go 结构体：

```go
package models

import (
    "time"
    "github.com/google/uuid"
)

type ImplementationLog struct {
    ID        uuid.UUID      `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    SpecID    *uuid.UUID     `gorm:"type:uuid;index" json:"spec_id"`
    Spec      *Spec          `gorm:"foreignKey:SpecID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    TaskID    *uuid.UUID     `gorm:"type:uuid;index" json:"task_id"`
    Task      *Task          `gorm:"foreignKey:TaskID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    ProjectID uuid.UUID      `gorm:"type:uuid;not null;index" json:"project_id"`
    Project   Project        `gorm:"foreignKey:ProjectID;references:ID;constraint:OnDelete:CASCADE" json:"-"`
    UserID    uuid.UUID      `gorm:"type:uuid;not null;index" json:"user_id"`
    User      User           `gorm:"foreignKey:UserID;references:ID;constraint:OnDelete:SET NULL" json:"user"`
    Content   string         `gorm:"type:text;not null" json:"content"`
    Artifacts map[string]any `gorm:"type:jsonb;default:'{}';not null" json:"artifacts"`
    CreatedAt time.Time      `gorm:"autoCreateTime" json:"created_at"`
}
```

### 3.7. MCP 服务相关表

#### `tools` 表

```go
package models

import (
    "github.com/google/uuid"
)

type Tool struct {
    ID          uuid.UUID      `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    Name        string         `gorm:"type:varchar(255);uniqueIndex;not null" json:"name"`
    Description string         `gorm:"type:text" json:"description"`
    Parameters  map[string]any `gorm:"type:jsonb;default:'{}'" json:"parameters"`
    IsEnabled   bool           `gorm:"default:true" json:"is_enabled"`
    Category    string         `gorm:"type:varchar(100)" json:"category"`
}
```

#### `prompts` 表

```go
package models

import (
    "github.com/google/uuid"
)

type Prompt struct {
    ID          uuid.UUID `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    Name        string    `gorm:"type:varchar(255);uniqueIndex;not null" json:"name"`
    Content     string    `gorm:"type:text;not null" json:"content"`
    Description string    `gorm:"type:text" json:"description"`
    Category    string    `gorm:"type:varchar(100)" json:"category"`
    IsSystem    bool      `gorm:"default:false" json:"is_system"`
}
```

## 4. 数据关系图

```
users ───────────────────┐
  │                      │
  ├─► user_sessions      │
  │                      │
  ├─► projects (owner)   │
  │                      │
  ├─► project_members    │
  │                      │
  ├─► specs (created_by, last_modified_by) │
  │                      │
  ├─► tasks (assignee, created_by)         │
  │                      │
  ├─► approvals (requested_by, approved_by)│
  │                      │
  └─► implementation_logs (user_id)        │
                      │
projects ──────────────┤
  │                    │
  ├─► specs            │
  │                    │
  ├─► tasks            │
  │                    │
  └─► approvals        │
                      │
specs ────────────────┤
  │                    │
  ├─► spec_versions    │
  │                    │
  ├─► tasks            │
  │                    │
  ├─► approvals        │
  │                    │
  └─► implementation_logs
                      │
tasks ────────────────┤
  │                    │
  ├─► task_dependencies│
  │                    │
  └─► implementation_logs
                      │
approvals ────────────┘
  │
  └─► approval_history
```

## 5. 数据库初始化和迁移

### 5.1. 数据库初始化脚本

```sql
-- 创建扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 创建表空间（可选）
CREATE TABLESPACE spec_workflow_ts OWNER postgres LOCATION '/var/lib/postgresql/data/tablespaces/spec_workflow';

-- 创建数据库
CREATE DATABASE spec_workflow WITH OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = spec_workflow_ts
    CONNECTION LIMIT = -1;

-- 创建架构
CREATE SCHEMA IF NOT EXISTS spec_workflow;

-- 设置默认架构
ALTER DATABASE spec_workflow SET search_path TO spec_workflow, public;
```

### 5.2. 数据迁移策略

1. **数据提取**：从现有文件系统中提取项目、规范、任务等数据
2. **数据转换**：将提取的数据转换为符合新数据模型的格式
3. **用户映射**：创建初始用户并映射到相应的资源
4. **关系建立**：确保所有外键关系正确建立
5. **验证**：验证迁移后的数据完整性和一致性

## 6. 数据库优化建议

### 6.1. 索引优化
- 为频繁查询的列创建索引
- 为外键列创建索引以加速连接操作
- 考虑复合索引以优化多列查询

### 6.2. 查询优化
- 使用 `EXPLAIN ANALYZE` 分析查询性能
- 避免全表扫描，确保查询使用适当的索引
- 考虑使用分区表优化大型表的查询性能

### 6.3. 连接池管理
- 配置适当的连接池大小
- 实现连接复用机制
- 监控连接使用情况

### 6.4. 缓存策略
- 实现应用层缓存以减少数据库查询
- 考虑使用 Redis 缓存热点数据
- 实现缓存失效策略以确保数据一致性

## 7. 总结

本数据模型设计提供了一个完整的 PostgreSQL 数据库结构，用于支持 Spec Workflow MCP 系统的所有核心功能。设计考虑了多用户支持、数据完整性、查询性能和可扩展性等方面，为系统迁移提供了坚实的数据基础。

---

*文档创建日期：2024年*
*版本：1.0.0*