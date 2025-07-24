# FastGPT 数据库架构深度分析

## 🗄️ 数据库架构概述

FastGPT 采用**多数据库分层架构**，针对不同的数据类型和性能需求选择最适合的存储方案：

### 核心数据库组件

- **MongoDB** - 应用数据、用户管理、元数据的主数据库
- **向量数据库** - 存储和检索嵌入向量（支持 PostgreSQL + pgvector、Milvus、OceanBase）
- **Redis** - 缓存、会话管理和队列操作

### 数据库连接配置

#### MongoDB 连接配置
**文件路径**: `packages/service/common/mongo/index.ts`

```typescript
export const MONGO_URL = process.env.MONGODB_URI as string
export const MONGO_LOG_URL = (process.env.MONGODB_LOG_URI ?? process.env.MONGODB_URI) as string

export const connectionMongo = (() => {
  if (!global.mongodb) {
    global.mongodb = new Mongoose()
  }
  return global.mongodb
})()

// 连接参数配置
const options = {
  bufferCommands: true,
  maxConnecting: maxConnecting,
  maxPoolSize: maxConnecting,
  minPoolSize: 20,
  connectTimeoutMS: 60000,
  waitQueueTimeoutMS: 60000,
  socketTimeoutMS: 60000,
  maxIdleTimeMS: 300000,
  retryWrites: true,
  retryReads: true
}
```

#### Redis 连接配置
**文件路径**: `packages/service/common/redis/index.ts`

```typescript
const REDIS_URL = process.env.REDIS_URL ?? 'redis://localhost:6379'
export const FASTGPT_REDIS_PREFIX = 'fastgpt:'

export const getGlobalRedisConnection = () => {
  if (global.redisClient) return global.redisClient
  
  global.redisClient = new Redis(REDIS_URL, { 
    keyPrefix: FASTGPT_REDIS_PREFIX 
  })
  return global.redisClient
}
```

#### 向量数据库配置
**文件路径**: `packages/service/common/vectorDB/constants.ts`

```typescript
export const PG_ADDRESS = process.env.PG_URL
export const OCEANBASE_ADDRESS = process.env.OCEANBASE_URL
export const MILVUS_ADDRESS = process.env.MILVUS_ADDRESS
export const MILVUS_TOKEN = process.env.MILVUS_TOKEN
```

系统根据环境变量自动选择向量数据库，默认使用 PostgreSQL。

## 🧑‍💼 用户管理架构

### 用户表 (users)

**文件路径**: `packages/service/support/user/schema.ts`

```typescript
const UserSchema = new Schema({
  status: {
    type: String,
    enum: Object.keys(userStatusMap),
    default: UserStatusEnum.active
  },
  username: {
    type: String,
    required: true,
    unique: true
  },
  phonePrefix: Number,
  password: {
    type: String,
    required: true,
    set: (val: string) => hashStr(val),    // 自动哈希
    get: (val: string) => hashStr(val),    // 自动哈希
    select: false                          // 查询时不返回
  },
  passwordUpdateTime: Date,
  createTime: {
    type: Date,
    default: () => new Date()
  },
  promotionRate: {
    type: Number,
    default: 0
  },
  timezone: {
    type: String,
    default: 'Asia/Shanghai'
  },
  lastLoginTmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members'
  },
  inviterId: {
    type: Schema.Types.ObjectId,
    ref: 'users'
  },
  openaiAccount: {
    type: {
      key: String,
      baseUrl: String
    }
  },
  contact: String,
  sourceDomain: String
})

// 索引优化
UserSchema.index({ createTime: -1 })
UserSchema.index({ username: 1 }, { unique: true })
```

### 团队表 (teams)

**文件路径**: `packages/service/support/user/team/teamSchema.ts`

