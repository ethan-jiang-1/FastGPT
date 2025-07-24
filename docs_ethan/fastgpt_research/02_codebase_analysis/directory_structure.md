# FastGPT 目录结构深度解析

## 🏗️ 整体目录架构

FastGPT 采用标准的 **Monorepo** 架构，使用 pnpm workspace 进行包管理。整体目录结构清晰地体现了**分层架构**和**关注点分离**的设计理念。

### 顶层目录结构

```
FastGPT/
├── 📁 packages/          # 核心共享包 (业务逻辑层)
├── 📁 projects/          # 具体应用项目 (应用层)  
├── 📁 plugins/           # 插件和扩展 (扩展层)
├── 📁 docSite/          # 文档站点 (Hugo静态站点)
├── 📁 deploy/           # 部署配置 (Docker/K8s/Helm)
├── 📁 scripts/          # 构建和工具脚本
├── 📁 test/             # 测试用例和测试数据
├── 📄 package.json      # 项目根配置
├── 📄 pnpm-workspace.yaml # Monorepo工作区配置
├── 📄 tsconfig.json     # TypeScript全局配置
└── 📄 vitest.config.mts # 测试框架配置
```

## 📦 核心包结构分析 (packages/)

packages 目录包含了项目的核心业务逻辑，按功能模块划分为不同的包：

### packages/global/ - 全局类型和工具

```
packages/global/
├── 📁 common/           # 通用工具函数
│   ├── error/          # 错误处理和状态码
│   ├── file/           # 文件处理工具
│   ├── string/         # 字符串处理工具
│   ├── system/         # 系统配置和常量
│   └── time/           # 时间处理工具
├── 📁 core/            # 核心业务类型定义
│   ├── ai/            # AI服务相关类型
│   ├── app/           # 应用配置类型
│   ├── chat/          # 聊天相关类型
│   ├── dataset/       # 知识库类型
│   └── workflow/      # 工作流类型
└── 📁 support/         # 支持服务类型
    ├── user/          # 用户和团队类型
    ├── wallet/        # 计费和订阅类型
    └── permission/    # 权限管理类型
```

**关键文件分析**:
- `common/error/errorCode.ts` - 统一的错误码定义
- `core/workflow/type/` - 工作流节点和连接的类型系统
- `support/permission/type.d.ts` - 权限系统的类型架构

### packages/service/ - 后端服务层

```
packages/service/
├── 📁 common/           # 服务层通用组件
│   ├── api/            # API调用封装
│   ├── mongo/          # MongoDB连接和工具
│   ├── redis/          # Redis缓存操作
│   ├── vectorDB/       # 向量数据库操作
│   ├── bullmq/         # 任务队列服务
│   └── middle/         # 中间件层
├── 📁 core/            # 核心业务服务
│   ├── ai/            # AI模型服务
│   │   ├── config/    # 模型配置管理
│   │   ├── embedding/ # 向量化服务  
│   │   ├── rerank/    # 重排序服务
│   │   └── functions/ # AI功能函数
│   ├── app/           # 应用管理服务
│   ├── chat/          # 聊天服务
│   ├── dataset/       # 知识库服务
│   │   ├── training/  # 数据训练处理
│   │   ├── search/    # 检索服务
│   │   └── collection/ # 数据集合管理
│   └── workflow/      # 工作流执行引擎
│       └── dispatch/  # 节点调度器
├── 📁 support/         # 支持服务
│   ├── user/          # 用户管理
│   ├── permission/    # 权限控制
│   ├── wallet/        # 计费系统
│   └── openapi/       # API管理
└── 📁 worker/          # 后台任务处理
    ├── readFile/      # 文件解析处理
    ├── text2Chunks/   # 文本分块处理
    └── htmlStr2Md/    # HTML转Markdown
```

**核心服务模块深度分析**:

#### AI服务层 (`core/ai/`)
```typescript
// 模型提供商配置结构
core/ai/config/provider/
├── OpenAI.json        # OpenAI模型配置
├── Claude.json        # Anthropic Claude配置  
├── Qwen.json         # 阿里通义千问配置
├── ChatGLM.json      # 智谱GLM配置
└── ...               # 其他20+模型提供商
```

#### 工作流引擎 (`core/workflow/dispatch/`)
```typescript
workflow/dispatch/
├── index.ts          # 主要调度逻辑
├── ai/              # AI节点处理器
│   ├── chat.ts      # AI对话节点
│   ├── extract.ts   # 内容提取节点
│   └── classifyQuestion.ts # 问题分类节点
├── dataset/         # 数据集节点处理器
│   ├── search.ts    # 知识库检索节点
│   └── concat.ts    # 数据拼接节点
├── interactive/     # 交互节点处理器
│   ├── userSelect.ts # 用户选择节点
│   └── formInput.ts  # 表单输入节点
└── tools/           # 工具节点处理器
    ├── http468.ts   # HTTP请求节点
    ├── runLaf.ts    # Laf云函数节点
    └── codeSandbox.ts # 代码沙箱节点
```

### packages/web/ - 前端共享组件

```
packages/web/
├── 📁 components/      # 共享UI组件库
│   ├── common/        # 通用组件
│   │   ├── Avatar/    # 头像组件
│   │   ├── Icon/      # 图标组件
│   │   ├── MyModal/   # 模态框组件
│   │   ├── MySelect/  # 选择器组件
│   │   └── ...        # 其他通用组件
│   └── core/          # 业务相关组件
│       └── workflow/  # 工作流专用组件
├── 📁 hooks/          # React自定义Hooks
│   ├── useConfirm.tsx  # 确认对话框Hook
│   ├── useLoading.tsx  # 加载状态Hook
│   ├── usePagination.tsx # 分页Hook
│   └── useRequest.tsx  # 请求封装Hook
├── 📁 i18n/           # 国际化资源
│   ├── en/           # 英文翻译
│   ├── zh-CN/        # 简体中文
│   └── zh-Hant/      # 繁体中文
└── 📁 styles/         # 全局样式
    └── theme.ts       # Chakra UI主题配置
```

### packages/templates/ - 应用模板

```
packages/templates/src/
├── 📁 simpleDatasetChat/    # 简单知识库问答模板
├── 📁 TranslateRobot/       # 翻译机器人模板
├── 📁 chatGuide/            # 聊天引导模板
├── 📁 plugin-dalle/         # DALL-E图像生成模板
├── 📁 timeBot/              # 时间助手模板
├── 📁 CQ/                   # 问题分类模板
└── 📁 ...                   # 更多业务模板
```

每个模板都包含完整的工作流定义：
```json
{
  "name": "简单知识库问答",
  "intro": "最简单的知识库 + AI 问答模式",
  "modules": [
    // 完整的节点配置
  ]
}
```

## 🚀 应用项目结构 (projects/)

### projects/app/ - 主前端应用

