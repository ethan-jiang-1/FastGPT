# FastGPT 第三方集成生态深度分析

## 🌍 第三方集成生态概述

FastGPT 构建了**全面的第三方集成生态系统**，支持20+个AI提供商、多种数据库、企业通信平台、认证服务等。通过模块化设计和标准化接口，实现了**灵活配置**、**安全集成**和**动态扩展**。

### 集成架构特点

- **AI提供商无关** - 支持OpenAI、Claude、Gemini等20+主流AI服务
- **多数据库支持** - MongoDB、PostgreSQL、Redis、向量数据库全覆盖  
- **企业级集成** - 飞书、钉钉、微信等企业通信平台
- **安全认证** - OAuth、SSO、API密钥管理
- **监控可观测** - OpenTelemetry、SignoZ集成

## 🤖 AI服务提供商集成

### 1. 主流国际AI提供商

**OpenAI全系产品集成**:
**文件路径**: `packages/service/core/ai/config/provider/OpenAI.json`

```json
{
  "provider": "OpenAI",
  "baseUrl": "https://api.openai.com/v1",
  "models": [
    {
      "model": "gpt-4o",
      "name": "GPT-4o",
      "avatar": "core/ai/model/openai",
      "maxContext": 128000,
      "maxResponse": 4096,
      "vision": true,
      "toolChoice": true,
      "functionCall": false,
      "customCQPrompt": "",
      "defaultConfig": {
        "temperature": 0.3,
        "max_tokens": 4000,
        "frequency_penalty": 0,
        "presence_penalty": 0
      }
    },
    {
      "model": "o1-preview",
      "name": "GPT-o1-preview", 
      "maxContext": 128000,
      "maxResponse": 32768,
      "vision": false,
      "toolChoice": false,
      "charsPointsPrice": 20
    }
  ]
}
```

**Anthropic Claude集成**:
**文件路径**: `packages/service/core/ai/config/provider/Claude.json`

```json
{
  "provider": "Anthropic",
  "baseUrl": "https://api.anthropic.com",
  "models": [
    {
      "model": "claude-3-5-sonnet-20241022",
      "name": "Claude-3.5-Sonnet",
      "avatar": "core/ai/model/claude",
      "maxContext": 200000,
      "maxResponse": 8192,
      "vision": true,
      "toolChoice": true,
      "charsPointsPrice": 8
    },
    {
      "model": "claude-3-5-haiku-20241022", 
      "name": "Claude-3.5-Haiku",
      "maxContext": 200000,
      "maxResponse": 8192,
      "charsPointsPrice": 1
    }
  ]
}
```

**Google Gemini集成**:
```json
{
  "provider": "Google",
  "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
  "models": [
    {
      "model": "gemini-2.0-flash-exp",
      "name": "Gemini-2.0-Flash-Exp",
      "maxContext": 1048576,
      "maxResponse": 8192,
      "vision": true,
      "toolChoice": true
    },
    {
      "model": "gemini-1.5-pro",
      "name": "Gemini-1.5-Pro",
      "maxContext": 2097152,
      "maxResponse": 8192,
      "vision": true
    }
  ]
}
```

### 2. 国产AI服务提供商

**智谱ChatGLM集成**:
```json
{
  "provider": "ChatGLM",
  "baseUrl": "https://open.bigmodel.cn/api/paas/v4",
  "models": [
    {
      "model": "glm-4-plus",
      "name": "GLM-4-Plus",
      "maxContext": 128000,
      "maxResponse": 4095,
      "vision": true,
      "toolChoice": true,
      "charsPointsPrice": 10
    },
    {
      "model": "glm-4-air",
      "name": "GLM-4-Air",
      "maxContext": 128000,
      "maxResponse": 4095,
      "charsPointsPrice": 1
    }
  ]
}
```

**阿里通义千问集成**:
```json
{
  "provider": "Alibaba",
  "baseUrl": "https://dashscope.aliyuncs.com/api/v1",
  "models": [
    {
      "model": "qwen-max",
      "name": "通义千问-Max",
      "maxContext": 30000,
      "maxResponse": 2000,
      "toolChoice": true,
      "charsPointsPrice": 8
    },
    {
      "model": "qwen2.5-72b-instruct",
      "name": "通义千问2.5-72B",
      "maxContext": 131072,
      "maxResponse": 8192,
      "charsPointsPrice": 4
    }
  ]
}
```

### 3. AI服务统一管理

**AI服务工厂模式** (`packages/service/core/ai/config.ts`):

```typescript
// AI API统一管理
export const getAIApi = (props?: {
  userKey?: OpenaiAccountType
  timeout?: number
}): OpenAI => {
  const { userKey, timeout } = props || {}
  
  // 优先级: 用户密钥 > 全局OneAPI > 系统配置
  const baseUrl = userKey?.baseUrl || 
                  global?.systemEnv?.oneapiUrl || 
                  openaiBaseUrl
                  
  const apiKey = userKey?.key || 
                 global?.systemEnv?.chatApiKey || 
                 openaiBaseKey

  return new OpenAI({
    baseURL: baseUrl,
    apiKey,
    httpAgent: global.httpsAgent,
    timeout: timeout || 60000,
    maxRetries: 2,
    defaultHeaders: {
      'User-Agent': 'FastGPT/1.0',
      'X-Request-Source': 'fastgpt'
    }
  })
}

// AI服务健康检查
export const checkAIServiceHealth = async (
  provider: string,
  model: string
): Promise<boolean> => {
  try {
    const api = getAIApi()
    const response = await api.chat.completions.create({
      model,
      messages: [{ role: 'user', content: 'test' }],
      max_tokens: 1
    })
    
    return !!response.choices[0]?.message
  } catch (error) {
    console.warn(`AI服务健康检查失败 ${provider}:${model}:`, error)
    return false
  }
}

// AI服务成本计算
export const calculateAICost = (
  model: string,
  inputTokens: number,
  outputTokens: number
): number => {
  const modelConfig = getModelConfig(model)
  if (!modelConfig) return 0

  const inputCost = (inputTokens / 1000) * modelConfig.inputPrice
  const outputCost = (outputTokens / 1000) * modelConfig.outputPrice
  
  return Math.ceil((inputCost + outputCost) * modelConfig.pointsRatio)
}
```

### 4. AI代理集成

**AI代理服务** (`packages/service/core/ai/config.ts`):

