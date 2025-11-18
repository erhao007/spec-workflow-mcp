# Spec Workflow MCP 核心架构深度分析

## 概述

本文档提供了对 Spec Workflow MCP 核心服务层的深度架构分析，涵盖了所有核心服务类的设计理念、实现细节和最佳实践。

## 核心服务层架构概览

### 目录结构

```
src/core/
├── archive-service.ts          # 规范归档服务
├── dashboard-session.ts       # 仪表板会话管理
├── implementation-log-migrator.ts # 实现日志迁移器
├── parser.ts                  # 规范解析器
├── path-utils.ts              # 路径工具类
├── project-registry.ts        # 项目注册表
├── task-parser.ts             # 任务解析器
└── workspace-initializer.ts   # 工作空间初始化器
```

### 架构特点

- **8个核心服务类**：每个服务专注特定功能领域，遵循单一职责原则
- **安全性优先**：全面的输入验证、路径安全、系统目录访问控制
- **原子性操作**：临时文件+重命名机制确保数据完整性
- **跨平台兼容**：统一的路径处理和系统调用，支持 Windows/Linux/macOS
- **类型安全**：完整的 TypeScript 类型定义，编译时错误检查

## 详细服务分析

### 1. PathUtils - 路径工具服务

**文件位置**: `src/core/path-utils.ts`

**核心职责**：
- 提供安全的路径操作和验证
- 防止目录遍历攻击
- 跨平台路径兼容性处理

**关键接口**：
```typescript
// 安全路径连接
static safeJoin(basePath: string, ...paths: string[]): string

// 工作空间路径获取
static getWorkflowRoot(projectPath: string): string
static getSpecPath(projectPath: string, specName: string): string
static getSteeringPath(projectPath: string): string

// 路径验证函数
export async function validateProjectPath(projectPath: string): Promise<string>
export async function ensureWorkflowDirectory(projectPath: string): Promise<string>
```

**安全特性**：
- **输入验证**：所有路径输入都经过严格验证和清理
- **目录遍历防护**：防止 `../` 等路径遍历攻击
- **系统目录保护**：禁止访问敏感系统目录
- **原子性目录创建**：确保目录创建的原子性
- **跨平台路径处理**：自动处理不同操作系统的路径分隔符

### 2. WorkspaceInitializer - 工作空间初始化服务

**文件位置**: `src/core/workspace-initializer.ts`

**核心职责**：
- 创建完整的工作空间目录结构
- 初始化模板文件
- 执行数据迁移任务

**初始化流程**：
```typescript
async initializeWorkspace(): Promise<void> {
  // 1. 创建目录结构
  await this.initializeDirectories();

  // 2. 复制模板文件
  await this.initializeTemplates();

  // 3. 创建用户模板说明
  await this.createUserTemplatesReadme();

  // 4. 迁移实现日志
  await this.migrateImplementationLogs();
}
```

**目录结构创建**：
```
.spec-workflow/
├── approvals/          # 审批目录
├── archive/           # 归档目录
│   └── specs/        # 归档规范
├── specs/             # 活跃规范
├── steering/          # 指导文档
├── templates/         # 默认模板
└── user-templates/    # 用户自定义模板
```

**错误处理策略**：
- **模板复制失败**：记录错误但不中断整体初始化流程
- **实现日志迁移失败**：优雅降级，不影响其他功能
- **用户模板README创建**：避免覆盖现有文件

### 3. ProjectRegistry - 项目注册表服务

**文件位置**: `src/core/project-registry.ts`

**核心职责**：
- 全局项目注册和管理
- 进程生命周期跟踪
- 多项目实例协调

**数据结构**：
```typescript
export interface ProjectRegistryEntry {
  projectId: string;      // 基于路径的哈希ID
  projectPath: string;    // 绝对路径
  projectName: string;    // 目录名
  pid: number;            // 进程ID
  registeredAt: string;   // 注册时间
}
```

**关键特性**：
- **项目ID生成**：使用 SHA-1 哈希算法生成16字符唯一标识
- **原子性操作**：临时文件+重命名机制确保数据一致性
- **进程健康检查**：通过信号0检测进程存活状态
- **数据完整性保证**：自动检测和恢复损坏的注册表文件
- **备份机制**：自动创建注册表文件备份

**核心方法**：
```typescript
// 项目管理
async registerProject(projectPath: string, pid: number): Promise<string>
async unregisterProject(projectPath: string): Promise<void>
async cleanupStaleProjects(): Promise<number>

// 查询接口
async getAllProjects(): Promise<ProjectRegistryEntry[]>
async getProject(projectPath: string): Promise<ProjectRegistryEntry | null>
```

### 4. SpecArchiveService - 规范归档服务

**文件位置**: `src/core/archive-service.ts`

**核心职责**：
- 规范文档的归档和恢复
- 规范状态查询和管理
- 文档生命周期管理

