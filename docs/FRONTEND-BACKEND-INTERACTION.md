# 前后端交互和远程管理功能规划

## 1. 概述

本文档详细规划了Spec Workflow MCP系统重构后的前后端交互架构和远程管理功能设计。主要关注前端架构设计、API交互模式、实时通信机制、权限控制策略以及远程管理功能实现方案，旨在为开发团队提供清晰的技术指导。

## 2. 前端架构设计

### 2.1 技术栈选择

| 分类 | 技术/库 | 版本 | 用途 |
|------|---------|------|------|
| 核心框架 | React | 18.x | UI组件库 |
| 状态管理 | Redux Toolkit | 1.9.x | 全局状态管理 |
| 路由管理 | React Router | 6.x | 前端路由 |
| UI组件库 | Ant Design | 5.x | UI组件 |
| API通信 | Axios | 1.x | HTTP请求 |
| WebSocket | Socket.IO Client | 4.x | 实时通信 |
| 表单处理 | React Hook Form | 7.x | 表单管理 |
| 类型系统 | TypeScript | 5.x | 类型安全 |
| 构建工具 | Vite | 4.x | 构建和开发 |

### 2.2 组件架构

```
├── App.tsx                # 应用入口
├── main.tsx               # 渲染入口
├── components/            # 通用组件
│   ├── common/            # 基础UI组件
│   ├── layouts/           # 布局组件
│   └── widgets/           # 业务组件
├── pages/                 # 页面组件
│   ├── Auth/              # 认证相关页面
│   ├── Projects/          # 项目管理页面
│   ├── Specifications/    # 规范管理页面
│   ├── Tasks/             # 任务管理页面
│   ├── Dashboard/         # 仪表盘页面
│   └── Settings/          # 设置页面
├── services/              # API服务
│   ├── api.ts             # Axios配置
│   ├── auth.ts            # 认证服务
│   ├── projects.ts        # 项目服务
│   ├── specifications.ts  # 规范服务
│   ├── tasks.ts           # 任务服务
│   └── websocket.ts       # WebSocket服务
├── store/                 # Redux状态管理
│   ├── slices/            # Redux切片
│   └── index.ts           # Store配置
├── hooks/                 # 自定义Hooks
├── utils/                 # 工具函数
└── types/                 # TypeScript类型定义
```

### 2.3 核心模块

#### 2.3.1 认证模块

- **功能**：用户登录、注册、登出、会话管理
- **组件**：LoginPage、RegisterPage、PasswordResetPage
- **状态管理**：authSlice
- **工具**：JWT处理、本地存储管理

#### 2.3.2 项目管理模块

- **功能**：项目列表、创建、编辑、删除、成员管理
- **组件**：ProjectList、ProjectCreate、ProjectDetail、ProjectMembers
- **状态管理**：projectsSlice

#### 2.3.3 规范管理模块

- **功能**：规范列表、创建、编辑、查看、版本管理、归档
- **组件**：SpecList、SpecCreate、SpecEditor、SpecViewer、VersionHistory
- **状态管理**：specificationsSlice
- **特殊功能**：Markdown编辑器、任务提取

#### 2.3.4 任务管理模块

- **功能**：任务列表、分配、状态更新、优先级调整
- **组件**：TaskBoard、TaskList、TaskDetail、TaskFilter
- **状态管理**：tasksSlice
- **特殊功能**：看板视图、拖放操作

## 3. API交互模式

### 3.1 基础配置

```typescript
// services/api.ts
import axios from 'axios';

const API_BASE_URL = '/api/v1';

const api = axios.create({
  baseURL: API_BASE_URL,
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 请求拦截器
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// 响应拦截器
api.interceptors.response.use(
  (response) => {
    return response.data;
  },
  (error) => {
    // 处理错误响应
    if (error.response?.status === 401) {
      // 未授权，清除token并跳转登录
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### 3.2 资源封装模式

```typescript
// services/projects.ts
import api from './api';
import { Project, ProjectCreate, ProjectUpdate } from '../types/project';

