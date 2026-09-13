# 儿童成长积分助手 - 技术开发需求文档

**文档版本**: v1.0  
**创建日期**: 2026-08-29  
**目标读者**: 开发团队、产品经理、UI设计师

---

## 一、技术架构设计

### 1.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              客户端层 (Client Layer)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │  微信小程序      │  │     H5页面      │  │   管理后台      │            │
│  │  (uni-app)      │  │  (uni-app)      │  │   (Vue.js)      │            │
│  │                 │  │                 │  │                 │            │
│  │  - 孩子端界面   │  │  - 分享页面     │  │  - 数据统计     │            │
│  │  - 家长管理端   │  │  - 登录页面     │  │  - 系统配置     │            │
│  │  - 任务管理     │  │                 │  │                 │            │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘            │
│           │                    │                    │                      │
│           └────────────────────┼────────────────────┘                      │
│                                │                                           │
└────────────────────────────────┼───────────────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              服务端层 (Server Layer)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     微信云开发 (Cloud Base)                          │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │   │
│  │  │   云函数    │  │  云数据库   │  │   云存储    │  │ 云调用    │ │   │
│  │  │  (Node.js)  │  │ (MongoDB)   │  │   (Files)   │  │  (AI)     │ │   │
│  │  │             │  │             │  │             │  │           │ │   │
│  │  │ - 用户认证  │  │ - 用户表    │  │ - 头像图片  │  │ - AI记账  │ │   │
│  │  │ - 任务逻辑  │  │ - 任务表    │  │ - 宠物图片  │  │ - 语音识别│ │   │
│  │  │ - 积分计算  │  │ - 积分表    │  │ - 成就徽章  │  │ - 数据分析│ │   │
│  │  │ - 通知推送  │  │ - 奖励表    │  │             │  │           │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘ │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              第三方服务 (Third-party)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │  微信登录   │  │  AI大模型   │  │  消息推送   │  │  统计分析        │  │
│  │   API       │  │   API       │  │   服务      │  │  (可选)          │  │
│  │             │  │             │  │             │  │                  │  │
│  │  - openid   │  │  - 通义千问 │  │  - 订阅消息 │  │  - Google        │  │
│  │  - 用户信息 │  │  - 文心一言 │  │  - 模板消息 │  │    Analytics     │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 技术选型详情

| 层级 | 技术 | 版本 | 选择理由 |
|------|------|------|---------|
| **前端框架** | uni-app | 3.0+ | 一套代码多端运行，生态成熟 |
| **UI框架** | Vue 3 | 3.4+ | 响应式、Composition API、性能好 |
| **状态管理** | Pinia | latest | uni-app官方推荐，比Vuex更简单 |
| **UI组件库** | TDesign MiniProgram | latest | 腾讯官方，设计规范，质量可靠 |
| **后端服务** | 微信云开发 | - | 免运维，快速上手，与微信生态集成好 |
| **数据库** | 云数据库 MongoDB | - | 灵活的数据模型，适合复杂业务 |
| **云函数** | Node.js | 16+ | 云开发原生支持，性能稳定 |
| **文件存储** | 云存储 | - | 图片、文件存储，CDN加速 |
| **AI服务** | 通义千问/文心一言 | API | 自然语言处理，智能记账 |
| **消息推送** | 微信订阅消息 | - | 官方消息推送，触达用户 |

### 1.3 目录结构规划