```typescript
const TeamSchema = new Schema({
  name: {
    type: String,
    required: true
  },
  ownerId: {
    type: Schema.Types.ObjectId,
    ref: 'users',
    required: true
  },
  avatar: {
    type: String,
    default: '/icon/logo.svg'
  },
  createTime: {
    type: Date,
    default: () => Date.now()
  },
  balance: Number,
  teamDomain: String,
  limit: {
    lastExportDatasetTime: Date,
    lastWebsiteSyncTime: Date
  },
  lafAccount: {
    token: String,
    appid: String,
    pat: String
  },
  openaiAccount: {
    type: {
      key: String,
      baseUrl: String
    }
  },
  externalWorkflowVariables: {
    type: Object,
    default: {}
  },
  notificationAccount: String
})

// 索引优化
TeamSchema.index({ name: 1 })
TeamSchema.index({ ownerId: 1 })
TeamSchema.index({ createTime: -1 })
```

### 团队成员表 (team_members)

**文件路径**: `packages/service/support/user/team/teamMemberSchema.ts`

```typescript
const TeamMemberSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'users',
    required: true
  },
  avatar: {
    type: String,
    default: () => getRandomUserAvatar()
  },
  name: {
    type: String,
    default: 'Member'
  },
  status: {
    type: String,
    enum: Object.keys(TeamMemberStatusMap)
  },
  createTime: {
    type: Date,
    default: () => new Date()
  },
  updateTime: Date
})

// 复合索引优化
TeamMemberSchema.index({ teamId: 1, userId: 1 }, { unique: true })
TeamMemberSchema.index({ userId: 1 })  // 快速查找用户的团队
```

## 🤖 应用管理架构

### 应用表 (apps)

**文件路径**: `packages/service/core/app/schema.ts`

```typescript
const AppSchema = new Schema({
  parentId: {
    type: Schema.Types.ObjectId,
    ref: 'apps',
    default: null
  },
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  name: {
    type: String,
    required: true
  },
  type: {
    type: String,
    default: AppTypeEnum.workflow,
    enum: Object.values(AppTypeEnum)
  },
  version: {
    type: String,
    enum: ['v1', 'v2']
  },
  avatar: {
    type: String,
    default: '/icon/logo.svg'
  },
  intro: String,
  updateTime: {
    type: Date,
    default: () => new Date()
  },
  teamTags: [String],
  
  // 工作流配置
  modules: {
    type: Array,
    default: []
  },
  edges: {
    type: Array,
    default: []
  },
  
  // 聊天配置
  chatConfig: {
    type: chatConfigType
  },
  
  // 插件配置
  pluginData: {
    type: {
      nodeVersion: String,
      pluginUniId: String,
      apiSchemaStr: String,
      customHeaders: String
    }
  },
  
  // 定时触发配置
  scheduledTriggerConfig: {
    cronString: String,
    timezone: String,
    defaultPrompt: String
  },
  scheduledTriggerNextTime: Date,
  
  inited: Boolean,
  inheritPermission: {
    type: Boolean,
    default: true
  }
})

// 性能关键索引
AppSchema.index({ type: 1 })
AppSchema.index({ teamId: 1, updateTime: -1 })
AppSchema.index({ teamId: 1, type: 1 })
AppSchema.index(
  { scheduledTriggerConfig: 1, scheduledTriggerNextTime: -1 },
  {
    partialFilterExpression: {
      scheduledTriggerConfig: { $exists: true }
    }
  }
)
```

## 💬 聊天系统架构

### 聊天会话表 (chat)

**文件路径**: `packages/service/core/chat/chatSchema.ts`

```typescript
const ChatSchema = new Schema({
  chatId: {
    type: String,
    required: true
  },
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'users'
  },
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  appId: {
    type: Schema.Types.ObjectId,
    ref: 'apps',
    required: true
  },
  createTime: {
    type: Date,
    default: () => new Date()
  },
  updateTime: {
    type: Date,
    default: () => new Date()
  },
  title: {
    type: String,
    default: '历史记录'
  },
  customTitle: String,
  top: {
    type: Boolean,
    default: false
  },
  source: {
    type: String,
    required: true,
    enum: Object.values(ChatSourceEnum)
  },
  sourceName: String,
  shareId: String,
  outLinkUid: String,
  variableList: Array,
  welcomeText: String,
  variables: {
    type: Object,
    default: {}
  },
  pluginInputs: Array,
  metadata: {
    type: Object,
    default: {}
  }
})

// 聊天性能关键索引
ChatSchema.index({ chatId: 1 })
ChatSchema.index({ tmbId: 1, appId: 1, top: -1, updateTime: -1 })
ChatSchema.index({ appId: 1, chatId: 1 })
ChatSchema.index({ teamId: 1, appId: 1, updateTime: -1 })
ChatSchema.index({ shareId: 1, outLinkUid: 1, updateTime: -1 })
```

