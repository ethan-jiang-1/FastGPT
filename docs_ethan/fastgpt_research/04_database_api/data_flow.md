# FastGPT 数据流架构深度分析

## 🌊 数据流架构概述

FastGPT 采用**分层数据流架构**，实现了从 API 端点到数据库的完整数据处理管道。系统通过**事件驱动**和**流式处理**模式，保证了高性能的实时 AI 对话体验和数据一致性。

### 核心数据流模式

```
用户请求 → 身份验证 → 权限检查 → 工作流调度 → 节点执行 → 数据持久化 → 实时响应
    ↓           ↓           ↓           ↓           ↓           ↓           ↓
  HTTP API  → JWT/API Key → RBAC系统  → 任务分发  → AI处理   → MongoDB   → SSE流
```

## 🔄 请求处理流程

### 1. 聊天API完整数据流

**入口文件**: `projects/app/src/pages/api/v1/chat/completions.ts`

```typescript
// 完整的聊天请求处理流程
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    // 1. 身份验证层
    const { teamId, tmbId, apikey, appId } = await authCert({
      req,
      authToken: true,
      authApiKey: true
    })

    // 2. 应用权限验证
    const { app } = await authApp({
      appId,
      per: ReadPermissionVal,
      teamId,
      tmbId
    })

    // 3. 消息历史处理
    const { messages, variables, ...restProps } = req.body
    const chatHistory = await getChatHistory(messages)

    // 4. 工作流调度
    const workflowResult = await dispatchWorkFlow({
      runningAppInfo: { id: appId, teamId, tmbId },
      runtimeNodes: app.modules,
      runtimeEdges: app.edges,
      variables: variables || {},
      histories: chatHistory,
      stream: req.body.stream || false,
      detail: req.body.detail || false
    })

    // 5. 响应处理
    if (req.body.stream) {
      await handleStreamResponse(res, workflowResult)
    } else {
      res.json(await formatResponse(workflowResult))
    }

  } catch (error) {
    handleApiError(res, error)
  }
}
```

### 2. 分层数据转换管道

#### 消息格式转换层
**文件路径**: `packages/global/core/chat/adapt.ts`

```typescript
// OpenAI格式 → FastGPT内部格式
export const GPTMessages2Chats = (
  messages: GPTMessages
): ChatItemType[] => {
  return messages.map((item, index) => ({
    dataId: getNanoid(24),
    obj: (() => {
      switch (item.role) {
        case 'system':
          return ChatRoleEnum.System
        case 'user':
          return ChatRoleEnum.Human
        case 'assistant':
          return ChatRoleEnum.AI
        default:
          return ChatRoleEnum.Human
      }
    })(),
    value: (() => {
      if (typeof item.content === 'string') {
        return [{ type: ChatItemValueTypeEnum.text, text: { content: item.content } }]
      }
      // 处理多模态内容
      return item.content.map(contentItem => {
        if (contentItem.type === 'text') {
          return {
            type: ChatItemValueTypeEnum.text,
            text: { content: contentItem.text }
          }
        } else if (contentItem.type === 'image_url') {
          return {
            type: ChatItemValueTypeEnum.image,
            image: { url: contentItem.image_url.url }
          }
        }
      })
    })()
  }))
}

// FastGPT内部格式 → 运行时提示词
export const chatValue2RuntimePrompt = (value: ChatItemValueType[]): string => {
  return value
    .map((item) => {
      if (item.type === ChatItemValueTypeEnum.text) {
        return item.text?.content || ''
      } else if (item.type === ChatItemValueTypeEnum.image) {
        return `[图片: ${item.image?.url}]`
      }
      return ''
    })
    .join('\n')
    .trim()
}
```

#### 工作流节点转换层
**文件路径**: `packages/service/core/workflow/dispatch/index.ts`