**主要操作**：
```typescript
// 归档规范
async archiveSpec(specName: string): Promise<void>

// 恢复归档
async unarchiveSpec(specName: string): Promise<void>

// 状态查询
async isSpecActive(specName: string): Promise<boolean>
async isSpecArchived(specName: string): Promise<boolean>
async getSpecLocation(specName: string): Promise<'active' | 'archived' | 'not-found'>
```

**安全特性**：
- **操作前验证**：检查源文件存在性和权限
- **冲突检测**：检测目标位置文件冲突
- **原子性移动**：使用原子性文件移动操作
- **详细错误信息**：提供具体的错误描述和解决建议

### 5. DashboardSessionManager - 仪表板会话管理

**文件位置**: `src/core/dashboard-session.ts`

**核心职责**：
- 仪表板实例生命周期管理
- 会话状态持久化
- 多实例冲突预防

**会话数据结构**：
```typescript
export interface DashboardSessionEntry {
  url: string;        // 仪表板URL
  port: number;        // 端口号
  pid: number;        // 进程ID
  startedAt: string;  // 启动时间
}
```

**关键特性**：
- **原子性写入**：临时文件+重命名机制防止数据损坏
- **进程健康监控**：实时监控仪表板进程状态
- **自动清理机制**：进程死亡时自动清理会话记录
- **会话恢复功能**：启动时验证现有会话的有效性
- **端口冲突检测**：自动检测和避免端口冲突

### 6. ImplementationLogMigrator - 实现日志迁移器

**文件位置**: `src/core/implementation-log-migrator.ts`

**核心职责**：
- JSON格式到Markdown格式的自动迁移
- 实现日志的结构化转换
- 数据完整性保证

**迁移流程**：
```typescript
// 全量迁移
async migrateAllSpecs(specsDir: string): Promise<{
  totalSpecs: number;
  migratedSpecs: number;
  totalEntries: number;
  errors: Array<{ spec: string; error: string }>;
}>
```

**文件命名策略**：
```
task-{taskId}_{timestamp}_{idPrefix}.md
例如: task-1-2_20250118_143022_a1b2c3d4.md
```

**Markdown结构模板**：
```markdown
# Implementation Log: Task {taskId}

**Summary:** {summary}
**Timestamp:** {timestamp}
**Log ID:** {id}

## Statistics
- **Lines Added:** +{linesAdded}
- **Lines Removed:** -{linesRemoved}
- **Files Changed:** {filesChanged}

## Files Modified
- {file1}
- {file2}

## Artifacts
### API Endpoints
#### {method} {path}
- **Purpose:** {purpose}
- **Location:** {location}

### Components
#### {name}
- **Type:** {type}
- **Purpose:** {purpose}
```

### 7. SpecParser - 规范解析器

**文件位置**: `src/core/parser.ts`

**核心职责**：
- 解析规范文档结构和状态
- 提取任务进度信息
- 项目指导文档状态检查

**数据模型**：
```typescript
export interface SpecData {
  name: string;
  createdAt: string;
  lastModified: string;
  phases: {
    requirements: PhaseStatus;
    design: PhaseStatus;
    tasks: PhaseStatus;
    implementation: PhaseStatus;
  };
  taskProgress?: {
    total: number;
    completed: number;
    pending: number;
  };
}
```

**解析能力**：
- **多阶段文档支持**：requirements.md, design.md, tasks.md
- **任务进度集成**：集成 task-parser 解析任务状态
- **指导文档检查**：检查 product.md, tech.md, structure.md 存在性
- **容错处理**：文件不存在时返回空状态而非抛出异常
- **缓存机制**：智能缓存解析结果提高性能

### 8. TaskParser - 任务解析器

**文件位置**: `src/core/task-parser.ts`

**核心职责**：
- 统一的任务解析和状态管理
- 结构化提示解析
- 任务元数据提取

**任务数据结构**：
```typescript
export interface ParsedTask {
  id: string;                          // 任务ID (如 "1.2")
  description: string;                 // 任务描述
  status: 'pending' | 'in-progress' | 'completed';
  lineNumber: number;                  // 文件中的行号
  indentLevel: number;                 // 缩进层级
  isHeader: boolean;                   // 是否为标题任务

  // 元数据
  requirements?: string[];             // 关联需求
  leverage?: string;                   // 可利用的代码
  files?: string[];                    // 要修改的文件
  purposes?: string[];                 // 目标说明
  implementationDetails?: string[];    // 实现细节
  prompt?: string;                     // AI提示
  promptStructured?: PromptSection[]; // 结构化提示
}
```

**解析特性**：
- **智能状态识别**：`[]` 待办、`[-]` 进行中、`[x]` 已完成
- **层级处理**：支持多级缩进的任务结构
- **元数据提取**：Requirements, Leverage, Files, Purpose
- **结构化提示**：解析管道分隔的提示内容
- **容错机制**：格式错误时跳过而非中断整个解析过程
- **版本兼容**：支持不同版本的任务格式

## 数据流和处理逻辑