### 聊天消息表 (chatitems)

**文件路径**: `packages/service/core/chat/chatItemSchema.ts`

```typescript
const ChatItemSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'users'
  },
  chatId: {
    type: String,
    required: true
  },
  dataId: {
    type: String,
    required: true,
    default: () => getNanoid(22)
  },
  appId: {
    type: Schema.Types.ObjectId,
    ref: 'apps',
    required: true
  },
  time: {
    type: Date,
    default: () => new Date()
  },
  hideInUI: {
    type: Boolean,
    default: false
  },
  obj: {
    type: String,
    required: true,
    enum: Object.keys(ChatRoleMap)  // 'Human', 'AI', 'System'
  },
  value: {
    type: Array,    // 支持多模态内容
    default: []
  },
  memories: Object,     // 记忆相关数据
  errorMsg: String,     // 错误信息
  
  // 用户反馈系统
  userGoodFeedback: String,
  userBadFeedback: String,
  customFeedbacks: [String],
  adminFeedback: {
    type: {
      datasetId: String,
      collectionId: String,
      dataId: String,
      q: String,
      a: String
    }
  },
  
  // 节点响应详情
  nodeResponse: {
    type: Array,
    default: []
  },
  durationSeconds: Number  // 响应时长
})

// 消息查询优化索引
ChatItemSchema.index({ dataId: 1 })
ChatItemSchema.index({ appId: 1, chatId: 1, dataId: 1 })
ChatItemSchema.index({ teamId: 1, time: -1 })
ChatItemSchema.index(
  { obj: 1, time: -1 }, 
  { partialFilterExpression: { obj: 'Human' } }
)
```

## 📚 知识库系统架构

### 数据集表 (datasets)

**文件路径**: `packages/service/core/dataset/schema.ts`

```typescript
const DatasetSchema = new Schema({
  parentId: {
    type: Schema.Types.ObjectId,
    ref: 'datasets',
    default: null
  },
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'users'
  },
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  type: {
    type: String,
    enum: Object.keys(DatasetTypeMap),
    required: true,
    default: DatasetTypeEnum.dataset
  },
  avatar: {
    type: String,
    default: '/icon/logo.svg'
  },
  name: {
    type: String,
    required: true
  },
  updateTime: {
    type: Date,
    default: () => new Date()
  },
  
  // AI 模型配置
  vectorModel: {
    type: String,
    required: true,
    default: 'text-embedding-3-small'
  },
  agentModel: {
    type: String,
    required: true,
    default: 'gpt-4o-mini'
  },
  vlmModel: String,    // 视觉语言模型
  
  intro: String,
  
  // 网站抓取配置
  websiteConfig: {
    type: {
      url: {
        type: String,
        required: true
      },
      selector: {
        type: String,
        default: 'body'
      }
    }
  },
  
  // 分块设置
  chunkSettings: {
    type: ChunkSettings
  },
  
  inheritPermission: {
    type: Boolean,
    default: true
  },
  
  apiDatasetServer: Object
})

// 数据集索引
DatasetSchema.index({ teamId: 1, updateTime: -1 })
DatasetSchema.index({ type: 1 })
DatasetSchema.index({ teamId: 1, type: 1 })
```

### 数据集合表 (dataset_collections)

**文件路径**: `packages/service/core/dataset/collection/schema.ts`