```
children-points-app/
├── miniprogram/                    # 小程序代码
│   ├── pages/                      # 页面
│   │   ├── index/                  # 首页
│   │   │   ├── index.vue
│   │   │   └── index.json
│   │   ├── tasks/                  # 任务页
│   │   │   ├── list.vue
│   │   │   ├── create.vue
│   │   │   └── detail.vue
│   │   ├── points/                 # 积分页
│   │   │   └── index.vue
│   │   ├── shop/                   # 商城页
│   │   │   ├── list.vue
│   │   │   └── detail.vue
│   │   ├── pet/                    # 宠物页
│   │   │   └── index.vue
│   │   ├── achievements/           # 成就页
│   │   │   └── index.vue
│   │   ├── analytics/              # 统计页
│   │   │   └── index.vue
│   │   ├── family/                 # 家庭页
│   │   │   ├── manage.vue
│   │   │   └── members.vue
│   │   ├── profile/                # 个人中心
│   │   │   └── index.vue
│   │   └── login/                  # 登录页
│   │       └── index.vue
│   │
│   ├── components/                 # 公共组件
│   │   ├── task-card/             # 任务卡片
│   │   ├── point-badge/           # 积分徽章
│   │   ├── pet-display/           # 宠物展示
│   │   ├── achievement-badge/     # 成就徽章
│   │   └── stat-chart/            # 统计图表
│   │
│   ├── utils/                      # 工具函数
│   │   ├── auth.js                # 认证相关
│   │   ├── storage.js             # 本地存储
│   │   ├── request.js             # 网络请求
│   │   ├── ai.js                  # AI相关
│   │   └── format.js              # 格式化工具
│   │
│   ├── stores/                     # Pinia状态管理
│   │   ├── user.js                # 用户状态
│   │   ├── family.js              # 家庭状态
│   │   └── points.js              # 积分状态
│   │
│   ├── api/                        # API接口
│   │   ├── auth.js
│   │   ├── task.js
│   │   ├── points.js
│   │   ├── reward.js
│   │   ├── pet.js
│   │   ├── achievement.js
│   │   └── analytics.js
│   │
│   ├── static/                     # 静态资源
│   │   ├── images/                # 图片资源
│   │   ├── icons/                 # 图标
│   │   └── fonts/                 # 字体
│   │
│   ├── app.vue                     # 应用入口
│   ├── app.js                      # 应用配置
│   ├── app.json                    # 配置文件
│   ├── project.config.json       # 项目配置
│   └── sitemap.json              # 站点地图
│
├── cloudfunctions/                 # 云函数
│   ├── login/                      # 登录函数
│   │   ├── index.js
│   │   └── package.json
│   ├── tasks/                      # 任务函数
│   │   ├── create.js
│   │   ├── list.js
│   │   ├── update.js
│   │   ├── delete.js
│   │   └── approve.js
│   ├── points/                     # 积分函数
│   │   ├── calculate.js
│   │   ├── record.js
│   │   └── balance.js
│   ├── rewards/                    # 奖励函数
│   │   ├── create.js
│   │   ├── list.js
│   │   └── redeem.js
│   ├── pets/                       # 宠物函数
│   │   ├── create.js
│   │   ├── feed.js
│   │   └── evolve.js
│   ├── achievements/               # 成就函数
│   │   ├── list.js
│   │   └── unlock.js
│   ├── analytics/                  # 统计函数
│   │   ├── overview.js
│   │   └── report.js
│   └── ai/                         # AI函数
│       ├── record.js
│       └── query.js
│
├── src/                            # 管理后台 (Vue.js)
│   ├── views/
│   ├── components/
│   ├── router/
│   ├── store/
│   └── api/
│
├── docs/                           # 文档
│   ├── PRD.md
│   ├── API.md
│   └── DB_SCHEMA.md
│
├── package.json
├── vite.config.js
└── README.md
```

---

## 二、数据库设计

### 2.1 ER关系图

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│    users     │──────│   families   │──────│family_members│
│   (用户)     │      │   (家庭)     │      │  (家庭成员)   │
└──────┬───────┘      └──────┬───────┘      └──────┬───────┘
       │                     │                     │
       │                     │                     │
       ▼                     ▼                     ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│    tasks     │◄─────│  point_...   │──────│    pets      │
│   (任务)     │      │    records   │      │   (宠物)     │
└──────┬───────┘      └──────────────┘      └──────┬───────┘
       │                                           │
       │                     ┌──────────────┐      │
       └────────────────────►│  rewards     │◄─────┘
                             │ (奖励商品)   │
                             └──────────────┘
                                   │
                                   ▼
                             ┌──────────────┐
                             │ redemptions  │
                             │  (兑换记录)   │
                             └──────────────┘

┌──────────────┐      ┌──────────────┐
│ achievements │◄─────│ member_...   │
│   (成就定义) │      │ achievements │
└──────────────┘      │  (成员成就)   │
                      └──────────────┘
```

### 2.2 数据表详细设计

#### 2.2.1 用户表 (users)

```javascript
{
  _id: String,              // 用户ID (自动)
  openid: String,           // 微信openid
  unionid: String,          // 微信unionid (可选)
  phone: String,            // 手机号 (加密存储)
  nickname: String,         // 昵称
  avatar: String,           // 头像URL
  role: String,             // 角色: parent/ kid
  family_id: String,        // 所属家庭ID
  status: String,           // 状态: active/inactive
  created_at: Date,         // 创建时间
  updated_at: Date,         // 更新时间
  last_login: Date          // 最后登录时间
}

