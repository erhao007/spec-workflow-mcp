# Spec Workflow MCP 仪表板系统深度分析

## 概述

本文档详细分析了 Spec Workflow MCP 的仪表板系统，包括后端服务器架构、前端React应用、实时通信机制和项目管理功能。仪表板为规范驱动的开发工作流提供了可视化的Web界面。

## 系统架构概览

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    仪表板系统架构                             │
├─────────────────────────────────────────────────────────────┤
│  前端 (React + TypeScript)                                  │
│  ├── 实时UI组件 (WebSocket)                                  │
│  ├── 多页面路由 (React Router)                              │
│  ├── 国际化支持 (i18n)                                      │
│  └── 状态管理 (Context + Hooks)                             │
├─────────────────────────────────────────────────────────────┤
│  API层 (HTTP + WebSocket)                                   │
│  ├── REST API (Fastify)                                     │
│  ├── 实时通信 (WebSocket)                                   │
│  ├── 静态文件服务                                           │
│  └── 跨域处理 (CORS)                                        │
├─────────────────────────────────────────────────────────────┤
│  业务逻辑层                                                  │
│  ├── 项目管理器 (ProjectManager)                            │
│  ├── 文件监控器 (SpecWatcher)                               │
│  ├── 审批存储 (ApprovalStorage)                             │
│  ├── 任务调度器 (JobScheduler)                              │
│  └── 会话管理 (DashboardSessionManager)                     │
├─────────────────────────────────────────────────────────────┤
│  核心服务层                                                  │
│  ├── 规范解析器 (SpecParser)                                │
│  ├── 任务解析器 (TaskParser)                                │
│  ├── 路径工具 (PathUtils)                                   │
│  └── 归档服务 (SpecArchiveService)                          │
└─────────────────────────────────────────────────────────────┘
```

### 模块结构

#### 后端模块 (`src/dashboard/`)
```
src/dashboard/
├── multi-server.ts          # 主服务器入口，HTTP/WebSocket服务
├── project-manager.ts       # 项目管理和多项目协调
├── parser.ts                # 规范文档解析器
├── watcher.ts               # 文件系统监控和实时更新
├── approval-storage.ts      # 审批存储和版本管理
├── job-scheduler.ts         # 定时任务调度器
├── settings-manager.ts     # 全局配置管理
├── implementation-log-manager.ts  # 实现日志管理
├── utils.ts                 # 工具函数和端口管理
└── public/
    ├── claude-icon.svg      # 图标资源
    └── claude-icon-dark.svg # 深色图标资源
```

#### 前端模块 (`src/dashboard_frontend/`)
```
src/dashboard_frontend/
├── src/
│   ├── App.tsx              # 主应用组件
│   ├── main.tsx            # 应用入口
│   ├── components/         # 可复用组件库
│   │   ├── ui/            # 基础UI组件
│   │   ├── forms/         # 表单组件
│   │   ├── layout/        # 布局组件
│   │   └── charts/        # 图表组件
│   ├── pages/             # 页面组件
│   │   ├── Dashboard.tsx  # 主仪表板
│   │   ├── Specs.tsx      # 规范管理
│   │   ├── Tasks.tsx      # 任务管理
│   │   └── Settings.tsx   # 设置页面
│   ├── hooks/             # 自定义Hooks
│   ├── services/          # API服务层
│   ├── types/             # TypeScript类型定义
│   ├── utils/             # 工具函数
│   └── locales/           # 国际化资源
├── public/                # 静态资源
├── package.json           # 依赖配置
└── vite.config.ts         # 构建配置
```

## 后端服务器深度分析

### MultiProjectDashboardServer 主服务器

#### 核心特性
- **多项目支持**：同时管理多个开发项目
- **实时通信**：WebSocket + 事件驱动架构
- **RESTful API**：24个API端点覆盖完整功能
- **静态文件服务**：支持SPA和资源服务
- **跨项目协调**：项目间的状态同步和事件传播

#### 服务器配置
```typescript
const app = fastify({
  logger: false  // 生产环境中关闭内部日志
});

// 静态文件服务配置
await app.register(fastifyStatic, {
  root: join(__dirname, 'public'),
  prefix: '/',
});

// WebSocket插件注册
await app.register(fastifyWebsocket);

// 多项目协调器初始化
this.projectManager = new ProjectManager();
```

#### 项目管理架构
```typescript
class ProjectManager extends EventEmitter {
  private projects: Map<string, ProjectContext> = new Map();

  // 项目上下文结构
  interface ProjectContext {
    projectId: string;
    projectPath: string;
    projectName: string;
    parser: SpecParser;
    watcher: SpecWatcher;
    approvalStorage: ApprovalStorage;
    archiveService: SpecArchiveService;
  }
}
```

### REST API端点详细分析

#### 项目管理API (4个端点)
```typescript
// 获取所有项目
GET /api/projects/list
Response: {
  projects: Array<{
    projectId: string;
    projectPath: string;
    projectName: string;
    activeSpecs: number;
    totalSpecs: number;
  }>
}

// 添加新项目
POST /api/projects/add
Body: { projectPath: string }
Response: { success: boolean, projectId: string }