```typescript
// 静态节点 → 运行时节点
export const storeNodes2RuntimeNodes = (
  nodes: FlowNodeType[],
  variables: Record<string, any>
): RuntimeNodeType[] => {
  return nodes.map((node) => {
    // 变量注入处理
    const runtimeInputs = node.inputs.map((input) => {
      if (input.value && typeof input.value === 'string') {
        // 替换模板变量 {{variable}}
        const processedValue = input.value.replace(
          /\{\{(.*?)\}\}/g,
          (match, varName) => {
            return variables[varName.trim()] || match
          }
        )
        return { ...input, value: processedValue }
      }
      return input
    })

    return {
      ...node,
      inputs: runtimeInputs,
      outputs: [],
      status: RuntimeNodeStatusEnum.waiting
    }
  })
}

// 历史记录重写节点输出
export const rewriteNodeOutputByHistories = (
  runtimeNodes: RuntimeNodeType[],
  histories: ChatItemType[]
): RuntimeNodeType[] => {
  return runtimeNodes.map((node) => {
    // 查找历史中的相同节点执行记录
    const historyNodeResponse = histories
      .flatMap(item => item.nodeResponse || [])
      .find(response => response.nodeId === node.nodeId)

    if (historyNodeResponse) {
      return {
        ...node,
        outputs: historyNodeResponse.outputs || [],
        status: RuntimeNodeStatusEnum.completed
      }
    }

    return node
  })
}
```

## 🎯 事件驱动架构

### 1. Server-Sent Events 实现

**文件路径**: `packages/service/core/workflow/dispatch/utils.ts`

```typescript
// SSE事件类型定义
export enum SseResponseEventEnum {
  answer = 'answer',                    // 回答内容
  flowNodeStatus = 'flowNodeStatus',    // 节点状态
  interactive = 'interactive',          // 交互组件
  flowResponses = 'flowResponses',      // 工作流响应
  toolCall = 'toolCall',               // 工具调用
  toolResponse = 'toolResponse'         // 工具响应
}

// 流式响应处理器
export const workflowStreamResponse = (props: {
  event: SseResponseEventEnum
  data: any
}) => {
  const { event, data } = props

  // 格式化SSE数据
  const sseData = {
    event,
    data: JSON.stringify(data),
    id: getNanoid(24),
    timestamp: Date.now()
  }

  // 发送到客户端
  if (global.sseResponse) {
    global.sseResponse.write(`event: ${event}\n`)
    global.sseResponse.write(`data: ${sseData.data}\n`)
    global.sseResponse.write(`id: ${sseData.id}\n\n`)
  }
}

// 心跳机制
const sendStreamTimerSign = () => {
  const timer = setTimeout(() => {
    if (global.sseResponse && !global.sseResponse.destroyed) {
      workflowStreamResponse({
        event: SseResponseEventEnum.answer,
        data: textAdaptGptResponse({ text: '' })
      })
      sendStreamTimerSign() // 递归调用
    }
  }, 10000) // 10秒心跳

  // 清理定时器
  global.sseCleanup = () => {
    clearTimeout(timer)
  }
}
```

### 2. 实时状态更新机制

```typescript
// 节点执行状态实时推送
const updateNodeStatus = async (
  nodeId: string,
  status: RuntimeNodeStatusEnum,
  data?: any
) => {
  // 更新本地状态
  const node = runtimeNodes.find(n => n.nodeId === nodeId)
  if (node) {
    node.status = status
  }

  // 推送状态更新
  workflowStreamResponse({
    event: SseResponseEventEnum.flowNodeStatus,
    data: {
      nodeId,
      status,
      name: node?.name || '',
      data: data || {}
    }
  })

  // 如果是完成状态，推送详细响应
  if (status === RuntimeNodeStatusEnum.completed && data) {
    workflowStreamResponse({
      event: SseResponseEventEnum.flowResponses,
      data: {
        nodeId,
        response: filterPublicNodeResponseData(data)
      }
    })
  }
}
```

## 🚀 队列系统与后台处理

### 1. BullMQ队列架构

