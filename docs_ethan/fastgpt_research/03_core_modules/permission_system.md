# FastGPT 权限系统深度分析

## 🔐 权限系统概述

FastGPT 的权限系统是保障企业级应用安全性的**核心防线**，实现了细粒度的资源访问控制和多层级的组织架构管理。该系统采用 **RBAC (Role-Based Access Control)** 模型，结合**资源继承**和**动态权限**机制，为企业提供灵活而安全的权限管理能力。

### 核心设计原则

- **最小权限原则** - 用户只拥有完成工作所需的最小权限
- **权限继承** - 支持组织架构的权限继承机制
- **动态控制** - 基于条件的动态权限验证
- **审计追踪** - 完整的权限操作日志记录
- **分离职责** - 权限管理与业务逻辑的清晰分离

## 🏗️ 权限系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    权限系统架构                               │
├─────────────────────────────────────────────────────────────┤
│  权限控制层 (Permission Control Layer)                       │
│  ├── 权限验证器 (Permission Validator)                       │
│  ├── 角色管理器 (Role Manager)                               │
│  ├── 资源控制器 (Resource Controller)                       │
│  └── 动态权限引擎 (Dynamic Permission Engine)               │
├─────────────────────────────────────────────────────────────┤
│  组织架构层 (Organization Structure Layer)                   │
│  ├── 组织管理 (Organization Management)                     │
│  ├── 团队管理 (Team Management)                             │
│  ├── 成员组管理 (Member Group Management)                   │
│  └── 权限继承机制 (Permission Inheritance)                  │
├─────────────────────────────────────────────────────────────┤
│  资源定义层 (Resource Definition Layer)                      │
│  ├── 应用资源 (App Resources)                               │
│  ├── 知识库资源 (Dataset Resources)                         │
│  ├── 对话资源 (Chat Resources)                              │
│  └── 系统资源 (System Resources)                            │
├─────────────────────────────────────────────────────────────┤
│  数据持久化层 (Data Persistence Layer)                       │
│  ├── 权限配置存储 (Permission Config Storage)               │
│  ├── 角色定义存储 (Role Definition Storage)                 │
│  ├── 用户权限缓存 (User Permission Cache)                   │
│  └── 审计日志存储 (Audit Log Storage)                       │
└─────────────────────────────────────────────────────────────┘
```

## 👥 组织架构与角色体系

### 组织层级设计

**组织架构模型** (`packages/service/support/permission/org/orgSchema.ts`)
```typescript
// 组织架构 Schema
const OrgSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'Team',
    required: true
  },
  parentId: {
    type: Schema.Types.ObjectId,
    ref: 'Org',
    default: null
  },
  name: {
    type: String,
    required: true
  },
  avatar: {
    type: String,
    default: ''
  },
  intro: {
    type: String,
    default: ''
  },
  metadata: {
    type: Map,
    of: Schema.Types.Mixed,
    default: {}
  }
}, {
  timestamps: true
})

// 组织成员 Schema
const OrgMemberSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'Team',
    required: true
  },
  orgId: {
    type: Schema.Types.ObjectId,
    ref: 'Org',
    required: true
  },
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  role: {
    type: String,
    enum: ['owner', 'admin', 'member'],
    default: 'member'
  },
  permissions: [{
    resource: {
      type: String,
      required: true
    },
    actions: [{
      type: String,
      required: true
    }]
  }],
  status: {
    type: String,
    enum: ['active', 'inactive', 'pending'],
    default: 'active'
  }
}, {
  timestamps: true
})
```

### 角色权限矩阵

**角色定义系统** (`packages/service/support/permission/constant.ts`)
```typescript
// 系统预定义角色
export enum SystemRoleEnum {
  owner = 'owner',           // 拥有者
  admin = 'admin',           // 管理员
  editor = 'editor',         // 编辑者
  viewer = 'viewer'          // 查看者
}

