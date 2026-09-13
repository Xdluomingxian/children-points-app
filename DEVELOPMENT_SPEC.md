# 儿童成长积分助手 · 项目开发文档
## （AI 智能体驱动开发版 · MVP/M1 优先）

**文档版本**: v1.1（后端由「微信云开发」切换为「uniCloud」）
**创建日期**: 2026-09-13
**上游基线**: `REQUIREMENTS_SPEC.md` v1.2（需求说明书）
**文档定位**: 将需求翻译为**可直接编码执行**的工程规格：目录结构、数据模型、云对象契约、核心算法、任务拆分、验收标准。本文与需求冲突时，**以本文为实现口径**，并以本文件 §14「变更记录」沉淀差异。
**读者对象**: AI 开发智能体 / 人类开发者

---

## 0. 智能体执行须知（必读）

> 本节是给「驱动代码开发的智能体」的操作约定，决定你如何消费本文。

1. **单一事实源**：需求语义看 `REQUIREMENTS_SPEC.md`；工程实现看本文。若实现细节本文未覆盖，**以本文的设计原则（§1.3）推断**，并在代码注释与 §14 记录推断。
2. **禁止自由发挥范围**：只实现本文标注 `P0` 且归属 `M1` 的任务（§10）。`P1/P2` 不实现，只保证**架构可扩展**（预留字段/目录，不写死）。
3. **一次一个任务**：按 §10 的任务编号（T-xx）顺序推进；每个任务完成后必须：① 通过该任务「验收标准」；② 更新 §14 变更记录；③ 执行 §9.4 的提交规范。
4. **先跑通再优化**：M1 目标是「任务 → 打卡 → 积分 → 兑换」闭环可用，**不追求 UI 精美**，但必须满足 §8 的基础设计令牌（一年级可读性）。
5. **不确定即标注**：遇到歧义，选择**最简单可运行**的方案，用 `// TODO(spec):` 注释标注，并记入 §14，**不要阻塞等待**。
6. **数据写入必须留痕**：任何积分变动必须经 `points` 云对象写入 `point_records`，**禁止前端直改余额**（§7.5）。
7. **平台约束**：后端为 **uniCloud**。**禁止使用 `wx.cloud.*`**（微信云开发专有 API，跨端不可用）；统一用 `uniCloud.*` 与云对象。

---

## 1. 技术栈与总体架构

### 1.1 技术选型（Q1，2026-09-13 v1.2 修订为 uniCloud）

| 层次 | 技术 | 说明 |
|------|------|------|
| 前端框架 | **uni-app (Vue 3)** | 一套码出 微信小程序 / H5 / App |
| 构建 | Vite（uni-app CLI 版） | `@dcloudio/vite-plugin-uni` |
| 状态管理 | Pinia | 轻量、Vue3 原生 |
| 后端 | **uniCloud**（腾讯云 / 阿里云服务空间） | 云对象 + 云数据库 + 云存储 |
| 后端形态 | **云对象（Cloud Object）** | 一个业务域一个云对象，方法即接口 |
| 数据库 | 云数据库（文档型，MongoDB 语法 + JQL） | 免运维 |
| 数据权限 | **DB Schema（`.schema.json`）声明式** | 权限与校验配置化，替代隐式 openid 权限 |
| 账号体系 | **uni-id**（`uni-id-co` 云对象） | 小程序走 `loginByWeixin`；H5/App 可切手机号/一键登录 |
| 提醒 | 微信订阅消息 | 需申请模板 |
| 目标端优先级 | **微信小程序 → H5 → App** | MP-WEIXIN 为主 |

> **为什么不是微信云开发**：微信云开发天然只服务微信小程序（H5 端 Web SDK 仅未登录模式、无登录态；App 端需「多端应用 + 环境共享」），**前端能跨端而后端调用层锁死微信**，与「保留多端 / 为 App 预留」冲突。uniCloud 在同一套腾讯云 TCB 底层上做跨端统一封装，**腾讯云版价格与微信云开发相同**，全端原生可用。

### 1.2 架构分层

```
┌─────────────────────────────────────────────┐
│  uni-app 前端 (Vue 3 + Pinia)                │
│  pages ─ components ─ store ─ api ─ utils     │
└───────────────────┬─────────────────────────┘
                    │  uniCloud.importObject('xxx')
                    │  （云对象调用，全端一致）
┌───────────────────▼─────────────────────────┐
│  uniCloud 服务空间                             │
│  ┌──────────────┐  ┌────────────────┐        │
│  │ 云对象(写)     │→ │ 云数据库(读写)   │        │
│  │ checkin/points│  │ 9 集合 + Schema │        │
│  └──────────────┘  └────────────────┘        │
│  ┌──────────────┐  ┌────────────────┐        │
│  │ 定时触发器     │  │ 云存储(照片)     │        │
│  └──────────────┘  └────────────────┘        │
└─────────────────────────────────────────────┘
```

### 1.3 四条设计原则（本文未覆盖时的推断依据）

1. **写操作全部走云对象**：保证事务性、鉴权、积分留痕。**DB Schema 将前端直读权限全部关闭**（`read/write: false`），只有云函数/云对象可访问数据。
2. **数据按 `family_id` 隔离**：所有业务集合必带 `family_id`，云对象入口校验当前用户归属家庭。
3. **幂等优先**：凡是「打卡 / 发分 / 兑换」等写操作，必须能安全重试（§7.6）。
4. **孩子无独立身份**：孩子是「被观察对象」，用 `child_id` 标识；所有操作由家长账号发起。
5. **跨端不写死平台 API**：只用 `uniCloud.*` 与 uni-app 通用 API；平台差异用条件编译（`#ifdef MP-WEIXIN`）隔离，禁止在业务层散落 `wx.*` 调用。

---

## 2. 仓库结构与目录规范

### 2.1 目标目录结构（M1 需建立）

> uniCloud 的云目录**按所选服务商命名**：阿里云 `uniCloud-aliyun/`、腾讯云 `uniCloud-tencent/`。下文统称 `uniCloud-{provider}/`。

