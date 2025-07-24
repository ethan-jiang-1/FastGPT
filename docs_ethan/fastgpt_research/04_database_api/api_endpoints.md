# FastGPT API 端点架构深度分析

## 🌐 API 架构概述

FastGPT 基于 **Next.js API Routes** 构建了完整的 RESTful API 架构，采用 **monorepo 结构** 分离关注点，将全局类型、服务逻辑和 Web 组件组织到不同的包中。

### 核心 API 路径结构

- **主要 API 路由**: `projects/app/src/pages/api/` - Next.js API 路由
- **服务层**: `packages/service/` - 业务逻辑和数据库操作
- **全局类型**: `packages/global/` - 共享类型定义和常量

## 🚀 核心 API 分类

### 1. 聊天与补全 API

#### 聊天补全端点
**路径**: `/api/v1/chat/completions`
**文件**: `projects/app/src/pages/api/v1/chat/completions.ts`

```typescript
// OpenAI 兼容的聊天补全接口
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const {
      messages,
      stream = false,
      model,
      temperature = 0,
      max_tokens,
      top_p,
      frequency_penalty,
      presence_penalty,
      stop,
      user,
      ...otherParams
    } = req.body

    // 身份验证
    const { teamId, tmbId, apikey, appId } = await authCert({
      req,
      authToken: true,
      authApiKey: true
    })

    // 流式响应设置
    if (stream) {
      res.setHeader('Content-Type', 'text/event-stream')
      res.setHeader('Cache-Control', 'no-cache')
      res.setHeader('Connection', 'keep-alive')
    }

    // 调用聊天服务
    const result = await dispatchWorkFlow({
      teamId,
      tmbId,
      appId,
      chatId: undefined,
      variables: {},
      histories: messages,
      stream,
      detail: false
    })

    if (stream) {
      // 流式响应处理
      for await (const chunk of result) {
        res.write(`data: ${JSON.stringify(chunk)}\n\n`)
      }
      res.write('data: [DONE]\n\n')
      res.end()
    } else {
      res.json(result)
    }

  } catch (error) {
    res.status(500).json({
      error: {
        message: error.message,
        type: 'internal_error'
      }
    })
  }
}
```

#### 文本嵌入端点
**路径**: `/api/v1/embeddings`
**文件**: `projects/app/src/pages/api/v1/embeddings.ts`

```typescript
interface EmbeddingRequest {
  input: string | string[]
  model: string
  user?: string
}

interface EmbeddingResponse {
  object: 'list'
  data: Array<{
    object: 'embedding'
    embedding: number[]
    index: number
  }>
  model: string
  usage: {
    prompt_tokens: number
    total_tokens: number
  }
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<EmbeddingResponse>
) {
  const { input, model, user } = req.body as EmbeddingRequest

  // 身份验证
  const { teamId, tmbId } = await authCert({
    req,
    authToken: true,
    authApiKey: true
  })

  // 调用嵌入服务
  const embeddings = await getEmbedding({
    model,
    input: Array.isArray(input) ? input : [input],
    userId: user
  })

  // 格式化响应
  const response: EmbeddingResponse = {
    object: 'list',
    data: embeddings.map((embedding, index) => ({
      object: 'embedding',
      embedding,
      index
    })),
    model,
    usage: {
      prompt_tokens: calculateTokens(input),
      total_tokens: calculateTokens(input)
    }
  }

  res.json(response)
}
```

### 2. 应用管理 API

#### 应用创建端点
**路径**: `/api/core/app/create`
**文件**: `projects/app/src/pages/api/core/app/create.ts`

```typescript
interface CreateAppBody {
  parentId?: string
  name: string
  avatar?: string
  type: AppTypeEnum
  modules?: FlowNodeType[]
  edges?: Edge[]
  chatConfig?: AppChatConfig
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<any>
) {
  try {
    const {
      parentId,
      name,
      avatar,
      type,
      modules = [],
      edges = [],
      chatConfig
    } = req.body as CreateAppBody

    // 身份验证和权限检查
    const { teamId, tmbId } = await authCert({
      req,
      authToken: true,
      per: ReadPermissionVal
    })

    // 验证父应用权限
    if (parentId) {
      await authApp({
        appId: parentId,
        per: ManagePermissionVal,
        teamId,
        tmbId
      })
    }

    // 创建应用
    const app = await MongoApp.create({
      ...req.body,
      teamId,
      tmbId,
      modules: modules || [],
      edges: edges || [],
      version: 'v2'
    })

    // 初始化权限
    await addResourcePermission({
      teamId,
      resourceType: PerResourceTypeEnum.app,
      resourceId: app._id,
      permission: OwnerPermissionVal,
      tmbId
    })

    jsonRes(res, {
      data: app._id
    })

  } catch (error) {
    jsonRes(res, {
      code: 500,
      error
    })
  }
}
```