### 项目初始化流程
```
用户启动 → 验证路径 → 创建工作空间 → 初始化模板 → 迁移数据 → 注册项目
```

### 规范管理流程
```
规范操作 → 检查状态 → 验证路径 → 执行操作 → 更新注册表 → 返回结果
```

### 仪表板集成流程
```
仪表板启动 → 注册会话 → 进程监控 → 状态同步 → 连接管理 → 实时更新
```

### 实现日志管理流程
```
任务完成 → 解析日志 → 验证任务 → 格式转换 → 文件存储 → 索引更新
```

## 与其他模块的集成

### 与 Tools 模块集成
- **项目路径验证**：PathUtils.validateProjectPath()
- **规范状态查询**：SpecParser.getAllSpecs(), SpecParser.getSpec()
- **任务进度解析**：TaskParser.parseTasksFromMarkdown()
- **实现日志查询**：ImplementationLogMigrator 迁移和查询功能

### 与 Dashboard 模块集成
- **会话管理**：DashboardSessionManager 提供会话生命周期管理
- **项目注册**：ProjectRegistry 支持多项目注册和查询
- **规范归档**：SpecArchiveService 提供归档和恢复功能
- **实时状态**：各服务的状态变化触发仪表板实时更新

### 与 Prompts 模块集成
- **任务解析**：TaskParser 处理任务提示和元数据
- **规范状态**：SpecParser 提供文档状态和进度信息
- **工作流指导**：各服务状态驱动提示生成

## 配置管理和环境变量

### 配置文件结构
```typescript
export interface SpecWorkflowConfig {
  projectDir?: string;    // 项目目录
  port?: number;          // 端口号
  dashboardOnly?: boolean; // 仅仪表板模式
  lang?: string;          // 语言设置
}
```

### 配置加载优先级
1. **命令行参数** (最高优先级)
2. **TOML配置文件**
3. **默认值** (最低优先级)

### 环境变量使用
- `homedir()`: 用户主目录路径获取
- 进程ID和信号机制用于健康检查
- 跨平台路径分隔符自动处理
- 系统环境变量自动适配

## 异常处理机制

### 路径安全
- **目录遍历攻击防护**：严格的路径验证和规范化
- **系统目录访问控制**：禁止访问敏感系统路径
- **输入验证和清理**：所有路径输入都经过安全处理

### 文件操作
- **原子性写入操作**：临时文件+重命名确保数据完整性
- **文件存在性检查**：操作前验证文件状态
- **权限验证**：确保操作权限足够
- **并发控制**：避免多进程同时操作同一文件

### 数据完整性
- **注册表文件损坏检测**：自动检测和备份恢复
- **迁移操作的回滚机制**：失败时自动回滚
- **会话数据的一致性保证**：确保会话状态准确性
- **实现日志的版本管理**：支持多种格式的迁移

### 错误处理策略
- **可恢复错误**：记录日志并继续执行其他操作
- **致命错误**：抛出异常中断操作流程
- **降级处理**：功能降级而非完全失败
- **用户友好错误消息**：提供清晰的错误描述和解决建议

## 性能优化特点

### 缓存策略
- **进程状态缓存**：避免频繁的系统调用
- **路径计算结果缓存**：重复使用路径计算结果
- **文件存在性检查优化**：批量检查减少IO操作
- **解析结果缓存**：缓存规范和任务解析结果

### 批量操作
- **多目录一次性创建**：减少系统调用次数
- **批量规范状态查询**：一次操作获取多个规范状态
- **并行化的数据迁移**：支持并发迁移多个规范
- **批量文件操作**：合并多个小文件操作

### 资源管理
- **延迟加载的文件读取**：按需读取文件内容
- **及时释放文件句柄**：避免文件句柄泄漏
- **内存使用优化**：避免大对象长期占用内存
- **连接池管理**：复用数据库连接和网络连接

## 总结

Spec Workflow MCP 的核心服务层展现了一个设计良好、功能完整的企业级应用架构：

### 优势
- **安全性**：全方面的输入验证和路径保护机制
- **可靠性**：原子性操作和数据完整性保证
- **可扩展性**：模块化设计和清晰的接口分离
- **维护性**：完善的错误处理和日志记录系统
- **兼容性**：跨平台支持和向后兼容性保证

### 架构特点
- **单一职责原则**：每个服务类专注特定功能领域
- **依赖倒置原则**：通过接口而非具体实现进行依赖
- **容错设计**：多层次的错误处理和降级机制
- **性能优化**：智能缓存策略和批量操作优化

### 最佳实践应用
- **SOLID原则**：严格遵循面向对象设计原则
- **DRY原则**：通过工具类和服务复用避免重复
- **KISS原则**：保持简洁的接口设计和清晰的代码结构
- **错误处理**：统一的异常处理和用户友好的错误消息

这套核心服务层为整个 Spec Workflow MCP 系统提供了坚实的基础，确保了系统的稳定性、安全性和可维护性，是企业级软件开发最佳实践的典型体现。