```
children-points-app/
├── uniCloud-aliyun/                 # 云服务空间目录（按服务商命名）
│   ├── cloudfunctions/
│   │   ├── common/                  # 公共模块（各云对象以 npm 依赖方式引用）
│   │   │   ├── app-common/          # 鉴权 / 错误码 / 响应 / 常量
│   │   │   │   ├── index.js
│   │   │   │   └── package.json
│   │   │   └── points-rules/        # 积分/按时判定纯函数（可单测）
│   │   │       ├── index.js
│   │   │       └── package.json
│   │   ├── uni-id-co/               # uni-id 官方账号云对象（登录/注册/改密）
│   │   ├── user/                    # 云对象：用户资料 / 订阅设置
│   │   ├── family/                  # 云对象：家庭 / 规则 / 邀请码
│   │   ├── child/                   # 云对象：孩子档案
│   │   ├── task/                    # 云对象：任务定义 + 模板库
│   │   ├── instance/                # 云对象：每日任务实例（懒生成）
│   │   ├── checkin/                 # 云对象：打卡 + 按时判定 + 触发发分 ★
│   │   ├── points/                  # 云对象：积分流水 / 余额 / 手动调整 ★
│   │   ├── reward/                  # 云对象：奖励 + 兑换
│   │   └── scheduler/               # 定时触发器云函数（每日实例生成 + 提醒）
│   └── database/                    # DB Schema（权限 + 字段校验 + 索引）
│       ├── families.schema.json
│       ├── users.schema.json
│       ├── children.schema.json
│       ├── tasks.schema.json
│       ├── task_instances.schema.json
│       ├── completions.schema.json
│       ├── point_records.schema.json
│       ├── rewards.schema.json
│       └── redemptions.schema.json
├── src/                             # uni-app 前端源码
│   ├── api/                         # 云对象调用封装（每个云对象一个模块）
│   ├── store/                       # Pinia stores
│   ├── components/                  # 通用组件
│   ├── pages/
│   │   ├── parent/                  # 家长端页面
│   │   └── child/                   # 孩子展示视图页面
│   ├── utils/                       # 工具（时间、格式化、常量）
│   ├── static/                      # 图标、图片、音效
│   ├── App.vue
│   ├── main.js                      # 挂载 Pinia（uniCloud 无需 init）
│   ├── pages.json                   # 路由 + tabBar
│   └── manifest.json                # appid / 各端配置
├── docs/                            # 文档归档（需求/开发/设计）
├── project.config.json              # 微信开发者工具配置（miniprogramRoot 指向编译产物）
├── package.json
└── .gitignore
```

### 2.2 目录与命名规范

| 项 | 规范 |
|----|------|
| 云对象目录 | 小写单词，语义单数（`checkin` 非 `checkIns`）；文件固定为 `index.obj.js` |
| 云函数目录 | 小写单词，入口固定为 `index.js` |
| 公共模块 | kebab-case，发布后以 npm 包名引用（`points-rules`） |
| 前端页面目录 | 小写，家长端 `pages/parent/*`、孩子端 `pages/child/*` |
| 组件文件 | PascalCase（`TaskCard.vue`） |
| 工具/接口文件 | camelCase（`formatDate.js`） |
| 集合名 | 下划线复数（`task_instances`），Schema 文件名同名 + `.schema.json` |
| 常量 | 大写下划线（`CHECKIN_STATUS.DONE`） |

### 2.3 `.gitignore` 要点

```
node_modules/
dist/
unpackage/
.DS_Store
uniCloud-*/cloudfunctions/common/uni-id/config.json   # 含 appsecret，勿提交
```

> ⚠️ 小程序 `appsecret` 属敏感信息，**不提交仓库**；提供 `config.example.json` 供他人复制。

---

## 3. 环境搭建与本地运行

### 3.1 前置条件

| 项 | 要求 |
|----|------|
| Node.js | ≥ 18（本项目环境：22.22.2 / 24.3.0 均可） |
| HBuilderX | 最新稳定版（用于 uniCloud 服务空间关联与云对象部署，**可直接打开 CLI 工程**） |
| 微信开发者工具 | 最新稳定版（打开小程序编译产物） |
| 微信小程序 AppID | 需注册（个人/企业均可） |
| uniCloud 服务空间 | 在 uniCloud web 控制台创建（腾讯云 / 阿里云），记下空间 ID |
| 微信小程序 appsecret | 在微信公众平台获取，用于 uni-id 微信登录 |

### 3.2 初始化步骤（智能体参照执行）

```bash
# 1) 安装依赖
npm install

# 2) 启动微信小程序编译（watch 模式）
npm run dev:mp-weixin
# 产物输出：dist/dev/mp-weixin

# 3) 打开微信开发者工具
#    导入项目根目录 → 自动读取 project.config.json
#    miniprogramRoot 指向 dist/dev/mp-weixin
```

**uniCloud 关联与部署（在 HBuilderX 中完成）**：

```
1) HBuilderX 打开项目根目录
2) 右键 uniCloud-aliyun/ → 「关联云服务空间或项目」→ 选择服务空间
3) 配置 uni-id：编辑 cloudfunctions/common/uni-id/config.json
   填入 mp-weixin: { appid, appsecret }
4) 右键 cloudfunctions/common/app-common → 「上传公共模块」
   同理上传 points-rules
5) 右键各云对象 → 「上传部署」
6) 在 uniCloud web 控制台 → 服务空间 → 「一键配置微信小程序域名」
   （自动写入 request 合法域名，避免手工配置遗漏）
```

> 📌 **uni-id 配置**：`uni-id-co` 依赖配置文件（`uni-id-co/config.json` 或公共模块 `uni-id/config.json`，视版本），必须填对 `appid` / `appsecret`，否则微信登录会失败。

### 3.3 关键 `package.json` 脚本约定

| 脚本 | 作用 |
|------|------|
| `dev:mp-weixin` | 小程序开发编译（watch） |
| `dev:h5` | H5 开发（多端验证用） |
| `build:mp-weixin` | 小程序生产构建 |
| `build:h5` | H5 生产构建 |
| `lint` | 代码检查（建议 ESLint + Prettier） |
| `test:rules` | 单测核心算法（§11） |

### 3.4 客户端无需初始化云环境

**uniCloud 前端不需要也不应调用 `wx.cloud.init()`**。云对象调用直接使用：

```js
// src/api/_request.js 内的调用方式（全端一致，无平台判断）
const checkin = uniCloud.importObject('checkin')
const res = await checkin.submit(instanceId)
```

- **登录态**：`uni-id-co` 登录成功后把 token 存入本地 `uni_id_token`；uniCloud 客户端 SDK 会自动附加该 token，云对象内通过 `this.getClientInfo().uid` 取到用户身份。
- **多端一致**：同一段调用代码在小程序 / H5 / App 均可运行，**无需条件编译**（这是选 uniCloud 的核心收益）。

---

## 4. 数据模型（云数据库集合明细）

> 共 **9 张集合**。字段类型按云数据库类型：`string / number / boolean / date / array / object`。
> **权限全部通过 DB Schema 关闭前端读写**（`read/create/update/delete: false`），仅云对象可访问 —— 这是本项目的安全基线。

### 4.1 集合定义