#### 应用列表端点
**路径**: `/api/core/app/list`
**文件**: `projects/app/src/pages/api/core/app/list.ts`

```typescript
interface GetAppListQuery {
  parentId?: string
  type?: AppTypeEnum
  searchKey?: string
  pageNum?: number
  pageSize?: number
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const {
      parentId,
      type,
      searchKey,
      pageNum = 1,
      pageSize = 20
    } = req.query as GetAppListQuery

    // 身份验证
    const { teamId, tmbId } = await authCert({
      req,
      authToken: true,
      per: ReadPermissionVal
    })

    // 构建查询条件
    const match: any = {
      teamId,
      ...(parentId !== undefined && { parentId: parentId || null }),
      ...(type && { type })
    }

    // 搜索条件
    if (searchKey) {
      match.name = new RegExp(searchKey, 'i')
    }

    // 聚合查询（包含权限过滤）
    const aggregation = [
      { $match: match },
      {
        $lookup: {
          from: 'resource_permissions',
          let: { appId: '$_id' },
          pipeline: [
            {
              $match: {
                $expr: {
                  $and: [
                    { $eq: ['$resourceId', '$$appId'] },
                    { $eq: ['$resourceType', PerResourceTypeEnum.app] },
                    { $eq: ['$teamId', teamId] },
                    {
                      $or: [
                        { $eq: ['$tmbId', tmbId] },
                        { $in: ['$groupId', userGroupIds] }
                      ]
                    }
                  ]
                }
              }
            }
          ],
          as: 'permissions'
        }
      },
      {
        $match: {
          'permissions.0': { $exists: true }
        }
      },
      {
        $sort: { updateTime: -1 }
      },
      {
        $skip: (pageNum - 1) * pageSize
      },
      {
        $limit: pageSize
      },
      {
        $project: {
          _id: 1,
          name: 1,
          avatar: 1,
          type: 1,
          updateTime: 1,
          permission: { $arrayElemAt: ['$permissions.permission', 0] }
        }
      }
    ]

    const apps = await MongoApp.aggregate(aggregation)
    const total = await MongoApp.countDocuments(match)

    jsonRes(res, {
      data: {
        list: apps,
        total,
        pageNum,
        pageSize
      }
    })

  } catch (error) {
    jsonRes(res, { code: 500, error })
  }
}
```

### 3. 数据集管理 API

#### 数据集数据上传端点
**路径**: `/api/core/dataset/data/pushData`
**文件**: `projects/app/src/pages/api/core/dataset/data/pushData.ts`

```typescript
interface PushDatasetDataProps {
  datasetId: string
  collectionId: string
  data?: PushDatasetDataChunkProps[]
  trainingMode?: TrainingModeEnum
  prompt?: string
  billId?: string
}

interface PushDatasetDataChunkProps {
  q: string  // 问题
  a?: string // 答案
  indexes?: DatasetDataIndexItemType[]
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<any>
) {
  try {
    const {
      datasetId,
      collectionId,
      data = [],
      trainingMode = TrainingModeEnum.chunk,
      prompt,
      billId
    } = req.body as PushDatasetDataProps

    // 身份验证和权限检查
    const { teamId, tmbId } = await authCert({
      req,
      authToken: true,
      authApiKey: true,
      per: WritePermissionVal
    })

    // 验证数据集权限
    await authDataset({
      datasetId,
      per: WritePermissionVal,
      teamId,
      tmbId
    })

    // 数据预处理和验证
    const processedData = data.map((item, index) => ({
      ...item,
      chunkIndex: index,
      dataId: getNanoid(24)
    }))

    // 批量插入数据
    const insertResult = await Promise.allSettled(
      processedData.map(async (item) => {
        // 创建数据记录
        const datasetData = await MongoDatasetData.create({
          teamId,
          tmbId,
          datasetId,
          collectionId,
          q: item.q,
          a: item.a || '',
          chunkIndex: item.chunkIndex,
          indexes: item.indexes || []
        })

        // 生成向量嵌入
        if (item.q) {
          await pushDatasetDataToVector({
            teamId,
            datasetId,
            collectionId,
            dataId: datasetData._id.toString(),
            text: item.q
          })
        }

        return datasetData._id
      })
    )

    // 统计结果
    const successIds = insertResult
      .filter(result => result.status === 'fulfilled')
      .map(result => (result as PromiseFulfilledResult<any>).value)

    const failedCount = insertResult.length - successIds.length

    jsonRes(res, {
      data: {
        insertedIds: successIds,
        successCount: successIds.length,
        failedCount
      }
    })

  } catch (error) {
    jsonRes(res, {
      code: 500,
      error: error.message
    })
  }
}
```