// 资源类型枚举
export enum ResourceTypeEnum {
  team = 'team',             // 团队资源
  app = 'app',               // 应用资源
  dataset = 'dataset',       // 知识库资源
  chat = 'chat',             // 对话资源
  plugin = 'plugin',         // 插件资源
  apikey = 'apikey'          // API密钥资源
}

// 操作类型枚举
export enum PermissionActionEnum {
  read = 'read',             // 读取
  write = 'write',           // 写入
  delete = 'delete',         // 删除
  share = 'share',           // 分享
  manage = 'manage'          // 管理
}

// 角色权限映射表
export const RolePermissionMatrix: Record<
  SystemRoleEnum, 
  Record<ResourceTypeEnum, PermissionActionEnum[]>
> = {
  [SystemRoleEnum.owner]: {
    [ResourceTypeEnum.team]: ['read', 'write', 'delete', 'share', 'manage'],
    [ResourceTypeEnum.app]: ['read', 'write', 'delete', 'share', 'manage'],
    [ResourceTypeEnum.dataset]: ['read', 'write', 'delete', 'share', 'manage'],
    [ResourceTypeEnum.chat]: ['read', 'write', 'delete', 'share', 'manage'],
    [ResourceTypeEnum.plugin]: ['read', 'write', 'delete', 'share', 'manage'],
    [ResourceTypeEnum.apikey]: ['read', 'write', 'delete', 'manage']
  },
  [SystemRoleEnum.admin]: {
    [ResourceTypeEnum.team]: ['read', 'write', 'share', 'manage'],
    [ResourceTypeEnum.app]: ['read', 'write', 'delete', 'share'],
    [ResourceTypeEnum.dataset]: ['read', 'write', 'delete', 'share'],
    [ResourceTypeEnum.chat]: ['read', 'write', 'delete'],
    [ResourceTypeEnum.plugin]: ['read', 'write', 'share'],
    [ResourceTypeEnum.apikey]: ['read', 'write']
  },
  [SystemRoleEnum.editor]: {
    [ResourceTypeEnum.team]: ['read'],
    [ResourceTypeEnum.app]: ['read', 'write'],
    [ResourceTypeEnum.dataset]: ['read', 'write'],
    [ResourceTypeEnum.chat]: ['read', 'write'],
    [ResourceTypeEnum.plugin]: ['read'],
    [ResourceTypeEnum.apikey]: ['read']
  },
  [SystemRoleEnum.viewer]: {
    [ResourceTypeEnum.team]: ['read'],
    [ResourceTypeEnum.app]: ['read'],
    [ResourceTypeEnum.dataset]: ['read'],
    [ResourceTypeEnum.chat]: ['read'],
    [ResourceTypeEnum.plugin]: ['read'],
    [ResourceTypeEnum.apikey]: []
  }
}
```

### 团队成员管理

**团队成员控制器** (`packages/service/support/user/team/controller.ts`)
```typescript
export class TeamMemberController {
  
  // 邀请团队成员
  async inviteMember(params: {
    teamId: string
    inviterUserId: string
    inviteeEmail: string
    role: SystemRoleEnum
    orgId?: string
    customPermissions?: Permission[]
  }): Promise<InvitationResult> {
    
    const { teamId, inviterUserId, inviteeEmail, role, orgId, customPermissions } = params
    
    // 验证邀请者权限
    await this.verifyInvitePermission(inviterUserId, teamId)
    
    // 检查被邀请用户是否已存在
    const inviteeUser = await UserModel.findOne({ email: inviteeEmail })
    
    // 创建邀请记录
    const invitation = await TeamInvitationModel.create({
      teamId,
      inviterUserId,
      inviteeEmail,
      role,
      orgId,
      customPermissions,
      status: 'pending',
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7天过期
    })
    
    // 发送邀请邮件
    await this.sendInvitationEmail({
      email: inviteeEmail,
      teamName: await this.getTeamName(teamId),
      inviterName: await this.getUserName(inviterUserId),
      invitationToken: invitation.token,
      expiresAt: invitation.expiresAt
    })
    
    return {
      invitationId: invitation._id,
      status: 'sent',
      expiresAt: invitation.expiresAt
    }
  }
  