// 索引
{ openid: 1 },
{ phone: 1 },
{ family_id: 1 }
```

#### 2.2.2 家庭表 (families)

```javascript
{
  _id: String,              // 家庭ID
  name: String,             // 家庭名称
  invite_code: String,      // 邀请码 (唯一)
  owner_id: String,         // 创建者ID
  rules_config: {           // 家庭规则配置
    max_daily_points: Number,      // 每日积分上限
    weekend_multiplier: Number,    // 周末积分倍数
    birthday_multiplier: Number,   // 生日积分倍数
    reset_type: String,            // 重置类型: none/month/semester/year
    auto_approve: Boolean,         // 是否自动审批
    notification_enabled: Boolean  // 是否开启通知
  },
  theme: String,            // 主题设置
  created_at: Date,
  updated_at: Date
}

// 索引
{ invite_code: 1 },
{ owner_id: 1 }
```

#### 2.2.3 家庭成员表 (family_members)

```javascript
{
  _id: String,
  family_id: String,
  user_id: String,
  role: String,           // admin/parent/kid
  child_profile: {        // 儿童档案
    name: String,         // 姓名
    age: Number,          // 年龄
    birthday: Date,       // 生日
    avatar: String,       // 头像
    favorite_pet: String, // 喜欢的宠物类型
    grade: String         // 年级
  },
  permissions: {          // 权限配置
    can_create_tasks: Boolean,
    can_approve: Boolean,
    can_manage_rewards: Boolean,
    can_view_analytics: Boolean
  },
  created_at: Date,
  updated_at: Date
}

// 索引
{ family_id: 1 },
{ user_id: 1 }
```

#### 2.2.4 任务表 (tasks)

```javascript
{
  _id: String,
  family_id: String,
  creator_id: String,
  title: String,            // 任务标题
  description: String,      // 任务描述
  category: String,         // 分类: chore/habit/study/sport/social/creative
  subcategory: String,      // 子分类
  points_value: Number,     // 积分值
  max_points: Number,       // 最大积分 (可浮动)
  frequency: String,        // 频率: daily/weekly/once/custom
  schedule: {               // 时间安排
    days_of_week: Array,    // 星期几
    time_range: Object,     // 时间范围
    deadline: Date          // 截止时间
  },
  auto_approve: Boolean,    // 是否自动审批
  assign_to: [String],      // 分配给哪些孩子(user_id数组)
  status: String,           // active/inactive/archived
  completion_count: Number, // 完成次数
  created_at: Date,
  updated_at: Date
}

// 索引
{ family_id: 1, status: 1 },
{ creator_id: 1 },
{ assign_to: 1 }
```

#### 2.2.5 任务完成记录表 (task_completions)

```javascript
{
  _id: String,
  task_id: String,
  member_id: String,
  family_id: String,
  status: String,           // pending/approved/rejected
  actual_points: Number,    // 实际获得积分
  submit_time: Date,        // 提交时间
  approve_time: Date,       // 审批时间
  reject_reason: String,    // 拒绝原因
  remark: String,           // 备注
  proof: String,            //  proof图片URL
  created_at: Date
}

// 索引
{ task_id: 1, member_id: 1, status: 1 },
{ member_id: 1, submit_time: -1 },
{ family_id: 1, status: 1 }
```

#### 2.2.6 积分流水表 (point_records)

```javascript
{
  _id: String,
  member_id: String,
  family_id: String,
  type: String,             // earn/spend/adjust
  amount: Number,           // 积分数 (正数为获得，负数为消耗)
  related_type: String,     // 关联类型: task/reward/manual/achievement/pet
  related_id: String,       // 关联ID
  description: String,      // 描述
  balance_before: Number,   // 变动前余额
  balance_after: Number,    // 变动后余额
  created_at: Date
}

// 索引
{ member_id: 1, created_at: -1 },
{ family_id: 1, created_at: -1 }
```

#### 2.2.7 奖励商品表 (rewards)

```javascript
{
  _id: String,
  family_id: String,
  creator_id: String,
  title: String,
  description: String,
  category: String,         // toy/snack/privilege/experience/money/digital
  points_cost: Number,      // 所需积分
  stock: Number,            // 库存 (-1表示无限)
  limit_per_person: Number, // 每人限购数量
  cooldown_days: Number,    // 冷却时间
  is_featured: Boolean,     // 是否精选
  image: String,            // 商品图片
  status: String,           // active/inactive/sold_out
  created_at: Date,
  updated_at: Date
}