const projectService = {
  // 获取项目列表
  getProjects: async (params: { page?: number; limit?: number; search?: string }) => {
    return api.get<Project[]>('/projects', { params });
  },

  // 创建项目
  createProject: async (project: ProjectCreate) => {
    return api.post<Project>('/projects', project);
  },

  // 获取项目详情
  getProject: async (id: string) => {
    return api.get<Project>(`/projects/${id}`);
  },

  // 更新项目
  updateProject: async (id: string, project: ProjectUpdate) => {
    return api.put<Project>(`/projects/${id}`, project);
  },

  // 删除项目
  deleteProject: async (id: string) => {
    return api.delete(`/projects/${id}`);
  },
};

export default projectService;
```

### 3.3 错误处理模式

- **统一错误处理**：通过Axios拦截器统一处理HTTP错误
- **错误展示组件**：创建错误提示组件
- **重试机制**：对特定错误实现自动重试
- **用户友好提示**：将技术错误转换为用户可理解的信息

## 4. WebSocket实时通信

### 4.1 WebSocket服务配置

```typescript
// services/websocket.ts
import { io, Socket } from 'socket.io-client';

class WebSocketService {
  private socket: Socket | null = null;

  connect(token: string) {
    if (!this.socket) {
      this.socket = io(process.env.REACT_APP_WS_URL || '/', {
        auth: {
          token,
        },
        reconnection: true,
        reconnectionAttempts: 5,
        reconnectionDelay: 1000,
      });

      this.setupEventListeners();
    }
    return this.socket;
  }

  private setupEventListeners() {
    if (!this.socket) return;

    this.socket.on('connect', () => {
      console.log('WebSocket connected');
    });

    this.socket.on('disconnect', () => {
      console.log('WebSocket disconnected');
    });

    this.socket.on('error', (error) => {
      console.error('WebSocket error:', error);
    });
  }

  subscribe(channels: string[]) {
    if (this.socket) {
      this.socket.emit('subscribe', { channels });
    }
  }

  unsubscribe(channels: string[]) {
    if (this.socket) {
      this.socket.emit('unsubscribe', { channels });
    }
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }
}

export default new WebSocketService();
```

### 4.2 实时事件订阅

```typescript
// hooks/useWebSocket.ts
import { useEffect } from 'react';
import { useAppDispatch } from '../store';
import websocketService from '../services/websocket';

interface UseWebSocketOptions {
  channels?: string[];
  eventHandlers?: Record<string, (data: any) => void>;
}

const useWebSocket = ({ channels = [], eventHandlers = {} }: UseWebSocketOptions) => {
  const dispatch = useAppDispatch();

  useEffect(() => {
    const token = localStorage.getItem('authToken');
    if (!token) return;

    const socket = websocketService.connect(token);
    if (!socket) return;

    // 订阅指定通道
    if (channels.length > 0) {
      websocketService.subscribe(channels);
    }

    // 注册事件处理器
    Object.entries(eventHandlers).forEach(([event, handler]) => {
      socket.on(event, handler);
    });

    return () => {
      // 取消事件监听
      Object.entries(eventHandlers).forEach(([event]) => {
        socket.off(event);
      });

      // 取消订阅
      if (channels.length > 0) {
        websocketService.unsubscribe(channels);
      }

      // 组件卸载时不需要断开连接，使用全局连接池
    };
  }, [channels, eventHandlers, dispatch]);
};