  // 处理邀请接受
  async acceptInvitation(params: {
    invitationToken: string
    userId: string
  }): Promise<void> {
    
    const { invitationToken, userId } = params
    
    // 查找有效邀请
    const invitation = await TeamInvitationModel.findOne({
      token: invitationToken,
      status: 'pending',
      expiresAt: { $gt: new Date() }
    })
    
    if (!invitation) {
      throw new Error('邀请无效或已过期')
    }
    
    // 验证用户邮箱匹配
    const user = await UserModel.findById(userId)
    if (!user || user.email !== invitation.inviteeEmail) {
      throw new Error('用户邮箱不匹配')
    }
    
    // 检查是否已是团队成员
    const existingMember = await TeamMemberModel.findOne({
      teamId: invitation.teamId,
      userId
    })
    
    if (existingMember) {
      throw new Error('用户已是团队成员')
    }
    
    // 开始事务
    const session = await mongoose.startSession()
    session.startTransaction()
    
    try {
      // 创建团队成员记录
      await TeamMemberModel.create([{
        teamId: invitation.teamId,
        userId,
        role: invitation.role,
        orgId: invitation.orgId,
        permissions: invitation.customPermissions || [],
        status: 'active',
        joinedAt: new Date()
      }], { session })
      
      // 如果指定了组织，添加到组织成员
      if (invitation.orgId) {
        await OrgMemberModel.create([{
          teamId: invitation.teamId,
          orgId: invitation.orgId,
          userId,
          role: invitation.role,
          status: 'active'
        }], { session })
      }
      
      // 更新邀请状态
      await TeamInvitationModel.findByIdAndUpdate(invitation._id, {
        status: 'accepted',
        acceptedAt: new Date()
      }, { session })
      
      // 清理用户权限缓存
      await this.clearUserPermissionCache(userId)
      
      await session.commitTransaction()
      
    } catch (error) {
      await session.abortTransaction()
      throw error
    } finally {
      session.endSession()
    }
  }
  
  // 更新成员权限
  async updateMemberPermissions(params: {
    teamId: string
    targetUserId: string
    operatorUserId: string
    role?: SystemRoleEnum
    customPermissions?: Permission[]
  }): Promise<void> {
    
    const { teamId, targetUserId, operatorUserId, role, customPermissions } = params
    
    // 验证操作者权限
    const hasPermission = await this.checkPermission({
      userId: operatorUserId,
      resource: `team:${teamId}`,
      action: 'manage'
    })
    
    if (!hasPermission) {
      throw new Error('无权限修改成员权限')
    }
    
    // 验证目标成员存在
    const targetMember = await TeamMemberModel.findOne({
      teamId,
      userId: targetUserId,
      status: 'active'
    })
    
    if (!targetMember) {
      throw new Error('目标成员不存在')
    }
    
    // 防止自己修改自己的权限
    if (operatorUserId === targetUserId) {
      throw new Error('不能修改自己的权限')
    }
    
    // 更新权限
    const updateData: any = {}
    if (role) updateData.role = role
    if (customPermissions) updateData.permissions = customPermissions
    
    await TeamMemberModel.findByIdAndUpdate(targetMember._id, updateData)
    
    // 清理权限缓存
    await this.clearUserPermissionCache(targetUserId)
    
    // 记录权限变更日志
    await this.logPermissionChange({
      operatorUserId,
      targetUserId,
      teamId,
      action: 'update_permissions',
      changes: updateData
    })
  }
  
  // 验证邀请权限
  private async verifyInvitePermission(
    userId: string, 
    teamId: string
  ): Promise<void> {
    
    const hasPermission = await this.checkPermission({
      userId,
      resource: `team:${teamId}`,
      action: 'manage'
    })
    
    if (!hasPermission) {
      throw new Error('无权限邀请团队成员')
    }
  }
}
```

## 🔑 权限验证与控制

### 权限验证引擎

**权限检查器** (`packages/service/support/permission/controller.ts`)
```typescript
export class PermissionController {
  private cache: PermissionCache
  