// 索引
{ family_id: 1, status: 1 },
{ creator_id: 1 }
```

#### 2.2.8 兑换记录表 (redemptions)

```javascript
{
  _id: String,
  reward_id: String,
  member_id: String,
  family_id: String,
  points_spent: Number,
  status: String,           // pending/approved/completed/cancelled
  create_time: Date,
  approve_time: Date,
  completion_time: Date,
  remark: String,
  created_at: Date
}

// 索引
{ member_id: 1, status: 1 },
{ reward_id: 1, family_id: 1 }
```

#### 2.2.9 宠物表 (pets)

```javascript
{
  _id: String,
  member_id: String,
  family_id: String,
  type: String,             // cat/dog/fox/rabbit/dragon/unicorn/panda/penguin
  level: Number,            // 等级 (1-50)
  exp: Number,              // 当前经验
  mood: Number,             // 心情值 (0-100)
  last_feed_time: Date,     // 最后喂食时间
  outfit: String,           // 当前装扮
  is_unlocked: Boolean,     // 是否已解锁
  created_at: Date,
  updated_at: Date
}

// 索引
{ member_id: 1 },
{ family_id: 1 }
```

#### 2.2.10 成就定义表 (achievements)

```javascript
{
  _id: String,
  family_id: String,
  title: String,
  description: String,
  icon: String,             // 徽章图标
  category: String,         // labor/habit/study/sport/social/special
  condition_type: String,   // task_count/points_total/streak/days_active/special
  condition_value: Number,  // 条件值
  points_reward: Number,    // 解锁奖励积分
  is_hidden: Boolean,       // 是否隐藏
  sort_order: Number,       // 排序
  created_at: Date
}

// 索引
{ family_id: 1, category: 1 }
```

#### 2.2.11 成员成就表 (member_achievements)

```javascript
{
  _id: String,
  member_id: String,
  achievement_id: String,
  unlocked_at: Date,
  created_at: Date
}

// 索引
{ member_id: 1 },
{ achievement_id: 1 }
```

---

## 三、API接口设计

### 3.1 API规范

| 项目 | 规范 |
|------|------|
| **基础URL** | `wxml://cloud://` (云函数调用) |
| **请求格式** | JSON |
| **响应格式** | JSON |
| **认证方式** | 微信登录态 (自动传递) |
| **错误码** | 统一错误码规范 |

### 3.2 核心API列表

#### 3.2.1 用户认证接口

```javascript
// 登录
POST /cloudfunctions/login/index
{
  "code": "wx_login_code"  // 微信登录code
}
Response:
{
  "success": true,
  "data": {
    "user_id": "xxx",
    "openid": "xxx",
    "token": "xxx",
    "profile": {...}
  }
}

// 绑定手机号
POST /cloudfunctions/auth/bind-phone
{
  "phone": "13800138000",
  "code": "sms_code"  // 短信验证码
}

// 获取用户信息
GET /cloudfunctions/user/info
Response:
{
  "success": true,
  "data": {
    "user_id": "xxx",
    "nickname": "xxx",
    "avatar": "xxx",
    "role": "parent",
    "family_id": "xxx"
  }
}
```

#### 3.2.2 任务管理接口

```javascript
// 创建任务
POST /cloudfunctions/tasks/create
{
  "title": "整理房间",
  "description": "保持房间整洁",
  "category": "chore",
  "points_value": 2,
  "frequency": "daily",
  "assign_to": ["user_id1", "user_id2"]
}

// 获取任务列表
GET /cloudfunctions/tasks/list
{
  "family_id": "xxx",
  "status": "active",
  "category": "all",
  "page": 1,
  "page_size": 20
}

// 更新任务
PUT /cloudfunctions/tasks/update
{
  "task_id": "xxx",
  "title": "新标题",
  "points_value": 3
}

// 删除任务
DELETE /cloudfunctions/tasks/delete
{
  "task_id": "xxx"
}

// 提交任务完成
POST /cloudfunctions/tasks/submit
{
  "task_id": "xxx",
  "proof": "image_url"  // 可选
}

// 审批任务
POST /cloudfunctions/tasks/approve
{
  "completion_id": "xxx",
  "status": "approved",  // approved/rejected
  "actual_points": 2,    // 实际积分
  "remark": "做得很好！"
}
```

#### 3.2.3 积分接口