```typescript
// AI代理配置
interface AIProxyConfig {
  endpoint: string      // 代理服务端点
  token: string        // 代理服务令牌
  timeout: number      // 请求超时
  retries: number      // 重试次数
}

// AI代理请求处理
export const requestAIProxy = async (params: {
  model: string
  messages: ChatMessage[]
  stream?: boolean
  temperature?: number
  max_tokens?: number
}): Promise<any> => {
  const proxyConfig = getAIProxyConfig()
  
  const requestData = {
    ...params,
    source: 'fastgpt',
    timestamp: Date.now()
  }

  try {
    const response = await fetch(`${proxyConfig.endpoint}/v1/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${proxyConfig.token}`,
        'X-Request-ID': generateRequestId()
      },
      body: JSON.stringify(requestData),
      timeout: proxyConfig.timeout
    })

    if (!response.ok) {
      throw new Error(`AI代理请求失败: ${response.status} ${response.statusText}`)
    }

    return await response.json()

  } catch (error) {
    console.error('AI代理请求失败:', error)
    throw error
  }
}
```

## 💾 数据库集成架构

### 1. MongoDB集成

**连接管理** (`packages/service/common/mongo/index.ts`):

```typescript
// MongoDB连接配置
export const MONGO_URL = process.env.MONGODB_URI as string
export const MONGO_LOG_URL = (process.env.MONGODB_LOG_URI ?? process.env.MONGODB_URI) as string

// 全局连接管理
export const connectionMongo = (() => {
  if (!global.mongodb) {
    global.mongodb = new Mongoose()
  }
  return global.mongodb
})()

// 连接初始化
export const connectToMongoDB = async (): Promise<void> => {
  try {
    const options = {
      bufferCommands: true,
      maxConnecting: 30,        // 最大并发连接数
      maxPoolSize: 30,          // 连接池最大大小
      minPoolSize: 5,           // 连接池最小大小
      connectTimeoutMS: 60000,  // 连接超时
      waitQueueTimeoutMS: 60000,// 等待队列超时
      socketTimeoutMS: 60000,   // Socket超时
      maxIdleTimeMS: 300000,    // 最大空闲时间
      retryWrites: true,        // 启用重试写入
      retryReads: true          // 启用重试读取
    }

    await connectionMongo.connect(MONGO_URL, options)
    console.log('✅ MongoDB连接成功')

    // 连接监控
    connectionMongo.connection.on('error', (error) => {
      console.error('❌ MongoDB连接错误:', error)
    })

    connectionMongo.connection.on('disconnected', () => {
      console.warn('⚠️  MongoDB连接断开')
    })

  } catch (error) {
    console.error('❌ MongoDB连接失败:', error)
    process.exit(1)
  }
}

// 性能监控中间件
const addCommonMiddleware = (schema: mongoose.Schema) => {
  const operations = ['find', 'findOne', 'findOneAndUpdate', 'aggregate']
  
  operations.forEach((op: any) => {
    schema.pre(op, function(this: any) {
      this._startTime = Date.now()
    })

    schema.post(op, function(this: any, result: any, next) {
      if (this._startTime) {
        const duration = Date.now() - this._startTime
        if (duration > 1000) {
          addLog.warn(`MongoDB慢查询 ${duration}ms`, {
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

### 2. Redis集成

**连接与缓存管理** (`packages/service/common/redis/index.ts`):

```typescript
// Redis连接配置
const REDIS_URL = process.env.REDIS_URL ?? 'redis://localhost:6379'
export const FASTGPT_REDIS_PREFIX = 'fastgpt:'

// 全局Redis连接
export const getGlobalRedisConnection = (): Redis => {
  if (global.redisClient) return global.redisClient
  
  global.redisClient = new Redis(REDIS_URL, {
    keyPrefix: FASTGPT_REDIS_PREFIX,
    maxRetriesPerRequest: 3,      // 最大重试次数
    retryDelayOnFailover: 100,    // 故障转移延迟
    lazyConnect: true,            // 延迟连接
    keepAlive: 30000,             // 保持连接
    connectTimeout: 10000,        // 连接超时
    commandTimeout: 5000          // 命令超时
  })

  // 连接监控
  global.redisClient.on('connect', () => {
    console.log('✅ Redis连接成功')
  })

  global.redisClient.on('error', (error) => {
    console.error('❌ Redis连接错误:', error)
  })

  return global.redisClient
}

// 缓存操作接口
export class RedisCache {
  private redis = getGlobalRedisConnection()

  // 获取缓存
  async get<T = any>(key: string): Promise<T | null> {
    try {
      const cached = await this.redis.get(key)
      return cached ? JSON.parse(cached) : null
    } catch (error) {
      console.warn(`Redis获取缓存失败: ${key}`, error)
      return null
    }
  }

  // 设置缓存
  async set(key: string, value: any, ttl: number = 300): Promise<void> {
    try {
      await this.redis.setex(key, ttl, JSON.stringify(value))
    } catch (error) {
      console.warn(`Redis设置缓存失败: ${key}`, error)
    }
  }

  // 批量删除
  async delByPattern(pattern: string): Promise<void> {
    try {
      const keys = await this.redis.keys(pattern)
      if (keys.length > 0) {
        await this.redis.del(...keys)
      }
    } catch (error) {
      console.warn(`Redis批量删除失败: ${pattern}`, error)
    }
  }

  // 增量计数
  async incr(key: string, increment: number = 1): Promise<number> {
    try {
      return await this.redis.incrby(key, increment)
    } catch (error) {
      console.warn(`Redis计数失败: ${key}`, error)
      return 0
    }
  }
}
```

### 3. 向量数据库集成

**多向量数据库支持** (`packages/service/common/vectorDB/controller.ts`):

```typescript
// 向量数据库工厂
export const getVectorObj = (): VectorController => {
  // 按优先级选择向量数据库
  if (PG_ADDRESS) {
    return new PgVectorCtrl()
  }
  if (OCEANBASE_ADDRESS) {
    return new ObVectorCtrl() 
  }
  if (MILVUS_ADDRESS) {
    return new MilvusCtrl()
  }
  
  // 默认使用PostgreSQL
  return new PgVectorCtrl()
}

// 向量数据库统一接口
export abstract class VectorController {
  // 插入向量
  abstract insertDatasetDataVector(props: {
    id: string
    teamId: string
    datasetId: string
    collectionId: string
    vector: number[]
    retry?: number
  }): Promise<{ insertId: string }>

  // 删除向量
  abstract deleteDatasetDataVector(props: {
    teamId: string
    id: string
  }): Promise<void>

  // 向量检索
  abstract recallFromVectorStore(props: {
    teamId: string
    datasetIds: string[]
    vector: number[]
    limit: number
    efSearch?: number
  }): Promise<{
    id: string
    score: number
  }[]>