export default useWebSocket;
```

### 4.3 实时事件类型

| 事件名称 | 描述 | 数据结构 |
|---------|------|----------|
| `task_status_changed` | 任务状态变更 | `{ task_id: string, status: string }` |
| `spec_updated` | 规范内容更新 | `{ spec_id: string, title: string }` |
| `approval_status_changed` | 审批状态变更 | `{ approval_id: string, status: string }` |
| `new_comment` | 新评论通知 | `{ project_id: string, user: User, content: string }` |
| `member_added` | 新成员加入 | `{ project_id: string, member: Member }` |

## 5. 远程管理功能

### 5.1 用户管理

#### 5.1.1 用户认证流程

1. **登录流程**：
   - 用户输入凭据
   - 前端验证并发送到后端
   - 后端验证凭据并返回JWT
   - 前端存储JWT并设置用户状态
   - 建立WebSocket连接

2. **会话管理**：
   - JWT存储在localStorage
   - 自动刷新即将过期的令牌
   - 长时间不活动自动登出

#### 5.1.2 用户角色管理

- **角色类型**：admin、manager、editor、viewer
- **权限分配**：
  - 基于角色的API访问控制
  - 基于角色的UI组件显示控制
  - 动态权限检查

### 5.2 项目远程管理

#### 5.2.1 项目仪表盘

- **核心指标**：
  - 项目进度概览
  - 规范完成率
  - 任务状态分布
  - 最近活动记录

- **数据可视化**：
  - 使用Chart.js绘制项目统计图表
  - 实时更新项目状态

#### 5.2.2 远程协作功能

- **实时编辑**：
  - 规范内容实时协作编辑
  - 编辑冲突解决
  - 编辑历史记录

- **通知系统**：
  - 项目活动通知
  - 任务分配通知
  - 审批请求通知
  - 支持邮件和系统内通知

### 5.3 规范远程管理

#### 5.3.1 规范在线编辑器

- **功能特性**：
  - Markdown实时预览
  - 语法高亮
  - 自动保存
  - 版本历史对比
  - 任务自动提取

- **协作功能**：
  - 多用户同时编辑
  - 用户光标位置显示
  - 编辑锁定机制

#### 5.3.2 规范版本控制

- **版本管理**：
  - 自动版本创建
  - 版本命名和标签
  - 版本比较和差异显示
  - 版本回滚功能

### 5.4 任务远程管理

#### 5.4.1 任务看板

- **功能特性**：
  - 拖拽任务变更状态
  - 按状态、优先级、负责人分组
  - 任务卡片展示
  - 搜索和筛选功能

- **数据同步**：
  - 实时状态更新
  - 多用户操作同步

#### 5.4.2 任务提醒和通知

- **提醒类型**：
  - 任务截止提醒
  - 任务分配通知
  - 任务评论通知

- **提醒方式**：
  - 系统内消息
  - 邮件通知
  - 浏览器桌面通知

## 6. 权限控制策略

### 6.1 前端权限控制

#### 6.1.1 路由权限

```typescript
// routes/index.tsx
import { Navigate, useRoutes } from 'react-router-dom';
import { useSelector } from 'react-redux';
import { selectUserRole } from '../store/slices/authSlice';