  constructor() {
    this.cache = new PermissionCache()
  }
  
  // 核心权限检查方法
  async checkPermission(params: {
    userId: string
    resource: string
    action: PermissionActionEnum
    context?: PermissionContext
  }): Promise<boolean> {
    
    const { userId, resource, action, context } = params
    
    // 1. 检查缓存
    const cacheKey = this.generateCacheKey(userId, resource, action)
    const cachedResult = await this.cache.get(cacheKey)
    
    if (cachedResult !== null) {
      return cachedResult
    }
    
    // 2. 执行权限检查
    const hasPermission = await this.performPermissionCheck({
      userId,
      resource,
      action,
      context
    })
    
    // 3. 缓存结果
    await this.cache.set(cacheKey, hasPermission, 5 * 60) // 5分钟缓存
    
    return hasPermission
  }
  
  // 执行权限检查逻辑
  private async performPermissionCheck(params: {
    userId: string
    resource: string
    action: PermissionActionEnum
    context?: PermissionContext
  }): Promise<boolean> {
    
    const { userId, resource, action, context } = params
    
    // 解析资源类型和ID
    const { resourceType, resourceId } = this.parseResource(resource)
    
    // 1. 检查用户是否为资源拥有者
    if (await this.isResourceOwner(userId, resourceType, resourceId)) {
      return true
    }
    
    // 2. 检查直接权限授予
    const directPermission = await this.checkDirectPermission(
      userId, resourceType, resourceId, action
    )
    if (directPermission) {
      return true
    }
    
    // 3. 检查角色权限
    const rolePermission = await this.checkRolePermission(
      userId, resourceType, resourceId, action
    )
    if (rolePermission) {
      return true
    }
    
    // 4. 检查继承权限
    const inheritedPermission = await this.checkInheritedPermission(
      userId, resourceType, resourceId, action
    )
    if (inheritedPermission) {
      return true
    }
    
    // 5. 检查动态权限
    if (context) {
      const dynamicPermission = await this.checkDynamicPermission(
        userId, resourceType, resourceId, action, context
      )
      if (dynamicPermission) {
        return true
      }
    }
    
    // 默认拒绝访问
    return false
  }
  
  // 检查直接权限
  private async checkDirectPermission(
    userId: string,
    resourceType: string,
    resourceId: string,
    action: PermissionActionEnum
  ): Promise<boolean> {
    
    const permission = await PermissionModel.findOne({
      userId,
      resourceType,
      resourceId,
      actions: action,
      status: 'active'
    })
    
    return !!permission
  }
  
  // 检查角色权限
  private async checkRolePermission(
    userId: string,
    resourceType: string,
    resourceId: string,
    action: PermissionActionEnum
  ): Promise<boolean> {
    
    // 查找用户在相关团队中的角色
    const userRoles = await this.getUserRoles(userId, resourceType, resourceId)
    
    for (const role of userRoles) {
      const rolePermissions = RolePermissionMatrix[role.role]
      if (rolePermissions?.[resourceType as ResourceTypeEnum]?.includes(action)) {
        return true
      }
    }
    
    return false
  }
  
  // 检查继承权限
  private async checkInheritedPermission(
    userId: string,
    resourceType: string,
    resourceId: string,
    action: PermissionActionEnum
  ): Promise<boolean> {
    
    // 获取资源的父级资源
    const parentResources = await this.getParentResources(resourceType, resourceId)
    
    for (const parentResource of parentResources) {
      const hasParentPermission = await this.performPermissionCheck({
        userId,
        resource: `${parentResource.type}:${parentResource.id}`,
        action
      })
      
      if (hasParentPermission) {
        return true
      }
    }
    
    return false
  }
  