#### ① `families` 家庭
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | 家庭 ID |
| `name` | string | ✔ | 家庭名（如「嘟嘟的家」） |
| `invite_code` | string | ✔ | 6 位邀请码，唯一 |
| `owner_uid` | string | ✔ | 主账号 `users._id` |
| `rules_config` | object | ✔ | 见下 |
| `created_at` | date | ✔ | 创建时间 |

`rules_config` 默认值：
```js
{ max_daily_points: 50, weekend_multiplier: 1, on_time_bonus: 1, grace_minutes: 10 }
```

#### ② `users` 家长用户
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | 用户 ID |
| `uid` | string | ✔ | **uni-id 用户 ID**（唯一索引）；即 `getClientInfo().uid` |
| `openid` | string | ✖ | 微信 openid（微信端登录后冗余，便于排查） |
| `nickname` | string | ✖ | 昵称 |
| `avatar` | string | ✖ | 头像 URL |
| `family_id` | string | ✖ | 所属家庭（首次登录未建家庭时为空） |
| `role` | string | ✔ | `admin`（主） / `member`（家人） |
| `subscription` | object | ✖ | `{ task_remind: bool, remind_time: "19:00" }` |
| `created_at` | date | ✔ | |

#### ③ `children` 孩子档案
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | 孩子 ID |
| `family_id` | string | ✔ | |
| `name` | string | ✔ | 昵称（可匿名化） |
| `avatar` | string | ✖ | |
| `birthday` | date | ✖ | |
| `grade` | string | ✔ | 默认 `"一年级"` |
| `pet_id` | string | ✖ | 二期预留 |
| `points_balance` | number | ✔ | **当前余额（唯一可信源）**，默认 0 |
| `streak_days` | number | ✔ | 当前连续天数 |
| `streak_best` | number | ✔ | 历史最长 |
| `created_at` | date | ✔ | |

#### ④ `tasks` 任务定义
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | |
| `family_id` | string | ✔ | |
| `creator_id` | string | ✔ | user `_id` |
| `title` | string | ✔ | 任务名（≤6 字优先） |
| `icon` | string | ✔ | emoji 或图标 key |
| `category` | string | ✔ | `study/move/life/habit/read` |
| `points_value` | number | ✔ | 基础分（1–5） |
| `frequency` | string | ✔ | `daily` / `once` |
| `due_time` | string | ✖ | `"19:00"`，`once` 可空 |
| `grace_minutes` | number | ✖ | 覆盖家庭默认（空则用 `rules_config`） |
| `auto_approve` | boolean | ✔ | M1 恒 `true`（Q3：免审核） |
| `assign_to` | array | ✔ | `[child_id, ...]` |
| `status` | string | ✔ | `active` / `inactive` |
| `created_at` | date | ✔ | |

#### ⑤ `task_instances` 每日任务实例 ★
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | |
| `task_id` | string | ✔ | |
| `child_id` | string | ✔ | |
| `family_id` | string | ✔ | |
| `date` | string | ✔ | `"2026-09-13"`（**本地日期字符串**，非 Date） |
| `due_time` | string | ✖ | 实例化时从 task 快照 |
| `status` | string | ✔ | `todo` / `done` / `late` / `missed` |
| `submit_time` | date | ✖ | 打卡时间 |
| `completion_id` | string | ✖ | 关联 `completions._id` |

> **唯一性约束**：`(task_id, child_id, date)` 三元组唯一 → 云数据库无业务唯一索引，需**云对象内先查重**保证（§7.4）。

#### ⑥ `completions` 打卡记录
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | |
| `instance_id` | string | ✔ | 关联实例 |
| `task_id` / `child_id` / `family_id` | string | ✔ | 冗余便于查询 |
| `date` | string | ✔ | `"2026-09-13"` |
| `submit_time` | date | ✔ | |
| `on_time` | boolean | ✔ | 按时判定结果 |
| `proof_url` | string | ✖ | 云存储 fileID（P1） |
| `remark` | string | ✖ | |
| `status` | string | ✔ | M1 恒 `approved`（免审核） |
| `approver_id` | string | ✖ | M1 空 |
| `approve_time` | date | ✖ | M1 空 |
| `points_awarded` | number | ✔ | 本次实际发放积分 |
| `created_by` | string | ✔ | 操作人 user `_id`（家人也可打卡） |

#### ⑦ `point_records` 积分流水（**不可篡改**）
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | |
| `child_id` | string | ✔ | |
| `family_id` | string | ✔ | |
| `type` | string | ✔ | `earn` / `spend` / `adjust` |
| `amount` | number | ✔ | 正负均可 |
| `balance_before` | number | ✔ | 变动前余额 |
| `balance_after` | number | ✔ | 变动后余额 |
| `reason` | string | ✔ | 人类可读（如「按时完成作业」） |
| `related_id` | string | ✖ | 关联 completion / redemption `_id` |
| `created_at` | date | ✔ | |
| `created_by` | string | ✔ | 操作人 |

#### ⑧ `rewards` 奖励
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | |
| `family_id` | string | ✔ | |
| `title` | string | ✔ | |
| `icon` | string | ✔ | emoji |
| `category` | string | ✔ | `thing/privilege/experience/money/companion` |
| `points_cost` | number | ✔ | 所需积分 |
| `stock` | number | ✖ | 空 = 无限 |
| `limit_per_person` | number | ✖ | 限购（P2） |
| `status` | string | ✔ | `active` / `inactive` |
| `created_at` | date | ✔ | |

#### ⑨ `redemptions` 兑换记录
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `_id` | string | 自 | |
| `reward_id` / `child_id` / `family_id` | string | ✔ | |
| `points_spent` | number | ✔ | |
| `status` | string | ✔ | `done`（M1：直接完成） |
| `created_at` / `handled_at` | date | ✔ | |
| `handled_by` | string | ✔ | 操作人 |

### 4.2 索引建议（写在 Schema 的 `index` 字段中）

| 集合 | 索引字段 | 类型 |
|------|---------|------|
| `users` | `uid` | 唯一 |
| `users` | `family_id` | 普通 |
| `children` | `family_id` | 普通 |
| `task_instances` | `child_id + date` | 组合 |
| `task_instances` | `family_id + date` | 组合 |
| `completions` | `child_id + date` | 组合 |
| `point_records` | `child_id + created_at` | 组合 |
| `rewards` | `family_id` | 普通 |
| `families` | `invite_code` | 唯一 |

### 4.3 DB Schema 示例（`uniCloud-aliyun/database/tasks.schema.json`）