// 获取项目信息
GET /api/projects/:projectId/info
Response: {
  project: ProjectContext;
  stats: {
    totalSpecs: number;
    activeSpecs: number;
    archivedSpecs: number;
    completedSpecs: number;
  }
}

// 删除项目
DELETE /api/projects/:projectId
Response: { success: boolean, message: string }
```

#### 规范管理API (6个端点)
```typescript
// 获取规范列表
GET /api/projects/:projectId/specs
Response: {
  specs: Array<{
    name: string;
    displayName: string;
    status: string;
    phases: PhaseStatus;
    taskProgress?: TaskProgress;
  }>;
  archivedSpecs: string[];
}

// 获取规范详情
GET /api/projects/:projectId/specs/:name
Response: {
  spec: ParsedSpec;
  documents: {
    requirements?: string;
    design?: string;
    tasks?: string;
  };
}

// 获取所有文档内容
GET /api/projects/:projectId/specs/:name/all
Response: {
  requirements: string | null;
  design: string | null;
  tasks: string | null;
  implementationLogs: Array<LogEntry>;
}

// 保存文档
PUT /api/projects/:projectId/specs/:name/:document
Body: { content: string }
Response: { success: boolean, message: string }

// 归档规范
POST /api/projects/:projectId/specs/:name/archive
Response: { success: boolean, message: string }

// 获取归档规范
GET /api/projects/:projectId/specs/archived
Response: { archivedSpecs: string[] }
```

#### 审批工作流API (4个端点)
```typescript
// 获取审批列表
GET /api/projects/:projectId/approvals
Response: {
  approvals: Array<{
    id: string;
    title: string;
    filePath: string;
    type: 'document' | 'action';
    status: 'pending' | 'approved' | 'rejected' | 'needs-revision';
    createdAt: string;
    respondedAt?: string;
  }>;
}

// 创建审批请求
POST /api/projects/:projectId/approvals
Body: {
  title: string;
  filePath: string;
  type: 'document' | 'action';
  categoryName: string;
}
Response: { success: boolean, approvalId: string }

// 处理审批
PUT /api/projects/:projectId/approvals/:id
Body: {
  action: 'approve' | 'reject' | 'request-revision';
  response?: string;
}
Response: { success: boolean, message: string }

// 获取审批快照
GET /api/projects/:projectId/approvals/:id/snapshots
Response: {
  snapshots: Array<{
    timestamp: string;
    content: string;
    type: 'original' | 'current';
  }>;
}
```

#### 任务管理API (2个端点)
```typescript
// 获取任务进度
GET /api/projects/:projectId/specs/:name/tasks/progress
Response: {
  tasks: Array<{
    id: string;
    description: string;
    status: 'pending' | 'in-progress' | 'completed';
    lineNumber: number;
    indentLevel: number;
  }>;
  progress: {
    total: number;
    completed: number;
    pending: number;
    percentage: number;
  };
}

// 更新任务状态
PUT /api/projects/:projectId/specs/:name/tasks/:taskId/status
Body: { status: 'pending' | 'in-progress' | 'completed' }
Response: { success: boolean, message: string }
```

#### 实现日志API (2个端点)
```typescript
// 添加实现日志
POST /api/projects/:projectId/specs/:name/implementation-log
Body: {
  taskId: string;
  summary: string;
  filesModified: string[];
  filesCreated: string[];
  statistics: { linesAdded: number; linesRemoved: number };
  artifacts: LogArtifacts;
}
Response: { success: boolean, logId: string }

// 查询实现日志
GET /api/projects/:projectId/specs/:name/implementation-log
Query: { taskId?: string; search?: string }
Response: {
  entries: Array<{
    id: string;
    taskId: string;
    summary: string;
    timestamp: string;
    artifacts: LogArtifacts;
  }>;
}
```

#### 自动化任务API (6个端点)
```typescript
// 获取所有任务
GET /api/jobs
Response: {
  jobs: Array<{
    id: string;
    name: string;
    schedule: string;
    enabled: boolean;
    lastRun?: string;
    nextRun?: string;
  }>;
}

// 创建任务
POST /api/jobs
Body: {
  name: string;
  schedule: string;
  command: string;
  enabled: boolean;
}
Response: { success: boolean, jobId: string }

// 更新任务
PUT /api/jobs/:jobId
Body: { name?: string; schedule?: string; enabled?: boolean }
Response: { success: boolean, message: string }

// 删除任务
DELETE /api/jobs/:jobId
Response: { success: boolean, message: string }

// 手动执行任务
POST /api/jobs/:jobId/run
Response: { success: boolean, executionId: string }

// 获取执行历史
GET /api/jobs/:jobId/history
Response: {
  executions: Array<{
    id: string;
    startTime: string;
    endTime?: string;
    status: 'running' | 'completed' | 'failed';
    output?: string;
  }>;
}
```

### WebSocket实时通信系统

#### 连接管理
```typescript
interface WebSocketConnection {
  socket: WebSocket;
  projectId?: string;
}

class MultiProjectDashboardServer {
  private clients: Set<WebSocketConnection> = new Set();