const ProtectedRoute = ({ children, requiredRole }: { children: React.ReactNode; requiredRole?: string }) => {
  const userRole = useSelector(selectUserRole);
  const isAuthenticated = !!userRole;

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  // 角色权限检查
  if (requiredRole && !hasPermission(userRole, requiredRole)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return children;
};

const roleHierarchy = {
  viewer: 1,
  editor: 2,
  manager: 3,
  admin: 4,
};

const hasPermission = (userRole: string, requiredRole: string) => {
  return roleHierarchy[userRole] >= roleHierarchy[requiredRole];
};
```

#### 6.1.2 组件权限控制

```typescript
// components/common/PermissionGate.tsx
import { useSelector } from 'react-redux';
import { selectUserRole, selectCurrentProjectRole } from '../store/slices/authSlice';

interface PermissionGateProps {
  children: React.ReactNode;
  role?: string; // 全局角色要求
  projectRole?: string; // 项目内角色要求
}

const PermissionGate: React.FC<PermissionGateProps> = ({ children, role, projectRole }) => {
  const globalRole = useSelector(selectUserRole);
  const currentProjectRole = useSelector(selectCurrentProjectRole);

  // 全局角色检查
  if (role && !hasPermission(globalRole, role)) {
    return null;
  }

  // 项目内角色检查
  if (projectRole && !hasPermission(currentProjectRole, projectRole)) {
    return null;
  }

  return children;
};

export default PermissionGate;
```

### 6.2 后端权限验证

- **中间件验证**：在Go Gin中使用中间件进行权限验证
- **数据库查询过滤**：确保查询结果只包含用户有权访问的数据
- **操作权限检查**：验证用户是否有权执行特定操作
- **API端点保护**：根据角色限制API访问

## 7. 用户体验设计

### 7.1 响应式设计

- **断点设计**：
  - 移动端：< 768px
  - 平板：768px - 1024px
  - 桌面端：> 1024px

- **布局适配**：
  - 侧边栏自动折叠
  - 任务看板布局变化
  - 表单字段重排

### 7.2 性能优化

- **懒加载**：
  - 路由懒加载
  - 组件懒加载
  - 图片懒加载

- **数据缓存**：
  - Redux持久化
  - API响应缓存
  - 本地数据存储

- **渲染优化**：
  - 虚拟列表
  - 组件拆分和按需渲染
  - 避免不必要的重渲染

### 7.3 交互体验

- **加载状态**：
  - 全局加载指示器
  - 组件级加载状态
  - 骨架屏

- **微交互**：
  - 按钮反馈
  - 表单验证提示
  - 状态切换动画

- **错误处理**：
  - 用户友好的错误提示
  - 操作撤销功能
  - 自动保存草稿

## 8. 数据同步策略

### 8.1 离线支持

- **数据缓存**：
  - 关键数据本地存储
  - 操作队列管理
  - 冲突检测和解决

- **同步机制**：
  - 定期同步检查
  - 网络恢复时自动同步
  - 手动触发同步

### 8.2 实时数据同步

- **WebSocket同步**：
  - 增量数据更新
  - 事件驱动更新
  - 乐观UI更新

- **冲突解决**：
  - 基于时间戳的冲突检测
  - 用户选择的冲突解决
  - 自动合并策略

## 9. 安全性考虑

### 9.1 前端安全

- **XSS防护**：
  - 内容安全策略(CSP)
  - 输入验证和净化
  - React自动转义

- **CSRF防护**：
  - CSRF令牌
  - SameSite Cookie策略

- **敏感信息保护**：
  - 本地存储数据加密
  - 避免在客户端存储敏感数据

### 9.2 传输安全

- **HTTPS**：全程使用HTTPS
- **证书验证**：确保服务器证书有效
- **API密钥管理**：安全存储和传输API密钥

## 10. 测试策略

### 10.1 单元测试

- **组件测试**：使用Jest和React Testing Library
- **服务测试**：API服务和WebSocket服务测试
- **状态测试**：Redux状态和逻辑测试

### 10.2 集成测试

- **API集成测试**：前后端API交互测试
- **端到端测试**：使用Cypress进行E2E测试
- **性能测试**：页面加载和响应时间测试

## 11. 部署与CI/CD

### 11.1 前端部署

- **构建优化**：
  - 代码分割
  - 资源压缩
  - 树摇优化

- **部署选项**：
  - CDN部署
  - 容器化部署
  - 静态托管服务

### 11.2 CI/CD流程

- **自动化流程**：
  - 代码提交触发构建
  - 自动化测试
  - 代码质量检查
  - 自动部署到测试环境
  - 手动确认后部署到生产环境

- **监控告警**：
  - 前端错误监控
  - 性能指标监控
  - 用户行为分析

## 12. 结论与建议

### 12.1 关键建议

1. **渐进式实现**：先实现核心功能，再逐步添加高级特性
2. **用户体验优先**：确保远程管理功能易用性和效率
3. **安全性重视**：尤其注意身份验证和授权部分的实现
4. **性能优化**：为远程用户提供流畅的使用体验
5. **可扩展性设计**：为未来功能扩展预留空间

### 12.2 注意事项

1. **网络依赖性**：考虑弱网络环境下的用户体验
2. **数据一致性**：确保多用户并发操作时的数据一致性
3. **安全性保障**：实施全面的安全措施，保护用户数据
4. **可访问性**：确保系统符合可访问性标准
5. **国际化支持**：考虑未来的多语言需求

---

*文档创建日期：2024年*
*版本：1.0.0*