# Hydro 项目架构与代码文档

## 一、项目概述

Hydro 是一个高效的信息学在线测评系统（Online Judge），采用模块化设计，支持插件系统和功能热插拔。

### 技术栈

| 分类 | 技术 | 版本要求 |
|------|------|----------|
| 语言 | TypeScript | - |
| 运行时 | Node.js | >= 22 |
| 框架 | Cordis | 4.0.0-rc.4 |
| Web服务器 | Koa | ^3.2.0 |
| 数据库 | MongoDB | ^7.2.0 |
| 构建工具 | Yarn Workspaces | - |

---

## 二、项目架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Hydro 项目架构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐      │
│   │   前端 UI   │       │   Web API   │       │   评测机    │      │
│   │  ui-default │──────>│   hydrooj   │<─────>│ hydrojudge  │      │
│   └─────────────┘       └──────┬──────┘       └─────────────┘      │
│                                │                                    │
│                                ▼                                    │
│                       ┌─────────────┐                               │
│                       │  MongoDB    │                               │
│                       └─────────────┘                               │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  Packages:                                                          │
│    ├── hydrooj/        # 主应用核心                                  │
│    ├── hydrojudge/     # 评测引擎                                    │
│    ├── common/         # 通用类型和状态                              │
│    ├── ui-default/     # 默认前端界面                                │
│    └── ... (插件)      # 各种功能插件                                │
│                                                                     │
│  Framework:                                                         │
│    ├── framework/      # 核心框架（API、路由、服务器）                │
│    ├── utils/          # 工具函数                                   │
│    └── register/       # 模块注册器                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
/workspace/
├── .devcontainer/       # Dev Container 配置
├── .github/             # GitHub CI/CD 配置
├── .vscode/             # VS Code 配置
├── build/               # 构建脚本
├── examples/            # 示例配置文件
├── framework/           # 核心框架
│   ├── eslint-config/   # ESLint 配置
│   ├── framework/       # 框架核心代码
│   ├── register/        # 模块注册器
│   └── utils/           # 工具函数
├── install/             # 安装脚本和配置
│   ├── docker/          # Docker 配置
│   ├── helm-single/     # Helm Chart
│   └── nix/             # Nix 配置
├── packages/            # 功能包
│   ├── a11y/            # 可访问性测试工具
│   ├── blog/            # 博客模块
│   ├── center/          # 中心模块
│   ├── common/          # 通用类型和常量
│   ├── components/      # UI 组件
│   ├── elastic/         # Elasticsearch 集成
│   ├── fps-importer/    # FPS 格式导入器
│   ├── geoip/           # GeoIP 支持
│   ├── hydrojudge/      # 评测引擎
│   ├── hydrooj/         # 主应用
│   ├── import-hoj/      # HOJ 导入器
│   ├── import-qduoj/    # QDUOJ 导入器
│   ├── login-with-*/    # 第三方登录插件
│   ├── migrate/         # 数据迁移工具
│   ├── onlyoffice/      # OnlyOffice 集成
│   ├── onsite-toolkit/  # 现场比赛工具包
│   ├── prom-client/     # Prometheus 监控
│   ├── scoreboard-xcpcio/# XCPC IO 风格榜单
│   ├── sonic/           # Sonic 搜索集成
│   ├── telegram/        # Telegram 通知
│   └── ui-default/      # 默认前端界面
```

---

## 三、核心模块详解

### 3.1 Framework 层

Framework 提供系统的基础服务和抽象层。

#### 3.1.1 API 服务 (`framework/framework/api.ts`)

**核心类：**

| 类名 | 职责 | 关键字段/方法 |
|------|------|--------------|
| `ApiService` | API 注册和执行服务 | `provide()`, `execute()`, `serialize()` |
| `ApiHandler` | HTTP API 请求处理器 | `all()` |
| `ApiConnectionHandler` | WebSocket API 连接处理器 | `prepare()`, `message()`, `cleanup()` |

**API 类型定义：**

```typescript
export type ApiType = 'Query' | 'Mutation' | 'Subscription';