#### 向量检索端点
**路径**: `/api/core/dataset/searchTest`
**文件**: `projects/app/src/pages/api/core/dataset/searchTest.ts`

```typescript
interface SearchTestProps {
  datasetId: string
  text: string
  limit?: number
  searchMode?: DatasetSearchModeEnum
  reRankQuery?: string
  datasetSearchUsingReRank?: boolean
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const {
      datasetId,
      text,
      limit = 5,
      searchMode = DatasetSearchModeEnum.embedding,
      reRankQuery,
      datasetSearchUsingReRank = false
    } = req.body as SearchTestProps

    // 身份验证和权限检查
    const { teamId, tmbId } = await authCert({
      req,
      authToken: true,
      per: ReadPermissionVal
    })

    // 验证数据集权限
    await authDataset({
      datasetId,
      per: ReadPermissionVal,
      teamId,
      tmbId
    })

    // 执行向量检索
    const searchResults = await searchDatasetData({
      teamId,
      datasetId,
      text,
      searchMode,
      limit,
      usingReRank: datasetSearchUsingReRank,
      reRankQuery
    })

    // 格式化检索结果
    const formattedResults = searchResults.map((item) => ({
      id: item.id,
      datasetId: item.datasetId,
      collectionId: item.collectionId,
      sourceName: item.sourceName || '',
      sourceId: item.sourceId || '',
      q: item.q,
      a: item.a,
      chunkIndex: item.chunkIndex || 0,
      score: item.score,
    }))

    jsonRes(res, {
      data: {
        list: formattedResults,
        searchMode,
        limit,
        searchTime: Date.now()
      }
    })

  } catch (error) {
    jsonRes(res, {
      code: 500,
      error: error.message
    })
  }
}
```

### 4. 文件管理 API

#### 文件上传端点
**路径**: `/api/common/file/upload`
**文件**: `projects/app/src/pages/api/common/file/upload.ts`

```typescript
// Next.js 配置
export const config = {
  api: {
    bodyParser: false,        // 禁用默认body解析器
    sizeLimit: '20mb',       // 最大请求大小
    responseLimit: '20mb'    // 最大响应大小
  }
}

interface FileUploadQuery {
  bucketName?: BucketNameEnum
  metadata?: string
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const { bucketName, metadata } = req.query as FileUploadQuery

    // 身份验证
    const { teamId, tmbId } = await authCert({
      req,
      authToken: true,
      per: WritePermissionVal
    })

    // 解析上传数据
    const form = formidable({
      maxFileSize: 20 * 1024 * 1024, // 20MB
      maxFiles: 1,
      uploadDir: '/tmp'
    })

    const [fields, files] = await form.parse(req)
    const file = Array.isArray(files.file) ? files.file[0] : files.file

    if (!file) {
      throw new Error('No file uploaded')
    }

    // 文件验证
    const allowedTypes = [
      'image/jpeg', 'image/png', 'image/gif',
      'application/pdf', 'text/plain',
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
    ]

    if (!allowedTypes.includes(file.mimetype)) {
      throw new Error(`File type ${file.mimetype} not supported`)
    }

    // 读取文件数据
    const fileBuffer = await fs.readFile(file.filepath)

    // 上传到存储系统
    const uploadResult = await uploadFile({
      bucketName: bucketName || BucketNameEnum.chat,
      fileName: file.originalFilename || 'unknown',
      buffer: fileBuffer,
      metadata: metadata ? JSON.parse(metadata) : {},
      teamId
    })

    // 清理临时文件
    await fs.unlink(file.filepath)

    jsonRes(res, {
      data: {
        fileId: uploadResult.fileId,
        filename: uploadResult.filename,
        size: file.size,
        mimetype: file.mimetype,
        uploadTime: new Date()
      }
    })

  } catch (error) {
    jsonRes(res, {
      code: 500,
      error: error.message
    })
  }
}
```