  // 检查动态权限
  private async checkDynamicPermission(
    userId: string,
    resourceType: string,
    resourceId: string,
    action: PermissionActionEnum,
    context: PermissionContext
  ): Promise<boolean> {
    
    // 获取动态权限规则
    const dynamicRules = await this.getDynamicPermissionRules(
      resourceType, resourceId
    )
    
    for (const rule of dynamicRules) {
      if (await this.evaluateRule(rule, { userId, action, context })) {
        return true
      }
    }
    
    return false
  }
  
  // 批量权限检查
  async checkBatchPermissions(params: {
    userId: string
    permissions: Array<{
      resource: string
      action: PermissionActionEnum
      context?: PermissionContext
    }>
  }): Promise<Record<string, boolean>> {
    
    const { userId, permissions } = params
    const results: Record<string, boolean> = {}
    
    // 并行检查所有权限
    const checks = permissions.map(async (perm) => {
      const key = `${perm.resource}:${perm.action}`
      const hasPermission = await this.checkPermission({
        userId,
        resource: perm.resource,
        action: perm.action,
        context: perm.context
      })
      return { key, hasPermission }
    })
    
    const checkResults = await Promise.all(checks)
    
    checkResults.forEach(({ key, hasPermission }) => {
      results[key] = hasPermission
    })
    
    return results
  }
  
  // 获取用户权限列表
  async getUserPermissions(params: {
    userId: string
    resourceType?: string
    includeInherited?: boolean
  }): Promise<UserPermissionSummary> {
    
    const { userId, resourceType, includeInherited = true } = params
    
    // 直接权限
    const directPermissions = await PermissionModel.find({
      userId,
      ...(resourceType && { resourceType }),
      status: 'active'
    })
    
    // 角色权限
    const rolePermissions = await this.getUserRolePermissions(userId, resourceType)
    
    // 继承权限
    const inheritedPermissions = includeInherited 
      ? await this.getUserInheritedPermissions(userId, resourceType)
      : []
    
    return {
      userId,
      directPermissions: directPermissions.map(this.formatPermission),
      rolePermissions: rolePermissions.map(this.formatPermission),
      inheritedPermissions: inheritedPermissions.map(this.formatPermission),
      summary: this.generatePermissionSummary([
        ...directPermissions,
        ...rolePermissions,
        ...inheritedPermissions
      ])
    }
  }
}
```

### 权限中间件

**API权限中间件** (`packages/service/support/permission/auth/common.ts`)
```typescript
export function authPermission(params: {
  resource: string | ((req: any) => string)
  action: PermissionActionEnum
  optional?: boolean
}) {
  return async (req: any, res: any, next: any) => {
    
    try {
      // 获取用户ID
      const userId = req.user?.userId
      if (!userId) {
        return res.status(401).json({
          code: 401,
          message: '未登录'
        })
      }
      
      // 解析资源
      const resource = typeof params.resource === 'function' 
        ? params.resource(req)
        : params.resource
      
      // 替换资源中的参数
      const resolvedResource = this.resolveResourceParams(resource, req)
      
      // 检查权限
      const permissionController = new PermissionController()
      const hasPermission = await permissionController.checkPermission({
        userId,
        resource: resolvedResource,
        action: params.action,
        context: {
          ip: req.ip,
          userAgent: req.headers['user-agent'],
          timestamp: new Date(),
          requestData: req.body
        }
      })
      
      if (!hasPermission) {
        if (params.optional) {
          req.hasPermission = false
          return next()
        }
        
        return res.status(403).json({
          code: 403,
          message: '权限不足'
        })
      }
      
      req.hasPermission = true
      next()
      
    } catch (error) {
      console.error('权限验证错误:', error)
      res.status(500).json({
        code: 500,
        message: '权限验证失败'
      })
    }
  }
}

// 使用示例
// GET /api/apps/:appId
app.get('/api/apps/:appId', 
  authUser(),
  authPermission({
    resource: 'app:{{appId}}',
    action: PermissionActionEnum.read
  }),
  getAppHandler
)