**文件路径**: `packages/service/common/bullmq/index.ts`

```typescript
// 队列名称定义
export enum QueueNames {
  datasetSync = 'datasetSync',      // 数据集同步
  evaluation = 'evaluation',        // 评估任务
  websiteSync = 'websiteSync',      // 网站同步
  training = 'training',           // 训练任务
  vectorGeneration = 'vectorGeneration' // 向量生成
}

// 队列工厂
class QueueFactory {
  private queues = new Map<string, Queue>()
  private workers = new Map<string, Worker>()

  // 创建队列
  createQueue(name: QueueNames): Queue {
    if (this.queues.has(name)) {
      return this.queues.get(name)!
    }

    const queue = new Queue(name, {
      connection: {
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT || '6379'),
        password: process.env.REDIS_PASSWORD
      },
      defaultJobOptions: {
        removeOnComplete: 10,  // 保留10个已完成任务
        removeOnFail: 5,       // 保留5个失败任务
        attempts: 3,           // 最大重试3次
        backoff: {
          type: 'exponential',
          delay: 2000
        }
      }
    })

    this.queues.set(name, queue)
    return queue
  }

  // 创建工作器
  createWorker(name: QueueNames, processor: (job: Job) => Promise<any>): Worker {
    if (this.workers.has(name)) {
      return this.workers.get(name)!
    }

    const worker = new Worker(name, processor, {
      connection: {
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT || '6379'),
        password: process.env.REDIS_PASSWORD
      },
      concurrency: 5,  // 并发处理5个任务
      maxStalledCount: 1,
      stalledInterval: 30000
    })

    // 错误处理
    worker.on('failed', (job, err) => {
      console.error(`Job ${job?.id} failed:`, err)
    })

    worker.on('completed', (job) => {
      console.log(`Job ${job.id} completed`)
    })

    this.workers.set(name, worker)
    return worker
  }
}
```

### 2. 数据集训练队列处理

**文件路径**: `packages/service/core/dataset/training/controller.ts`

```typescript
// 训练数据推送到队列
export const pushDataListToTrainingQueue = async (props: {
  teamId: string
  tmbId: string
  datasetId: string
  collectionId: string
  agentModel: string
  vectorModel: string
  data: TrainingDataItemType[]
  billId?: string
  mode: TrainingModeEnum
}): Promise<string> => {
  const { teamId, tmbId, datasetId, collectionId, data, mode } = props

  // 创建训练任务
  const trainingQueue = QueueFactory.createQueue(QueueNames.training)
  
  const job = await trainingQueue.add('processTrainingData', {
    ...props,
    batchSize: 500,  // 每批处理500条数据
    priority: mode === TrainingModeEnum.chunk ? 1 : 2 // 分块训练优先级更高
  }, {
    priority: mode === TrainingModeEnum.chunk ? 10 : 5,
    delay: mode === TrainingModeEnum.auto ? 1000 : 0 // 自动模式延迟1秒
  })

  return job.id!
}

// 训练队列处理器
const trainingWorker = QueueFactory.createWorker(
  QueueNames.training,
  async (job: Job) => {
    const {
      teamId,
      tmbId,
      datasetId,
      collectionId,
      agentModel,
      vectorModel,
      data,
      batchSize,
      mode
    } = job.data

    console.log(`开始处理训练任务: ${job.id}`)

    try {
      // 分批处理数据
      const batches = chunkArray(data, batchSize)
      let processedCount = 0

      for (let i = 0; i < batches.length; i++) {
        const batch = batches[i]
        
        // 更新任务进度
        await job.updateProgress(Math.floor((i / batches.length) * 100))

        // 处理当前批次
        await processBatch({
          teamId,
          tmbId,
          datasetId,
          collectionId,
          batch,
          agentModel,
          vectorModel,
          mode
        })

        processedCount += batch.length
        console.log(`已处理 ${processedCount}/${data.length} 条数据`)
      }

      return {
        success: true,
        processedCount,
        totalCount: data.length
      }

    } catch (error) {
      console.error(`训练任务 ${job.id} 失败:`, error)
      throw error
    }
  }
)

// 批次处理函数
const processBatch = async (props: {
  teamId: string
  tmbId: string
  datasetId: string
  collectionId: string
  batch: TrainingDataItemType[]
  agentModel: string
  vectorModel: string
  mode: TrainingModeEnum
}) => {
  const { teamId, tmbId, datasetId, collectionId, batch, vectorModel, mode } = props

  // 使用事务确保数据一致性
  await mongoSessionRun(async (session) => {
    // 批量插入数据
    const insertedDocs = await MongoDatasetData.insertMany(
      batch.map((item, index) => ({
        teamId,
        tmbId,
        datasetId,
        collectionId,
        q: item.q,
        a: item.a || '',
        chunkIndex: item.chunkIndex || index,
        indexes: []
      })),
      { session }
    )

    // 批量生成向量
    const vectorPromises = insertedDocs.map(async (doc) => {
      if (doc.q) {
        return generateAndSaveVector({
          teamId,
          datasetId,
          collectionId,
          dataId: doc._id.toString(),
          text: doc.q,
          model: vectorModel
        })
      }
    })

    await Promise.all(vectorPromises)
  })
}
```