```json
{
  "bsonType": "object",
  "required": ["family_id", "title", "category", "points_value", "assign_to", "status"],
  "permission": {
    "read": false,
    "create": false,
    "update": false,
    "delete": false
  },
  "index": [
    {
      "IndexName": "family_status",
      "MgoKeySchema": {
        "MgoIndexKeys": [
          { "Name": "family_id", "Direction": "1" },
          { "Name": "status", "Direction": "1" }
        ],
        "MgoIsUnique": false
      }
    }
  ],
  "properties": {
    "_id": { "description": "ID，系统自动生成" },
    "family_id": { "bsonType": "string", "description": "所属家庭 ID" },
    "creator_id": { "bsonType": "string", "description": "创建人 users._id" },
    "title": { "bsonType": "string", "description": "任务名", "maxLength": 20 },
    "icon": { "bsonType": "string", "description": "图标 emoji 或 key" },
    "category": {
      "bsonType": "string",
      "description": "分类",
      "enum": ["study", "move", "life", "habit", "read"]
    },
    "points_value": { "bsonType": "int", "description": "基础积分", "minimum": 1, "maximum": 5 },
    "frequency": { "bsonType": "string", "enum": ["daily", "once"], "defaultValue": "daily" },
    "due_time": { "bsonType": "string", "description": "截止时间 HH:mm" },
    "grace_minutes": { "bsonType": "int", "description": "宽限期(分钟)" },
    "auto_approve": { "bsonType": "bool", "defaultValue": true },
    "assign_to": { "bsonType": "array", "description": "分配到的孩子 ID 列表" },
    "status": { "bsonType": "string", "enum": ["active", "inactive"], "defaultValue": "active" },
    "created_at": { "bsonType": "timestamp", "defaultValue": { "$env": "now" } }
  }
}
```

> **权限说明**：`read/create/update/delete` 全为 `false` 表示**客户端不可直接读写**；云对象/云函数中的数据库操作**不受 Schema 权限限制**，仍可正常读写。这正是「写操作全走云对象」的落地方式。

**其余集合**：按 §4.1 字段表同构编写，`permission` 一律四个 `false`；`users.uid` 与 `families.invite_code` 加唯一索引；`task_instances` / `completions` / `point_records` 加组合索引。

---

## 5. 云对象设计

### 5.1 统一约定

**调用方式**（前端）：一个云对象一个业务域，**方法名即接口名**，参数直接传递：

```js
// src/api/checkin.js
const checkin = uniCloud.importObject('checkin', { customUI: true })
const res = await checkin.submit(instanceId, { remark: '完成啦' })
// res 即云对象方法的 return 值（无需再解 {code,data} 包装）
```

**返回与错误**：云对象**直接 `return` 业务数据**；出错时 `throw` 带 `errCode` 的错误，客户端 `catch` 到 `{ errCode, errMsg }`。

```js
// common/app-common/errors.js
const CODE = {
  PARAM_ERROR:   'PARAM_ERROR',    // 参数缺失/非法
  UNAUTHORIZED:  'UNAUTHORIZED',   // 未登录 / 无 uid
  NO_FAMILY:     'NO_FAMILY',      // 尚未加入家庭
  NO_PERMISSION: 'NO_PERMISSION',  // 无权访问该家庭
  NOT_FOUND:     'NOT_FOUND',      // 资源不存在
  CONFLICT:      'CONFLICT',       // 冲突（重复打卡 / 重复实例）
  INTERNAL:      'INTERNAL'        // 服务内部错误
}

class BizError extends Error {
  constructor(code, message) {
    super(message)
    this.errCode = code
    this.errMsg = message
    this.name = 'BizError'
  }
}

module.exports = { CODE, BizError }
```

> 云对象内 `throw new BizError(CODE.CONFLICT, '该任务今日已打卡')`，客户端 `catch(e)` 得到 `e.errCode === 'CONFLICT'`。

**鉴权（`_before` 钩子统一处理）**：

```js
// cloudfunctions/checkin/index.obj.js
const { requireUser } = require('app-common')

module.exports = {
  async _before() {
    // uniCloud 已自动校验 uni_id_token，uid 注入 getClientInfo()
    this.ctx = await requireUser(this)   // { uid, user, familyId, role }
  },
  async submit(instanceId, extra = {}) { /* ... */ }
}
```

`requireUser(ctx)` 逻辑：取 `ctx.getClientInfo().uid` → 无则抛 `UNAUTHORIZED`；查 `users` 表（按 `uid`）→ 无则抛 `UNAUTHORIZED`；返回 `{ uid, user, familyId: user.family_id, role }`。

### 5.2 云对象清单（M1 范围）

| 云对象 | 方法 | 优先级 | 说明 |
|--------|------|--------|------|
| `uni-id-co`（官方） | `loginByWeixin` | P0 | 微信登录，返回 token + uid |
| `uni-id-co` | `updateUser` | P1 | 更新昵称/头像 |
| `user` | `getProfile` | P0 | 登录后拉取 user + family + children |
| `user` | `updateSubscription` | P1 | 订阅消息设置 |
| `family` | `create` | P0 | 创建家庭（生成 invite_code） |
| `family` | `get` | P0 | 获取家庭信息 + rules_config |
| `family` | `updateRules` | P1 | 修改积分规则 |
| `family` | `join` | P1 | 用邀请码加入家庭 |
| `child` | `list` | P0 | 家庭下孩子列表 |
| `child` | `create` | P0 | 新增孩子 |
| `child` | `update` / `remove` | P1 | 编辑/删除 |
| `task` | `templates` | P0 | 返回内置模板库（只读常量） |
| `task` | `list` | P0 | 家庭任务列表 |
| `task` | `create` | P0 | 创建任务 |
| `task` | `update` / `toggle` | P0 | 编辑 / 启停 |
| `instance` | `ensureToday` | P0 | **懒生成**今日实例（见 §7.4） |
| `instance` | `listByDate` | P0 | 查某日实例（默认今天） |
| `checkin` | `submit` | P0 | 打卡 + 按时判定 + 触发发分 ★ |
| `checkin` | `undo` | P1 | 撤销打卡（回滚积分） |
| `points` | `balance` | P0 | 查孩子余额 + 简要统计 |
| `points` | `records` | P0 | 积分明细分页 |
| `points` | `adjust` | P1 | 手动加/减分 |
| `reward` | `list` | P0 | 奖励列表 |
| `reward` | `create` / `update` | P0 | 建/改奖励 |
| `reward` | `redeem` | P0 | 兑换（扣分 + 写流水）★ |
| `scheduler` | （定时触发） | P1 | 每日生成实例 + 提醒（M2） |

### 5.3 核心接口契约（M1 必做的三个）