#### 文件访问端点
**路径**: `/api/common/file/read`
**文件**: `projects/app/src/pages/api/common/file/read.ts`

```typescript
interface FileReadQuery {
  fileId: string
  token?: string
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const { fileId, token } = req.query as FileReadQuery

    // Token验证（如果提供）
    let teamId: string | undefined

    if (token) {
      const decoded = await verifyFileToken(token)
      teamId = decoded.teamId
    } else {
      // 身份验证
      const auth = await authCert({
        req,
        authToken: true,
        per: ReadPermissionVal
      })
      teamId = auth.teamId
    }

    // 获取文件信息
    const fileData = await getFileById({
      fileId,
      teamId
    })

    if (!fileData) {
      return res.status(404).json({
        error: 'File not found'
      })
    }

    // 设置响应头
    res.setHeader('Content-Type', fileData.contentType || 'application/octet-stream')
    res.setHeader('Content-Length', fileData.length)
    res.setHeader('Cache-Control', 'public, max-age=86400') // 24小时缓存

    // 如果是图片，设置额外的头部
    if (fileData.contentType?.startsWith('image/')) {
      res.setHeader('Content-Disposition', 'inline')
    } else {
      res.setHeader('Content-Disposition', `attachment; filename="${fileData.filename}"`)
    }

    // 流式返回文件内容
    const stream = getFileStream(fileId)
    stream.pipe(res)

  } catch (error) {
    res.status(500).json({
      error: error.message
    })
  }
}
```

## 🔐 身份验证与授权

### 身份验证类型枚举

```typescript
enum AuthUserTypeEnum {
  token = 'token',     // 会话令牌认证
  apikey = 'apikey',   // API密钥认证
  outLink = 'outLink', // 分享链接认证
  root = 'root'        // 超级用户认证
}
```

### 统一身份验证中间件

**文件路径**: `packages/service/support/permission/controller.ts`

```typescript
interface AuthModeType {
  req: NextApiRequest
  authToken?: boolean      // 是否启用Token认证
  authApiKey?: boolean     // 是否启用API密钥认证
  authRoot?: boolean       // 是否启用Root认证
  per?: number            // 所需权限级别
}

const authCert = async (props: AuthModeType): Promise<{
  teamId: string
  tmbId: string
  userId?: string
  appId?: string
  apikey?: string
  isOwner: boolean
  canWrite: boolean
}> => {
  const { req, authToken, authApiKey, authRoot, per } = props

  // 解析认证信息
  const parseResult = await parseHeaderCert({
    req,
    authToken,
    authApiKey,
    authRoot
  })

  // 权限验证
  if (per !== undefined) {
    await checkPermission({
      ...parseResult,
      per
    })
  }

  return {
    ...parseResult,
    isOwner: parseResult.permission >= OwnerPermissionVal,
    canWrite: parseResult.permission >= WritePermissionVal
  }
}

// API密钥格式解析
const parseApiKey = (apikey: string): {
  appId?: string
  teamId?: string
} => {
  // 格式: fastgpt-{key}-{appId}
  const keyParts = apikey.split('-')
  
  if (keyParts.length >= 3 && keyParts[0] === 'fastgpt') {
    return {
      appId: keyParts[2],
      teamId: keyParts[1]
    }
  }
  
  throw new Error('Invalid API key format')
}
```

### JWT Token 管理

```typescript
// JWT Token 生成
const generateAccessToken = (payload: {
  userId: string
  teamId: string
  tmbId: string
}): string => {
  return jwt.sign(payload, process.env.TOKEN_KEY!, {
    expiresIn: '7d'
  })
}

// JWT Token 验证
const verifyAccessToken = (token: string): Promise<{
  userId: string
  teamId: string
  tmbId: string
}> => {
  return new Promise((resolve, reject) => {
    jwt.verify(token, process.env.TOKEN_KEY!, (err, decoded) => {
      if (err) {
        reject(new Error('Invalid token'))
      } else {
        resolve(decoded as any)
      }
    })
  })
}
```