## 💾 缓存机制与数据一致性

### 1. Redis缓存架构

**文件路径**: `packages/service/common/redis/cache.ts`

```typescript
// 缓存键枚举
export enum CacheKeyEnum {
  team_vector_count = 'team_vector_count',      // 团队向量数量
  team_point_surplus = 'team_point_surplus',    // 团队剩余积分
  team_point_total = 'team_point_total',        // 团队总积分
  app_info = 'app_info',                        // 应用信息
  dataset_info = 'dataset_info',                // 数据集信息
  user_session = 'user_session'                 // 用户会话
}

// 缓存过期时间（秒）
export enum CacheKeyEnumTime {
  team_vector_count = 30 * 60,     // 30分钟
  team_point_surplus = 60,         // 1分钟
  team_point_total = 60,           // 1分钟
  app_info = 10 * 60,              // 10分钟
  dataset_info = 10 * 60,          // 10分钟
  user_session = 24 * 60 * 60      // 24小时
}

// 缓存操作类
class CacheManager {
  private redis = getGlobalRedisConnection()

  // 获取缓存
  async getCache<T = any>(key: string): Promise<T | null> {
    try {
      const cached = await this.redis.get(key)
      if (cached) {
        return JSON.parse(cached)
      }
      return null
    } catch (error) {
      console.warn(`缓存获取失败: ${key}`, error)
      return null
    }
  }

  // 设置缓存
  async setCache(
    key: string, 
    value: any, 
    ttl: number = 300
  ): Promise<void> {
    try {
      await this.redis.setex(key, ttl, JSON.stringify(value))
    } catch (error) {
      console.warn(`缓存设置失败: ${key}`, error)
    }
  }

  // 删除缓存
  async delCache(key: string): Promise<void> {
    try {
      await this.redis.del(key)
    } catch (error) {
      console.warn(`缓存删除失败: ${key}`, error)
    }
  }

  // 增量更新缓存
  async incrCache(key: string, increment: number = 1): Promise<number> {
    try {
      return await this.redis.incrby(key, increment)
    } catch (error) {
      console.warn(`缓存递增失败: ${key}`, error)
      return 0
    }
  }

  // 批量删除缓存（模式匹配）
  async delCachesByPattern(pattern: string): Promise<void> {
    try {
      const keys = await this.redis.keys(pattern)
      if (keys.length > 0) {
        await this.redis.del(...keys)
      }
    } catch (error) {
      console.warn(`批量缓存删除失败: ${pattern}`, error)
    }
  }
}

// 全局缓存管理器实例
export const cacheManager = new CacheManager()
```

### 2. 缓存一致性策略