```typescript
const DatasetCollectionSchema = new Schema({
  parentId: {
    type: Schema.Types.ObjectId,
    ref: 'dataset_collections',
    default: null
  },
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  datasetId: {
    type: Schema.Types.ObjectId,
    ref: 'datasets',
    required: true
  },
  type: {
    type: String,
    enum: Object.keys(DatasetCollectionTypeMap),
    required: true
  },
  name: {
    type: String,
    required: true
  },
  tags: {
    type: [String],
    default: []
  },
  createTime: {
    type: Date,
    default: () => new Date()
  },
  updateTime: {
    type: Date,
    default: () => new Date()
  },
  
  // 文件关联
  fileId: {
    type: Schema.Types.ObjectId,
    ref: 'dataset.files'
  },
  rawLink: String,
  apiFileId: String,
  externalFileId: String,
  externalFileUrl: String,
  
  // 文本处理相关
  rawTextLength: Number,
  hashRawText: String,
  metadata: {
    type: Object,
    default: {}
  },
  
  forbid: Boolean,            // 是否禁用
  customPdfParse: Boolean,    // 自定义PDF解析
  apiFileParentId: String,
  
  // 继承分块设置
  ...ChunkSettings
})

// 集合查询优化索引
DatasetCollectionSchema.index({ teamId: 1, fileId: 1 })
DatasetCollectionSchema.index({
  teamId: 1,
  datasetId: 1,
  parentId: 1,
  updateTime: -1
})
DatasetCollectionSchema.index({ teamId: 1, datasetId: 1, tags: 1 })
DatasetCollectionSchema.index({ teamId: 1, datasetId: 1, createTime: 1 })
```

### 数据集数据表 (dataset_datas)

**文件路径**: `packages/service/core/dataset/data/schema.ts`

```typescript
const DatasetDataSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  datasetId: {
    type: Schema.Types.ObjectId,
    ref: 'datasets',
    required: true
  },
  collectionId: {
    type: Schema.Types.ObjectId,
    ref: 'dataset_collections',
    required: true
  },
  
  // QA 对内容
  q: {
    type: String,
    required: true    // 问题/查询内容
  },
  a: String,          // 答案内容（可选）
  
  // 图像相关
  imageId: String,
  imageDescMap: Object,
  
  // 历史版本管理
  history: {
    type: [
      {
        q: String,
        a: String,
        updateTime: Date
      }
    ]
  },
  
  // 索引数据
  indexes: {
    type: [
      {
        defaultIndex: Boolean,
        type: {
          type: String,
          enum: Object.values(DatasetDataIndexTypeEnum),
          default: DatasetDataIndexTypeEnum.custom
        },
        dataId: {
          type: String,
          required: true
        },
        text: {
          type: String,
          required: true
        }
      }
    ],
    default: []
  },
  
  updateTime: {
    type: Date,
    default: () => new Date()
  },
  chunkIndex: {
    type: Number,
    default: 0
  },
  rebuilding: Boolean
})

// 数据检索关键索引
DatasetDataSchema.index({
  teamId: 1,
  datasetId: 1,
  collectionId: 1,
  chunkIndex: 1,
  updateTime: -1
})
DatasetDataSchema.index({ 
  teamId: 1, 
  datasetId: 1, 
  collectionId: 1, 
  'indexes.dataId': 1 
})
DatasetDataSchema.index({ 
  rebuilding: 1, 
  teamId: 1, 
  datasetId: 1 
})
```

## 🔐 权限系统架构

### 资源权限表 (resource_permissions)

**文件路径**: `packages/service/support/permission/schema.ts`

```typescript
const ResourcePermissionSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members'
  },
  groupId: {
    type: Schema.Types.ObjectId,
    ref: 'member_groups'
  },
  orgId: {
    type: Schema.Types.ObjectId,
    ref: 'orgs'
  },
  resourceType: {
    type: String,
    enum: Object.values(PerResourceTypeEnum),
    required: true
  },
  permission: {
    type: Number,         // 权限位掩码
    required: true
  },
  resourceId: {
    type: Schema.Types.ObjectId    // 关联的资源ID
  }
})

// 权限查询优化的唯一索引
ResourcePermissionSchema.index(
  {
    resourceType: 1,
    teamId: 1,
    resourceId: 1,
    tmbId: 1
  },
  {
    unique: true,
    partialFilterExpression: {
      tmbId: { $exists: true }
    }
  }
)

ResourcePermissionSchema.index(
  {
    resourceType: 1,
    teamId: 1,
    resourceId: 1,
    groupId: 1
  },
  {
    unique: true,
    partialFilterExpression: {
      groupId: { $exists: true }
    }
  }
)
```

## 💰 计费与使用统计架构

### 使用统计表 (usages)

**文件路径**: `packages/service/support/wallet/usage/schema.ts`