  // 连接处理
  app.get('/ws', { websocket: true }, (connection, req) => {
    const wsConnection: WebSocketConnection = {
      socket: connection.socket,
      projectId: undefined
    };

    this.clients.add(wsConnection);

    // 连接关闭时清理
    connection.socket.on('close', () => {
      this.clients.delete(wsConnection);
    });
  });
}
```

#### 实时事件类型
```typescript
// 项目级别事件
interface ProjectEvent {
  type: 'projects-update';
  data: {
    projects: ProjectContext[];
  };
}

// 规范变更事件
interface SpecEvent {
  type: 'spec-update';
  projectId: string;
  data: {
    specs: ParsedSpec[];
    archivedSpecs: string[];
  };
}

// 任务状态变更事件
interface TaskEvent {
  type: 'task-status-update';
  projectId: string;
  specName: string;
  data: {
    taskId: string;
    status: string;
    progress: TaskProgress;
  };
}

// 审批变更事件
interface ApprovalEvent {
  type: 'approval-update';
  projectId: string;
  data: {
    approvalId: string;
    status: string;
    response?: string;
  };
}

// 指导文档变更事件
interface SteeringEvent {
  type: 'steering-update';
  projectId: string;
  data: {
    documents: {
      product?: boolean;
      tech?: boolean;
      structure?: boolean;
    };
  };
}

// 实现日志更新事件
interface LogEvent {
  type: 'implementation-log-update';
  projectId: string;
  specName: string;
  data: {
    logId: string;
    taskId: string;
    summary: string;
  };
}
```

#### 事件广播机制
```typescript
// 广播到所有客户端
private broadcastToAll(message: any) {
  const messageStr = JSON.stringify(message);
  this.clients.forEach((connection) => {
    if (connection.socket.readyState === 1) { // WebSocket.OPEN
      connection.socket.send(messageStr);
    }
  });
}

// 广播到特定项目
private broadcastToProject(projectId: string, message: any) {
  const messageStr = JSON.stringify(message);
  this.clients.forEach((connection) => {
    if (connection.socket.readyState === 1 &&
        connection.projectId === projectId) {
      connection.socket.send(messageStr);
    }
  });
}

// 项目变更事件处理
this.projectManager.on('project-added', async (project) => {
  const projects = this.projectManager.getAllProjects();
  this.broadcastToAll({
    type: 'projects-update',
    data: { projects }
  });
});

// 规范变更事件处理
this.projectManager.on('spec-change', async (event) => {
  const { projectId, ...data } = event;
  const project = this.projectManager.getProject(projectId);
  if (project) {
    const specs = await project.parser.getAllSpecs();
    const archivedSpecs = await project.parser.getAllArchivedSpecs();
    this.broadcastToProject(projectId, {
      type: 'spec-update',
      projectId,
      data: { specs, archivedSpecs, ...data }
    });
  }
});
```

### 文件系统监控

#### SpecWatcher 实现
```typescript
class SpecWatcher extends EventEmitter {
  private watchers: Map<string, FSWatcher> = new Map();
  private debounceTimers: Map<string, NodeJS.Timeout> = new Map();

  // 监控项目目录
  watchProject(projectPath: string, projectId: string) {
    const workflowPath = join(projectPath, '.spec-workflow');

    const watcher = watch(workflowPath, {
      recursive: true,
      ignored: /node_modules|\.git/,
      persistent: true
    });

    watcher.on('all', (eventType, filename) => {
      this.debounceEvent(projectId, filename, eventType);
    });

    this.watchers.set(projectId, watcher);
  }

  // 防抖事件处理
  private debounceEvent(projectId: string, filename: string, eventType: string) {
    const key = `${projectId}:${filename}`;

    // 清除之前的定时器
    if (this.debounceTimers.has(key)) {
      clearTimeout(this.debounceTimers.get(key)!);
    }

    // 设置新的防抖定时器
    const timer = setTimeout(() => {
      this.handleFileChange(projectId, filename, eventType);
      this.debounceTimers.delete(key);
    }, 300); // 300ms防抖延迟

    this.debounceTimers.set(key, timer);
  }

  // 文件变更处理
  private handleFileChange(projectId: string, filename: string, eventType: string) {
    // 判断文件类型和触发相应事件
    if (filename.includes('specs/') && filename.endsWith('.md')) {
      this.emit('spec-change', { projectId, filename, eventType });
    } else if (filename.includes('approvals/')) {
      this.emit('approval-change', { projectId, filename, eventType });
    } else if (filename.includes('steering/')) {
      this.emit('steering-change', { projectId, filename, eventType });
    }
  }
}
```

### 审批存储系统

#### ApprovalStorage 实现
```typescript
class ApprovalStorage {
  private approvalPath: string;

  // 创建审批请求
  async createRequest(request: ApprovalRequest): Promise<string> {
    const approvalId = this.generateId();
    const approval: ApprovalRequest = {
      ...request,
      id: approvalId,
      status: 'pending',
      createdAt: new Date().toISOString()
    };

    // 保存到文件系统
    const filePath = join(this.approvalPath, `${approvalId}.json`);
    await fs.writeFile(filePath, JSON.stringify(approval, null, 2));

    // 创建内容快照
    await this.createSnapshot(approvalId, request.filePath);

    return approvalId;
  }