#### ① `user` / `getProfile`
```js
// 前端：const user = uniCloud.importObject('user'); await user.getProfile()
// 入参：无（身份来自 token）
// 出参：
{
  user: { _id, nickname, avatar, family_id, role },
  family: { _id, name, rules_config } | null,   // null = 需引导创建家庭
  children: [ { _id, name, avatar, points_balance, streak_days } ]
}
```

#### ② `instance` / `ensureToday`
```js
// 入参：ensureToday(childId, date?)      // date 默认今天
// 逻辑：查 tasks(status=active, assign_to 含 childId)
//       × 该日 → 缺失则批量插入 task_instances（status=todo）
//       → 已存在的 overdue 实例标记 missed（见 §7.4）
// 出参：
{ date: "2026-09-13",
  instances: [ { _id, task_id, title, icon, category, due_time, status, points_value } ] }
```

#### ③ `checkin` / `submit` ★
```js
// 入参：submit(instanceId, { remark?, proofUrl? })
// 逻辑（在同一云对象方法内串行执行，保证一致性）
//  1) 校验归属；查实例，status 非 todo → 抛 CONFLICT（幂等保护）
//  2) submitTime = now；比较 due_time + grace → on_time
//  3) 写 completions（status=approved）
//  4) 更新 instance: status = on_time ? done : late, submitTime, completionId
//  5) 调用积分引擎算分 → 写 point_records + 更新 children.points_balance（事务）
//  6) 更新 streak
// 出参：
{
  onTime: true, pointsAwarded: 4,
  detail: { base: 3, onTimeBonus: 1, streakBonus: 0, multiplier: 1 },
  balanceAfter: 27, streakDays: 5
}
```

---

## 6. 前端架构

### 6.1 页面清单（pages.json）

```
pages/parent/home/index        家长·首页（今日概览 + 快捷打卡）      P0
pages/parent/task/list         任务列表                            P0
pages/parent/task/edit         创建/编辑任务                       P0
pages/parent/task/templates    模板库                              P0
pages/parent/reward/list       奖励管理                            P0
pages/parent/reward/edit       创建/编辑奖励                       P0
pages/parent/me/index          我的（孩子/家人/规则）               P0
pages/child/home/index         孩子·今日任务（大卡片）              P0
pages/child/points/index       孩子·我的积分 + 可兑换               P0

（P1，M2 再做）
pages/parent/report/index      报表（日历 + 趋势）
pages/parent/family/join       加入家庭（邀请码）
```

**tabBar（家长端默认）**：首页 / 任务 / 奖励 / 我的

### 6.2 组件清单（`src/components/`）

| 组件 | 用途 | 优先级 |
|------|------|--------|
| `TaskCard.vue` | 任务卡片（图标+标题+截止+状态） | P0 |
| `CheckButton.vue` | 打卡按钮（含 loading/成功动画钩子） | P0 |
| `PointsBadge.vue` | 积分展示（家长/孩子双样式） | P0 |
| `ChildSwitcher.vue` | 多孩子切换 | P0 |
| `EmptyState.vue` | 空态 | P0 |
| `PointsFlyAnimation.vue` | 加分飞入动画 | P0 |
| `CategoryTag.vue` | 任务分类标签 | P1 |

### 6.3 Pinia Store 设计

| store | 状态 | 关键 action |
|-------|------|-------------|
| `userStore` | `user, family, children, currentChildId` | `login()`, `getProfile()`, `switchChild()`, `ensureFamily()` |
| `taskStore` | `todayInstancesByChild, tasks, templates` | `loadToday(childId)`, `loadTasks()`, `createTask()` |
| `pointsStore` | `balance, records` | `loadBalance(childId)`, `loadRecords()` |
| `rewardStore` | `rewards` | `loadRewards()`, `redeem()` |

### 6.4 API 层封装（`src/api/`）

```js
// src/api/_request.js —— 统一云对象调用 + 错误处理
export function co(name) {
  // customUI: true 关闭 uniCloud 默认错误弹窗，由我们统一提示
  return uniCloud.importObject(name, { customUI: true })
}

export async function call(name, method, ...args) {
  try {
    return await co(name)[method](...args)
  } catch (e) {
    uni.showToast({ title: e.errMsg || '出错了', icon: 'none' })
    throw e
  }
}

// src/api/checkin.js
import { call } from './_request'
export const checkinApi = {
  submit: (instanceId, extra = {}) => call('checkin', 'submit', instanceId, extra),
  undo:   (completionId)           => call('checkin', 'undo', completionId)
}
```

> 每个云对象对应 `src/api/<name>.js`，**页面不直接调用 `uniCloud.importObject`**。

---

## 7. 核心业务逻辑实现细则 ★

> 本章是智能体最容易实现错的点，给出**纯函数**与**伪代码**，便于单测（§11）。

### 7.1 按时判定（`common/points-rules/index.js`）

```js
/**
 * 判定是否按时
 * @param {string|Date} submitTime 打卡时间
 * @param {string} dueTime     "19:00"
 * @param {string} date        "2026-09-13"  实例所属日期
 * @param {number} graceMinutes 宽限期（默认 10）
 * @returns {boolean}
 */
function isOnTime(submitTime, dueTime, date, graceMinutes = 10) {
  if (!dueTime) return true                      // 无截止时间视为按时
  const due = new Date(`${date}T${dueTime}:00`)  // 按本地时区解释
  due.setMinutes(due.getMinutes() + graceMinutes)
  return new Date(submitTime).getTime() <= due.getTime()
}
```

**边界规则**：
- 跨天补录（今天补昨天的）→ 该实例 `date` 为昨天，用昨天的 `date` 计算，且**标记为超时**（补录一律按是否超出 `due+grace` 判，通常为 late）。
- 无 `due_time` 的任务（如「主动阅读」）→ 恒 `on_time = true`。

### 7.2 积分计算引擎（`common/points-rules/index.js`）

```js
/**
 * @returns {{ total:number, detail:{base,multiplier,onTimeBonus,streakBonus,capped} }}
 */
function calcPoints({ base, isOnTime, date, streakDaysAfter, rules, earnedToday }) {
  const multiplier  = isWeekend(date) ? (rules.weekend_multiplier || 1) : 1
  const onTimeBonus = isOnTime ? (rules.on_time_bonus || 0) : 0
  const streakBonus = streakBonusFor(streakDaysAfter)   // 7→10 / 14→15 / 30→70，否则 0
  let raw = base * multiplier + onTimeBonus + streakBonus
  const limit = rules.max_daily_points || 50
  const remain = Math.max(0, limit - earnedToday)
  const capped = Math.min(raw, remain)
  return { total: capped, detail: { base, multiplier, onTimeBonus, streakBonus, capped: raw > capped } }
}

function streakBonusFor(days) {
  if (days === 7)  return 10
  if (days === 14) return 15
  if (days === 30) return 70
  return 0
}
```