```javascript
// 查询积分余额
GET /cloudfunctions/points/balance
{
  "member_id": "xxx"
}
Response:
{
  "success": true,
  "data": {
    "balance": 120,
    "today_earned": 15,
    "today_spent": 5,
    "monthly_total": 320
  }
}

// 获取积分历史
GET /cloudfunctions/points/records
{
  "member_id": "xxx",
  "start_date": "2026-08-01",
  "end_date": "2026-08-29",
  "type": "all",  // earn/spend/all
  "page": 1,
  "page_size": 20
}

// 更新积分配置
PUT /cloudfunctions/points/config
{
  "family_id": "xxx",
  "max_daily_points": 50,
  "weekend_multiplier": 2,
  "birthday_multiplier": 3,
  "reset_type": "month"
}
```

#### 3.2.4 奖励商城接口

```javascript
// 创建奖励商品
POST /cloudfunctions/rewards/create
{
  "title": "玩具车",
  "description": "遥控赛车一辆",
  "category": "toy",
  "points_cost": 50,
  "stock": 10,
  "image": "url"
}

// 获取奖励列表
GET /cloudfunctions/rewards/list
{
  "family_id": "xxx",
  "category": "all",
  "is_featured": false,
  "page": 1,
  "page_size": 20
}

// 兑换奖励
POST /cloudfunctions/rewards/redeem
{
  "reward_id": "xxx",
  "member_id": "xxx"
}
Response:
{
  "success": true,
  "data": {
    "redemption_id": "xxx",
    "points_deducted": 50,
    "remaining_balance": 70
  }
}

// 审批兑换
POST /cloudfunctions/rewards/approve
{
  "redemption_id": "xxx",
  "status": "approved"
}
```

#### 3.2.5 宠物接口

```javascript
// 创建宠物
POST /cloudfunctions/pets/create
{
  "member_id": "xxx",
  "type": "cat"
}

// 获取宠物信息
GET /cloudfunctions/pets/info
{
  "member_id": "xxx"
}

// 喂食宠物
POST /cloudfunctions/pets/feed
{
  "pet_id": "xxx",
  "points_cost": 10
}
Response:
{
  "success": true,
  "data": {
    "mood": 85,
    "exp": 15,
    "level": 2
  }
}

// 宠物进化
POST /cloudfunctions/pets/evolve
{
  "pet_id": "xxx"
}
```

#### 3.2.6 成就接口

```javascript
// 获取成就列表
GET /cloudfunctions/achievements/list
{
  "family_id": "xxx",
  "category": "all"
}

// 获取已解锁成就
GET /cloudfunctions/achievements/unlocked
{
  "member_id": "xxx"
}

// 检查并解锁成就
POST /cloudfunctions/achievements/check
{
  "member_id": "xxx",
  "trigger_event": "task_completed"
}
```

#### 3.2.7 数据分析接口

```javascript
// 获取概览数据
GET /cloudfunctions/analytics/overview
{
  "family_id": "xxx",
  "period": "week"  // day/week/month
}
Response:
{
  "total_tasks": 156,
  "completed_tasks": 135,
  "completion_rate": 86.5,
  "total_points": 2340,
  "pending_approval": 3
}

// 获取趋势数据
GET /cloudfunctions/analytics/trends
{
  "family_id": "xxx",
  "type": "points",  // points/tasks/achievements
  "days": 30
}

// 生成报告
POST /cloudfunctions/analytics/report
{
  "family_id": "xxx",
  "type": "weekly",
  "start_date": "2026-08-25",
  "end_date": "2026-08-29"
}
```

#### 3.2.8 智能记账接口

```javascript
// AI智能记账
POST /cloudfunctions/ai/record
{
  "text": "今天孩子主动整理了书桌，还帮妈妈刷了碗",
  "member_id": "xxx"  // 可选，未指定则让AI判断
}
Response:
{
  "success": true,
  "data": {
    "records": [
      {
        "behavior": "整理书桌",
        "points": 1,
        "category": "chore"
      },
      {
        "behavior": "帮妈妈刷碗",
        "points": 1,
        "category": "chore"
      }
    ],
    "total_points": 2,
    "member": "小明"
  }
}

// 积分查询
POST /cloudfunctions/ai/query
{
  "text": "我有多少积分？",
  "member_id": "xxx"
}
Response:
{
  "success": true,
  "data": {
    "answer": "你有120积分哦！本周获得了45分~",
    "points": 120,
    "trend": "+45"
  }
}
```

---

## 四、核心业务逻辑

### 4.1 积分计算逻辑