  // 处理审批
  async processApproval(approvalId: string, action: ApprovalAction, response?: string): Promise<void> {
    const approval = await this.getApproval(approvalId);
    if (!approval) {
      throw new Error(`Approval ${approvalId} not found`);
    }

    approval.status = this.mapActionToStatus(action);
    approval.respondedAt = new Date().toISOString();
    approval.response = response;

    // 添加修订历史
    if (action === 'request-revision') {
      approval.revisionHistory = approval.revisionHistory || [];
      approval.revisionHistory.push({
        timestamp: new Date().toISOString(),
        type: 'revision_requested',
        response: response
      });
    }

    await this.saveApproval(approval);
    this.emit('approval-updated', { approvalId, approval });
  }

  // 创建文件快照
  private async createSnapshot(approvalId: string, filePath: string): Promise<void> {
    const snapshotDir = join(this.approvalPath, 'snapshots', approvalId);
    await fs.mkdir(snapshotDir, { recursive: true });

    const originalContent = await fs.readFile(filePath, 'utf-8');
    const snapshotPath = join(snapshotDir, 'original.md');
    await fs.writeFile(snapshotPath, originalContent, 'utf-8');
  }
}
```

### 任务调度系统

#### JobScheduler 实现
```typescript
class JobScheduler extends EventEmitter {
  private jobs: Map<string, ScheduledJob> = new Map();
  private running = false;

  // 添加定时任务
  addJob(job: ScheduledJob): string {
    const jobId = this.generateId();
    job.id = jobId;
    job.nextRun = this.calculateNextRun(job.schedule);

    this.jobs.set(jobId, job);
    this.saveJobs();

    this.emit('job-added', { job });
    return jobId;
  }

  // 启动调度器
  start() {
    if (this.running) return;

    this.running = true;
    this.scheduleNextRun();

    // 每分钟检查一次任务
    this.interval = setInterval(() => {
      this.checkAndRunJobs();
    }, 60 * 1000);
  }

  // 检查并运行到期任务
  private async checkAndRunJobs() {
    const now = new Date();

    for (const [jobId, job] of this.jobs) {
      if (!job.enabled) continue;

      if (job.nextRun && new Date(job.nextRun) <= now) {
        await this.runJob(job);
      }
    }
  }

  // 执行任务
  private async runJob(job: ScheduledJob): Promise<void> {
    const execution: JobExecution = {
      id: this.generateId(),
      jobId: job.id,
      startTime: new Date().toISOString(),
      status: 'running'
    };

    job.lastRun = execution.startTime;
    job.nextRun = this.calculateNextRun(job.schedule);

    this.emit('job-started', { job, execution });

    try {
      // 执行任务命令
      const { stdout, stderr } = await exec(job.command);

      execution.endTime = new Date().toISOString();
      execution.status = 'completed';
      execution.output = stdout;

      if (stderr) {
        execution.error = stderr;
      }

      this.emit('job-completed', { job, execution });

    } catch (error) {
      execution.endTime = new Date().toISOString();
      execution.status = 'failed';
      execution.error = error.message;

      this.emit('job-failed', { job, execution });
    }

    // 记录执行历史
    await this.recordExecution(execution);
  }
}
```

## 前端React应用深度分析

### 应用架构设计

#### 主应用组件 (App.tsx)
```typescript
function App() {
  const [projects, setProjects] = useState<Project[]>([]);
  const [selectedProject, setSelectedProject] = useState<string | null>(null);
  const [wsConnection, setWsConnection] = useState<WebSocket | null>(null);
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [language, setLanguage] = useState<string>('zh-cn');

  // WebSocket连接管理
  useEffect(() => {
    const ws = new WebSocket(`ws://localhost:${PORT}/ws`);

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      handleWebSocketMessage(message);
    };

    setWsConnection(ws);

    return () => {
      ws.close();
    };
  }, []);

  // WebSocket消息处理
  const handleWebSocketMessage = (message: any) => {
    switch (message.type) {
      case 'projects-update':
        setProjects(message.data.projects);
        break;
      case 'spec-update':
        if (message.projectId === selectedProject) {
          // 更新规范列表
          updateSpecs(message.data);
        }
        break;
      case 'task-status-update':
        // 更新任务状态
        updateTaskStatus(message.data);
        break;
      // ... 其他事件处理
    }
  };

  return (
    <Router>
      <div className={`app ${theme}`}>
        <Header
          projects={projects}
          selectedProject={selectedProject}
          onProjectChange={setSelectedProject}
        />

        <main className="main-content">
          <Routes>
            <Route path="/" element={<Dashboard projectId={selectedProject} />} />
            <Route path="/specs" element={<SpecsPage projectId={selectedProject} />} />
            <Route path="/tasks" element={<TasksPage projectId={selectedProject} />} />
            <Route path="/approvals" element={<ApprovalsPage projectId={selectedProject} />} />
            <Route path="/settings" element={<SettingsPage />} />
          </Routes>
        </main>
      </div>
    </Router>
  );
}
```

#### 自定义Hooks

##### useWebSocket Hook
```typescript
function useWebSocket(projectId: string | null) {
  const [socket, setSocket] = useState<WebSocket | null>(null);
  const [connectionStatus, setConnectionStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');

  useEffect(() => {
    if (!projectId) return;

    const ws = new WebSocket(`ws://localhost:${PORT}/ws`);

    ws.onopen = () => {
      setConnectionStatus('connected');
      // 发送项目订阅消息
      ws.send(JSON.stringify({
        type: 'subscribe',
        projectId
      }));
    };

    ws.onclose = () => {
      setConnectionStatus('disconnected');
    };

    ws.onerror = () => {
      setConnectionStatus('disconnected');
    };

    setSocket(ws);

    return () => {
      ws.close();
    };
  }, [projectId]);

  const sendMessage = useCallback((message: any) => {
    if (socket && socket.readyState === WebSocket.OPEN) {
      socket.send(JSON.stringify(message));
    }
  }, [socket]);

  return { socket, connectionStatus, sendMessage };
}
```

##### useProjectData Hook
```typescript
function useProjectData(projectId: string | null) {
  const [project, setProject] = useState<Project | null>(null);
  const [specs, setSpecs] = useState<Spec[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!projectId) return;

    const fetchProjectData = async () => {
      setLoading(true);
      setError(null);

      try {
        const [projectResponse, specsResponse] = await Promise.all([
          fetch(`/api/projects/${projectId}/info`),
          fetch(`/api/projects/${projectId}/specs`)
        ]);

        const projectData = await projectResponse.json();
        const specsData = await specsResponse.json();

        setProject(projectData.project);
        setSpecs(specsData.specs);

      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchProjectData();
  }, [projectId]);

  return { project, specs, loading, error };
}
```

#### API服务层

##### ProjectsService
```typescript
class ProjectsService {
  // 获取所有项目
  static async getAllProjects(): Promise<Project[]> {
    const response = await fetch('/api/projects/list');
    const data = await response.json();
    return data.projects;
  }

  // 添加项目
  static async addProject(projectPath: string): Promise<{ projectId: string }> {
    const response = await fetch('/api/projects/add', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ projectPath })
    });

    if (!response.ok) {
      throw new Error(`Failed to add project: ${response.statusText}`);
    }

    return response.json();
  }

  // 删除项目
  static async deleteProject(projectId: string): Promise<void> {
    const response = await fetch(`/api/projects/${projectId}`, {
      method: 'DELETE'
    });

    if (!response.ok) {
      throw new Error(`Failed to delete project: ${response.statusText}`);
    }
  }
}
```

##### SpecsService
```typescript
class SpecsService {
  // 获取规范列表
  static async getSpecs(projectId: string): Promise<Spec[]> {
    const response = await fetch(`/api/projects/${projectId}/specs`);
    const data = await response.json();
    return data.specs;
  }