> ⚠️ **不含随机分**（需求 §5.2 明确去掉）。`earnedToday` = 该孩子当日已发放积分合计（查 `point_records`）。

### 7.3 连续打卡 streak 计算

```
规则（需求 §5.3）：按「孩子整体」统计；中断归零，保留 best。
实现（在 checkin 成功后）：
  取该孩子最近一次完成的 date（即上一次 done/late 的实例）
  若 lastDate == 昨天  → streak_days += 1
  若 lastDate == 今天  → 不变（同日多次打卡不重复累加）
  否则                 → streak_days = 1（中断重置为本次）
  streak_best = max(streak_best, streak_days)
```

> ⚠️ **streak_days 基于「当天是否有任意任务完成」判断，而非单个任务**（M1 简化口径，§14 记录）。

### 7.4 每日任务实例生成（`instance.ensureToday`）

**M1 采用「懒生成」**（避免依赖定时器，降低首次可用门槛）：

```
ensureToday(childId, date = 今天):
  1) 校验归属
  2) 查 tasks: { family_id, status:'active', assign_to: childId 包含 }
  3) 查已存在的 task_instances: { child_id, date }
  4) 差集 → 批量插入新实例（due_time 从 task 快照，status='todo'）
  5) 【补漏】查该孩子 date < 今天 且 status='todo' 的实例
     → 批量更新为 'missed'
  6) 返回当日全部实例（含 task 的 title/icon/category/points_value 冗余字段，减少前端联查）
```

**并发保护**：步骤 3–4 之间可能并发重复插入 → 用「先查后插 + 插入前再查一次」双检；若仍重复，`completions` 以 `instance_id` 幂等（§7.6）。
**定时兜底**：M2 增加 `scheduler` 云函数 + uniCloud **定时触发器**（每日 00:05）批量生成，作为懒生成的补充。

### 7.5 积分写入唯一通道

```
任何积分变动 → 必须经 points 逻辑：
  1) 开启数据库事务：const transaction = await db.startTransaction()
  2) 读 children.points_balance（before）
  3) 插入 point_records（含 balance_before/after）
  4) 更新 children.points_balance = after
  5) await transaction.commit()
禁止：前端直改 balance；禁止只更新余额不写流水。
```

> uniCloud 事务用法：`db.startTransaction()` / `transaction.collection().doc().update()` / `transaction.commit()` / `transaction.rollback()`。**事务方法名与普通方法不同**（用 `transaction.collection` 而非 `db.collection`）。

### 7.6 幂等与并发（必做）

| 场景 | 幂等策略 |
|------|---------|
| 重复打卡 | `instance.status !== 'todo'` → 直接抛 `CONFLICT` 或返回上次结果 |
| 重复生成实例 | `(task_id, child_id, date)` 先查后插；重复则跳过 |
| 重复兑换 | 以 `redemptions` 唯一键 `(reward_id, child_id, 分钟级时间戳)` 或前端 requestId 去重 |
| 弱网重试 | 云对象写操作基于「状态机」判断（status 转换），天然幂等 |

---

## 8. UI 设计规范（一年级可读性）

### 8.1 设计令牌（`src/utils/theme.js`）

```js
export const theme = {
  color: {
    primary:   '#FF9500',   // 温暖橙（主）
    secondary: '#5AC8FA',   // 天空蓝（辅）
    success:   '#34C759',   // 完成
    late:      '#FF3B30',   // 超时/未完成
    text:      '#1C1C1E',
    textSub:   '#8E8E93',
    bg:        '#FFF9F0',   // 暖底
    card:      '#FFFFFF'
  },
  radius: { lg: '24rpx', xl: '32rpx', pill: '999rpx' },
  font:   { xs: '24rpx', sm: '28rpx', md: '32rpx', lg: '40rpx', xl: '56rpx' },
  space:  { s: '16rpx', m: '24rpx', l: '32rpx' }
}
```

### 8.2 页面通用规则

| 规则 | 要求 |
|------|------|
| 按钮 | 高度 ≥ 88rpx，圆角，主色填充 |
| 孩子视图 | 卡片大图标（emoji ≥ 80rpx 字号），文字 ≤ 6 字 |
| 打卡反馈 | 即时动画（`PointsFlyAnimation`）+ 可选音效，**先出动画再落库** |
| 家长视图 | 信息密度可高，但核心操作 ≤ 3 步 |
| 分类配色 | 学习蓝 / 运动绿 / 生活橙 / 习惯紫 / 阅读青 |

### 8.3 状态色约定

| 状态 | 颜色 | 文案 |
|------|------|------|
| `todo` | 灰 | 待完成 |
| `done` | 绿 `#34C759` | 已完成 ✅ |
| `late` | 橙红 `#FF3B30` | 已完成（超时） |
| `missed` | 灰红 | 未完成 |

---

## 9. 编码规范与 Git 工作流

### 9.1 代码规范

- ESLint（`eslint:recommended` + `plugin:vue/vue3-recommended`）+ Prettier（单引号、无分号、2 空格）。
- 组件用 `<script setup>`；组合式 API。
- **云对象**用 CommonJS，导出**方法对象**；公共逻辑抽到 `common/` 公共模块并以 npm 依赖引用。
- 所有中文文案集中在 `src/utils/i18n.js`（为后续多端/多语言预留），**页面不硬编码中文**。
- **禁止** `wx.cloud.*`；平台专有 API 一律条件编译隔离。

### 9.2 云对象开发约定

```js
// uniCloud-aliyun/cloudfunctions/checkin/index.obj.js
const { requireUser } = require('app-common')
const { BizError, CODE } = require('app-common')
const rules = require('points-rules')

module.exports = {
  async _before() {
    this.ctx = await requireUser(this)     // { uid, user, familyId, role }
  },
  async submit(instanceId, extra = {}) {
    const { familyId } = this.ctx
    if (!instanceId) throw new BizError(CODE.PARAM_ERROR, '缺少 instanceId')
    // ...业务逻辑
    return { onTime, pointsAwarded, balanceAfter, streakDays }
  }
}
```

**公共模块引用**：在云对象的 `package.json` 中声明依赖，例如：
```json
{
  "name": "checkin",
  "dependencies": {
    "app-common": "file:../common/app-common",
    "points-rules": "file:../common/points-rules"
  }
}
```

### 9.3 分支与提交