```typescript
// 写透缓存模式（Write-Through）
const updateTeamPointSurplus = async (
  teamId: string, 
  change: number
): Promise<number> => {
  const cacheKey = `${CacheKeyEnum.team_point_surplus}:${teamId}`

  try {
    // 1. 更新数据库
    const team = await MongoTeam.findByIdAndUpdate(
      teamId,
      { $inc: { balance: change } },
      { new: true }
    )

    if (!team) {
      throw new Error('Team not found')
    }

    // 2. 更新缓存
    await cacheManager.setCache(
      cacheKey,
      team.balance,
      CacheKeyEnumTime.team_point_surplus
    )

    return team.balance

  } catch (error) {
    // 3. 删除缓存（确保一致性）
    await cacheManager.delCache(cacheKey)
    throw error
  }
}

// 懒加载缓存模式（Cache-Aside）
const getTeamVectorCount = async (teamId: string): Promise<number> => {
  const cacheKey = `${CacheKeyEnum.team_vector_count}:${teamId}`

  // 1. 尝试从缓存获取
  const cached = await cacheManager.getCache<number>(cacheKey)
  if (cached !== null) {
    return cached
  }

  // 2. 缓存未命中，查询数据库
  const count = await Vector.getVectorCountByTeamId(teamId)

  // 3. 更新缓存
  await cacheManager.setCache(
    cacheKey,
    count,
    CacheKeyEnumTime.team_vector_count
  )

  return count
}

// 缓存失效策略
const invalidateRelatedCaches = async (teamId: string, type: 'app' | 'dataset' | 'vector') => {
  const patterns = []

  switch (type) {
    case 'app':
      patterns.push(`${CacheKeyEnum.app_info}:${teamId}:*`)
      break
    case 'dataset':
      patterns.push(`${CacheKeyEnum.dataset_info}:${teamId}:*`)
      patterns.push(`${CacheKeyEnum.team_vector_count}:${teamId}`)
      break
    case 'vector':
      patterns.push(`${CacheKeyEnum.team_vector_count}:${teamId}`)
      break
  }

  // 批量删除相关缓存
  await Promise.all(
    patterns.map(pattern => cacheManager.delCachesByPattern(pattern))
  )
}
```

## 📊 事务管理与数据完整性

### 1. MongoDB事务封装

**文件路径**: `packages/service/common/mongo/sessionRun.ts`

```typescript
// 事务会话包装器
export const mongoSessionRun = async <T>(
  fn: (session: ClientSession) => Promise<T>,
  connection = connectionMongo
): Promise<T> => {
  const session = await connection.startSession()
  const timeout = 60000 // 60秒超时

  try {
    // 开始事务
    session.startTransaction({
      maxCommitTimeMS: timeout,
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority', j: true }
    })

    // 执行业务逻辑
    const result = await fn(session)

    // 提交事务
    await session.commitTransaction()
    return result

  } catch (error) {
    // 事务回滚
    if (!session.transaction.isCommitted) {
      await session.abortTransaction()
    }
    
    console.error('事务执行失败:', error)
    return Promise.reject(error)

  } finally {
    // 清理会话
    await session.endSession()
  }
}

// 带重试的事务执行
export const mongoSessionRunWithRetry = async <T>(
  fn: (session: ClientSession) => Promise<T>,
  maxRetries: number = 3
): Promise<T> => {
  let lastError: any

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await mongoSessionRun(fn)
    } catch (error) {
      lastError = error
      
      // 检查是否为可重试错误
      if (
        error.hasErrorLabel('TransientTransactionError') ||
        error.hasErrorLabel('UnknownTransactionCommitResult')
      ) {
        console.warn(`事务重试 ${attempt}/${maxRetries}:`, error.message)
        
        // 指数退避
        const delay = Math.min(1000 * Math.pow(2, attempt - 1), 5000)
        await new Promise(resolve => setTimeout(resolve, delay))
        
        continue
      }
      
      // 非可重试错误，直接抛出
      throw error
    }
  }

  throw lastError
}
```