## 🚦 频率限制与验证

### IP频率限制实现

**文件路径**: `packages/service/common/middleware/requestLimit.ts`

```typescript
export function useIPFrequencyLimit(props: {
  id: string           // 限制标识符
  seconds: number      // 时间窗口（秒）
  limit: number        // 最大请求数
  force?: boolean      // 强制启用
}) {
  return async (req: ApiRequestProps, res: NextApiResponse) => {
    const { id, seconds, limit, force = false } = props

    // 获取客户端IP
    const ip = requestIp.getClientIp(req) || 'unknown'
    
    // 构建事件ID
    const eventId = `ip-qps-limit-${id}-${ip}`

    try {
      // 检查频率限制
      await authFrequencyLimit({
        eventId,
        maxAmount: limit,
        expiredTime: addSeconds(new Date(), seconds),
        force
      })
    } catch (error) {
      res.status(429).json({
        code: 429,
        statusText: 'Too Many Requests',
        message: `Rate limit exceeded: ${limit} requests per ${seconds} seconds`,
        data: null
      })
      return
    }
  }
}

// 频率限制检查器
const authFrequencyLimit = async (props: {
  eventId: string
  maxAmount: number
  expiredTime: Date
  force?: boolean
}): Promise<void> => {
  const { eventId, maxAmount, expiredTime, force } = props

  // 从Redis获取当前计数
  const currentCount = await redis.get(`freq_limit:${eventId}`)
  const count = currentCount ? parseInt(currentCount) : 0

  if (count >= maxAmount) {
    throw new Error('Frequency limit exceeded')
  }

  // 递增计数器
  const newCount = await redis.incr(`freq_limit:${eventId}`)
  
  // 设置过期时间（仅首次）
  if (newCount === 1) {
    await redis.expireat(`freq_limit:${eventId}`, Math.floor(expiredTime.getTime() / 1000))
  }
}
```

### 团队使用量限制

```typescript
interface TeamUsageLimit {
  maxTokensPerDay: number      // 每日最大Token数
  maxRequestsPerHour: number   // 每小时最大请求数
  maxConcurrentRequests: number // 最大并发请求数
}

const checkTeamUsageLimit = async (teamId: string): Promise<void> => {
  // 获取团队限制配置
  const limits = await getTeamUsageLimits(teamId)
  
  // 检查日Token限制
  const todayUsage = await getTodayTokenUsage(teamId)
  if (todayUsage > limits.maxTokensPerDay) {
    throw new Error('Daily token limit exceeded')
  }
  
  // 检查小时请求限制
  const hourlyRequests = await getHourlyRequestCount(teamId)
  if (hourlyRequests > limits.maxRequestsPerHour) {
    throw new Error('Hourly request limit exceeded')
  }
  
  // 检查并发限制
  const concurrentRequests = await getConcurrentRequestCount(teamId)
  if (concurrentRequests >= limits.maxConcurrentRequests) {
    throw new Error('Concurrent request limit exceeded')
  }
}
```

## 📊 请求/响应架构

### 标准化响应格式

```typescript
interface ApiResponse<T = any> {
  code: number        // 状态码
  data?: T           // 响应数据
  message?: string   // 响应消息
  error?: any        // 错误信息
}

// 成功响应辅助函数
const jsonRes = <T = any>(
  res: NextApiResponse,
  props: {
    code?: number
    message?: string
    error?: any
    data?: T
  } = {}
) => {
  const { code = 200, message, error, data } = props

  res.status(code).json({
    code,
    statusText: message || getStatusText(code),
    message: error?.message || message,
    data: data || null
  })
}

// 错误响应类型
interface ErrType {
  code: number
  statusText: string
  message: string
  data: null
}

// 特定错误类别
enum ChatErrEnum {
  unAuthChat = 'unAuthChat',                    // 未授权聊天
  contextLimitExceeded = 'contextLimitExceeded', // 上下文长度超限
  insufficientQuota = 'insufficientQuota'       // 配额不足
}
```