export interface ApiCall<Type extends ApiType, Arg, Res, Progress = void> {
    readonly type: Type;
    readonly input: Schema<Arg>;
    readonly func: /* 执行函数 */;
    readonly hooks: ApiCall<'Query', Arg, void>[];
}
```

**核心方法说明：**

- `ApiService.provide(calls, namespace)`: 注册 API 调用
- `ApiService.execute(context, callOrName, args)`: 执行 API 调用
- `projection(input, schema, context)`: 结果投影，根据 schema 过滤返回字段

#### 3.1.2 Server 服务 (`framework/framework/server.ts`)

**核心类：**

| 类名 | 职责 | 关键字段/方法 |
|------|------|--------------|
| `WebService` | Web 服务器核心服务 | `Route()`, `Connection()`, `listen()` |
| `Handler` | HTTP 请求处理器基类 | `init()`, `onerror()`, `renderHTML()` |
| `ConnectionHandler` | WebSocket 连接处理器基类 | `send()`, `close()`, `onerror()` |
| `HandlerCommon` | 处理器公共基类 | `url()`, `translate()`, `renderHTML()` |

**请求处理流程：**

```
请求进入 → serverLayers → routes → handlerLayers → Handler处理 → 响应
```

**Handler 生命周期步骤：**

```
log/__init → init → handler/init
→ handler/before-prepare/{name} → __prepare → _prepare → prepare
→ handler/before/{name} → all → {method}
→ handler/before-operation/{name} → post{operation} (仅POST操作)
→ after → handler/after/{name} → cleanup → handler/finish/{name}
```

#### 3.1.3 路由系统 (`framework/framework/router.ts`)

基于 `@koa/router` 封装，支持 HTTP 和 WebSocket 路由。

**关键方法：**

| 方法 | 说明 |
|------|------|
| `Route(name, path, HandlerClass, ...checkers)` | 注册 HTTP 路由 |
| `Connection(name, path, HandlerClass, ...checkers)` | 注册 WebSocket 连接 |
| `addLayer(name, func)` | 添加中间件层 |
| `applyMixin(name, MixinClass)` | 为 Handler 应用 Mixin |

#### 3.1.4 验证器 (`framework/framework/validator.ts`)

提供参数验证功能，支持多种类型：

| 类型 | 说明 |
|------|------|
| `Types.String` | 字符串 |
| `Types.Number` | 数字 |
| `Types.Boolean` | 布尔值 |
| `Types.ObjectId` | MongoDB ObjectId |
| `Types.DomainId` | 域 ID |
| `Types.UserId` | 用户 ID |

#### 3.1.5 装饰器 (`framework/framework/decorators.ts`)

| 装饰器 | 说明 |
|--------|------|
| `@param(name, type, required?)` | 声明请求参数 |
| `@route(method, path)` | 声明路由方法 |

---

### 3.2 HydroOJ 核心模块 (`packages/hydrooj/`)

#### 3.2.1 上下文系统 (`packages/hydrooj/src/context.ts`)

基于 Cordis 框架构建，提供依赖注入和生命周期管理。

**核心服务：**

| 服务 | 说明 |
|------|------|
| `ApiMixin` | API 混合服务 |
| `Loader` | 模块加载器 |
| `CheckService` | 检查服务 |
| `MigrationService` | 数据迁移服务 |

**全局对象：**

```typescript
global.Hydro = {
    version: { node, hydrooj },
    model: {},        // 数据模型
    script: {},       // 脚本
    module: {},       // 模块
    ui: {},           // UI 组件
    error: {},        // 错误定义
    locales: {},      // 国际化
};
```

#### 3.2.2 数据模型

HydroOJ 使用 MongoDB 作为数据库，主要模型包括：

| 模型 | 文件路径 | 职责 |
|------|----------|------|
| `UserModel` | `model/user.ts` | 用户管理 |
| `ProblemModel` | `model/problem.ts` | 题目管理 |
| `RecordModel` | `model/record.ts` | 提交记录 |
| `ContestModel` | `model/contest.ts` | 比赛管理 |
| `DomainModel` | `model/domain.ts` | 域（空间）管理 |
| `TaskModel` | `model/task.ts` | 定时任务 |
| `DocumentModel` | `model/document.ts` | 文档管理 |
| `DiscussionModel` | `model/discussion.ts` | 讨论区 |
| `SolutionModel` | `model/solution.ts` | 题解管理 |
| `MessageModel` | `model/message.ts` | 消息管理 |
| `OauthModel` | `model/oauth.ts` | OAuth 认证 |

#### 3.2.3 服务层

| 服务 | 文件路径 | 职责 |
|------|----------|------|
| `db` | `service/db.ts` | 数据库连接和操作 |
| `server` | `service/server.ts` | 服务器服务扩展 |
| `bus` | `service/bus.ts` | 事件总线 |
| `storage` | `service/storage.ts` | 文件存储服务 |
| `check` | `service/check.ts` | 参数校验服务 |
| `migration` | `service/migration.ts` | 数据迁移服务 |

---

### 3.3 HydroJudge 评测引擎 (`packages/hydrojudge/`)

#### 3.3.1 评测任务 (`packages/hydrojudge/src/task.ts`)

**`JudgeTask` 类：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `source` | string | 题目来源标识 |
| `rid` | string | 记录 ID |
| `lang` | string | 编程语言 |
| `code` | CopyInFile | 源代码 |
| `data` | FileInfo[] | 测试数据文件列表 |
| `config` | ParsedConfig | 评测配置 |
| `meta` | JudgeMeta | 评测元信息 |
| `folder` | string | 缓存目录路径 |

**核心方法：**

| 方法 | 说明 |
|------|------|
| `handle()` | 处理评测请求入口 |
| `cacheOpen()` | 打开并缓存题目数据 |
| `doSubmission()` | 执行评测流程 |
| `compile()` | 编译源代码 |
| `compileLocalFile()` | 编译本地文件（checker等） |

#### 3.3.2 评测类型 (`packages/hydrojudge/src/judge/`)

支持多种评测类型：

| 类型 | 文件 | 说明 |
|------|------|------|
| `default` | `default.ts` | 默认评测（标准输入输出） |
| `generate` | `generate.ts` | 数据生成器模式 |
| `interactive` | `interactive.ts` | 交互式评测 |
| `communication` | `communication.ts` | 通信题评测 |
| `run` | `run.ts` | 仅运行（自测） |
| `submit_answer` | `submit_answer.ts` | 提交答案题 |
| `objective` | `objective.ts` | 客观题（选择题等） |
| `hack` | `hack.ts` | Hack 攻击模式 |

#### 3.3.3 沙箱系统 (`packages/hydrojudge/src/sandbox/`)

提供安全的代码执行环境，支持资源限制。

| 功能 | 说明 |
|------|------|
| 内存限制 | 通过 cgroup 限制进程内存 |
| 时间限制 | 通过 timeout 限制执行时间 |
| 文件限制 | 限制文件读写 |
| 网络限制 | 默认禁用网络访问 |

#### 3.3.4 编译系统 (`packages/hydrojudge/src/compile.ts`)

支持多种编程语言的编译：

| 语言 | 编译命令 |
|------|----------|
| C++ | g++ |
| C | gcc |
| Java | javac |
| Python | python |
| JavaScript | node |
| Rust | rustc |
| Go | go build |

---

### 3.4 公共模块 (`packages/common/`)

#### 3.4.1 状态定义 (`packages/common/status.ts`)

**评测状态枚举：**

| 状态码 | 常量 | 说明 |
|--------|------|------|
| 0 | `STATUS_WAITING` | 等待评测 |
| 1 | `STATUS_ACCEPTED` | 通过 |
| 2 | `STATUS_WRONG_ANSWER` | 答案错误 |
| 3 | `STATUS_TIME_LIMIT_EXCEEDED` | 超时 |
| 4 | `STATUS_MEMORY_LIMIT_EXCEEDED` | 内存超限 |
| 5 | `STATUS_OUTPUT_LIMIT_EXCEEDED` | 输出超限 |
| 6 | `STATUS_RUNTIME_ERROR` | 运行时错误 |
| 7 | `STATUS_COMPILE_ERROR` | 编译错误 |
| 8 | `STATUS_SYSTEM_ERROR` | 系统错误 |
| 9 | `STATUS_CANCELED` | 已取消 |
| 20 | `STATUS_JUDGING` | 评测中 |
| 21 | `STATUS_COMPILING` | 编译中 |

#### 3.4.2 权限定义 (`packages/common/permission.ts`)

权限系统基于位运算，支持细粒度权限控制：

| 权限常量 | 说明 |
|----------|------|
| `PERM_VIEW` | 查看权限 |
| `PERM_EDIT` | 编辑权限 |
| `PERM_DELETE` | 删除权限 |
| `PERM_ADMIN` | 管理员权限 |

#### 3.4.3 题目类型 (`packages/common/types.ts`)

**题目类型枚举：**

```typescript
export enum ProblemType {
    Default = 'default',           // 标准题目
    SubmitAnswer = 'submit_answer', // 提交答案题
    Interactive = 'interactive',   // 交互式题目
    Communication = 'communication', // 通信题
    Objective = 'objective',       // 客观题
    Remote = 'remote_judge',       // 远程评测
}
```

**评测结果接口：**

```typescript
export interface JudgeResultBody {
    key: string;
    domainId: string;
    rid: string;
    status?: number;
    score?: number;
    time?: number;      // 毫秒
    memory?: number;    // KB
    message?: string | JudgeMessage;
    compilerText?: string;
    case?: TestCase;
    subtasks?: Record<number, SubtaskResult>;
}
```

---

## 四、依赖关系

### 4.1 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `cordis` | 4.0.0-rc.4 | 服务框架和依赖注入 |
| `koa` | ^3.2.0 | Web 服务器 |
| `@koa/router` | ^15.4.0 | 路由管理 |
| `mongodb` | ^7.2.0 | 数据库驱动 |
| `schemastery` | ^3.18.0 | 数据验证 |
| `ws` | ^8.20.0 | WebSocket 支持 |
| `lodash` | ^4.18.1 | 工具函数 |
| `moment-timezone` | ^0.6.2 | 时间处理 |

### 4.2 工作区依赖关系

```
hydro-workspace
├── @hydrooj/framework    # 核心框架
│   └── @hydrooj/utils
├── @hydrooj/utils        # 工具函数
├── hydrooj               # 主应用
│   ├── @hydrooj/framework
│   ├── @hydrooj/common
│   └── @hydrooj/utils
├── @hydrooj/hydrojudge   # 评测引擎
│   └── @hydrooj/common
├── @hydrooj/common       # 公共类型
└── @hydrooj/ui-default   # 前端界面
    ├── @hydrooj/common
    └── @hydrooj/framework