  // 获取向量数量
  abstract getVectorCountByTeamId(teamId: string): Promise<number>
}

// PostgreSQL向量实现
export class PgVectorCtrl extends VectorController {
  async insertDatasetDataVector(props: {
    id: string
    teamId: string
    datasetId: string
    collectionId: string
    vector: number[]
    retry?: number
  }): Promise<{ insertId: string }> {
    const { id, teamId, datasetId, collectionId, vector, retry = 3 } = props

    try {
      const sql = `
        INSERT INTO modeldata 
        (id, vector, team_id, dataset_id, collection_id, create_time) 
        VALUES ($1, $2, $3, $4, $5, $6)
      `
      
      await pgClient.query(sql, [
        id,
        `[${vector.join(',')}]`,
        teamId,
        datasetId,
        collectionId,
        Date.now()
      ])

      return { insertId: id }

    } catch (error) {
      if (retry > 0) {
        console.warn(`向量插入重试: ${retry}`, error)
        await new Promise(resolve => setTimeout(resolve, 1000))
        return this.insertDatasetDataVector({ ...props, retry: retry - 1 })
      }
      throw error
    }
  }

  async recallFromVectorStore(props: {
    teamId: string
    datasetIds: string[]
    vector: number[]
    limit: number
    efSearch?: number
  }): Promise<{ id: string; score: number }[]> {
    const { teamId, datasetIds, vector, limit, efSearch = 100 } = props

    try {
      // 设置ef_search参数优化检索性能
      await pgClient.query(`SET LOCAL hnsw.ef_search = ${efSearch}`)

      const sql = `
        SELECT id, 1 - (vector <=> $1) as score
        FROM modeldata 
        WHERE team_id = $2 
          AND dataset_id = ANY($3)
        ORDER BY vector <=> $1
        LIMIT $4
      `

      const result = await pgClient.query(sql, [
        `[${vector.join(',')}]`,
        teamId,
        datasetIds,
        limit
      ])

      return result.rows.map(row => ({
        id: row.id,
        score: parseFloat(row.score)
      }))

    } catch (error) {
      console.error('向量检索失败:', error)
      throw error
    }
  }
}
```

## 🗄️ 存储系统集成

### 1. MinIO对象存储

**MinIO集成配置** (Docker Compose):

```yaml
# MinIO对象存储服务
minio:
  image: minio/minio:RELEASE.2023-12-20T01-00-02Z
  container_name: minio
  restart: always
  networks:
    - fastgpt
  ports:
    - "9000:9000"    # API端口
    - "9001:9001"    # 控制台端口
  environment:
    - MINIO_ROOT_USER=minioadmin
    - MINIO_ROOT_PASSWORD=minioadmin
  volumes:
    - ./fastgpt-minio:/data
  command: server /data --console-address ":9001"
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 30s
    timeout: 20s
    retries: 3
```

### 2. GridFS文件存储

**GridFS集成** (`packages/service/common/file/gridfs/controller.ts`):

```typescript
// GridFS存储控制器
export class GridFSController {
  private bucket: GridFSBucket

  constructor(connection: Connection, bucketName: string = 'files') {
    this.bucket = new GridFSBucket(connection.db, { 
      bucketName,
      chunkSizeBytes: 255 * 1024 // 255KB块大小
    })
  }

  // 上传文件
  async uploadFile(props: {
    filename: string
    buffer: Buffer
    metadata?: Record<string, any>
    contentType?: string
  }): Promise<{ fileId: string }> {
    const { filename, buffer, metadata = {}, contentType } = props

    return new Promise((resolve, reject) => {
      const uploadStream = this.bucket.openUploadStream(filename, {
        metadata: {
          ...metadata,
          contentType,
          uploadTime: new Date(),
          size: buffer.length
        }
      })

      uploadStream.on('finish', () => {
        resolve({ fileId: uploadStream.id.toString() })
      })

      uploadStream.on('error', reject)

      // 写入数据
      uploadStream.end(buffer)
    })
  }

  // 下载文件
  async downloadFile(fileId: string): Promise<{
    stream: GridFSBucketReadStream
    metadata: any
  }> {
    try {
      // 获取文件信息
      const files = await this.bucket.find({ 
        _id: new ObjectId(fileId) 
      }).toArray()

      if (files.length === 0) {
        throw new Error('文件不存在')
      }

      const file = files[0]
      const downloadStream = this.bucket.openDownloadStream(new ObjectId(fileId))

      return {
        stream: downloadStream,
        metadata: {
          filename: file.filename,
          contentType: file.metadata?.contentType,
          size: file.length,
          uploadDate: file.uploadDate
        }
      }

    } catch (error) {
      console.error('GridFS下载文件失败:', error)
      throw error
    }
  }

  // 删除文件
  async deleteFile(fileId: string): Promise<void> {
    try {
      await this.bucket.delete(new ObjectId(fileId))
    } catch (error) {
      console.error('GridFS删除文件失败:', error)
      throw error
    }
  }

  // 批量清理过期文件
  async cleanupExpiredFiles(days: number = 30): Promise<number> {
    const cutoffDate = new Date()
    cutoffDate.setDate(cutoffDate.getDate() - days)

    try {
      const expiredFiles = await this.bucket.find({
        uploadDate: { $lt: cutoffDate }
      }).toArray()

      let deletedCount = 0
      for (const file of expiredFiles) {
        await this.bucket.delete(file._id)
        deletedCount++
      }

      console.log(`清理过期文件: ${deletedCount}个`)
      return deletedCount

    } catch (error) {
      console.error('清理过期文件失败:', error)
      return 0
    }
  }
}
```

## 📱 企业通信平台集成

### 1. 飞书(Lark)集成

**飞书数据集集成** (`packages/service/core/dataset/apiDataset/feishuDataset/api.ts`):

```typescript
// 飞书API客户端
export class FeishuApiClient {
  private appId: string
  private appSecret: string
  private accessToken?: string
  private tokenExpiry?: Date

  constructor(appId: string, appSecret: string) {
    this.appId = appId
    this.appSecret = appSecret
  }