// POST /api/apps/:appId/update
app.post('/api/apps/:appId/update',
  authUser(),
  authPermission({
    resource: 'app:{{appId}}',
    action: PermissionActionEnum.write
  }),
  updateAppHandler
)
```

## 🔍 动态权限与条件控制

### 条件权限系统

**动态权限规则引擎** (`packages/service/support/permission/dynamicRule.ts`)
```typescript
export class DynamicPermissionEngine {
  
  // 评估权限规则
  async evaluateRule(
    rule: PermissionRule,
    context: EvaluationContext
  ): Promise<boolean> {
    
    const { userId, action, resource, requestContext } = context
    
    switch (rule.type) {
      case 'time_based':
        return this.evaluateTimeBasedRule(rule, requestContext)
        
      case 'ip_based':
        return this.evaluateIpBasedRule(rule, requestContext)
        
      case 'usage_based':
        return await this.evaluateUsageBasedRule(rule, userId)
        
      case 'custom_script':
        return await this.evaluateCustomScript(rule, context)
        
      case 'approval_required':
        return await this.evaluateApprovalRule(rule, context)
        
      default:
        return false
    }
  }
  
  // 时间基础规则
  private evaluateTimeBasedRule(
    rule: TimeBasedRule,
    context: RequestContext
  ): boolean {
    
    const now = new Date()
    const currentTime = now.getHours() * 60 + now.getMinutes()
    
    // 检查时间范围
    if (rule.timeRange) {
      const startTime = this.parseTime(rule.timeRange.start)
      const endTime = this.parseTime(rule.timeRange.end)
      
      if (currentTime < startTime || currentTime > endTime) {
        return false
      }
    }
    
    // 检查日期范围
    if (rule.dateRange) {
      const startDate = new Date(rule.dateRange.start)
      const endDate = new Date(rule.dateRange.end)
      
      if (now < startDate || now > endDate) {
        return false
      }
    }
    
    // 检查星期限制
    if (rule.weekDays && rule.weekDays.length > 0) {
      const currentWeekDay = now.getDay()
      if (!rule.weekDays.includes(currentWeekDay)) {
        return false
      }
    }
    
    return true
  }
  
  // IP基础规则
  private evaluateIpBasedRule(
    rule: IpBasedRule,
    context: RequestContext
  ): boolean {
    
    const clientIp = context.ip
    
    // 白名单检查
    if (rule.whitelist && rule.whitelist.length > 0) {
      return rule.whitelist.some(pattern => this.matchIpPattern(clientIp, pattern))
    }
    
    // 黑名单检查
    if (rule.blacklist && rule.blacklist.length > 0) {
      return !rule.blacklist.some(pattern => this.matchIpPattern(clientIp, pattern))
    }
    
    // 地理位置检查
    if (rule.geoRestriction) {
      return this.checkGeoRestriction(clientIp, rule.geoRestriction)
    }
    
    return true
  }
  
  // 使用量基础规则
  private async evaluateUsageBasedRule(
    rule: UsageBasedRule,
    userId: string
  ): Promise<boolean> {
    
    const usage = await this.getUserUsage(userId, rule.timeWindow)
    
    switch (rule.metric) {
      case 'api_calls':
        return usage.apiCalls < rule.limit
        
      case 'tokens_consumed':
        return usage.tokensConsumed < rule.limit
        
      case 'concurrent_sessions':
        return usage.concurrentSessions < rule.limit
        
      case 'storage_used':
        return usage.storageUsed < rule.limit
        
      default:
        return false
    }
  }
  
  // 自定义脚本规则
  private async evaluateCustomScript(
    rule: CustomScriptRule,
    context: EvaluationContext
  ): Promise<boolean> {
    
    try {
      // 创建安全的执行环境
      const sandbox = this.createSandbox(context)
      
      // 执行自定义脚本
      const result = await this.executeInSandbox(rule.script, sandbox)
      
      return Boolean(result)
    } catch (error) {
      console.error('自定义规则执行失败:', error)
      return false
    }
  }
  