### 流式响应处理

```typescript
// Server-Sent Events 流式响应
const handleStreamResponse = async (
  req: NextApiRequest,
  res: NextApiResponse,
  chatStreamGenerator: AsyncGenerator<any>
) => {
  // 设置流式响应头
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')
  res.setHeader('Access-Control-Allow-Origin', '*')

  try {
    // 处理流式数据块
    for await (const chunk of chatStreamGenerator) {
      if (res.destroyed) break

      // 格式化SSE数据
      const sseData = {
        id: chunk.id || generateId(),
        event: chunk.event || 'message',
        data: chunk.data || chunk
      }

      // 发送数据块
      res.write(`data: ${JSON.stringify(sseData)}\n\n`)
    }

    // 发送结束标记
    res.write('data: [DONE]\n\n')
    res.end()

  } catch (error) {
    // 发送错误信息
    res.write(`data: ${JSON.stringify({
      error: {
        message: error.message,
        type: 'stream_error'
      }
    })}\n\n`)
    res.end()
  }
}
```

## 🔌 外部集成与Webhook

### 第三方平台集成

```typescript
// 飞书集成端点
// /api/support/outLink/feishu/[token].ts
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { token } = req.query as { token: string }

  try {
    // 验证飞书签名
    const signature = req.headers['x-feishu-signature'] as string
    const timestamp = req.headers['x-feishu-request-timestamp'] as string
    
    if (!verifyFeishuSignature(signature, timestamp, req.body)) {
      return res.status(401).json({ error: 'Invalid signature' })
    }

    // 解析webhook载荷
    const { type, event } = req.body

    switch (type) {
      case 'event_callback':
        await handleFeishuEvent(token, event)
        break
      case 'url_verification':
        return res.json({ challenge: event.challenge })
      default:
        console.log('Unknown Feishu event type:', type)
    }

    res.json({ success: true })

  } catch (error) {
    console.error('Feishu webhook error:', error)
    res.status(500).json({ error: error.message })
  }
}

// 钉钉集成端点
// /api/support/outLink/dingtalk/[token].ts
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { token } = req.query as { token: string }

  try {
    // 钉钉签名验证
    const signature = req.headers['x-dingtalk-signature'] as string
    const timestamp = req.headers['timestamp'] as string
    
    if (!verifyDingTalkSignature(signature, timestamp, req.body)) {
      return res.status(401).json({ error: 'Invalid signature' })
    }

    // 处理钉钉消息
    await handleDingTalkMessage(token, req.body)
    
    res.json({ success: true })

  } catch (error) {
    console.error('DingTalk webhook error:', error)
    res.status(500).json({ error: error.message })
  }
}
```

### Webhook 签名验证

```typescript
// 飞书签名验证
const verifyFeishuSignature = (
  signature: string,
  timestamp: string,
  body: any
): boolean => {
  const secret = process.env.FEISHU_WEBHOOK_SECRET!
  const rawBody = JSON.stringify(body)
  
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(timestamp + rawBody)
    .digest('hex')
  
  return signature === expectedSignature
}

// 钉钉签名验证
const verifyDingTalkSignature = (
  signature: string,
  timestamp: string,
  body: any
): boolean => {
  const secret = process.env.DINGTALK_WEBHOOK_SECRET!
  const rawBody = JSON.stringify(body)
  
  const stringToSign = timestamp + '\n' + secret
  const expectedSignature = crypto
    .createHmac('sha256', stringToSign)
    .update(rawBody)
    .digest('base64')
  
  return signature === expectedSignature
}
```

## 📋 OpenAPI/Swagger 文档

### 文档结构

**位置**: `projects/app/public/openapi/index.html`
**规范**: `scripts/openapi/openapi.json`