  // 获取访问令牌
  async getAccessToken(): Promise<string> {
    if (this.accessToken && this.tokenExpiry && this.tokenExpiry > new Date()) {
      return this.accessToken
    }

    try {
      const response = await fetch('https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          app_id: this.appId,
          app_secret: this.appSecret
        })
      })

      const data = await response.json()
      
      if (data.code === 0) {
        this.accessToken = data.tenant_access_token
        this.tokenExpiry = new Date(Date.now() + (data.expire - 60) * 1000) // 提前60秒过期
        return this.accessToken
      } else {
        throw new Error(`获取飞书访问令牌失败: ${data.msg}`)
      }

    } catch (error) {
      console.error('飞书认证失败:', error)
      throw error
    }
  }

  // 获取文件夹列表
  async getFolderList(parentFolderId?: string): Promise<FeishuFolder[]> {
    const token = await this.getAccessToken()
    
    const url = new URL('https://open.feishu.cn/open-apis/drive/v1/files')
    url.searchParams.set('folder_token', parentFolderId || '')
    url.searchParams.set('page_size', '200')

    try {
      const response = await fetch(url.toString(), {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      })

      const data = await response.json()
      
      if (data.code === 0) {
        return data.data.files.map((file: any) => ({
          token: file.token,
          name: file.name,
          type: file.type,
          parentToken: file.parent_token,
          createTime: new Date(file.created_time * 1000),
          modifyTime: new Date(file.modified_time * 1000)
        }))
      } else {
        throw new Error(`获取飞书文件夹列表失败: ${data.msg}`)
      }

    } catch (error) {
      console.error('获取飞书文件夹列表失败:', error)
      throw error
    }
  }

  // 读取文档内容
  async readDocumentContent(docToken: string, docType: FeishuDocType): Promise<string> {
    const token = await this.getAccessToken()
    
    let apiUrl: string
    switch (docType) {
      case 'doc':
        apiUrl = `https://open.feishu.cn/open-apis/docx/v1/documents/${docToken}/content`
        break
      case 'sheet':
        apiUrl = `https://open.feishu.cn/open-apis/sheets/v1/spreadsheets/${docToken}/values_batch_get`
        break
      case 'bitable':
        apiUrl = `https://open.feishu.cn/open-apis/bitable/v1/apps/${docToken}/tables`
        break
      default:
        throw new Error(`不支持的文档类型: ${docType}`)
    }

    try {
      const response = await fetch(apiUrl, {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      })

      const data = await response.json()
      
      if (data.code === 0) {
        return this.parseDocumentContent(data.data, docType)
      } else {
        throw new Error(`读取飞书文档内容失败: ${data.msg}`)
      }

    } catch (error) {
      console.error('读取飞书文档内容失败:', error)
      throw error
    }
  }

  // 解析文档内容
  private parseDocumentContent(data: any, docType: FeishuDocType): string {
    switch (docType) {
      case 'doc':
        return this.parseDocxContent(data)
      case 'sheet':
        return this.parseSheetContent(data)
      case 'bitable':
        return this.parseBitableContent(data)
      default:
        return JSON.stringify(data)
    }
  }

  private parseDocxContent(data: any): string {
    // 解析飞书文档格式
    const blocks = data.document?.body?.blocks || []
    
    return blocks.map((block: any) => {
      switch (block.block_type) {
        case 'paragraph':
          return block.paragraph?.elements?.map((element: any) => {
            return element.text_run?.content || ''
          }).join('') || ''
        case 'heading1':
        case 'heading2':
        case 'heading3':
          const level = parseInt(block.block_type.slice(-1))
          const headingText = block[block.block_type]?.elements?.map((element: any) => {
            return element.text_run?.content || ''
          }).join('') || ''
          return '#'.repeat(level) + ' ' + headingText
        default:
          return ''
      }
    }).filter(Boolean).join('\n\n')
  }
}

// 飞书Webhook处理
export default async function feishuWebhookHandler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { token } = req.query as { token: string }

  try {
    // 验证飞书签名
    const signature = req.headers['x-lark-signature'] as string
    const timestamp = req.headers['x-lark-request-timestamp'] as string
    const nonce = req.headers['x-lark-request-nonce'] as string

    if (!verifyFeishuSignature(signature, timestamp, nonce, JSON.stringify(req.body))) {
      return res.status(401).json({ error: 'Invalid signature' })
    }

    const { type, event } = req.body

    switch (type) {
      case 'event_callback':
        await handleFeishuEvent(token, event)
        break
      case 'url_verification':
        return res.json({ challenge: event.challenge })
      default:
        console.log('未知的飞书事件类型:', type)
    }

    res.json({ success: true })

  } catch (error) {
    console.error('飞书Webhook处理失败:', error)
    res.status(500).json({ error: error.message })
  }
}

// 飞书签名验证
function verifyFeishuSignature(
  signature: string,
  timestamp: string,
  nonce: string,
  body: string
): boolean {
  const secret = process.env.FEISHU_WEBHOOK_SECRET!
  const stringToSign = timestamp + nonce + secret + body
  
  const expectedSignature = crypto
    .createHash('sha256')
    .update(stringToSign)
    .digest('hex')
  
  return signature === expectedSignature
}
```

### 2. 钉钉集成

**钉钉Webhook处理** (`projects/app/src/pages/api/support/outLink/dingtalk/[token].ts`):

```typescript
export default async function dingTalkWebhookHandler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { token } = req.query as { token: string }

  try {
    // 钉钉签名验证
    const signature = req.headers['x-dingtalk-signature'] as string
    const timestamp = req.headers['timestamp'] as string

    if (!verifyDingTalkSignature(signature, timestamp, JSON.stringify(req.body))) {
      return res.status(401).json({ error: 'Invalid signature' })
    }

    // 处理钉钉消息
    const { msgtype, text, at } = req.body

    switch (msgtype) {
      case 'text':
        await handleDingTalkTextMessage(token, text.content, at)
        break
      case 'markdown':
        await handleDingTalkMarkdownMessage(token, req.body.markdown)
        break
      default:
        console.log('未支持的钉钉消息类型:', msgtype)
    }

    res.json({ success: true })

  } catch (error) {
    console.error('钉钉Webhook处理失败:', error)
    res.status(500).json({ error: error.message })
  }
}

// 钉钉签名验证
function verifyDingTalkSignature(
  signature: string,
  timestamp: string,
  body: string
): boolean {
  const secret = process.env.DINGTALK_WEBHOOK_SECRET!
  const stringToSign = timestamp + '\n' + secret
  
  const hmac = crypto.createHmac('sha256', secret)
  hmac.update(stringToSign)
  const expectedSignature = hmac.digest('base64')
  
  return signature === expectedSignature
}
```

### 3. 微信集成

**微信支付集成** (`packages/global/support/wallet/bill/constants.ts`):

```typescript
// 支付方式枚举
export enum BillPayWayEnum {
  balance = 'balance',    // 余额支付
  wx = 'wx',             // 微信支付
  alipay = 'alipay',     // 支付宝
  bank = 'bank',         // 银行转账
  coupon = 'coupon'      // 优惠券
}