| 项 | 规范 |
|----|------|
| 分支 | `main` 保护；功能用 `feat/<taskId>-<slug>`，如 `feat/T05-checkin` |
| 提交信息 | `feat(T05): 实现打卡提交与按时判定` / `fix(T05): 修正宽限期边界` |
| 提交粒度 | **每个任务（T-xx）一次提交**，便于回溯与验收 |

### 9.4 每次提交前检查清单

- [ ] `npm run lint` 无错误
- [ ] `npm run test:rules` 通过（若涉及 §7 纯函数）
- [ ] 该任务「验收标准」已自测通过
- [ ] §14 变更记录已更新（若有实现口径调整）
- [ ] 无硬编码密钥（`appsecret` 等）、无 `console.log` 残留
- [ ] 全局搜索确认**无 `wx.cloud` 残留**

---

## 10. 开发任务拆分（M1 · 智能体可逐条认领）

> 每任务含：**目标 / 交付物 / 依赖 / 验收标准**。
> 建议按 T01→T16 顺序执行；`依赖` 为空的先做。
> 状态标记：`[ ]` 待做 / `[~]` 进行中 / `[x]` 完成

### 阶段一：工程底座（T01–T03）

| ID | 任务 | 交付物 | 依赖 | 验收标准 |
|----|------|--------|------|---------|
| `[ ]` T01 | 初始化 uni-app 工程 | `package.json` / `src/` 骨架 / `pages.json` / `manifest.json` | — | `npm run dev:mp-weixin` 编译成功，开发者工具可打开首页 |
| `[ ]` T02 | uniCloud 服务空间接入 | `uniCloud-aliyun/` 目录、HBuilderX 关联服务空间 | T01 | 服务空间关联成功；可上传并调用一个 demo 云对象 |
| `[ ]` T03 | 建立 9 张集合 + Schema + 索引 | `uniCloud-aliyun/database/*.schema.json` | T02 | 9 个 Schema 上传成功；客户端直读被拒（权限为 false）；索引生效 |
| `[ ]` T03b | 接入 uni-id | `uni-id-co` 上传、`uni-id/config.json` 填 appid/appsecret | T03 | 小程序端 `loginByWeixin` 返回 token 与 uid |

### 阶段二：账号与家庭（T04–T06）

| ID | 任务 | 交付物 | 依赖 | 验收标准 |
|----|------|--------|------|---------|
| `[ ]` T04 | 公共模块 `app-common` / `points-rules` | `cloudfunctions/common/*`，含 `requireUser`、错误码 | T03b | 上传成功；云对象可 `require` 且鉴权生效（未登录抛 UNAUTHORIZED） |
| `[ ]` T05 | 登录 + `user` 云对象 + 前端登录态 | `cloudfunctions/user`、`api/user.js`、`store/userStore` | T04 | 首次登录建 user（按 uid）；二次登录复用；`family=null` 时前端进入建家庭引导 |
| `[ ]` T06 | `family` / `child` 云对象 + 页面 | `cloudfunctions/family`、`cloudfunctions/child`、对应页面 | T05 | 3 步内完成「建家庭 → 加孩子 → 进入首页」 |

### 阶段三：任务与实例（T07–T09）

| ID | 任务 | 交付物 | 依赖 | 验收标准 |
|----|------|--------|------|---------|
| `[ ]` T07 | 任务模板库 + `task` 云对象 | 模板常量、`task` 的 `templates/list/create/update/toggle` | T06 | 可一键启用模板；可创建自定义任务（含 due_time） |
| `[ ]` T08 | 任务管理页面 | `pages/parent/task/*` | T07 | 列表/创建/编辑/启停均可用；必填校验生效 |
| `[ ]` T09 | `instance.ensureToday` + 今日列表 | `cloudfunctions/instance`、`pages/parent/home` | T08 | 打开首页自动生成当天实例；历史 todo 自动转 missed |

### 阶段四：打卡与积分（T10–T12）★核心

| ID | 任务 | 交付物 | 依赖 | 验收标准 |
|----|------|--------|------|---------|
| `[ ]` T10 | 纯函数 + 单测 | `common/points-rules/index.js`、`test/rules.test.js` | T04 | `isOnTime`（含宽限/边界）、`calcPoints`（含上限/连奖）单测全绿 |
| `[ ]` T11 | `checkin.submit` 全链路 | `cloudfunctions/checkin`、`api/checkin.js` | T09,T10 | 打卡 ≤3 步；按时判定正确；积分即时到账；流水/余额/实例状态一致；重复打卡幂等 |
| `[ ]` T12 | `points` 余额与明细 | `cloudfunctions/points`、`pages/child/points` | T11 | 余额实时刷新；明细含每笔原因/时间；家长可见 |

### 阶段五：奖励兑换与孩子视图（T13–T14）

| ID | 任务 | 交付物 | 依赖 | 验收标准 |
|----|------|--------|------|---------|
| `[ ]` T13 | `reward` 奖励 + 兑换 | `cloudfunctions/reward`、`pages/parent/reward/*` | T12 | 建奖励；积分足够可兑换；兑换扣分写流水；余额不足有明确提示 |
| `[ ]` T14 | 孩子展示视图 | `pages/child/home`、`ChildSwitcher`、加分动画 | T12 | 家长一键切换到孩子视图（免登录）；大卡片+一键完成+积分动画；文字 ≤6 字 |

### 阶段六：提醒与验收（T15–T16）

| ID | 任务 | 交付物 | 依赖 | 验收标准 |
|----|------|--------|------|---------|
| `[ ]` T15 | 订阅消息（P1，M1 可选） | `user.updateSubscription`、`scheduler` + 定时触发器 | T06 | 家长可授权；能推送「今日任务未完成」 |
| `[ ]` T16 | M1 端到端验收 | 验收清单回归 | T01–T14 | 新家长 3 分钟内跑通「建家庭→建任务→打卡得积分→兑换」；核心指标见 §10.1 |

### 10.1 M1 验收总纲（DoD）

- [ ] 完整闭环：创建家庭 → 添加孩子 → 启用模板任务 → 打**到今天**首页待办 → 点击打卡 → 积分即时到账并出现动画 → 到奖励页兑换成功且余额扣减
- [ ] 按时判定：截止前打卡记「已完成」，超时记「已完成（超时）」，隔日未做记「未完成」
- [ ] 积分可解释：每笔变动可在明细中看到原因与余额变化，**无随机分**
- [ ] 幂等：重复点击打卡不重复发分；重复兑换不重复扣分
- [ ] **多端可构建**：`build:mp-weixin` 与 `build:h5` 均成功；**代码中无 `wx.cloud` 残留**
- [ ] 合规：无定位/人脸/通讯录权限申请

---