  // 审批必需规则
  private async evaluateApprovalRule(
    rule: ApprovalRule,
    context: EvaluationContext
  ): Promise<boolean> {
    
    const { userId, resource, action } = context
    
    // 查找相关的审批记录
    const approval = await ApprovalModel.findOne({
      userId,
      resource,
      action,
      status: 'approved',
      expiresAt: { $gt: new Date() }
    })
    
    if (approval) {
      return true
    }
    
    // 如果没有审批，创建审批请求
    if (rule.autoCreateRequest) {
      await this.createApprovalRequest({
        userId,
        resource,
        action,
        reason: rule.defaultReason || '需要权限审批',
        approvers: rule.approvers
      })
    }
    
    return false
  }
  
  // IP模式匹配
  private matchIpPattern(ip: string, pattern: string): boolean {
    if (pattern.includes('/')) {
      // CIDR notation
      return this.matchCIDR(ip, pattern)
    } else if (pattern.includes('*')) {
      // Wildcard pattern
      return this.matchWildcard(ip, pattern)
    } else {
      // Exact match
      return ip === pattern
    }
  }
}
```

## 📊 权限审计与监控

### 审计日志系统

**权限操作记录** (`packages/service/support/permission/audit.ts`)
```typescript
export class PermissionAuditLogger {
  
  // 记录权限检查
  async logPermissionCheck(params: {
    userId: string
    resource: string
    action: PermissionActionEnum
    result: boolean
    context?: PermissionContext
    duration?: number
  }): Promise<void> {
    
    const { userId, resource, action, result, context, duration } = params
    
    await PermissionAuditModel.create({
      userId,
      resource,
      action,
      result,
      ip: context?.ip,
      userAgent: context?.userAgent,
      timestamp: new Date(),
      duration,
      metadata: {
        requestId: context?.requestId,
        sessionId: context?.sessionId
      }
    })
  }
  
  // 记录权限变更
  async logPermissionChange(params: {
    operatorUserId: string
    targetUserId?: string
    resource: string
    action: 'grant' | 'revoke' | 'update'
    changes: any
    reason?: string
  }): Promise<void> {
    
    const { operatorUserId, targetUserId, resource, action, changes, reason } = params
    
    await PermissionChangeLogModel.create({
      operatorUserId,
      targetUserId,
      resource,
      action,
      changes,
      reason,
      timestamp: new Date()
    })
  }
  
  // 生成权限报告
  async generatePermissionReport(params: {
    teamId?: string
    userId?: string
    startDate: Date
    endDate: Date
    resourceType?: string
  }): Promise<PermissionReport> {
    
    const { teamId, userId, startDate, endDate, resourceType } = params
    
    // 构建查询条件
    const matchConditions: any = {
      timestamp: { $gte: startDate, $lte: endDate }
    }
    
    if (teamId) matchConditions.teamId = teamId
    if (userId) matchConditions.userId = userId
    if (resourceType) matchConditions.resource = new RegExp(`^${resourceType}:`)
    
    // 聚合查询
    const aggregationPipeline = [
      { $match: matchConditions },
      {
        $group: {
          _id: {
            userId: '$userId',
            resource: '$resource',
            action: '$action'
          },
          totalChecks: { $sum: 1 },
          successfulChecks: {
            $sum: { $cond: ['$result', 1, 0] }
          },
          failedChecks: {
            $sum: { $cond: ['$result', 0, 1] }
          },
          avgDuration: { $avg: '$duration' },
          lastCheck: { $max: '$timestamp' }
        }
      },
      {
        $group: {
          _id: '$_id.userId',
          permissions: {
            $push: {
              resource: '$_id.resource',
              action: '$_id.action',
              totalChecks: '$totalChecks',
              successfulChecks: '$successfulChecks',
              failedChecks: '$failedChecks',
              avgDuration: '$avgDuration',
              lastCheck: '$lastCheck'
            }
          },
          totalPermissionChecks: { $sum: '$totalChecks' },
          totalSuccessful: { $sum: '$successfulChecks' },
          totalFailed: { $sum: '$failedChecks' }
        }
      }
    ]
    
    const results = await PermissionAuditModel.aggregate(aggregationPipeline)
    
    return {
      period: { startDate, endDate },
      summary: {
        totalUsers: results.length,
        totalChecks: results.reduce((sum, r) => sum + r.totalPermissionChecks, 0),
        successRate: this.calculateSuccessRate(results),
        mostAccessedResources: await this.getMostAccessedResources(matchConditions),
        suspiciousActivities: await this.detectSuspiciousActivities(matchConditions)
      },
      userDetails: results.map(this.formatUserPermissionReport)
    }
  }
  