```

---

## 五、项目运行方式

### 5.1 环境要求

- Node.js >= 22
- MongoDB >= 7.0
- Yarn >= 4.0

### 5.2 安装步骤

```bash
# 安装依赖
yarn install

# 构建项目
yarn build

# 构建前端
yarn build:ui

# 启动开发服务器
yarn debug
```

### 5.3 命令行接口

```bash
# 启动服务
yarn start

# 调试模式
yarn debug

# 运行测试
yarn test

# 代码检查
yarn lint

# 构建前端（开发模式）
yarn build:ui:dev

# 构建前端（生产模式）
yarn build:ui:production
```

### 5.4 配置文件

主要配置文件：

| 文件 | 说明 |
|------|------|
| `install/config.yaml` | 安装配置 |
| `packages/hydrooj/setting.yaml` | 系统设置 |
| `packages/hydrojudge/judge.yaml` | 评测配置 |

---

## 六、关键设计模式

### 6.1 依赖注入（Dependency Injection）

使用 Cordis 框架实现依赖注入：

```typescript
export class MyService extends Service {
    constructor(ctx: Context) {
        super(ctx, 'myService');
        ctx.mixin('myService', ['method1', 'method2']);
    }
    
    method1() { /* ... */ }
    method2() { /* ... */ }
}
```

### 6.2 中间件模式（Middleware）

支持多层中间件：

```typescript
server.addServerLayer('myLayer', async (ctx, next) => {
    // 请求前处理
    await next();
    // 请求后处理
});
```

### 6.3 Hook 机制

通过事件总线实现 Hook 扩展：

```typescript
ctx.on('handler/before/ProblemEdit', (handler) => {
    // 在题目编辑前执行
});
```

### 6.4 插件系统

支持热插拔插件：

```typescript
ctx.inject(['server'], ({ Route }) => {
    Route('myRoute', '/my-path', MyHandler);
});
```

---

## 七、扩展开发

### 7.1 开发新插件

```typescript
import { Service } from 'cordis';
import { Route } from '@hydrooj/framework';