// 微信支付配置
interface WeChatPayConfig {
  appId: string          // 应用ID
  mchId: string         // 商户号
  apiKey: string        // API密钥
  notifyUrl: string     // 回调地址
  certPath?: string     // 证书路径
}

// 微信支付处理
export class WeChatPayHandler {
  private config: WeChatPayConfig

  constructor(config: WeChatPayConfig) {
    this.config = config
  }

  // 创建支付订单
  async createPayment(props: {
    orderId: string
    amount: number
    description: string
    userId: string
  }): Promise<{ qrCode: string; orderId: string }> {
    const { orderId, amount, description, userId } = props

    try {
      const paymentData = {
        appid: this.config.appId,
        mch_id: this.config.mchId,
        nonce_str: this.generateNonce(),
        body: description,
        out_trade_no: orderId,
        total_fee: amount * 100, // 转换为分
        spbill_create_ip: '127.0.0.1',
        notify_url: this.config.notifyUrl,
        trade_type: 'NATIVE',
        openid: userId
      }

      // 生成签名
      const sign = this.generateSign(paymentData)
      paymentData.sign = sign

      // 发送支付请求
      const xmlData = this.buildXML(paymentData)
      const response = await fetch('https://api.mch.weixin.qq.com/pay/unifiedorder', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/xml'
        },
        body: xmlData
      })

      const result = await this.parseXML(await response.text())
      
      if (result.return_code === 'SUCCESS' && result.result_code === 'SUCCESS') {
        return {
          qrCode: result.code_url,
          orderId: result.prepay_id
        }
      } else {
        throw new Error(`微信支付创建失败: ${result.err_code_des || result.return_msg}`)
      }

    } catch (error) {
      console.error('微信支付创建失败:', error)
      throw error
    }
  }

  // 处理支付回调
  async handlePaymentNotify(xmlData: string): Promise<{
    success: boolean
    orderId?: string
    amount?: number
  }> {
    try {
      const data = await this.parseXML(xmlData)
      
      // 验证签名
      if (!this.verifySign(data)) {
        throw new Error('微信支付回调签名验证失败')
      }

      if (data.return_code === 'SUCCESS' && data.result_code === 'SUCCESS') {
        return {
          success: true,
          orderId: data.out_trade_no,
          amount: parseInt(data.total_fee) / 100
        }
      } else {
        return { 
          success: false 
        }
      }

    } catch (error) {
      console.error('微信支付回调处理失败:', error)
      return { success: false }
    }
  }

  private generateSign(data: any): string {
    const keys = Object.keys(data).sort()
    const stringA = keys.map(key => `${key}=${data[key]}`).join('&')
    const stringSignTemp = `${stringA}&key=${this.config.apiKey}`
    
    return crypto
      .createHash('md5')
      .update(stringSignTemp)
      .digest('hex')
      .toUpperCase()
  }
}
```

## 🔐 认证与SSO集成

### 1. OAuth提供商集成

**OAuth配置** (`packages/global/common/system/types/index.d.ts`):

```typescript
// OAuth提供商配置
interface OAuthConfig {
  github?: string              // GitHub OAuth应用ID
  google?: string              // Google OAuth客户端ID
  wechat?: string              // 微信OAuth应用ID
  microsoft?: {
    clientId?: string          // Microsoft应用ID
    tenantId?: string          // Azure AD租户ID
    customButton?: string      // 自定义按钮文本
  }
}

// SSO配置
interface SSOConfig {
  icon?: string               // SSO图标
  title?: string              // SSO标题
  url?: string                // SSO登录地址
  autoLogin?: boolean         // 自动登录
}

// 系统环境配置
interface SystemEnvType {
  oauth?: OAuthConfig
  sso?: SSOConfig
  // ... 其他配置
}
```

**OAuth认证处理** (`packages/service/support/user/auth/controller.ts`):

```typescript
// GitHub OAuth认证
export const githubOAuthLogin = async (code: string): Promise<{
  user: UserType
  token: string
}> => {
  try {
    // 1. 获取GitHub访问令牌
    const tokenResponse = await fetch('https://github.com/login/oauth/access_token', {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        client_id: process.env.GITHUB_CLIENT_ID,
        client_secret: process.env.GITHUB_CLIENT_SECRET,
        code
      })
    })

    const tokenData = await tokenResponse.json()
    if (tokenData.error) {
      throw new Error(`GitHub OAuth错误: ${tokenData.error_description}`)
    }

    // 2. 获取用户信息
    const userResponse = await fetch('https://api.github.com/user', {
      headers: {
        'Authorization': `Bearer ${tokenData.access_token}`,
        'User-Agent': 'FastGPT'
      }
    })

    const githubUser = await userResponse.json()
    if (!githubUser.id) {
      throw new Error('获取GitHub用户信息失败')
    }

    // 3. 查找或创建用户
    let user = await MongoUser.findOne({ 
      'oauth.github.id': githubUser.id 
    })

    if (!user) {
      // 创建新用户
      user = await MongoUser.create({
        username: githubUser.login,
        email: githubUser.email,
        avatar: githubUser.avatar_url,
        oauth: {
          github: {
            id: githubUser.id,
            username: githubUser.login,
            email: githubUser.email
          }
        },
        status: UserStatusEnum.active,
        createTime: new Date()
      })

      // 创建默认团队
      await createDefaultTeam(user._id)
    }

    // 4. 生成JWT令牌
    const token = generateAccessToken({
      userId: user._id,
      teamId: user.defaultTeamId,
      tmbId: user.defaultTmbId
    })

    return { user, token }

  } catch (error) {
    console.error('GitHub OAuth登录失败:', error)
    throw error
  }
}

// Google OAuth认证
export const googleOAuthLogin = async (idToken: string): Promise<{
  user: UserType
  token: string
}> => {
  try {
    // 1. 验证Google ID Token
    const ticket = await googleAuthClient.verifyIdToken({
      idToken,
      audience: process.env.GOOGLE_CLIENT_ID
    })

    const payload = ticket.getPayload()
    if (!payload) {
      throw new Error('无效的Google ID Token')
    }

    // 2. 查找或创建用户
    let user = await MongoUser.findOne({ 
      'oauth.google.id': payload.sub 
    })

    if (!user) {
      user = await MongoUser.create({
        username: payload.name || payload.email,
        email: payload.email,
        avatar: payload.picture,
        oauth: {
          google: {
            id: payload.sub,
            email: payload.email,
            name: payload.name
          }
        },
        status: UserStatusEnum.active,
        createTime: new Date()
      })

      await createDefaultTeam(user._id)
    }

    // 3. 生成令牌
    const token = generateAccessToken({
      userId: user._id,
      teamId: user.defaultTeamId,
      tmbId: user.defaultTmbId
    })

    return { user, token }

  } catch (error) {
    console.error('Google OAuth登录失败:', error)
    throw error
  }
}