```typescript
// OpenAPI 规范生成
const generateOpenAPISpec = () => {
  return {
    openapi: '3.0.0',
    info: {
      title: 'FastGPT API',
      version: '2.0.0',
      description: 'FastGPT AI Platform REST API'
    },
    servers: [
      {
        url: 'https://api.fastgpt.in',
        description: 'Production server'
      },
      {
        url: 'http://localhost:3000',
        description: 'Development server'
      }
    ],
    security: [
      {
        bearerAuth: []
      },
      {
        apiKeyAuth: []
      }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        },
        apiKeyAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'fastgpt-{key}-{appId}'
        }
      },
      schemas: {
        ChatCompletionRequest: {
          type: 'object',
          required: ['messages'],
          properties: {
            messages: {
              type: 'array',
              items: { $ref: '#/components/schemas/ChatMessage' }
            },
            model: { type: 'string' },
            temperature: { type: 'number', minimum: 0, maximum: 2 },
            max_tokens: { type: 'integer', minimum: 1 },
            stream: { type: 'boolean', default: false }
          }
        }
      }
    }
  }
}
```

## 🛡️ 安全考虑

### 输入验证

```typescript
// Joi 参数验证
import Joi from 'joi'

const chatCompletionSchema = Joi.object({
  messages: Joi.array().items(
    Joi.object({
      role: Joi.string().valid('system', 'user', 'assistant').required(),
      content: Joi.string().required().max(50000),
      name: Joi.string().optional()
    })
  ).required().min(1).max(50),
  model: Joi.string().optional(),
  temperature: Joi.number().min(0).max(2).optional(),
  max_tokens: Joi.number().integer().min(1).max(8000).optional(),
  stream: Joi.boolean().optional(),
  top_p: Joi.number().min(0).max(1).optional()
})

// 验证中间件
const validateRequest = (schema: Joi.ObjectSchema) => {
  return (req: NextApiRequest, res: NextApiResponse, next: () => void) => {
    const { error } = schema.validate(req.body)
    
    if (error) {
      return res.status(400).json({
        code: 400,
        message: 'Validation error',
        error: error.details[0].message
      })
    }
    
    next()
  }
}
```

### CORS 配置

```typescript
// CORS 处理中间件
const withNextCors = async (
  req: NextApiRequest, 
  res: NextApiResponse
) => {
  // 允许的源
  const allowedOrigins = [
    'https://fastgpt.in',
    'https://cloud.fastgpt.in',
    process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : null
  ].filter(Boolean)

  const origin = req.headers.origin
  
  if (allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin)
  }
  
  res.setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE,OPTIONS')
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type,Authorization')
  res.setHeader('Access-Control-Allow-Credentials', 'true')
  
  // 处理预检请求
  if (req.method === 'OPTIONS') {
    res.status(200).end()
    return
  }
}
```

## ⚡ 性能优化

### 连接池管理

```typescript
// MongoDB 连接池配置
const mongoOptions = {
  bufferCommands: true,
  maxConnecting: 30,
  maxPoolSize: 30,
  minPoolSize: 5,
  connectTimeoutMS: 60000,
  waitQueueTimeoutMS: 60000,
  socketTimeoutMS: 60000,
  maxIdleTimeMS: 300000,
  retryWrites: true,
  retryReads: true
}

// Redis 连接池
const redisOptions = {
  maxRetriesPerRequest: 3,
  retryDelayOnFailover: 100,
  lazyConnect: true,
  maxRetriesPerRequest: null,
  keepAlive: 30000
}
```

### 响应缓存

```typescript
// API 响应缓存
const getCachedResponse = async (key: string): Promise<any> => {
  const cached = await redis.get(`api_cache:${key}`)
  return cached ? JSON.parse(cached) : null
}

const setCachedResponse = async (
  key: string, 
  data: any, 
  ttl: number = 300
): Promise<void> => {
  await redis.setex(`api_cache:${key}`, ttl, JSON.stringify(data))
}

// 缓存中间件
const withCache = (ttl: number = 300) => {
  return async (req: NextApiRequest, res: NextApiResponse, next: () => void) => {
    const cacheKey = `${req.url}-${JSON.stringify(req.query)}-${JSON.stringify(req.body)}`
    
    const cached = await getCachedResponse(cacheKey)
    if (cached) {
      return res.json(cached)
    }
    
    // 修改 res.json 以自动缓存
    const originalJson = res.json
    res.json = function(data: any) {
      setCachedResponse(cacheKey, data, ttl)
      return originalJson.call(this, data)
    }
    
    next()
  }
}
```

---

FastGPT 的 API 架构展现了企业级应用的成熟设计，包含完整的安全性、可扩展性和可维护性特性。通过模块化设计和标准化实践，为 AI 应用提供了稳定可靠的 API 服务基础。