export class MyPlugin extends Service {
    constructor(ctx) {
        super(ctx, 'myPlugin');
        
        // 注册路由
        ctx.inject(['server'], ({ Route }) => {
            Route('myRoute', '/my-path', MyHandler);
        });
        
        // 注册 API
        ctx.inject(['api'], ({ api }) => {
            api.provide({
                'myPlugin.query': Query(Schema.object({}), () => ({ ok: true })),
            });
        });
    }
}
```

### 7.2 添加新评测类型

```typescript
// packages/hydrojudge/src/judge/myType.ts
export async function judge(ctx: Context) {
    // 实现评测逻辑
}

// packages/hydrojudge/src/judge/index.ts
import * as myType from './myType';

export = {
    default: def,
    // ...
    myType,
};
```

---

## 八、部署方式

### 8.1 Docker 部署

```bash
# 使用 Docker Compose
cd install/docker
docker-compose up -d
```

### 8.2 Kubernetes 部署

```bash
# 使用 Helm
helm install hydro ./install/helm-single
```

### 8.3 手动部署

```bash
# 安装依赖
yarn install

# 构建
yarn build && yarn build:ui:production

# 启动
yarn start
```

---

## 九、安全考虑

### 9.1 CSRF 防护

框架内置 CSRF 检查：

```typescript
async init() {
    if (this.request.method === 'post' && this.request.headers.referer) {
        const host = new URL(this.request.headers.referer).host;
        if (host !== this.request.host) throw new CsrfTokenError(host);
    }
}
```

### 9.2 输入验证

使用 Schemastery 进行参数验证：

```typescript
const schema = Schema.object({
    name: Schema.string().required(),
    age: Schema.number().min(0).max(150),
});
```

### 9.3 沙箱隔离

评测机使用沙箱环境限制代码执行权限，防止恶意代码攻击。

---

## 十、性能优化

### 10.1 缓存策略

- 测试数据缓存
- 编译结果缓存
- 页面渲染缓存

### 10.2 异步处理

使用异步队列处理评测任务，避免阻塞主线程。

### 10.3 负载均衡

支持多评测机部署，自动均衡任务分配。

---

## 附录：文件引用

| 文件 | 路径 |
|------|------|
| 主框架入口 | [framework/framework/index.ts](file:///workspace/framework/framework/index.ts) |
| API 服务 | [framework/framework/api.ts](file:///workspace/framework/framework/api.ts) |
| 服务器服务 | [framework/framework/server.ts](file:///workspace/framework/framework/server.ts) |
| 主应用入口 | [packages/hydrooj/src/plugin-api.ts](file:///workspace/packages/hydrooj/src/plugin-api.ts) |
| 评测任务 | [packages/hydrojudge/src/task.ts](file:///workspace/packages/hydrojudge/src/task.ts) |
| 公共类型 | [packages/common/types.ts](file:///workspace/packages/common/types.ts) |
| 状态定义 | [packages/common/status.ts](file:///workspace/packages/common/status.ts) |