```typescript
const UsageSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  tmbId: {
    type: Schema.Types.ObjectId,
    ref: 'team_members',
    required: true
  },
  source: {
    type: String,
    enum: Object.values(UsageSourceEnum),
    required: true
  },
  appName: {
    type: String,
    default: ''
  },
  appId: {
    type: Schema.Types.ObjectId,
    ref: 'apps'
  },
  pluginId: {
    type: Schema.Types.ObjectId,
    ref: 'plugins'
  },
  time: {
    type: Date,
    default: () => new Date()
  },
  totalPoints: {
    type: Number,
    required: true
  },
  list: {
    type: Array,
    default: []
  }
})

// 使用统计查询索引
UsageSchema.index({ teamId: 1, time: 1, tmbId: 1, source: 1 })
UsageSchema.index({ teamId: 1, time: 1, appName: 1 })

// TTL索引 - 360天后自动删除
UsageSchema.index(
  { time: 1 }, 
  { expireAfterSeconds: 360 * 24 * 60 * 60 }
)
```

## 🔍 向量数据库架构

### Milvus 实现

**文件路径**: `packages/service/common/vectorDB/milvus/index.ts`

**集合结构**:
```typescript
// 集合: modeldata  
// 数据库: fastgpt
const fields = [
  {
    name: 'id',
    data_type: DataType.Int64,
    is_primary_key: true,
    autoID: false
  },
  {
    name: 'vector',
    data_type: DataType.FloatVector,
    dim: 1536  // OpenAI embedding 维度
  },
  { name: 'teamId', data_type: DataType.VarChar, max_length: 64 },
  { name: 'datasetId', data_type: DataType.VarChar, max_length: 64 },
  { name: 'collectionId', data_type: DataType.VarChar, max_length: 64 },
  {
    name: 'createTime',
    data_type: DataType.Int64
  }
]

// 索引配置
const index_params = [
  {
    field_name: 'vector',
    index_name: 'vector_HNSW',
    index_type: 'HNSW',
    metric_type: 'IP',  // 内积相似度
    params: { efConstruction: 32, M: 64 }
  },
  {
    field_name: 'teamId',
    index_type: 'Trie'
  },
  {
    field_name: 'datasetId',
    index_type: 'Trie'  
  },
  {
    field_name: 'collectionId',
    index_type: 'Trie'
  },
  {
    field_name: 'createTime',
    index_type: 'STL_SORT'
  }
]
```

### PostgreSQL + pgvector 实现

**文件路径**: `packages/service/common/vectorDB/pg/controller.ts`

**连接配置**:
```typescript
const options = {
  connectionString: PG_ADDRESS,
  max: Number(process.env.DB_MAX_LINK || 20),
  min: 10,
  keepAlive: true,
  idleTimeoutMillis: 600000,
  connectionTimeoutMillis: 20000,
  query_timeout: 30000,
  statement_timeout: 40000,
  idle_in_transaction_session_timeout: 60000
}
```

**表结构**:
```sql
CREATE TABLE IF NOT EXISTS modeldata (
  id BIGSERIAL PRIMARY KEY,
  vector vector(1536),
  team_id varchar(50) NOT NULL,
  dataset_id varchar(50) NOT NULL,
  collection_id varchar(50) NOT NULL,
  create_time bigint NOT NULL
);

-- 向量索引 (HNSW)
CREATE INDEX IF NOT EXISTS modeldata_vector_hnsw_idx 
ON modeldata USING hnsw (vector vector_ip_ops);

-- 过滤索引
CREATE INDEX IF NOT EXISTS modeldata_team_id_idx 
ON modeldata (team_id);
CREATE INDEX IF NOT EXISTS modeldata_dataset_id_idx 
ON modeldata (dataset_id);
CREATE INDEX IF NOT EXISTS modeldata_collection_id_idx 
ON modeldata (collection_id);
```

## 📊 核心数据关系图

### 实体关系图

```
用户 (users)
    ↓ 1:N
团队成员 (team_members) ← 关联 → 团队 (teams)
    ↓ 1:N                      ↓ 1:N
应用 (apps)                    数据集 (datasets)
    ↓ 1:N                      ↓ 1:N
聊天会话 (chat)                集合 (dataset_collections)
    ↓ 1:N                      ↓ 1:N
聊天消息 (chatitems)           数据 (dataset_datas) → 向量库
```