### 2. 关键业务事务模式

```typescript
// 聊天保存事务
export const saveChatWithTransaction = async (props: {
  chatId: string
  appId: string
  teamId: string
  tmbId: string
  messages: ChatItemType[]
  variables: Record<string, any>
  updateTime: Date
}) => {
  const { chatId, appId, teamId, tmbId, messages, variables, updateTime } = props

  await mongoSessionRun(async (session) => {
    // 1. 更新或创建聊天记录
    await MongoChat.findOneAndUpdate(
      { chatId, appId },
      {
        $set: {
          teamId,
          tmbId,
          updateTime,
          variables,
          title: messages[0]?.value?.find(v => v.type === 'text')?.text?.content?.slice(0, 20) || '新对话'
        }
      },
      { 
        upsert: true, 
        new: true, 
        session 
      }
    )

    // 2. 批量插入聊天消息
    if (messages.length > 0) {
      await MongoChatItem.insertMany(
        messages.map(msg => ({
          ...msg,
          chatId,
          appId,
          teamId,
          tmbId,
          time: updateTime
        })),
        { session }
      )
    }

    // 3. 更新使用统计
    await MongoUsage.create([{
      teamId,
      tmbId,
      appId,
      source: UsageSourceEnum.chat,
      time: updateTime,
      totalPoints: calculateUsagePoints(messages),
      list: buildUsageList(messages)
    }], { session })
  })
}

// 数据集训练事务
export const processDatasetTrainingWithTransaction = async (props: {
  teamId: string
  tmbId: string
  datasetId: string
  collectionId: string
  data: TrainingDataType[]
}) => {
  const { teamId, tmbId, datasetId, collectionId, data } = props

  await mongoSessionRun(async (session) => {
    // 1. 批量插入数据
    const insertedDocs = await MongoDatasetData.insertMany(
      data.map((item, index) => ({
        teamId,
        tmbId,
        datasetId,
        collectionId,
        q: item.q,
        a: item.a || '',
        chunkIndex: index
      })),
      { session }
    )

    // 2. 生成向量并保存（外部向量数据库）
    const vectorPromises = insertedDocs.map(async (doc) => {
      if (doc.q) {
        const vector = await generateEmbedding(doc.q)
        await insertDatasetDataVector({
          id: doc._id.toString(),
          teamId,
          datasetId,
          collectionId,
          vector
        })
      }
    })

    await Promise.all(vectorPromises)

    // 3. 更新集合统计
    await MongoDatasetCollection.findByIdAndUpdate(
      collectionId,
      {
        $inc: { 'trainingCount': data.length },
        $set: { 'updateTime': new Date() }
      },
      { session }
    )

    // 4. 记录训练使用量
    await MongoUsage.create([{
      teamId,
      tmbId,
      source: UsageSourceEnum.training,
      time: new Date(),
      totalPoints: data.length * 2, // 每条数据2积分
      list: [{
        type: 'datasetTraining',
        name: '数据集训练',
        amount: data.length,
        price: 2
      }]
    }], { session })
  })
}
```

## 🔄 服务间通信模式

### 1. MCP协议集成

**文件路径**: `packages/service/core/app/mcp.ts`