  // 检测可疑活动
  private async detectSuspiciousActivities(
    matchConditions: any
  ): Promise<SuspiciousActivity[]> {
    
    const suspiciousActivities: SuspiciousActivity[] = []
    
    // 1. 检测异常频繁的权限检查
    const frequentChecks = await PermissionAuditModel.aggregate([
      { $match: matchConditions },
      {
        $group: {
          _id: { userId: '$userId', hour: { $hour: '$timestamp' } },
          count: { $sum: 1 }
        }
      },
      { $match: { count: { $gt: 1000 } } }, // 每小时超过1000次
      { $sort: { count: -1 } },
      { $limit: 10 }
    ])
    
    frequentChecks.forEach(item => {
      suspiciousActivities.push({
        type: 'frequent_permission_checks',
        userId: item._id.userId,
        severity: 'medium',
        description: `用户在一小时内进行了${item.count}次权限检查`,
        metadata: item
      })
    })
    
    // 2. 检测异常的失败率
    const highFailureRate = await PermissionAuditModel.aggregate([
      { $match: matchConditions },
      {
        $group: {
          _id: '$userId',
          total: { $sum: 1 },
          failed: { $sum: { $cond: ['$result', 0, 1] } }
        }
      },
      {
        $match: {
          total: { $gt: 50 },
          $expr: { $gt: [{ $divide: ['$failed', '$total'] }, 0.5] }
        }
      }
    ])
    
    highFailureRate.forEach(item => {
      const failureRate = (item.failed / item.total * 100).toFixed(2)
      suspiciousActivities.push({
        type: 'high_permission_failure_rate',
        userId: item._id,
        severity: 'high',
        description: `用户权限检查失败率达到${failureRate}%`,
        metadata: item
      })
    })
    
    return suspiciousActivities
  }
}
```

## 🚀 权限系统优势与展望

### 技术优势

1. **细粒度控制** - 支持资源级别的精确权限控制
2. **灵活架构** - RBAC + 动态权限的混合模型
3. **性能优化** - 多层缓存和批量检查机制
4. **审计完整** - 全面的权限操作日志记录
5. **扩展性强** - 支持自定义权限规则和条件

### 企业级特性

1. **组织管理** - 完整的企业组织架构支持
2. **协作友好** - 团队协作和权限共享机制
3. **合规支持** - 完整的审计日志和报告
4. **安全防护** - 多维度的安全控制机制
5. **易于管理** - 直观的权限管理界面

### 未来发展方向

#### 短期优化 (3-6个月)
- [ ] 增强动态权限规则引擎
- [ ] 优化权限检查性能
- [ ] 完善权限管理界面
- [ ] 增加更多审计报告类型

#### 中期规划 (6-12个月)
- [ ] 实现基于AI的权限推荐
- [ ] 支持更复杂的组织架构
- [ ] 构建权限风险评估系统
- [ ] 开发移动端权限管理

#### 长期愿景 (1-2年)
- [ ] 实现零信任安全架构
- [ ] 支持跨平台权限同步
- [ ] 构建智能化权限治理
- [ ] 建设权限即服务平台

---

FastGPT 的权限系统通过精心设计的架构和全面的功能实现，为企业级应用提供了强大而灵活的安全保障。这个系统不仅满足了当前的安全需求，更为未来的扩展和演进奠定了坚实的基础。