  // 获取规范详情
  static async getSpec(projectId: string, specName: string): Promise<SpecDetail> {
    const response = await fetch(`/api/projects/${projectId}/specs/${specName}`);
    const data = await response.json();
    return data;
  }

  // 获取规范的所有文档
  static async getSpecDocuments(projectId: string, specName: string): Promise<SpecDocuments> {
    const response = await fetch(`/api/projects/${projectId}/specs/${specName}/all`);
    const data = await response.json();
    return data;
  }

  // 保存文档
  static async saveDocument(
    projectId: string,
    specName: string,
    documentType: string,
    content: string
  ): Promise<void> {
    const response = await fetch(`/api/projects/${projectId}/specs/${specName}/${documentType}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ content })
    });

    if (!response.ok) {
      throw new Error(`Failed to save document: ${response.statusText}`);
    }
  }

  // 归档规范
  static async archiveSpec(projectId: string, specName: string): Promise<void> {
    const response = await fetch(`/api/projects/${projectId}/specs/${specName}/archive`, {
      method: 'POST'
    });

    if (!response.ok) {
      throw new Error(`Failed to archive spec: ${response.statusText}`);
    }
  }
}
```

#### 组件库设计

##### 基础UI组件

###### Button组件
```typescript
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
  loading?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  onClick,
  children
}) => {
  const className = [
    'btn',
    `btn--${variant}`,
    `btn--${size}`,
    disabled && 'btn--disabled',
    loading && 'btn--loading'
  ].filter(Boolean).join(' ');

  return (
    <button
      className={className}
      disabled={disabled || loading}
      onClick={onClick}
    >
      {loading ? <Spinner size="small" /> : children}
    </button>
  );
};
```

###### Card组件
```typescript
interface CardProps {
  title?: string;
  subtitle?: string;
  actions?: React.ReactNode;
  children: React.ReactNode;
  className?: string;
}

export const Card: React.FC<CardProps> = ({
  title,
  subtitle,
  actions,
  children,
  className = ''
}) => {
  return (
    <div className={`card ${className}`}>
      {(title || subtitle || actions) && (
        <div className="card__header">
          <div className="card__title-section">
            {title && <h3 className="card__title">{title}</h3>}
            {subtitle && <p className="card__subtitle">{subtitle}</p>}
          </div>
          {actions && <div className="card__actions">{actions}</div>}
        </div>
      )}
      <div className="card__content">
        {children}
      </div>
    </div>
  );
};
```

##### 业务组件

###### SpecCard组件
```typescript
interface SpecCardProps {
  spec: Spec;
  onView: (specName: string) => void;
  onEdit: (specName: string, documentType: string) => void;
  onArchive: (specName: string) => void;
}

export const SpecCard: React.FC<SpecCardProps> = ({
  spec,
  onView,
  onEdit,
  onArchive
}) => {
  const progressPercentage = spec.taskProgress
    ? Math.round((spec.taskProgress.completed / spec.taskProgress.total) * 100)
    : 0;

  const getStatusColor = (status: string) => {
    switch (status) {
      case 'completed': return 'green';
      case 'implementing': return 'blue';
      case 'design-needed': return 'orange';
      default: return 'gray';
    }
  };

  return (
    <Card
      title={spec.displayName}
      actions={
        <div className="spec-card__actions">
          <Button
            variant="secondary"
            size="small"
            onClick={() => onView(spec.name)}
          >
            查看
          </Button>
          <Button
            variant="secondary"
            size="small"
            onClick={() => onEdit(spec.name, 'tasks')}
          >
            编辑
          </Button>
          <Button
            variant="danger"
            size="small"
            onClick={() => onArchive(spec.name)}
          >
            归档
          </Button>
        </div>
      }
    >
      <div className="spec-card__content">
        <div className="spec-card__status">
          <span className={`status-badge status-badge--${getStatusColor(spec.status)}`}>
            {spec.status}
          </span>
          <span className="spec-card__date">
            {new Date(spec.lastModified).toLocaleDateString()}
          </span>
        </div>

        <div className="spec-card__phases">
          {Object.entries(spec.phases).map(([phase, status]) => (
            <div key={phase} className="phase-indicator">
              <div className={`phase-dot phase-dot--${status.status}`} />
              <span className="phase-name">{phase}</span>
            </div>
          ))}
        </div>

        {spec.taskProgress && (
          <div className="spec-card__progress">
            <div className="progress-bar">
              <div
                className="progress-bar__fill"
                style={{ width: `${progressPercentage}%` }}
              />
            </div>
            <div className="progress-text">
              {spec.taskProgress.completed}/{spec.taskProgress.total} 任务完成
            </div>
          </div>
        )}
      </div>
    </Card>
  );
};
```

###### TaskList组件
```typescript
interface TaskListProps {
  tasks: Task[];
  onTaskStatusChange: (taskId: string, status: TaskStatus) => void;
  onTaskEdit: (taskId: string) => void;
}

export const TaskList: React.FC<TaskListProps> = ({
  tasks,
  onTaskStatusChange,
  onTaskEdit
}) => {
  const groupedTasks = tasks.reduce((groups, task) => {
    const phase = task.id.split('.')[0];
    if (!groups[phase]) groups[phase] = [];
    groups[phase].push(task);
    return groups;
  }, {} as Record<string, Task[]>);

  return (
    <div className="task-list">
      {Object.entries(groupedTasks).map(([phase, phaseTasks]) => (
        <div key={phase} className="task-phase">
          <h4 className="task-phase__title">阶段 {phase}</h4>
          <div className="task-phase__tasks">
            {phaseTasks.map((task) => (
              <TaskItem
                key={task.id}
                task={task}
                onStatusChange={onTaskStatusChange}
                onEdit={onTaskEdit}
              />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};
```

#### 页面组件

##### Dashboard页面
```typescript
export const Dashboard: React.FC<{ projectId: string | null }> = ({ projectId }) => {
  const { project, specs, loading, error } = useProjectData(projectId);

  if (!projectId) {
    return <div className="dashboard--empty">请选择一个项目</div>;
  }

  if (loading) {
    return <div className="dashboard--loading">加载中...</div>;
  }

  if (error) {
    return <div className="dashboard--error">错误: {error}</div>;
  }

  const stats = {
    totalSpecs: specs.length,
    activeSpecs: specs.filter(spec => spec.status !== 'archived').length,
    completedSpecs: specs.filter(spec => spec.status === 'completed').length,
    totalTasks: specs.reduce((sum, spec) =>
      sum + (spec.taskProgress?.total || 0), 0
    ),
    completedTasks: specs.reduce((sum, spec) =>
      sum + (spec.taskProgress?.completed || 0), 0
    )
  };

  return (
    <div className="dashboard">
      <div className="dashboard__header">
        <h1>{project?.projectName} 仪表板</h1>
        <div className="dashboard__stats">
          <StatCard title="总规范数" value={stats.totalSpecs} />
          <StatCard title="活跃规范" value={stats.activeSpecs} />
          <StatCard title="已完成规范" value={stats.completedSpecs} />
          <StatCard title="任务进度" value={`${stats.completedTasks}/${stats.totalTasks}`} />
        </div>
      </div>

      <div className="dashboard__content">
        <div className="dashboard__section">
          <h2>最近更新的规范</h2>
          <div className="spec-grid">
            {specs
              .sort((a, b) => new Date(b.lastModified).getTime() - new Date(a.lastModified).getTime())
              .slice(0, 6)
              .map(spec => (
                <SpecCard
                  key={spec.name}
                  spec={spec}
                  onView={(name) => navigate(`/specs/${name}`)}
                  onEdit={(name, docType) => navigate(`/specs/${name}/${docType}`)}
                  onArchive={(name) => handleArchiveSpec(name)}
                />
              ))}
          </div>
        </div>

        <div className="dashboard__section">
          <h2>进度图表</h2>
          <ProgressChart specs={specs} />
        </div>
      </div>
    </div>
  );
};
```

### 国际化支持

#### i18n配置
```typescript
// i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

import zhCN from './locales/zh-cn.json';
import enUS from './locales/en-us.json';

i18n
  .use(initReactI18next)
  .init({
    resources: {
      'zh-cn': { translation: zhCN },
      'en-us': { translation: enUS }
    },
    lng: 'zh-cn',
    fallbackLng: 'en-us',
    interpolation: {
      escapeValue: false
    }
  });

export default i18n;
```

#### 语言资源文件
```json
// locales/zh-cn.json
{
  "common": {
    "save": "保存",
    "cancel": "取消",
    "delete": "删除",
    "edit": "编辑",
    "view": "查看",
    "loading": "加载中...",
    "error": "错误",
    "success": "成功"
  },
  "dashboard": {
    "title": "仪表板",
    "selectProject": "请选择项目",
    "totalSpecs": "总规范数",
    "activeSpecs": "活跃规范",
    "completedSpecs": "已完成规范",
    "taskProgress": "任务进度",
    "recentSpecs": "最近更新的规范",
    "progressChart": "进度图表"
  },
  "specs": {
    "title": "规范管理",
    "createSpec": "创建规范",
    "requirements": "需求文档",
    "design": "设计文档",
    "tasks": "任务文档",
    "archive": "归档",
    "status": {
      "notStarted": "未开始",
      "requirementsNeeded": "需要需求文档",
      "designNeeded": "需要设计文档",
      "tasksNeeded": "需要任务文档",
      "implementing": "实施中",
      "completed": "已完成"
    }
  }
}
```

### 状态管理

#### Context配置
```typescript
// AppContext.tsx
interface AppContextType {
  projects: Project[];
  selectedProject: string | null;
  theme: 'light' | 'dark';
  language: string;
  notifications: Notification[];

  // Actions
  setSelectedProject: (projectId: string | null) => void;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (language: string) => void;
  addNotification: (notification: Notification) => void;
  removeNotification: (id: string) => void;
}

const AppContext = createContext<AppContextType | undefined>(undefined);

export const AppProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, setState] = useState<AppContextType>({
    projects: [],
    selectedProject: null,
    theme: 'light',
    language: 'zh-cn',
    notifications: [],
    setSelectedProject: (projectId) => setState(prev => ({ ...prev, selectedProject: projectId })),
    setTheme: (theme) => setState(prev => ({ ...prev, theme })),
    setLanguage: (language) => setState(prev => ({ ...prev, language })),
    addNotification: (notification) => setState(prev => ({
      ...prev,
      notifications: [...prev.notifications, { ...notification, id: generateId() }]
    })),
    removeNotification: (id) => setState(prev => ({
      ...prev,
      notifications: prev.notifications.filter(n => n.id !== id)
    }))
  });

  return (
    <AppContext.Provider value={state}>
      {children}
    </AppContext.Provider>
  );
};

export const useApp = () => {
  const context = useContext(AppContext);
  if (!context) {
    throw new Error('useApp must be used within AppProvider');
  }
  return context;
};
```

## 性能优化和最佳实践

### 后端性能优化

#### 1. 缓存策略
```typescript
class CacheManager {
  private cache = new Map<string, CacheEntry>();
  private readonly DEFAULT_TTL = 5 * 60 * 1000; // 5分钟

  set<T>(key: string, value: T, ttl?: number): void {
    this.cache.set(key, {
      value,
      expiry: Date.now() + (ttl || this.DEFAULT_TTL)
    });
  }

  get<T>(key: string): T | null {
    const entry = this.cache.get(key);
    if (!entry || Date.now() > entry.expiry) {
      this.cache.delete(key);
      return null;
    }
    return entry.value as T;
  }

  // 缓存规范解析结果
  async getCachedSpec(projectPath: string, specName: string): Promise<SpecData | null> {
    const cacheKey = `spec:${projectPath}:${specName}`;
    const cached = this.get<SpecData>(cacheKey);

    if (cached) {
      return cached;
    }

    // 解析规范并缓存
    const parser = new SpecParser(projectPath);
    const spec = await parser.getSpec(specName);
    this.set(cacheKey, spec);

    return spec;
  }
}
```

#### 2. 批量操作
```typescript
class BatchProcessor {
  private batchQueue: Array<() => Promise<any>> = [];
  private processing = false;

  async addToBatch<T>(operation: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.batchQueue.push(async () => {
        try {
          const result = await operation();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });

      this.processBatch();
    });
  }

  private async processBatch(): Promise<void> {
    if (this.processing || this.batchQueue.length === 0) {
      return;
    }

    this.processing = true;
    const batch = this.batchQueue.splice(0, 10); // 每次处理10个

    try {
      await Promise.all(batch.map(op => op()));
    } catch (error) {
      console.error('Batch processing error:', error);
    } finally {
      this.processing = false;

      // 继续处理剩余队列
      if (this.batchQueue.length > 0) {
        setTimeout(() => this.processBatch(), 100);
      }
    }
  }
}
```

### 前端性能优化

#### 1. 组件懒加载
```typescript
// 路由级别的代码分割
const Dashboard = lazy(() => import('../pages/Dashboard'));
const SpecsPage = lazy(() => import('../pages/SpecsPage'));
const TasksPage = lazy(() => import('../pages/TasksPage'));
const ApprovalsPage = lazy(() => import('../pages/ApprovalsPage'));

// 组件级别的懒加载
const SpecEditor = lazy(() => import('../components/SpecEditor'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/specs" element={<SpecsPage />} />
        {/* ... 其他路由 */}
      </Routes>
    </Suspense>
  );
}
```

#### 2. 虚拟滚动
```typescript
// 大列表虚拟滚动
import { FixedSizeList as List } from 'react-window';

const VirtualTaskList: React.FC<{ tasks: Task[] }> = ({ tasks }) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <TaskItem task={tasks[index]} />
    </div>
  );

  return (
    <List
      height={600}
      itemCount={tasks.length}
      itemSize={80}
      width="100%"
    >
      {Row}
    </List>
  );
};
```

#### 3. 数据预取
```typescript
// 预取相关数据
const useDataPrefetch = (projectId: string | null) => {
  const queryClient = useQueryClient();

  useEffect(() => {
    if (projectId) {
      // 预取规范列表
      queryClient.prefetchQuery(
        ['specs', projectId],
        () => SpecsService.getSpecs(projectId)
      );

      // 预取审批列表
      queryClient.prefetchQuery(
        ['approvals', projectId],
        () => ApprovalsService.getApprovals(projectId)
      );
    }
  }, [projectId, queryClient]);
};
```

### 安全措施

#### 1. 输入验证
```typescript
// 后端输入验证
const validateProjectPath = (path: string): string => {
  // 规范化路径
  const normalizedPath = normalize(path);

  // 检查路径遍历
  if (normalizedPath.includes('..')) {
    throw new Error('Path traversal detected');
  }

  // 检查绝对路径
  if (!isAbsolute(normalizedPath)) {
    throw new Error('Absolute path required');
  }

  return normalizedPath;
};

// 前端输入清理
const sanitizeInput = (input: string): string => {
  return input
    .replace(/[<>]/g, '') // 移除HTML标签
    .trim()
    .substring(0, 1000); // 限制长度
};
```

#### 2. CORS配置
```typescript
// CORS安全配置
await app.register(cors, {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
});
```

#### 3. 认证授权（示例）
```typescript
// JWT认证中间件
const authMiddleware = async (request: FastifyRequest, reply: FastifyReply) => {
  try {
    const token = request.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return reply.code(401).send({ error: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    request.user = decoded;

  } catch (error) {
    return reply.code(401).send({ error: 'Invalid token' });
  }
};
```

## 部署和运维

### Docker容器化

#### Dockerfile (后端)
```dockerfile
FROM node:18-alpine

WORKDIR /app

# 复制package文件
COPY package*.json ./
COPY src/package*.json ./src/
COPY src/dashboard_frontend/package*.json ./src/dashboard_frontend/

# 安装依赖
RUN npm ci --only=production

# 复制源代码
COPY . .

# 构建前端
RUN cd src/dashboard_frontend && npm run build

# 构建后端
RUN npm run build

# 暴露端口
EXPOSE 3000

# 启动命令
CMD ["npm", "start"]
```

#### docker-compose.yml
```yaml
version: '3.8'

services:
  spec-workflow-mcp:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

### 监控和日志

#### 健康检查端点
```typescript
// 健康检查
app.get('/health', async (request, reply) => {
  const health = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    version: process.env.npm_package_version
  };

  // 检查数据库连接
  try {
    await checkDatabaseConnection();
    health.database = 'connected';
  } catch (error) {
    health.status = 'error';
    health.database = 'disconnected';
  }

  const statusCode = health.status === 'ok' ? 200 : 503;
  return reply.code(statusCode).send(health);
});
```

#### 结构化日志
```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'spec-workflow-mcp' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});
```

## 总结

Spec Workflow MCP 的仪表板系统是一个功能完整、架构良好的现代化Web应用：

### 技术优势
- **全栈TypeScript**：类型安全的端到端开发体验
- **实时通信**：WebSocket + 事件驱动架构
- **多项目支持**：灵活的项目管理和协调
- **响应式设计**：适配不同设备和屏幕尺寸
- **国际化支持**：完整的多语言支持

### 架构特点
- **模块化设计**：清晰的职责分离和接口定义
- **性能优化**：缓存、懒加载、虚拟滚动等优化策略
- **安全防护**：输入验证、CORS配置、认证授权
- **可扩展性**：插件化架构支持功能扩展
- **可维护性**：完整的日志、监控和错误处理

### 用户体验
- **直观界面**：清晰的布局和友好的交互
- **实时更新**：即时反映项目状态变化
- **高效操作**：批量操作和快捷键支持
- **离线友好**：本地缓存和离线功能
- **移动适配**：响应式设计支持移动设备

这个仪表板系统为规范驱动的软件开发提供了强大的可视化支持，显著提高了团队协作效率和项目管理水平。