```typescript
// MCP客户端管理器
export class MCPClientManager {
  private clients = new Map<string, MCPClient>()
  private connections = new Map<string, Promise<Client>>()

  // 创建MCP连接
  async createConnection(config: {
    serverUrl: string
    capabilities: ClientCapabilities
    teamId: string
  }): Promise<Client> {
    const { serverUrl, capabilities, teamId } = config
    const connectionKey = `${teamId}:${serverUrl}`

    // 复用已有连接
    if (this.connections.has(connectionKey)) {
      return this.connections.get(connectionKey)!
    }

    // 创建新连接
    const connectionPromise = this.establishConnection(serverUrl, capabilities)
    this.connections.set(connectionKey, connectionPromise)

    try {
      const client = await connectionPromise
      console.log(`MCP连接建立成功: ${serverUrl}`)
      return client
    } catch (error) {
      // 连接失败时清理
      this.connections.delete(connectionKey)
      throw error
    }
  }

  // 建立连接
  private async establishConnection(
    serverUrl: string,
    capabilities: ClientCapabilities
  ): Promise<Client> {
    const transport = new StreamableHTTPClientTransport(new URL(serverUrl))
    
    const client = new Client({
      name: 'FastGPT',
      version: '1.0.0'
    }, {
      capabilities
    })

    // 连接处理
    await client.connect(transport)

    // 设置错误处理
    client.on('error', (error) => {
      console.error('MCP连接错误:', error)
    })

    // 设置断开处理
    client.on('disconnect', () => {
      console.log('MCP连接断开')
      // 清理连接缓存
      for (const [key, conn] of this.connections.entries()) {
        if (conn === Promise.resolve(client)) {
          this.connections.delete(key)
          break
        }
      }
    })

    return client
  }

  // 调用MCP工具
  async callTool(props: {
    teamId: string
    serverUrl: string
    toolName: string
    arguments: Record<string, any>
  }): Promise<any> {
    const { teamId, serverUrl, toolName, arguments: args } = props

    const client = await this.createConnection({
      serverUrl,
      capabilities: { tools: {} },
      teamId
    })

    // 调用工具
    const result = await client.callTool({
      name: toolName,
      arguments: args
    })

    return result
  }

  // 清理连接
  async cleanup(teamId?: string) {
    if (teamId) {
      // 清理特定团队的连接
      const keysToDelete = Array.from(this.connections.keys())
        .filter(key => key.startsWith(`${teamId}:`))

      for (const key of keysToDelete) {
        const connection = this.connections.get(key)
        if (connection) {
          try {
            const client = await connection
            await client.disconnect()
          } catch (error) {
            console.warn('清理MCP连接失败:', error)
          }
          this.connections.delete(key)
        }
      }
    } else {
      // 清理所有连接
      for (const [key, connection] of this.connections.entries()) {
        try {
          const client = await connection
          await client.disconnect()
        } catch (error) {
          console.warn('清理MCP连接失败:', error)
        }
      }
      this.connections.clear()
    }
  }
}

// 全局MCP客户端管理器
export const mcpClientManager = new MCPClientManager()
```

### 2. 插件系统通信

```typescript
// 插件调用接口
export interface PluginRuntime {
  id: string
  name: string
  version: string
  apiSchema: any
  customHeaders?: Record<string, string>
}

// 插件调用管理器
export class PluginManager {
  private plugins = new Map<string, PluginRuntime>()

  // 注册插件
  registerPlugin(plugin: PluginRuntime): void {
    this.plugins.set(plugin.id, plugin)
  }

  // 调用插件
  async callPlugin(props: {
    pluginId: string
    action: string
    params: Record<string, any>
    teamId: string
    tmbId: string
  }): Promise<any> {
    const { pluginId, action, params, teamId, tmbId } = props

    const plugin = this.plugins.get(pluginId)
    if (!plugin) {
      throw new Error(`插件不存在: ${pluginId}`)
    }

    // 构建请求
    const requestUrl = `${plugin.apiSchema.baseUrl}/${action}`
    const headers = {
      'Content-Type': 'application/json',
      'X-Team-Id': teamId,
      'X-Member-Id': tmbId,
      ...plugin.customHeaders
    }

    try {
      // 发送HTTP请求
      const response = await fetch(requestUrl, {
        method: 'POST',
        headers,
        body: JSON.stringify(params),
        timeout: 30000
      })

      if (!response.ok) {
        throw new Error(`插件调用失败: ${response.status} ${response.statusText}`)
      }

      const result = await response.json()
      return result

    } catch (error) {
      console.error(`插件调用错误 ${pluginId}:`, error)
      throw new Error(`插件调用失败: ${error.message}`)
    }
  }

  // 健康检查
  async healthCheck(pluginId: string): Promise<boolean> {
    const plugin = this.plugins.get(pluginId)
    if (!plugin) return false

    try {
      const response = await fetch(`${plugin.apiSchema.baseUrl}/health`, {
        method: 'GET',
        timeout: 5000
      })
      return response.ok
    } catch {
      return false
    }
  }
}

// 全局插件管理器
export const pluginManager = new PluginManager()
```