```javascript
// 积分计算核心逻辑
function calculatePoints(task, completion, context) {
  let points = task.points_value;
  
  // 1. 基础积分
  if (points <= 0) return 0;
  
  // 2. 周末双倍
  const today = new Date();
  if (today.getDay() === 0 || today.getDay() === 6) {
    points *= context.weekend_multiplier || 2;
  }
  
  // 3. 生日三倍
  if (isBirthday(completion.member_id, today)) {
    points *= context.birthday_multiplier || 3;
  }
  
  // 4. 连续打卡奖励
  const streak = getStreak(completion.member_id, task.category);
  if (streak >= 7) points += 10;
  if (streak >= 14) points += 15;
  if (streak >= 30) points += 70;
  
  // 5. 每日积分上限
  const dailyTotal = getDailyPoints(completion.member_id, today);
  if (dailyTotal + points > context.max_daily_points) {
    points = context.max_daily_points - dailyTotal;
  }
  
  // 6. 随机奖励 (10%概率额外+1~5分)
  if (Math.random() < 0.1) {
    points += Math.floor(Math.random() * 5) + 1;
  }
  
  return Math.max(0, points);
}
```

### 4.2 宠物进化逻辑

```javascript
// 宠物进化核心逻辑
const PET_LEVELS = {
  1: { name: '蛋宝宝', exp_needed: 0 },
  2: { name: '破壳啦', exp_needed: 10 },
  3: { name: '成长中', exp_needed: 50 },
  4: { name: '进化了', exp_needed: 100 },
  5: { name: '完全体', exp_needed: 200 }
};

function evolvePet(pet, newExp) {
  let newLevel = pet.level;
  
  // 检查是否可以升级
  for (let level = 5; level >= 1; level--) {
    if (newExp >= PET_LEVELS[level].exp_needed && level > pet.level) {
      newLevel = level;
      break;
    }
  }
  
  // 触发进化通知
  if (newLevel > pet.level) {
    sendNotification({
      type: 'pet_evolve',
      member_id: pet.member_id,
      pet_type: pet.type,
      new_level: newLevel,
      new_name: PET_LEVELS[newLevel].name
    });
  }
  
  return {
    level: newLevel,
    exp: newExp,
    mood: Math.min(100, pet.mood + 5)
  };
}
```

### 4.3 成就解锁逻辑

```javascript
// 成就检查核心逻辑
async function checkAchievements(member_id, event_type, event_data) {
  const achievements = await db.collection('achievements')
    .where({ family_id: event_data.family_id })
    .get();
  
  const results = [];
  
  for (const achievement of achievements.data) {
    let unlocked = false;
    
    switch (achievement.condition_type) {
      case 'task_count':
        const count = await getTaskCount(member_id, achievement.category);
        unlocked = count >= achievement.condition_value;
        break;
      
      case 'points_total':
        const total = await getTotalPoints(member_id);
        unlocked = total >= achievement.condition_value;
        break;
      
      case 'streak':
        const streak = await getStreak(member_id, achievement.category);
        unlocked = streak >= achievement.condition_value;
        break;
      
      case 'days_active':
        const days = await getActiveDays(member_id);
        unlocked = days >= achievement.condition_value;
        break;
      
      case 'special':
        unlocked = checkSpecialCondition(achievement, event_data);
        break;
    }
    
    if (unlocked) {
      await unlockAchievement(member_id, achievement._id);
      results.push({
        achievement: achievement,
        unlocked: true
      });
      
      // 发放积分奖励
      if (achievement.points_reward > 0) {
        await addPoints(member_id, achievement.points_reward, 
          `成就解锁: ${achievement.title}`);
      }
    }
  }
  
  return results;
}
```

---

## 五、页面结构设计

### 5.1 小程序页面结构