// Microsoft OAuth认证
export const microsoftOAuthLogin = async (
  code: string,
  tenantId?: string
): Promise<{ user: UserType; token: string }> => {
  try {
    const tokenEndpoint = tenantId 
      ? `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/token`
      : 'https://login.microsoftonline.com/common/oauth2/v2.0/token'

    // 1. 获取访问令牌
    const tokenResponse = await fetch(tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        client_id: process.env.MICROSOFT_CLIENT_ID!,
        client_secret: process.env.MICROSOFT_CLIENT_SECRET!,
        code,
        grant_type: 'authorization_code',
        redirect_uri: process.env.MICROSOFT_REDIRECT_URI!,
        scope: 'https://graph.microsoft.com/User.Read'
      })
    })

    const tokenData = await tokenResponse.json()
    if (tokenData.error) {
      throw new Error(`Microsoft OAuth错误: ${tokenData.error_description}`)
    }

    // 2. 获取用户信息
    const userResponse = await fetch('https://graph.microsoft.com/v1.0/me', {
      headers: {
        'Authorization': `Bearer ${tokenData.access_token}`
      }
    })

    const msUser = await userResponse.json()
    if (!msUser.id) {
      throw new Error('获取Microsoft用户信息失败')
    }

    // 3. 处理用户登录逻辑
    let user = await MongoUser.findOne({ 
      'oauth.microsoft.id': msUser.id 
    })

    if (!user) {
      user = await MongoUser.create({
        username: msUser.displayName || msUser.userPrincipalName,
        email: msUser.userPrincipalName,
        oauth: {
          microsoft: {
            id: msUser.id,
            email: msUser.userPrincipalName,
            name: msUser.displayName,
            tenantId: tenantId
          }
        },
        status: UserStatusEnum.active,
        createTime: new Date()
      })

      await createDefaultTeam(user._id)
    }

    const token = generateAccessToken({
      userId: user._id,
      teamId: user.defaultTeamId,
      tmbId: user.defaultTmbId
    })

    return { user, token }

  } catch (error) {
    console.error('Microsoft OAuth登录失败:', error)
    throw error
  }
}
```

## 📊 监控与日志集成

### 1. OpenTelemetry集成

**链路追踪配置** (`packages/service/common/otel/trace/register.ts`):

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'
import { Resource } from '@opentelemetry/resources'
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions'

// OpenTelemetry初始化
const initOpenTelemetry = (): NodeSDK | null => {
  const serviceName = process.env.SIGNOZ_SERVICE_NAME || 'fastgpt'
  const otlpEndpoint = process.env.SIGNOZ_BASE_URL

  if (!otlpEndpoint) {
    console.log('OpenTelemetry未配置，跳过初始化')
    return null
  }

  try {
    const sdk = new NodeSDK({
      resource: new Resource({
        [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
        [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version || '1.0.0',
        [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development'
      }),
      instrumentations: [
        getNodeAutoInstrumentations({
          // HTTP请求追踪
          '@opentelemetry/instrumentation-http': {
            enabled: true,
            requestHook: (span, request) => {
              span.setAttributes({
                'http.request.size': request.headers['content-length'] || 0,
                'http.user_agent': request.headers['user-agent'] || 'unknown'
              })
            }
          },
          // MongoDB追踪
          '@opentelemetry/instrumentation-mongodb': {
            enabled: true,
            enhancedDatabaseReporting: true
          },
          // Redis追踪
          '@opentelemetry/instrumentation-redis': {
            enabled: true
          },
          // Express追踪
          '@opentelemetry/instrumentation-express': {
            enabled: true
          }
        })
      ],
      traceExporter: new OTLPTraceExporter({
        url: `${otlpEndpoint}/v1/traces`,
        headers: {
          'signoz-access-token': process.env.SIGNOZ_ACCESS_TOKEN || ''
        }
      }),
      metricReader: new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter({
          url: `${otlpEndpoint}/v1/metrics`,
          headers: {
            'signoz-access-token': process.env.SIGNOZ_ACCESS_TOKEN || ''
          }
        }),
        exportIntervalMillis: 10000 // 10秒导出间隔
      })
    })

    sdk.start()
    console.log('✅ OpenTelemetry初始化成功')
    
    return sdk

  } catch (error) {
    console.error('❌ OpenTelemetry初始化失败:', error)
    return null
  }
}

// 自定义追踪
export const createCustomTrace = (name: string, attributes?: Record<string, any>) => {
  const tracer = trace.getTracer('fastgpt-custom', '1.0.0')
  
  return tracer.startSpan(name, {
    attributes: {
      'service.name': 'fastgpt',
      'trace.type': 'custom',
      ...attributes
    }
  })
}

// 性能指标收集
export const recordPerformanceMetric = (
  name: string,
  value: number,
  attributes?: Record<string, any>
) => {
  const meter = metrics.getMeter('fastgpt-metrics', '1.0.0')
  const histogram = meter.createHistogram(name, {
    description: `Performance metric for ${name}`,
    unit: 'ms'
  })
  
  histogram.record(value, attributes)
}
```

### 2. SignoZ集成

**SignoZ监控配置**:

```typescript
// SignoZ配置接口
interface SignoZConfig {
  baseUrl: string           // SignoZ服务地址
  accessToken?: string      // 访问令牌
  serviceName: string       // 服务名称
  environment: string       // 环境标识
}

// 自定义指标上报
export class SignoZMetrics {
  private config: SignoZConfig

  constructor(config: SignoZConfig) {
    this.config = config
  }

  // 上报业务指标
  async reportCustomMetric(props: {
    metricName: string
    value: number
    timestamp?: number
    tags?: Record<string, string>
  }): Promise<void> {
    const { metricName, value, timestamp = Date.now(), tags = {} } = props

    try {
      const payload = {
        metrics: [{
          name: metricName,
          value,
          timestamp,
          tags: {
            service: this.config.serviceName,
            environment: this.config.environment,
            ...tags
          }
        }]
      }

      await fetch(`${this.config.baseUrl}/api/v1/metrics`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.config.accessToken}`
        },
        body: JSON.stringify(payload)
      })

    } catch (error) {
      console.warn('SignoZ指标上报失败:', error)
    }
  }

  // 上报错误事件
  async reportError(props: {
    error: Error
    context?: Record<string, any>
    userId?: string
    traceId?: string
  }): Promise<void> {
    const { error, context = {}, userId, traceId } = props

    await this.reportCustomMetric({
      metricName: 'error_count',
      value: 1,
      tags: {
        error_type: error.name,
        error_message: error.message,
        user_id: userId || 'unknown',
        trace_id: traceId || 'unknown',
        ...Object.fromEntries(
          Object.entries(context).map(([k, v]) => [k, String(v)])
        )
      }
    })
  }

  // 上报性能指标
  async reportPerformance(props: {
    operation: string
    duration: number
    success: boolean
    userId?: string
  }): Promise<void> {
    const { operation, duration, success, userId } = props

    await this.reportCustomMetric({
      metricName: 'operation_duration',
      value: duration,
      tags: {
        operation,
        status: success ? 'success' : 'failure',
        user_id: userId || 'unknown'
      }
    })
  }
}
```

### 3. 聊天日志集成

**外部日志集成** (`packages/service/common/system/log.ts`):

```typescript
// 聊天日志配置
interface ChatLogConfig {
  url: string               // 日志服务URL
  interval: number          // 发送间隔(秒)
  sourceIdPrefix: string    // 源ID前缀
  batchSize: number         // 批量大小
}

// 聊天日志管理器
export class ChatLogManager {
  private config: ChatLogConfig
  private logBuffer: ChatLogEntry[] = []
  private timer?: NodeJS.Timeout

  constructor(config: ChatLogConfig) {
    this.config = config
    this.startBatchSending()
  }

  // 记录聊天日志
  log(entry: {
    chatId: string
    userId: string
    teamId: string
    appId: string
    message: string
    role: 'user' | 'assistant' | 'system'
    timestamp?: Date
    metadata?: Record<string, any>
  }): void {
    const logEntry: ChatLogEntry = {
      ...entry,
      sourceId: `${this.config.sourceIdPrefix}_${entry.chatId}`,
      timestamp: entry.timestamp || new Date(),
      id: generateId()
    }

    this.logBuffer.push(logEntry)

    // 缓冲区满时立即发送
    if (this.logBuffer.length >= this.config.batchSize) {
      this.flushLogs()
    }
  }

  // 批量发送日志
  private async flushLogs(): Promise<void> {
    if (this.logBuffer.length === 0) return

    const logsToSend = [...this.logBuffer]
    this.logBuffer = []

    try {
      await fetch(this.config.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'User-Agent': 'FastGPT-Logger/1.0'
        },
        body: JSON.stringify({
          logs: logsToSend,
          source: 'fastgpt',
          timestamp: new Date().toISOString()
        }),
        timeout: 10000
      })

      console.log(`聊天日志发送成功: ${logsToSend.length}条`)

    } catch (error) {
      console.error('聊天日志发送失败:', error)
      // 发送失败时重新加入缓冲区
      this.logBuffer.unshift(...logsToSend)
    }
  }

  // 启动定时批量发送
  private startBatchSending(): void {
    this.timer = setInterval(() => {
      this.flushLogs()
    }, this.config.interval * 1000)
  }

  // 停止日志管理器
  stop(): void {
    if (this.timer) {
      clearInterval(this.timer)
      this.timer = undefined
    }
    // 发送剩余日志
    this.flushLogs()
  }
}

// 全局聊天日志实例
let globalChatLogger: ChatLogManager | null = null

export const initChatLogger = (config: ChatLogConfig): void => {
  if (globalChatLogger) {
    globalChatLogger.stop()
  }
  
  globalChatLogger = new ChatLogManager(config)
}

export const logChat = (entry: any): void => {
  if (globalChatLogger) {
    globalChatLogger.log(entry)
  }
}
```

## 🔗 HTTP插件与Webhook系统

### 1. HTTP请求工具

**完整HTTP客户端** (`packages/service/core/workflow/dispatch/tools/http468.ts`):

```typescript
// HTTP请求工具
export const dispatchHttpRequest = async (props: HttpRequestProps): Promise<HttpResponse> => {
  const {
    url,
    method = 'GET',
    headers = '{}',
    params = '{}',
    body = '',
    timeout = 30000,
    contentType = 'json'
  } = props

  try {
    // 1. 参数预处理
    const processedUrl = replaceVariables(url, props.variables || {})
    const processedHeaders = JSON.parse(replaceVariables(headers, props.variables || {}))
    const processedParams = JSON.parse(replaceVariables(params, props.variables || {}))
    const processedBody = replaceVariables(body, props.variables || {})

    // 2. URL参数处理
    const urlObj = new URL(processedUrl)
    Object.entries(processedParams).forEach(([key, value]) => {
      urlObj.searchParams.set(key, String(value))
    })

    // 3. 安全检查（防止SSRF攻击）
    if (isInternalAddress(urlObj.hostname)) {
      throw new Error('禁止访问内部地址')
    }

    // 4. 请求头处理
    const requestHeaders: Record<string, string> = {
      'User-Agent': 'FastGPT-HTTP-Tool/1.0',
      ...processedHeaders
    }

    // 5. 请求体处理
    let requestBody: string | FormData | undefined
    switch (contentType) {
      case 'json':
        requestHeaders['Content-Type'] = 'application/json'
        requestBody = processedBody ? JSON.stringify(JSON.parse(processedBody)) : undefined
        break
      case 'form':
        requestHeaders['Content-Type'] = 'application/x-www-form-urlencoded'
        requestBody = new URLSearchParams(JSON.parse(processedBody || '{}')).toString()
        break
      case 'form-data':
        const formData = new FormData()
        Object.entries(JSON.parse(processedBody || '{}')).forEach(([key, value]) => {
          formData.append(key, String(value))
        })
        requestBody = formData
        break
      case 'xml':
        requestHeaders['Content-Type'] = 'application/xml'
        requestBody = processedBody
        break
      case 'raw':
        requestBody = processedBody
        break
    }

    // 6. 发送HTTP请求
    const controller = new AbortController()
    const timeoutId = setTimeout(() => controller.abort(), timeout)

    const startTime = Date.now()
    const response = await fetch(urlObj.toString(), {
      method: method.toUpperCase(),
      headers: requestHeaders,
      body: requestBody,
      signal: controller.signal
    })

    clearTimeout(timeoutId)
    const duration = Date.now() - startTime

    // 7. 响应处理
    const responseText = await response.text()
    let responseData: any

    try {
      responseData = JSON.parse(responseText)
    } catch {
      responseData = responseText
    }

    // 8. 记录请求日志
    console.log(`HTTP请求完成: ${method} ${processedUrl} - ${response.status} (${duration}ms)`)

    return {
      success: response.ok,
      status: response.status,
      statusText: response.statusText,
      headers: Object.fromEntries(response.headers.entries()),
      data: responseData,
      duration,
      size: responseText.length
    }

  } catch (error) {
    console.error('HTTP请求失败:', error)
    
    if (error.name === 'AbortError') {
      throw new Error(`HTTP请求超时 (${timeout}ms)`)
    }
    
    throw new Error(`HTTP请求失败: ${error.message}`)
  }
}