## ⚡ 性能优化策略

### 1. 批处理优化

```typescript
// 数组分块处理
export const chunkArray = <T>(array: T[], chunkSize: number): T[][] => {
  const chunks: T[][] = []
  for (let i = 0; i < array.length; i += chunkSize) {
    chunks.push(array.slice(i, i + chunkSize))
  }
  return chunks
}

// 批量向量处理
export const batchVectorProcessing = async (
  data: { text: string; metadata: any }[],
  batchSize: number = 100
) => {
  const batches = chunkArray(data, batchSize)
  const results = []

  for (const batch of batches) {
    // 并行处理当前批次
    const batchPromises = batch.map(async (item) => {
      const vector = await generateEmbedding(item.text)
      return {
        ...item.metadata,
        vector,
        text: item.text
      }
    })

    const batchResults = await Promise.all(batchPromises)
    results.push(...batchResults)

    // 批次间延迟，避免API限流
    await new Promise(resolve => setTimeout(resolve, 100))
  }

  return results
}
```

### 2. 连接池管理

```typescript
// 数据库连接池配置
const mongoConnectionOptions = {
  // 连接池设置
  maxPoolSize: 30,        // 最大连接数
  minPoolSize: 5,         // 最小连接数
  maxConnecting: 10,      // 最大并发连接数
  
  // 超时设置
  connectTimeoutMS: 60000,      // 连接超时
  socketTimeoutMS: 60000,       // Socket超时
  maxIdleTimeMS: 300000,        // 最大空闲时间
  waitQueueTimeoutMS: 60000,    // 等待队列超时
  
  // 重试设置
  retryWrites: true,
  retryReads: true,
  
  // 其他优化
  bufferCommands: true,
  bufferMaxEntries: 0
}

// Redis连接池
const redisConnectionOptions = {
  maxRetriesPerRequest: 3,
  retryDelayOnFailover: 100,
  lazyConnect: true,
  keepAlive: 30000,
  maxRetriesPerRequest: null  // 禁用自动重试
}
```

### 3. 内存管理

```typescript
// 流式处理防止内存溢出
export const streamLargeDataProcessing = async function* (
  dataSource: AsyncIterable<any>
) {
  const buffer = []
  const maxBufferSize = 1000

  for await (const item of dataSource) {
    buffer.push(item)

    // 缓冲区满时，yield当前批次
    if (buffer.length >= maxBufferSize) {
      yield buffer.splice(0, maxBufferSize)
    }
  }

  // 处理剩余数据
  if (buffer.length > 0) {
    yield buffer
  }
}

// 大文件流式上传处理
export const handleLargeFileUpload = async (
  fileStream: Readable,
  processor: (chunk: Buffer) => Promise<void>
) => {
  const chunkSize = 1024 * 1024 // 1MB chunks

  return new Promise((resolve, reject) => {
    fileStream.on('data', async (chunk: Buffer) => {
      try {
        // 暂停流，处理当前块
        fileStream.pause()
        await processor(chunk)
        fileStream.resume()
      } catch (error) {
        reject(error)
      }
    })

    fileStream.on('end', resolve)
    fileStream.on('error', reject)
  })
}
```

---

FastGPT 的数据流架构体现了现代微服务架构的最佳实践，通过分层设计、事件驱动、缓存优化和事务管理，构建了一个高性能、高可用的 AI 应用平台。系统有效处理了复杂的 AI 工作流，同时保证了数据的一致性和实时响应能力。