```
app.json
├── pages/
│   ├── index/              # 首页 (孩子端)
│   │   ├── index.vue       # 主页面
│   │   └── index.json
│   ├── tasks/              # 任务相关
│   │   ├── list.vue        # 任务列表
│   │   ├── create.vue      # 创建任务
│   │   └── detail.vue      # 任务详情
│   ├── points/             # 积分相关
│   │   └── index.vue       # 积分页面
│   ├── shop/               # 商城
│   │   ├── list.vue        # 商品列表
│   │   └── detail.vue      # 商品详情
│   ├── pet/                # 宠物
│   │   └── index.vue       # 宠物页面
│   ├── achievements/       # 成就
│   │   └── index.vue       # 成就墙
│   ├── analytics/          # 数据统计
│   │   └── index.vue       # 统计页面
│   ├── family/             # 家庭管理
│   │   ├── manage.vue      # 家庭管理
│   │   └── members.vue     # 成员列表
│   ├── profile/            # 个人中心
│   │   └── index.vue       # 个人资料
│   └── login/              # 登录
│       └── index.vue       # 登录页面
│
├── components/             # 公共组件
│   ├── task-card/         # 任务卡片
│   ├── point-badge/       # 积分徽章
│   ├── pet-display/       # 宠物展示
│   ├── achievement-badge/ # 成就徽章
│   └── stat-chart/        # 统计图表
│
├── tabs/                   # 底部导航
│   ├── index.vue          # 首页Tab
│   ├── tasks.vue          # 任务Tab
│   ├── shop.vue           # 商城Tab
│   ├── pet.vue            # 宠物Tab
│   └── profile.vue        # 我的Tab
│
└── utils/                  # 工具函数
    ├── auth.js            # 认证
    ├── storage.js         # 本地存储
    ├── request.js         # 网络请求
    ├── ai.js              # AI功能
    └── format.js          # 格式化工具
```

### 5.2 页面流程图

```
                    ┌─────────────┐
                    │   启动页    │
                    │ (Logo动画)  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   登录页    │
                    │ (微信登录)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼────────┐ ┌─▼──────────┐
       │   家长端     │ │  孩子端   │ │  未绑定家庭 │
       │  (管理后台)  │ │ (任务主页)│ │   引导创建  │
       └──────┬──────┘ └────┬──────┘ └────────────┘
              │             │
    ┌─────────┼─────────────┼─────────┐
    │         │             │         │
┌───▼───┐ ┌──▼────┐   ┌───▼───┐ ┌───▼────┐
│任务管理│ │数据统计│   │今日任务│ │积分商城│
│审批   │ │分析   │   │查看   │ │兑换   │
└────────┘ └───────┘   └───────┘ └────────┘
    │
    ▼
┌──────────┐
│ 家族管理  │
│ 成员设置  │
│ 规则配置  │
└──────────┘
```

---

## 六、开发任务分解

### 6.1 Phase 1: 项目初始化 (Week 1-2)

#### 6.1.1 环境搭建
- [ ] 创建uni-app项目
- [ ] 配置TDesign组件库
- [ ] 搭建云开发环境
- [ ] 配置微信开发者工具
- [ ] 初始化Git仓库

#### 6.1.2 基础架构
- [ ] 设计数据库Schema
- [ ] 创建基础云函数
- [ ] 实现用户认证模块
- [ ] 搭建网络请求封装
- [ ] 配置状态管理(Pinia)

#### 6.1.3 UI框架
- [ ] 设计全局样式变量
- [ ] 创建基础组件库
- [ ] 搭建底部导航栏
- [ ] 实现路由配置
- [ ] 配置页面跳转

### 6.2 Phase 2: 核心功能 (Week 3-4)

#### 6.2.1 任务管理
- [ ] 任务列表页面
- [ ] 任务创建页面
- [ ] 任务审批流程
- [ ] 任务完成提交
- [ ] 任务分类管理

#### 6.2.2 积分系统
- [ ] 积分计算逻辑
- [ ] 积分历史记录
- [ ] 积分余额查询
- [ ] 积分配置管理

#### 6.2.3 奖励商城
- [ ] 商品列表页面
- [ ] 商品详情页
- [ ] 兑换流程
- [ ] 库存管理

### 6.3 Phase 3: 游戏化功能 (Week 5-6)

#### 6.3.1 宠物系统
- [ ] 宠物创建与展示
- [ ] 喂食互动功能
- [ ] 进化逻辑实现
- [ ] 装扮系统
- [ ] 心情值计算

#### 6.3.2 成就系统
- [ ] 成就定义管理
- [ ] 成就解锁检测
- [ ] 成就墙展示
- [ ] 隐藏成就设计
- [ ] 成就奖励发放

### 6.4 Phase 4: 智能功能 (Week 7-8)

#### 6.4.1 AI智能记账
- [ ] 对接大模型API
- [ ] 自然语言解析
- [ ] 积分自动计算
- [ ] 历史记录查询
- [ ] 语音输入支持

#### 6.4.2 数据分析
- [ ] 统计图表开发
- [ ] 趋势分析功能
- [ ] 报告生成功能
- [ ] 数据可视化

### 6.5 Phase 5: 完善优化 (Week 9-10)