```
projects/app/
├── 📁 public/          # 静态资源
│   ├── icon/          # 图标资源
│   ├── imgs/          # 图片资源
│   │   ├── app/       # 应用相关图片
│   │   ├── avatar/    # 头像图片
│   │   ├── workflow/  # 工作流图标
│   │   └── ...
│   └── js/            # 第三方JS库
├── 📁 src/            # 源代码
│   ├── 📁 components/ # 应用专用组件
│   │   ├── Layout/    # 布局组件
│   │   ├── SideBar/   # 侧边栏
│   │   ├── core/      # 核心业务组件
│   │   │   ├── ai/    # AI配置组件
│   │   │   ├── app/   # 应用管理组件
│   │   │   ├── chat/  # 聊天界面组件
│   │   │   └── dataset/ # 知识库组件
│   │   └── support/   # 支持功能组件
│   ├── 📁 pages/      # Next.js页面
│   │   ├── api/       # API路由
│   │   │   ├── v1/    # V1版本API
│   │   │   ├── core/  # 核心业务API
│   │   │   ├── support/ # 支持服务API
│   │   │   └── system/ # 系统API
│   │   ├── app/       # 应用相关页面
│   │   ├── chat/      # 聊天页面
│   │   ├── dataset/   # 知识库页面
│   │   └── account/   # 账户管理页面
│   ├── 📁 pageComponents/ # 页面级组件
│   ├── 📁 web/        # Web工具和API封装
│   ├── 📁 service/    # 前端服务层
│   ├── 📁 global/     # 全局类型和配置
│   └── 📁 types/      # TypeScript类型定义
├── 📄 next.config.js  # Next.js配置
├── 📄 next-i18next.config.js # 国际化配置
└── 📄 tsconfig.json   # TypeScript配置
```

**页面路由结构**:
```
页面路由映射:
├── / (首页)
├── /login (登录页)
├── /dashboard (控制台)
│   ├── /apps (应用列表)
│   └── /mcpServer (MCP服务器管理)
├── /app/detail (应用详情/编辑)
├── /chat (聊天界面)
├── /dataset (知识库管理)
│   ├── /list (知识库列表)
│   └── /detail (知识库详情)
└── /account (账户管理)
    ├── /info (个人信息)
    ├── /team (团队管理)
    ├── /bill (账单管理)
    └── /apikey (API密钥)
```

### projects/mcp_server/ - MCP协议服务

```
projects/mcp_server/
├── 📁 src/
│   ├── 📄 index.ts     # MCP服务器入口
│   ├── 📄 init.ts      # 初始化配置
│   ├── 📁 api/         # FastGPT API封装
│   │   ├── fastgpt.ts  # FastGPT接口调用
│   │   └── request.ts  # 请求封装
│   └── 📁 utils/       # 工具函数
├── 📄 package.json     # 依赖配置
└── 📄 Dockerfile       # 容器化配置
```

### projects/sandbox/ - 代码沙箱服务

```
projects/sandbox/
├── 📁 src/
│   ├── 📄 main.ts           # NestJS应用入口
│   ├── 📄 app.module.ts     # 应用模块定义
│   └── 📁 sandbox/          # 沙箱核心模块
│       ├── sandbox.controller.ts # 控制器
│       ├── sandbox.service.ts    # 业务逻辑
│       └── utils.ts              # 沙箱工具
├── 📄 requirements.txt      # Python依赖
├── 📄 nest-cli.json         # NestJS CLI配置
└── 📄 Dockerfile            # 容器化配置
```

## 🔌 插件生态结构 (plugins/)

### 模型插件 (plugins/model/)

```
plugins/model/
├── 📁 llm-ChatGLM2/         # ChatGLM2模型服务
├── 📁 llm-Baichuan2/        # 百川2模型服务
├── 📁 rerank-bge/           # BGE重排序模型
│   ├── bge-reranker-base/   # 基础版本
│   ├── bge-reranker-large/  # 大模型版本
│   └── bge-reranker-v2-m3/  # 多语言版本
├── 📁 stt-sensevoice/       # 语音转文字服务
├── 📁 tts-cosevoice/        # 文字转语音服务
├── 📁 ocr-surya/            # OCR识别服务
├── 📁 pdf-marker/           # PDF标记工具
└── 📁 pdf-mineru/           # PDF解析工具
```

每个模型插件都是独立的服务，包含：
- `Dockerfile` - 容器化配置
- `requirements.txt` - Python依赖
- `app.py` 或类似的服务入口文件

### 网络爬虫插件 (plugins/webcrawler/)