// 变量替换函数
const replaceVariables = (template: string, variables: Record<string, any>): string => {
  return template.replace(/\{\{([^}]+)\}\}/g, (match, varName) => {
    const keys = varName.trim().split('.')
    let value = variables
    
    for (const key of keys) {
      if (value && typeof value === 'object' && key in value) {
        value = value[key]
      } else {
        return match // 变量不存在时保持原样
      }
    }
    
    return String(value)
  })
}

// 内部地址检查（防SSRF）
const isInternalAddress = (hostname: string): boolean => {
  const internalRanges = [
    /^127\./,                    // 127.0.0.0/8
    /^10\./,                     // 10.0.0.0/8
    /^172\.(1[6-9]|2[0-9]|3[0-1])\./, // 172.16.0.0/12
    /^192\.168\./,               // 192.168.0.0/16
    /^169\.254\./,               // 169.254.0.0/16
    /^::1$/,                     // IPv6 loopback
    /^fe80::/,                   // IPv6 link-local
    /^fc00::/                    // IPv6 unique local
  ]

  return internalRanges.some(range => range.test(hostname)) ||
         hostname === 'localhost' ||
         hostname === '0.0.0.0'
}
```

### 2. Webhook处理系统

**通用Webhook处理器**:

```typescript
// Webhook事件类型
enum WebhookEventType {
  CHAT_MESSAGE = 'chat.message',
  USER_REGISTER = 'user.register',
  PAYMENT_SUCCESS = 'payment.success',
  PLUGIN_INSTALL = 'plugin.install',
  SYSTEM_ERROR = 'system.error'
}

// Webhook管理器  
export class WebhookManager {
  private endpoints = new Map<string, WebhookEndpoint>()

  // 注册Webhook端点
  registerEndpoint(props: {
    id: string
    url: string
    events: WebhookEventType[]
    secret?: string
    timeout?: number
    retries?: number
  }): void {
    const { id, url, events, secret, timeout = 10000, retries = 3 } = props

    this.endpoints.set(id, {
      url,
      events,
      secret,
      timeout,
      retries,
      active: true,
      createdAt: new Date()
    })
  }

  // 触发Webhook事件
  async triggerEvent(props: {
    event: WebhookEventType
    data: any
    source?: string
    timestamp?: Date
  }): Promise<void> {
    const { event, data, source = 'fastgpt', timestamp = new Date() } = props

    // 查找监听此事件的端点
    const endpoints = Array.from(this.endpoints.values())
      .filter(endpoint => endpoint.active && endpoint.events.includes(event))

    if (endpoints.length === 0) {
      console.log(`没有Webhook端点监听事件: ${event}`)
      return
    }

    // 并行发送到所有端点
    const promises = endpoints.map(endpoint => 
      this.sendWebhook(endpoint, { event, data, source, timestamp })
    )

    await Promise.allSettled(promises)
  }

  // 发送Webhook请求
  private async sendWebhook(
    endpoint: WebhookEndpoint,
    payload: WebhookPayload
  ): Promise<void> {
    let lastError: Error | null = null

    for (let attempt = 1; attempt <= endpoint.retries; attempt++) {
      try {
        const headers: Record<string, string> = {
          'Content-Type': 'application/json',
          'User-Agent': 'FastGPT-Webhook/1.0',
          'X-Webhook-Event': payload.event,
          'X-Webhook-Source': payload.source,
          'X-Webhook-Timestamp': payload.timestamp.toISOString(),
          'X-Webhook-Attempt': attempt.toString()
        }

        // 生成签名
        if (endpoint.secret) {
          const signature = this.generateSignature(JSON.stringify(payload), endpoint.secret)
          headers['X-Webhook-Signature'] = signature
        }

        const response = await fetch(endpoint.url, {
          method: 'POST',
          headers,
          body: JSON.stringify(payload),
          timeout: endpoint.timeout
        })

        if (response.ok) {
          console.log(`Webhook发送成功: ${endpoint.url} - ${payload.event}`)
          return
        } else {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`)
        }

      } catch (error) {
        lastError = error as Error
        console.warn(`Webhook发送失败 (尝试 ${attempt}/${endpoint.retries}): ${error.message}`)

        if (attempt < endpoint.retries) {
          // 指数退避重试
          const delay = Math.min(1000 * Math.pow(2, attempt - 1), 30000)
          await new Promise(resolve => setTimeout(resolve, delay))
        }
      }
    }

    // 所有重试都失败
    console.error(`Webhook发送彻底失败: ${endpoint.url} - ${lastError?.message}`)
  }

  // 生成HMAC签名
  private generateSignature(payload: string, secret: string): string {
    return crypto
      .createHmac('sha256', secret)
      .update(payload)
      .digest('hex')
  }

  // 验证Webhook签名
  static verifySignature(
    payload: string,
    signature: string,
    secret: string
  ): boolean {
    const expectedSignature = crypto
      .createHmac('sha256', secret)
      .update(payload)
      .digest('hex')

    return signature === expectedSignature
  }
}

// 全局Webhook管理器
export const globalWebhookManager = new WebhookManager()

// 便捷的事件触发函数
export const triggerWebhookEvent = (
  event: WebhookEventType,
  data: any,
  source?: string
): void => {
  globalWebhookManager.triggerEvent({ event, data, source }).catch(error => {
    console.error('Webhook事件触发失败:', error)
  })
}
```

---

FastGPT 的第三方集成生态展现了企业级软件的全面整合能力，通过**标准化接口**、**安全认证**、**错误处理**和**监控追踪**，构建了一个功能完整、安全可靠的集成平台。无论是AI服务、数据库、企业通信还是监控系统，都能在统一的架构下高效协作。