## 11. 测试策略

| 层级 | 范围 | 方式 |
|------|------|------|
| 单元测试 | §7 纯函数（`isOnTime` / `calcPoints` / `streakBonusFor`） | Jest / Vitest，`npm run test:rules` |
| 云对象测试 | 每个云对象的 `_before` 与业务方法 | HBuilderX「运行云对象本地调试」或 uniCloud web 控制台「云函数测试」 |
| 端到端 | M1 闭环 | 真机 + 开发者工具回归 §10.1 |

**必测用例（纯函数）**：

```
isOnTime:
  - 18:59 打卡 / due 19:00 / grace 10  → true
  - 19:05 打卡 / due 19:00 / grace 10  → true （宽限内）
  - 19:11 打卡 / due 19:00 / grace 10  → false
  - 无 due_time                        → true
calcPoints:
  - base 3 + 按时 + 无连奖 + 平日        → 4
  - base 3 + 按时 + 周末(×2) + 连奖7     → 3*2+1+10 = 17
  - 当日已得 48 / limit 50 / raw 17      → capped = 2
```

---

## 12. 发布与部署

### 12.1 部署顺序

```
1) 公共模块：HBuilderX 右键 cloudfunctions/common/app-common、points-rules → 上传
2) 云对象：逐个「上传部署」（uni-id-co 需先配置 config.json）
3) DB Schema：右键 database 目录 → 「上传所有 DB Schema」
4) 定时触发器（M2）：uniCloud web 控制台 → 云函数 → scheduler → 配置 cron
5) 小程序：上传体验版 → 提交审核 → 发布
6) 首次运行前：在 uniCloud web 控制台「一键配置微信小程序域名」
```

### 12.2 多端扩展路径（uniCloud 让这条路径变成「零重构」）

| 目标端 | 动作 |
|--------|------|
| H5 | `npm run build:h5` 直接出包；**云对象调用代码无需改动** |
| App | uni-app 云打包（iOS/Android）；**登录方式**可从微信改为手机号/一键登录（uni-id 原生支持），云对象与数据库**完全复用** |
| 其他小程序 | 加编译目标即可（uniCloud 全端支持），业务代码免改 |

> 这正是选择 uniCloud 的核心收益：**换端不改后端调用层**。

---

## 13. 与需求文档的映射（可追溯性）

| 需求编号 | 实现位置 |
|---------|---------|
| FR-101~105 账号家庭 | §3.2 uni-id、§5.2 `user/family`、T05/T06 |
| FR-201~208 任务模板 | §5.2 `task`、§15 内置模板、T07/T08 |
| FR-301~306 打卡按时 | §7.1、§7.4、`checkin`、T11 |
| FR-401~403 免审核 | §4.1 `completions.status` 恒 `approved`、§5.3 |
| FR-501~506 积分 | §7.2、§7.5、`points`、T10/T12 |
| FR-601~605 奖励兑换 | §5.2 `reward`、§7.6、T13 |
| FR-701~705 统计 | M2（`report` 页面） |
| FR-801~804 提醒 | T15（P1） |
| FR-901~903 游戏化 | M3（二期） |

---

## 14. 变更记录（实现口径调整 Leave-off）

> 智能体在实现中发现本文未覆盖或需调整之处，**必须在此登记**，格式：
> `日期 | 任务ID | 变更点 | 原因`

| 日期 | 任务 ID | 变更点 | 原因 |
|------|--------|--------|------|
| 2026-09-13 | — | 初始版本 | 依据 REQUIREMENTS_SPEC v1.1 生成 |
| 2026-09-13 | 全局 | 后端由**微信云开发**切换为 **uniCloud** | 微信云开发后端调用层锁死微信端，与「保留多端 / 为 App 预留」冲突（详见 REQUIREMENTS_SPEC §7.1） |
| 2026-09-13 | 全局 | 云函数（`{action,payload}` 单函数路由）改为**云对象（方法即接口）** | uniCloud 推荐形态，调用更直观、类型更清晰 |
| 2026-09-13 | 全局 | 数据权限由「控制台手工配置」改为 **DB Schema 声明式**（`permission` 全 false） | 权限与校验配置化，随代码一起版本管理 |
| 2026-09-13 | §4 数据模型 | `users` 主键关联由 `openid` 改为 **`uid`**（uni-id 用户 ID），`openid` 降级为可选冗余 | 接入 uni-id 统一账号体系，支持未来手机号/一键登录 |
| 2026-09-13 | T03 | 拆出 **T03b 接入 uni-id** | 微信登录依赖 uni-id 配置，需独立验证 |
| 2026-09-13 | 全局 | `streak_days` 采用「当天任意任务完成」口径 | M1 简化，按任务分类统计留 M2 |

### 待技术验证项（承接需求 §7.5）

1. `task_instances` 生成策略：M1 用**懒生成**（T09）；M2 加 `scheduler` + 定时触发器兜底。
2. 订阅消息模板申请与频次（T15 前需在公众平台申请模板 ID）。
3. 积分并发原子性：统一经 `db.startTransaction()`（§7.5），T11 需覆盖并发用例。
4. 服务商与额度：腾讯云 / 阿里云服务空间选择，确认免费额度足够（家庭自用量级极小）。

---

## 15. 附录：内置任务模板库常量（`src/utils/taskTemplates.js`）

```js
export const TASK_TEMPLATES = [
  { key: 'homework',  title: '完成作业',   icon: '📘', category: 'study', points: 3, due: '19:00' },
  { key: 'writing',   title: '写字练习',   icon: '✍️', category: 'study', points: 2, due: '20:00' },
  { key: 'reading',   title: '阅读20分钟', icon: '📖', category: 'read',  points: 2, due: '20:30' },
  { key: 'rope',      title: '跳绳100个',  icon: '🤸', category: 'move',  points: 2, due: '18:30' },
  { key: 'bag',       title: '整理书包',   icon: '🎒', category: 'life',  points: 1, due: '20:00' },
  { key: 'brush',     title: '早晚刷牙',   icon: '🪥', category: 'life',  points: 1, due: '21:00' },
  { key: 'sleep',     title: '9点前睡觉',  icon: '😴', category: 'habit', points: 2, due: '21:00' },
  { key: 'extraRead', title: '主动阅读',   icon: '⭐', category: 'habit', points: 1, due: null  }
]
```

> 工具函数建议：`src/utils/date.js` 提供 `today()`（返回 `YYYY-MM-DD` 本地日期）、`isWeekend(date)`、`formatTime(date)`。

---

**文档结束**

*本文为 M1 开发执行基线。智能体按 §10 任务序推进，每个任务完成即提交并更新 §14。*