#### 6.5.1 家庭管理
- [ ] 家庭创建与邀请
- [ ] 成员权限管理
- [ ] 家庭设置配置
- [ ] 多孩子切换

#### 6.5.2 消息通知
- [ ] 订阅消息配置
- [ ] 任务提醒
- [ ] 积分变动通知
- [ ] 审批提醒

#### 6.5.3 性能优化
- [ ] 图片压缩优化
- [ ] 列表虚拟滚动
- [ ] 请求缓存策略
- [ ] 包体积优化

### 6.6 Phase 6: 测试上线 (Week 11-12)

#### 6.6.1 测试
- [ ] 单元测试
- [ ] 集成测试
- [ ] 性能测试
- [ ] 安全测试
- [ ] 兼容性测试

#### 6.6.2 上线
- [ ] 提交审核
- [ ] 修复问题
- [ ] 正式发布
- [ ] 监控部署

---

## 七、开发规范

### 7.1 代码规范

```javascript
// 命名规范
// 文件命名: kebab-case (如: task-list.vue)
// 组件命名: PascalCase (如: TaskCard.vue)
// 变量命名: camelCase (如: taskList)
// 常量命名: UPPER_SNAKE_CASE (如: MAX_POINTS)

// 注释规范
// 单行注释: // 注释内容
// 多行注释: /* 注释内容 */
// JSDoc注释: /** 参数说明 */

// 代码风格
// 使用ESLint + Prettier
// 缩进: 2空格
// 引号: 单引号
// 分号: 必须
```

### 7.2 Git工作流

```bash
# 分支策略
main          # 生产环境
develop       # 开发环境
feature/***   # 功能分支
bugfix/***    # 修复分支
release/***   # 发布分支

# 提交规范
feat: 新功能
fix: 修复bug
docs: 文档更新
style: 代码格式
refactor: 重构
test: 测试
chore: 构建/工具

# 示例
git commit -m "feat: 添加宠物喂食功能"
git commit -m "fix: 修复积分计算精度问题"
```

### 7.3 测试规范

```javascript
// 单元测试 (Jest)
describe('积分计算', () => {
  it('应该正确计算周末双倍积分', () => {
    // test code
  });
});

// 集成测试
describe('任务审批流程', () => {
  it('应该完成完整的审批流程', async () => {
    // test code
  });
});
```

---

## 八、部署方案

### 8.1 开发环境

```bash
# 本地开发
npm run dev:mp-weixin

# 预览
npm run build:mp-weixin
# 在微信开发者工具中打开 dist/dev/mp-weixin
```

### 8.2 测试环境

```bash
# 测试构建
npm run build:mp-weixin:test

# 上传测试版本
# 使用微信开发者工具的上传功能
```

### 8.3 生产环境

```bash
# 生产构建
npm run build:mp-weixin:prod

# 上传代码
# 使用微信开发者工具的上传功能

# 提交审核
# 在微信公众平台提交审核
```

### 8.4 CI/CD

```yaml
# .github/workflows/mini-program.yml
name: Mini Program CI/CD

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '16'
      - run: npm install
      - run: npm run build:mp-weixin:prod
      - name: Upload to WeChat
        run: |
          # 使用微信CLI上传代码
```

---

## 九、附录

### 9.1 依赖清单

| 依赖 | 版本 | 用途 |
|------|------|------|
| uni-app | ^3.0.0 | 前端框架 |
| Vue | ^3.4.0 | UI框架 |
| Pinia | ^2.1.0 | 状态管理 |
| TDesign MiniProgram | ^1.0.0 | UI组件库 |
| dayjs | ^1.11.0 | 日期处理 |
| lodash | ^4.17.0 | 工具函数 |

### 9.2 参考资源

- 微信小程序官方文档: https://developers.weixin.qq.com/miniprogram/dev/framework/
- uni-app文档: https://uniapp.dcloud.net.cn/
- TDesign文档: https://tdesign.tencent.com/miniprogram/overview
- 微信云开发文档: https://developers.weixin.qq.com/miniprogram/dev/wxcloud/basis/getting-started.html

### 9.3 注意事项

1. **数据安全**: 所有敏感数据（手机号、积分）需要加密存储
2. **权限控制**: 严格区分家长和孩子权限
3. **儿童隐私**: 遵守《儿童个人信息网络保护规定》
4. **性能优化**: 注意图片压缩和请求缓存
5. **兼容性**: 测试不同微信版本和机型

---

**文档结束**

*如有疑问请联系项目负责人*