```
plugins/webcrawler/
├── 📁 SPIDER/               # 爬虫核心服务
│   ├── 📁 src/
│   │   ├── controllers/     # 控制器层
│   │   ├── engines/         # 搜索引擎适配
│   │   ├── routes/          # 路由定义
│   │   └── utils/           # 工具函数
│   └── 📄 package.json
├── 📁 searxng/              # SearXNG搜索引擎配置
├── 📄 docker-compose.yaml   # 服务编排
└── 📄 Dockerfile            # 容器化配置
```

## 📚 文档站点结构 (docSite/)

```
docSite/
├── 📁 content/zh-cn/docs/   # 中文文档内容
│   ├── development/         # 开发文档
│   ├── guide/              # 使用指南
│   ├── faq/                # 常见问题
│   └── use-cases/          # 使用案例
├── 📁 assets/              # 静态资源
│   ├── imgs/              # 文档图片
│   └── docs/              # 文档样式
├── 📁 layouts/             # Hugo模板
├── 📁 static/              # 静态文件
├── 📄 hugo.toml            # Hugo配置
└── 📄 nginx.conf           # Nginx配置
```

## 🚢 部署配置结构 (deploy/)

```
deploy/
├── 📁 docker/              # Docker部署
│   ├── docker-compose.yml  # 基础服务编排
│   ├── docker-compose-pgvector.yml # PostgreSQL向量版
│   ├── docker-compose-milvus.yml   # Milvus向量版
│   └── 📁 docker-compose/   # 完整部署配置
├── 📁 helm/                # Kubernetes Helm部署
│   └── 📁 fastgpt/         # Helm Chart
│       ├── Chart.yaml      # Chart元数据
│       ├── values.yaml     # 默认配置值
│       └── 📁 templates/   # Kubernetes模板
└── 📄 run.sh               # 快速启动脚本
```

## 🔧 工具脚本结构 (scripts/)

```
scripts/
├── 📁 openapi/             # OpenAPI文档生成
│   ├── index.ts           # 主要生成逻辑
│   ├── openapi.json       # API规范文件
│   └── template.md        # 文档模板
├── 📁 i18n/               # 国际化工具
├── 📁 icon/               # 图标处理工具
└── 📄 postinstall.sh      # 安装后脚本
```

## 🧪 测试结构 (test/)

```
test/
├── 📁 cases/              # 测试用例
│   ├── components/        # 组件测试
│   ├── pages/api/         # API测试
│   ├── global/core/       # 核心逻辑测试
│   └── service/           # 服务层测试
├── 📁 datas/              # 测试数据
├── 📁 mocks/              # Mock工具
├── 📄 globalSetup.ts      # 全局测试设置
└── 📄 setupModels.ts      # 模型测试设置
```

## 📊 目录结构设计原则

### 1. 分层架构原则
```
架构层次:
应用层 (projects/) → 业务逻辑层 (packages/) → 数据访问层
```

### 2. 职责单一原则
- 每个包都有明确的职责边界
- 避免循环依赖和紧耦合
- 便于独立开发和测试

### 3. 可扩展性原则
- 插件化的架构设计
- 标准化的接口定义
- 便于第三方扩展

### 4. 维护性原则
- 清晰的命名规范
- 完整的类型定义
- 充分的文档说明

## 🎯 目录结构优势

### 开发效率优势
1. **代码复用** - 共享包减少重复代码
2. **类型安全** - 统一的类型定义系统
3. **工具链统一** - 共享的构建和测试配置

### 维护性优势  
1. **模块化** - 独立的功能模块便于维护
2. **版本管理** - 统一的依赖版本控制
3. **测试覆盖** - 完整的测试体系

### 扩展性优势
1. **插件架构** - 灵活的功能扩展机制
2. **微服务就绪** - 服务拆分的基础架构
3. **多语言支持** - 完整的国际化架构

---

这个目录结构设计体现了现代化软件工程的最佳实践，通过合理的分层和模块化设计，为 FastGPT 的持续发展和扩展奠定了坚实的基础。每个目录都有其明确的职责和作用，形成了一个有机的整体架构。