### 权限继承关系

```
团队 (teams)
    ↓ 继承
组织 (orgs)
    ↓ 继承  
成员组 (member_groups)
    ↓ 继承
团队成员 (team_members)
    ↓ 访问
资源权限 (resource_permissions)
```

## ⚡ 性能优化策略

### 关键索引设计

#### 1. 聊天系统性能索引
```typescript
// 快速获取聊天历史
ChatSchema.index({ tmbId: 1, appId: 1, top: -1, updateTime: -1 })

// 聊天消息查找
ChatItemSchema.index({ appId: 1, chatId: 1, dataId: 1 })
```

#### 2. 数据集检索性能索引
```typescript
// 数据集合过滤
DatasetCollectionSchema.index({
  teamId: 1,
  datasetId: 1,
  parentId: 1,
  updateTime: -1
})

// 向量召回优化
DatasetDataSchema.index({ 
  teamId: 1, 
  datasetId: 1, 
  collectionId: 1, 
  'indexes.dataId': 1 
})
```

#### 3. 权限查找优化
```typescript
// 快速权限检查（唯一约束）
ResourcePermissionSchema.index(
  {
    resourceType: 1,
    teamId: 1,
    resourceId: 1,
    tmbId: 1
  },
  { unique: true }
)
```

### 缓存策略

#### Redis 缓存模式
```typescript
const getVectorCountByTeamId = async (teamId: string) => {
  const key = `${CacheKeyEnum.team_vector_count}:${teamId}`
  
  const countStr = await getRedisCache(key)
  if (countStr) {
    return Number(countStr)
  }
  
  const count = await Vector.getVectorCountByTeamId(teamId)
  await setRedisCache(key, count, CacheKeyEnumTime.team_vector_count)
  
  return count
}
```

### 慢查询监控

**文件路径**: `packages/service/common/mongo/index.ts`

```typescript
const addCommonMiddleware = (schema: mongoose.Schema) => {
  operations.forEach((op: any) => {
    schema.post(op, function (this: any, result: any, next) {
      if (this._startTime) {
        const duration = Date.now() - this._startTime
        if (duration > 1000) {
          addLog.warn(`Slow operation ${duration}ms`, {
            collectionName: this.collection?.name,
            op: this.op,
            query: this._query,
            duration
          })
        }
      }
      next()
    })
  })
}
```

## 🛡️ 安全设计

### 密码安全
```typescript
// 自动密码哈希
password: {
  type: String,
  required: true,
  set: (val: string) => hashStr(val),
  get: (val: string) => hashStr(val),
  select: false  // 查询时永不返回密码
}
```

### 数据隔离
- 所有主要集合都包含 `teamId` 实现多租户
- 自动团队范围查询
- 防止跨团队数据访问
- 团队级别的向量存储隔离

### 权限验证
- 分层权限验证系统
- 团队级权限
- 资源特定权限  
- 成员组继承
- 组织级控制

## 📋 环境变量配置

### 必需的数据库配置

```bash
# MongoDB
MONGODB_URI=mongodb://username:password@host:port/database
MONGODB_LOG_URI=mongodb://username:password@host:port/logs  # 可选，默认使用 MONGODB_URI

# Redis
REDIS_URL=redis://localhost:6379

# 向量数据库（选择其一）
PG_URL=postgresql://username:password@host:port/database
MILVUS_ADDRESS=http://localhost:19530
MILVUS_TOKEN=your_milvus_token
OCEANBASE_URL=your_oceanbase_connection_string

# 连接池设置  
DB_MAX_LINK=20  # 最大数据库连接数
SYNC_INDEX=1    # 启用自动索引同步
```

## 🔄 数据迁移与初始化

系统包含版本特定的初始化脚本，位于：
- `projects/app/src/pages/api/admin/initv*.ts`

处理内容：
- 版本间模式迁移
- 数据转换和清理
- 索引创建和优化
- 无效数据检测和移除

---

这个全面的数据库架构为 FastGPT 的核心功能提供支持，包括用户管理、AI应用、数据集管理、聊天系统和基于向量的检索，同时保持性能、安全性和可扩展性。