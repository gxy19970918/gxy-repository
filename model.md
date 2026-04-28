# ChatGXY AI 项目知识库（model.md）

> **项目路径**: `/Users/gxy/Downloads/gitee项目/chat-gpt-website`  
> **创建日期**: 2026-04-28  
> **最后更新**: 2026-04-29  
> **说明**: 本文档记录所有与模型接入、计费、防绕过相关的运维问题、根因分析及新增功能。后续任何修改都必须补充到本文档。

---

# 核心前提：数据库初始化（必须先做）

> ⚠️ **警告**：本项目**无法**在没有数据库的情况下启动。部署时**必须**最先完成这一步。

> **说明**：运行项目前，需要先完成以下步骤：
> 1. 启动 MySQL 服务
> 2. 创建数据库 `chatgpt_website`
> 3. 执行下方完整建表语句（按顺序执行）
> 4. `app.py` 可以自动兼容旧表升级，但首次部署时建议手工建表确保权限正确

### 步骤 1：创建数据库

```sql
CREATE DATABASE IF NOT EXISTS chatgpt_website
    DEFAULT CHARACTER SET utf8mb4
    DEFAULT COLLATE utf8mb4_unicode_ci;

USE chatgpt_website;
```

### 步骤 2：建表（按此顺序执行，因为外键依赖关系）

#### 表 1：users（用户表）——**最核心，必须先建**

```sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user',
    can_chat TINYINT(1) DEFAULT 1,
    billing_enabled TINYINT(1) DEFAULT 0,
    balance DECIMAL(12, 4) DEFAULT 0.00,
    proxy_api_key VARCHAR(100) DEFAULT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

> `proxy_api_key` 为空时，`init_db()` 会自动为每个用户生成唯一的 `sk-gxy-<uuid>`。

#### 表 2：shared_models（共享模型表）——管理员配置，计费用户使用

```sql
CREATE TABLE IF NOT EXISTS shared_models (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    api_url VARCHAR(500) NOT NULL,
    api_key VARCHAR(500) NOT NULL,
    model_type VARCHAR(50) DEFAULT 'openai',
    is_default TINYINT(1) DEFAULT 0,
    is_system TINYINT(1) DEFAULT 0 COMMENT '系统内置不可删除',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_shared_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 表 3：models（普通用户自管模型表）

```sql
CREATE TABLE IF NOT EXISTS models (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    name VARCHAR(50) NOT NULL,
    api_url VARCHAR(500) NOT NULL,
    api_key VARCHAR(500) NOT NULL,
    is_default TINYINT(1) DEFAULT 0,
    model_type VARCHAR(50) DEFAULT 'openai',
    price_input DECIMAL(12, 6) DEFAULT NULL COMMENT '输入价格/百万tokens(元)',
    price_output DECIMAL(12, 6) DEFAULT NULL COMMENT '输出价格/百万tokens(元)',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uk_user_model_name (user_id, name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 表 4：sessions（聊天会话持久化表）

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    session_key VARCHAR(100) NOT NULL,
    title VARCHAR(200) DEFAULT '新会话',
    messages_json LONGTEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uk_user_session (user_id, session_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 表 5：token_usage（Token 使用统计表）——所有人都有

```sql
CREATE TABLE IF NOT EXISTS token_usage (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    model_name VARCHAR(50) NOT NULL,
    prompt_tokens INT DEFAULT 0 COMMENT '输入token数',
    completion_tokens INT DEFAULT 0 COMMENT '输出token数',
    total_tokens INT DEFAULT 0,
    cost DECIMAL(12, 6) DEFAULT 0.000000 COMMENT '本次花费(元)',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_model (user_id, model_name),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 表 6：billing_records（消费明细表）——计费用户专属

```sql
CREATE TABLE IF NOT EXISTS billing_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    model_name VARCHAR(50) NOT NULL,
    prompt_tokens INT DEFAULT 0,
    completion_tokens INT DEFAULT 0,
    total_tokens INT DEFAULT 0,
    cost DECIMAL(12, 6) DEFAULT 0 COMMENT '本次扣费(元)',
    balance_after DECIMAL(12, 4) DEFAULT 0 COMMENT '扣费后余额',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user (user_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 表 7：recharges（充值记录表）——计费用户专属

```sql
CREATE TABLE IF NOT EXISTS recharges (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    amount DECIMAL(12, 4) NOT NULL COMMENT '充值金额(元)',
    operator_id INT NOT NULL COMMENT '操作员(admin)ID',
    remark VARCHAR(200) DEFAULT '',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (operator_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 步骤 3：插入初始数据（可选，`app.py` 会自动完成）

#### 插入默认管理员账号

```sql
INSERT INTO users (username, password_hash, role, can_chat)
VALUES ('admin', 'pbkdf2:sha256:260000$...', 'admin', 1);
```

> 实际密码需要用 Python 生成：`generate_password_hash("admin", method="pbkdf2:sha256")`
> 或者直接运行 `python app.py` 自动创建（数据库为空且无 admin 时自动插入）。

#### 插入默认共享模型（计费用户使用）

```sql
INSERT INTO shared_models (name, api_url, api_key, model_type, is_default, is_system)
VALUES
('glm5', 'http://10.99.134.35:50300/v1/chat/completions', 'sk-xxxx', 'openai', 1, 1),
('kimi25', 'http://10.99.134.35:50300/v1/chat/completions', 'sk-xxxx', 'openai', 0, 1);
```

### 各表用途与依赖关系

```
users（核心，所有表都依赖它）
  ├─ models        ← 外键 users.id（ON DELETE CASCADE）
  ├─ sessions      ← 外键 users.id
  ├─ token_usage   ← 外键 users.id
  ├─ billing_records ← 外键 users.id
  └─ recharges     ← 外键 users.id + operator_id

shared_models（独立表，无用户外键，管理员统一管理）
```

### 表 vs 功能对应关系

| 表名 | 必须建？ | 说明 |
|------|---------|------|
| `users` | ✅ 必须 | 用户登录、权限、计费开关、余额、代理 Key |
| `models` | ✅ 必须 | 普通用户自管模型（至少管理员要有一个） |
| `shared_models` | ✅ 必须 | 计费用户共享模型 |
| `sessions` | ✅ 必须 | 聊天会话持久化 |
| `token_usage` | ⚠️ 强烈建议 | Token 统计，不建则聊天正常但无统计 |
| `billing_records` | ❌ 可选 | 计费功能消费明细，不开计费则不需要 |
| `recharges` | ❌ 可选 | 充值记录，不开计费则不需要 |

---

# 启停项目

```bash
python app.py
```

- 端口：`50001`
- 修改 Python 代码后需要重启 Flask 才会生效

---

## 目录

- [核心前提：数据库初始化（必须先做）](#核心前提数据库初始化必须先做)
- [问题 1：模型接入修改后不生效（核心问题）](#问题-1模型接入修改后不生效核心问题)
- [问题 2：前端硬编码模型名（早期诊断）](#问题-2前端硬编码模型名早期诊断)
- [问题 3：模型接入导航权限控制](#问题-3模型接入导航权限控制)
- [问题 4：用户会话隔离（按用户存储、仅看自己数据）](#问题-4用户会话隔离按用户存储仅看自己数据)
- [问题 5：启动报错 Duplicate entry for models.uk_user_model_name](#问题-5启动报错-duplicate-entry-for-modelsuk_user_model_name)
- [功能 6：Token 使用统计模块](#功能-6token-使用统计模块)
- [问题 7：前端 API Key 绕过统计/计费](#问题-7前端-api-key-绕过统计计费)
- [功能 8：聊天页面模型选择器](#功能-8聊天页面模型选择器)
- [问题 9：OneAPI 添加渠道接口兼容（GET + 非流式 + Base URL）](#问题-9oneapi-添加渠道接口兼容get--非流式--base-url)
- [功能 10：模型接入页面 — 设为默认 + 防绕过代理](#功能-10模型接入页面--设为默认--防绕过代理)
- [功能 11：共享模型体系（计费用户专用）](#功能-11共享模型体系计费用户专用)
- [功能 12：计费功能体系（余额、充值、消费记录）](#功能-12计费功能体系余额充值消费记录)
- [功能 13：费用使用页面](#功能-13费用使用页面)
- [功能 14：管理员共享模型管理页面](#功能-14管理员共享模型管理页面)
- [功能 15：登录页「联系管理员」弹窗](#功能-15登录页联系管理员弹窗)
- [发现 16：`get_user_default_model()` 缺少 `def` 关键字](#发现-16get_user_default_model-缺少-def-关键字)
- [架构说明：模型配置数据流](#架构说明模型配置数据流)
- [涉及文件清单](#涉及文件清单)
- [快速排查清单](#快速排查清单)
- [修改记录汇总](#修改记录汇总)

---

## 问题 1：模型接入修改后不生效（核心问题）

### 现象

- 在「模型接入」页面将模型名改为 `glm5`
- 聊天时回复仍然是 `kimi25` 的
- 刷新页面或重新登录也不生效

### 根因（2026-04-28 最终定位）

**数据流问题**：前端页面加载时缓存了默认模型名，后续聊天请求把这个缓存值发给后端，后端优先使用前端传来的旧值，导致数据库里改了也无效。

```
页面加载时          用户发消息时           后端处理
┌─────────┐       ┌──────────────┐      ┌──────────────────┐
│ 请求默认模型 │ ──→ │ 发送 model=kimi25 │ ──→ │ actual_model =   │
│ 得到 kimi25 │       │（前端缓存的旧值）   │      │ model or db_name │
└─────────┘       └──────────────┘      │ = "kimi25" ❌     │
                                        └──────────────────┘
                                                        │
                                                       永远用旧值
```

**代码位置**：
- 前端 `static/js/custom.js`：加载时请求默认模型并缓存到 `defaultModelName`，发送消息时传给后端
- 后端 `app.py`：`actual_model = model or model_config["name"]` 优先使用前端传来的 `model`

### 修复方案

两处同时修改：

#### 1. 前端 `static/js/custom.js`

```javascript
// ❌ 修改前：把缓存的模型名发给后端
data.model = defaultModelName;

// ✅ 修改后：不再发送 model 参数，由后端自行查询数据库
data = {};  // 不设置 model 字段
```

#### 2. 后端 `app.py`

```python
# ❌ 修改前：优先使用前端传来的 model 参数
actual_model = model or model_config["name"]

# ✅ 修改后：始终使用数据库中最新的默认模型配置
actual_model = model_config["name"]
```

### 修复后的数据流

```
用户发消息时        后端处理
┌─────────┐      ┌────────────────────────┐
│  不传递 model │ ──→ │ 查数据库默认模型配置      │
└─────────┘      │ 取 model_config["name"] │
                 │ 得到 "glm5" ✅          │
                 └────────────────────────┘
                          │
                         调用 glm5
```

---

## 问题 2：前端硬编码模型名（早期诊断）

> ⚠️ **注意**：这是最初诊断时误以为的根因，实际根因是上述问题 1。但保留了修复记录以免遗漏。

### 现象（最初）

- 配置 `glm5` 后，前端 Network 面板看到请求体里 `model` 是 `kimi25`
- 以为是前端直接硬编码了字符串

### 实际修正

经过深入排查，发现实际代码（`custom.js`）是：

```javascript
if (defaultModelName) {
  data.model = defaultModelName;   // ← 真正问题：缓存的 defaultModelName 是旧值
}
```

**结论**：不是硬编码，而是页面加载时缓存的 `defaultModelName` 没有被刷新，导致一直发旧值。

---

## 问题 3：模型接入导航权限控制

### 需求日期

2026-04-28

### 需求描述

「模型接入」导航菜单目前对**所有用户可见**，但应该只对**允许使用 AI 对话功能的用户**（`can_chat=true`）或管理员可见。不允许 AI 对话的用户不应该看到「模型接入」。

### 原代码逻辑

原 `layout.html` 中：
- 「模型使用」已根据 `can_chat` 条件显示/隐藏 ✅
- 「用户管理」已根据 admin 条件显示/隐藏 ✅
- 「模型接入」**无条件显示** ❌（没有权限控制）

### 修复方案

修改 `templates/layout.html`：

1. 给「模型接入」导航链接添加 `id="modelsNavLink"`，初始 `style="display:none;"`
2. 在 JS 权限控制逻辑中，加入模型接入导航的显示/隐藏判断

```html
<a href="/models" id="modelsNavLink" class="nav-item ..." data-page="models" style="display:none;">
  <span class="nav-icon"><i class="fa fa-plug"></i></span>
  <span class="nav-label">模型接入</span>
</a>
```

```javascript
// 模型接入：只有允许 AI 对话的用户（含管理员）可见
var modelsNav = document.getElementById('modelsNavLink');
if (modelsNav) {
  modelsNav.style.display = (isAdmin || data.user.can_chat) ? '' : 'none';
}
```

### 权限对照表

| 用户类型 | 首页 | 模型接入 | 模型使用 | 用户管理 |
|---------|------|---------|---------|---------|
| 管理员 | ✅ | ✅ | ✅ | ✅ |
| 普通用户（允许 AI）| ✅ | ✅ | ✅ | ❌ |
| 访客（禁止 AI）| ✅ | ❌ | ❌ | ❌ |

---

## 问题 4：用户会话隔离（按用户存储、仅看自己数据）

### 需求日期

2026-04-28

### 需求描述

- 每个用户的聊天会话必须**保存到数据库**
- 登出后清掉该用户的**本地会话缓存**
- 不同用户登录时**看不到彼此的历史会话**
- 多用户共用同一台电脑时，切换用户后左侧会话栏不能残留上一用户的会话

### 之前的问题

- `localStorage` 的 `chatSessions_v2` 是**全局共享**的，所有用户共用同一个 key
- `loadFromDB()` 只有"追加合并"逻辑，不会清理本地已删除的会话
- 退出登录时不清本地缓存

### 修复方案

#### 1. `templates/chat.html` — 注入当前用户ID

```html
<script>window.currentUserId = {{ user_id }};</script>
```

#### 2. `static/js/custom.js` — localStorage 按用户隔离

```javascript
var LS_KEY = 'chatSessions_v2';
function getUserId() {
  return (typeof window.currentUserId !== 'undefined') ? String(window.currentUserId) : 'anonymous';
}
function getStorageKey() {
  return LS_KEY + '_' + getUserId();
}
```

#### 3. `static/js/custom.js` — `loadFromDB()` 以数据库为准

数据库有的更新或添加，数据库没有的本地也删除。

#### 4. `static/js/custom.js` — 退出登录时清除当前用户本地缓存

```javascript
$(document).on('click', '#logoutBtn', function() {
  var myKey = getStorageKey();
  localStorage.removeItem(myKey);
});
```

### 数据流

```
用户A 聊天                     用户B 登录
│                              │
▼                              ▼
┌──────────────┐              ┌──────────────┐
│ chatSessions │              │ chatSessions │
│ _v2_1        │              │ _v2_2        │
│ (用户A的key)  │              │ (用户B的key)  │
└──────────────┘              └──────────────┘
       │                              │
       ▼                              ▼
同步到数据库                     从数据库加载
(sessions表按user_id隔离)        到用户B的key
```

---

## 问题 5：启动报错 Duplicate entry for models.uk_user_model_name

### 问题日期

2026-04-28

### 报错信息

```
pymysql.err.IntegrityError: (1062, "Duplicate entry '1-glm5' for key 'models.uk_user_model_name'")
```

### 根因

`models` 表有唯一索引 `uk_user_model_name (user_id, name)`，保证同一用户的模型名不重复。

`init_db()` 中的逻辑：
1. 如果管理员没有模型 → INSERT 默认模型 ✅
2. 如果管理员已有模型 → UPDATE `is_default=1` 的那条，把 name 改成 `settings.py` 中的 `DEFAULT_MODEL_NAME` ❌

第二步的问题：管理员已经在「模型接入」页面创建了新的模型（如 `glm5`、`kimi25`），初始化时 UPDATE 默认模型 name，就会和这些已有记录冲突，触发唯一键报错。

### 修复方案（最终版）

**删掉 UPDATE 同步逻辑，只在数据库为空时 INSERT 默认模型。**

管理员后续自己在「模型接入」页面管理模型，`settings.py` 中的 `DEFAULT_MODEL_NAME` 只作为首次初始化的默认值。

### 教训

数据库初始化逻辑应该只做**兜底插入**（数据不存在时才创建），不应该做**强制同步**（覆盖已有数据）。管理类数据以用户实际配置为准，不应被配置文件反复覆盖。

---

## 功能 6：Token 使用统计模块

### 需求日期

2026-04-28

### 需求描述

新增「Token 使用统计」功能模块，用于：
- 统计每个用户的模型 Token 消耗量（输入 + 输出）
- 根据模型定价计算预估费用
- 支持按用户、模型、日期范围筛选
- 支持在「模型接入」页面自定义各模型价格

### 涉及的数据表

#### `models` 表新增字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `price_input` | `DECIMAL(12,6)` | 输入价格（元 / 百万 tokens） |
| `price_output` | `DECIMAL(12,6)` | 输出价格（元 / 百万 tokens） |

#### 新建 `token_usage` 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INT PK | 自增主键 |
| `user_id` | INT | 用户ID |
| `model_name` | VARCHAR(50) | 模型名称 |
| `prompt_tokens` | INT | 输入 token 数 |
| `completion_tokens` | INT | 输出 token 数 |
| `total_tokens` | INT | 总 token 数 |
| `cost` | DECIMAL(12,6) | 本次费用（元） |
| `created_at` | TIMESTAMP | 记录时间 |

### 核心实现

#### 1. 数据库（`app.py` `init_db()`）

- `models` 表添加 `price_input`、`price_output` 字段（兼容旧表 ALTER TABLE）
- 新建 `token_usage` 表

#### 2. 后端 API 流式捕获 usage（`app.py` `/api/chat`）

```python
def generate():
    usage_data = None
    for chunk in resp.iter_lines():
        # ... 原有流式输出逻辑 ...
        if streamDict.get("usage"):
            usage_data = streamDict["usage"]
    
    # 流结束后保存 usage
    if usage_data:
        _save_token_usage(current_user.id, actual_model, usage_data)
```

#### 3. 费用计算逻辑

```python
MODEL_PRICING = {
    "kimi-k2.6":       {"input": 6.50, "output": 27.00},
    "kimi25":          {"input": 2.50, "output": 10.00},
    "glm5":            {"input": 2.00, "output": 10.00},
    "glm-4":           {"input": 2.00, "output": 10.00},
    "glm-4-plus":      {"input": 5.00, "output": 50.00},
    "glm-4-air":       {"input": 0.50, "output": 2.00},
    "gpt-4o":          {"input": 2.50, "output": 10.00},
    "gpt-4o-mini":     {"input": 0.15, "output": 0.60},
}
```

先查用户 `models` 表定价，没有再查 `MODEL_PRICING` 兜底。

#### 4. 新增 API 接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/token-usage` | GET | Token 统计页面 |
| `/api/token-usage` | GET | 获取统计数据（汇总 + 按模型 + 最近记录），支持 `start_date`/`end_date`/`model` 筛选 |
| `/api/pricing/<model>` | GET | 查询模型定价信息 |

#### 5. 新增前端页面

`templates/token-usage.html`：
- 顶部汇总卡片：总调用次数、输入/输出/总 Token、预估费用
- 按模型汇总表格：各模型的消耗详情
- 最近调用记录：最近 30 条明细
- 筛选栏：日期范围、模型选择

#### 6. 时间显示修复

MySQL `TIMESTAMP` 存的是 UTC，`isoformat()` 返回的也是 UTC。后端转成北京时间后返回：

```python
from datetime import timedelta
created_at_cn = r["created_at"] + timedelta(hours=8)
created_at_str = created_at_cn.strftime("%Y-%m-%d %H:%M:%S")
```

### 定价参考

| 模型 | 输入价格 | 输出价格 | 来源 |
|------|---------|---------|------|
| kimi-k2.6 | ¥6.50/M | ¥27.00/M | Kimi 官网 |
| kimi25 | ¥2.50/M | ¥10.00/M | Kimi 官网 |
| glm5 / glm-4 | ¥2.00/M | ¥10.00/M | 智谱官网 |
| glm-4-plus | ¥5.00/M | ¥50.00/M | 智谱官网 |
| glm-4-air | ¥0.50/M | ¥2.00/M | 智谱官网 |
| gpt-4o | ¥2.50/M | ¥10.00/M | OpenAI |
| gpt-4o-mini | ¥0.15/M | ¥0.60/M | OpenAI |

> **注意**: 定价可能随官网调整而变化，用户可在「模型接入」页面自行修改。兜底字典仅供未配置时使用。

### 关键修复记录

| 问题 | 修复方案 |
|------|---------|
| API 不返回 usage（stream=true 时） | `finally` 中用字符数估算兜底（input_chars/1.5, output_chars/1.5） |
| `current_user` 在生成器闭包中失效 | 闭包外提前保存 `user_id_for_usage`、`model_name_for_usage` |
| 浏览器跳转导致 finally 不执行 | 把整个流式逻辑包在 `try...finally` 中 |
| 时间显示偏差（UTC vs 北京） | 后端 `strftime` 前加 8 小时 `timedelta(hours=8)` |

---

## 问题 7：前端 API Key 绕过统计/计费

### 现象

- 前端「设置」面板允许用户填写自己的 API Key，存在 `localStorage.apiKey`
- 用户在「模型接入」配置了模型，但又填了自己的 API Key
- 后端 `/api/chat` 接收 `apiKey` 参数，优先用前端传的 Key 去调外部 API
- Token 统计和计费都失效了，因为请求根本不经过后端数据库配置的模型

### 根因

**后端信任前端参数**：`actual_api_key = apiKey or model_config["api_key"]` 导致只要前端传了 Key，就会绕过数据库配置，直接调用用户自己的 Key 对应的服务。

这意味着：
1. 模型接入页面的配置被忽略
2. Token 统计无法收集
3. 计费用户余额不被检查
4. 管理员配置的共享模型也失效

### 修复

**强制所有请求走后端代理**：

1. **前端 `custom.js`**：删除 `localStorage.apiKey` 的读取和传入逻辑
2. **后端 `app.py`**：彻底删除 `apiKey = request.form.get(...)` 及 `or` 逻辑，强制 `actual_api_key = model_config["api_key"]`

```
前端的 apiKey ──→ ❌ 不再传给后端
后端的 or 逻辑 ──→ ❌ 彻底删除
所有外调 ──→ ✅ 必须走数据库配置
```

---

## 功能 8：聊天页面模型选择器

### 需求

普通用户在模型接入页面配置了多个模型（如 glm5 + kimi25），但聊天页面一直只用默认模型，需要支持手动切换。

### 实现

1. **后端新增 API**：`GET /api/available-models`
   - 普通用户返回自己 `models` 表中的所有模型
   - 计费用户返回 `shared_models` 中的共享模型
   - 按 `is_default DESC, id ASC` 排序，默认模型排第一个

2. **后端 `chat()` 函数**：`client_model = request.form.get("model", "").strip()`
   - 前端传了 model 参数 → 优先用 `get_user_model_by_name()` 查找对应配置
   - 未传或找不到 → fallback 到 `get_user_default_model()`

3. **前端 `custom.js`**：
   - `loadAvailableModels()`：页面加载时调 `/api/available-models` 获取列表
   - 多个模型 → 显示 `<select>` 下拉；仅一个 → 隐藏选择器，只显示文本
   - 发送消息时：`data.model = $('#modelSelector').val()`

---

## 问题 9：OneAPI 添加渠道接口兼容（GET + 非流式 + Base URL）

### 现象

OneAPI 添加渠道时的流程：
1. 先 `GET /v1/models` 验证 API 是否可访问
2. 然后 `POST /v1/chat/completions`（`stream=false`）测试

之前代理接口只支持 POST 和流式，导致 OneAPI：
- GET 验证 → 404（返回 HTML 而非 JSON）
- POST 测试 → SSE 流式返回（OneAPI 收到 `text/event-stream`，解析 choices 为 null）

### 根因

OneAPI 只需要 Base URL，自己拼接路径。如果连完整路径返回给工具，会导致双重拼接 404。

### 修复方案

1. **新增 `GET /v1/models`** → 返回模型列表 JSON：`{"object": "list", "data": [...]}`
2. **`_proxy_chat_json()` 非流式** → `stream=false` 时直接返回 `application/json`，非 SSE
3. **`proxy_url` 只返回 Base URL**（如 `http://host:50001`）

### API 行为

```
GET /v1/models
Authorization: Bearer <proxy_api_key>
→ 返回 { "object": "list", "data": [{"id": "glm5", ...}] }

POST /v1/chat/completions
{ "model": "glm5", "messages": [...], "stream": false }
→ 直接返回 JSON（非 SSE）

POST /v1/chat/completions
{ "model": "glm5", "messages": [...], "stream": true }
→ SSE 流式返回
```

---

## 功能 10：模型接入页面 — 设为默认 + 防绕过代理

### 需求 1：普通用户可以一键切换默认模型

在模型接入卡片上加「设为默认」按钮，点击直接切换，不用进编辑弹窗改 checkbox。

### 需求 2：防绕过 — 用户复制真实 API 信息后直接调用

**旧逻辑**：模型接入页面展示真实 API 地址 + 真实 API Key → 用户复制后填入第三方工具直接使用 → 平台完全无法统计

**新逻辑**：每个用户分配唯一 Proxy Key，页面只展示代理地址 + Proxy Key + 模型名称，所有外调必须通过 `/v1/chat/completions` 代理接口。

### 实现方案

#### 数据库层

1. `users` 表增加 `proxy_api_key VARCHAR(100)` 字段
2. `init_db()` 自动 ALTER + 为所有旧用户循环生成唯一的 Proxy Key：`sk-gxy-<uuid>`
3. 新建用户时自动生成 Proxy Key
4. 已有用户补上 Proxy Key

#### 后端代理 API

```
POST /v1/chat/completions
Authorization: Bearer <proxy_api_key>
Content-Type: application/json

{ "model": "glm5", "messages": [...], "stream": true }
```

- `_authenticate_proxy_key(proxy_key)`：通过 Proxy Key 找到用户
- `_proxy_chat()`：查找模型配置 → 外调真实 API → SSE 流式返回 → finally 中统计 Token + 计费扣费
- `_proxy_chat_json()`：非流式代理（OneAPI 兼容）

#### 模型接入页面改造

- 不再展示真实 API 地址和真实 API Key
- 每个模型卡片内增加**虚线框代理信息区**（`.proxy-section`）
  - 代理地址：`http://<host>:50001`（纯 Base URL）
  - 代理 Key：`sk-gxy-xxxxxxxx...`（用户唯一）
  - 模型名称：如 `glm5`
  - 每行带「复制」按钮
- 每个非默认模型增加「设为默认」按钮
- 保留编辑/删除/测试按钮

#### 后端 `/api/models` GET 改造

- 查询 `users` 表获取当前用户的 `proxy_api_key`
- 返回 `{id, name, api_url, is_default, price_input, price_output, proxy_url, proxy_api_key}`
- `api_url` 仍在返回中（编辑弹窗需要），但前端展示页面不展示它

### 涉及修改

- `app.py`：
  - `User` 类增加 `proxy_api_key`
  - `init_db()` 加 `proxy_api_key` ALTER + 补生成
  - `load_user()` 传入 `proxy_api_key`
  - `api_current_user()` 返回 `proxy_api_key`
  - 新建用户/默认管理员 `INSERT` 加 `proxy_api_key`
  - `api_list_models()` 返回 `proxy_url` + `proxy_api_key`（不暴露真实 `api_key`）
  - 新增 `api_set_default_model(model_id)` → 取消所有默认 + 设置指定为默认
  - 新增 `proxy_chat_completions()` → `/v1/chat/completions` OpenAI 兼容代理
  - 新增 `_authenticate_proxy_key()` + `_proxy_chat()` + `_proxy_chat_json()`
- `templates/model-config.html`：
  - 新样式 `.btn-set-default` + `.proxy-section`
  - 重写 `loadModels()` 渲染逻辑（展示代理信息，隐藏真实 API）
  - `bindModelEvents()` 增加「设为默认」事件
  - `bindCopyEvents()` 通用复制事件提取

---

## 功能 11：共享模型体系（计费用户专用）

### 需求日期

2026-04-28

### 需求描述

普通用户自己配置和管理模型（`models` 表），计费用户没有自己的模型管理权限，而是使用管理员配置的**共享模型**（`shared_models` 表）。

目的：
1. 计费用户不能自行修改 API 信息，防止绕过统计和计费
2. 管理员统一维护共享模型，方便集中管理
3. 计费切换开关后，用户的模型来源自动从 `models` 切换到 `shared_models`

### 数据库表

```sql
CREATE TABLE shared_models (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    api_url VARCHAR(500) NOT NULL,
    api_key VARCHAR(500) NOT NULL,
    model_type VARCHAR(50) DEFAULT 'openai',
    is_default TINYINT(1) DEFAULT 0,
    is_system TINYINT(1) DEFAULT 0 COMMENT '系统内置不可删除',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_shared_name (name)
)
```

默认共享模型：`glm5`、`kimi25`

### 涉及后端代码

- `init_db()`：自动建表 + 插入默认共享模型
- `get_user_default_model()`：计费用户查 `shared_models`，非计费用户查 `models`
- `get_user_model_by_name()`：同上
- `api_available_models()`：计费用户返回 `shared_models` 列表
- `api_my_shared_models()`：计费用户的模型接入页面返回代理信息
- `api_set_shared_default()`：计费用户切换默认模型

### 权限对照表

| 用户类型 | 模型接入页面 | 可管理模型 |
|---------|------------|-----------|
| 管理员 | `shared_models` 管理页 | ✅ 全部共享模型 |
| 计费用户 | 只读（查看代理信息 + 设为默认） | ❌ 只能切换默认 |
| 普通用户 | `models` 管理页 | ✅ 自己的模型 |

---

## 功能 12：计费功能体系（余额、充值、消费记录）

### 需求日期

2026-04-28

### 需求描述

可选计费功能，`billing_enabled` 开关控制：
- 开启计费后，用户每次调用模型按 Token 消耗扣费
- 管理员可以给用户充值
- 用户可以在「费用使用」页面查看余额和记录
- 余额不足 0.10 元时无法继续调用模型

### 数据库表

#### `users` 表新增字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `billing_enabled` | `TINYINT(1) DEFAULT 0` | 是否开启计费 |
| `balance` | `DECIMAL(12, 4) DEFAULT 0.00` | 账户余额（元） |

#### `recharges` 表（充值记录）

| 字段 | 说明 |
|------|------|
| `id` | 自增主键 |
| `user_id` | 充值用户 |
| `amount` | 充值金额 |
| `operator_id` | 操作员（管理员） |
| `remark` | 备注 |
| `created_at` | 时间 |

#### `billing_records` 表（消费明细）

| 字段 | 说明 |
|------|------|
| `id` | 自增主键 |
| `user_id` | 消费用户 |
| `model_name` | 使用模型 |
| `prompt_tokens` | 输入 token |
| `completion_tokens` | 输出 token |
| `total_tokens` | 总计 |
| `cost` | 本次扣费 |
| `balance_after` | 扣费后余额 |
| `created_at` | 时间 |

### 计费流程

```
用户发送消息 → /api/chat
              → 检查 billing_enabled
              → 开启计费 → 检查余额 >= 0.10元
              → 流式调用 API
              → finally 中 _save_token_usage()
              → billing_enabled=True + cost>0
              → UPDATE users SET balance = balance - cost
              → INSERT billing_records
```

### 涉及代码

- `app.py` `chat()`：余额检查 + `FOR UPDATE` 锁
- `app.py` `_save_token_usage()`：计费扣费逻辑
- `app.py` `api_admin_recharge()`：管理员充值
- `app.py` `api_billing_balance()`：查询余额和记录
- `app.py` `api_admin_add_user()`：新建用户时可选择是否开启计费
- `app.py` `api_admin_update_user()`：可修改计费开关

---

## 功能 13：费用使用页面

### 需求日期

2026-04-28

### 页面

- `templates/billing.html`：费用使用页面
- 路由：`GET /billing`
- API：`GET /api/billing/balance`

```javascript
// 加载时调用 /api/billing/balance
{
  "success": true,
  "balance": 100.50,
  "billing_enabled": true,
  "recharges": [...],
  "records": [...]
}
```

---

## 功能 14：管理员共享模型管理页面

### 需求日期

2026-04-28

### 页面

- `templates/admin-shared-models.html`
- 路由：`GET /admin/shared-models`

### API

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/admin/shared-models` | GET | 管理员获取共享模型列表 |
| `/api/admin/shared-models` | POST | 添加共享模型 |
| `/api/admin/shared-models/<id>` | DELETE | 删除共享模型（系统内置不能删） |

---

## 功能 15：登录页「联系管理员」弹窗

### 需求日期

2026-04-28

### 需求描述

移除登录页默认账号提示，改为「没有账号？联系管理员」按钮，点击后弹出联系方式弹窗，显示管理员手机号 `19800236191`，带一键复制。

### 实现

- `templates/login.html`：
  - 移除默认账号提示文字
  - 新增 `.contact-admin-btn` 按钮
  - 新增 `#adminContactModal` 弹窗
  - 弹窗内显示手机号 + 复制按钮（2秒后还原）

- `templates/home.html`：首页「联系作者」弹窗，同样显示手机号 `19800236191`

---

## 发现 16：`get_user_default_model()` 缺少 `def` 关键字

### 问题日期

2026-04-28

### 现象

在检查 `app.py` 时发现某次编辑中间状态中，`get_user_default_model(user_id):` 前面缺少 `def` 关键字，会导致 Python `NameError`。

### 确认修复

当前代码 `app.py` 第 1248 行：

```python
def get_user_default_model(user_id):
    """获取用户的默认模型配置（计费用户读 shared_models，普通用户读自己配置）"""
    conn = get_db()
    ...
```

确保 `def` 关键字存在。


---

## 架构说明：模型配置数据流

### 涉及的表

| 表名 | 说明 |
|------|------|
| `models` | 普通用户自己配置的模型列表（name, api_url, api_key, is_default...） |
| `shared_models` | 管理员配置的共享模型（仅计费用户使用） |
| `users` | 用户信息（含 billing_enabled, balance, proxy_api_key） |
| `token_usage` | 所有用户的 Token 消耗统计 |
| `billing_records` | 计费用户的消费明细 |
| `recharges` | 充值记录 |
| `sessions` | 聊天会话持久化 |

### 涉及的 API

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/models` | GET | 获取模型列表 |
| `/api/models` | POST | 添加模型 |
| `/api/models/<id>` | PUT | 修改模型 |
| `/api/models/<id>` | DELETE | 删除模型 |
| `/api/models/<id>/test` | POST | 测试模型连接 |
| `/api/models/<id>/set-default` | POST | 设为默认 |
| `/api/models/default` | GET | 获取默认模型 |
| `/api/available-models` | GET | 聊天页面可用模型列表 |
| `/api/shared-models/<id>/set-default` | POST | 计费用户设共享模型为默认 |
| `/api/shared-models/my` | GET | 计费用户获取共享模型代理信息 |
| `/api/chat` | POST | 发送聊天消息（流式SSE） |
| `/v1/models` | GET | OpenAI 兼容代理：模型列表 |
| `/v1/chat/completions` | POST/GET | OpenAI 兼容代理：聊天（流式+非流式） |
| `/api/admin/shared-models` | GET/POST | 管理员管理共享模型 |
| `/api/admin/recharge/<id>` | POST | 管理员给用户充值 |
| `/api/billing/balance` | GET | 查询余额和消费记录 |

### 完整数据流

```
┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
│ 管理员在模型接入 │    │ 写入 models 表  │    │ /api/chat 处理时  │
│ 页面添加/修改   │──→ │ (name, api_url) │──→ │ get_user_default  │
│ model=glm5   │    │ is_default=1   │    │ _model(user_id)   │
└──────────────┘    └──────────────┘    └──────────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │ 查数据库取最新配置  │
                                          │ model_config["name"]│
                                          │ = "glm5" ✅       │
                                          └──────────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │ 实际调用外部 API   │
                                          │ 发送 model: "glm5"│
                                          └──────────────────┘
                                          │
                                          ▼
                                    finally 中 _save_token_usage()
                                    统计入库 + 计费扣费（如开启）
```

---

## 涉及文件清单

| 文件 | 路径 | 角色 |
|------|------|------|
| `app.py` | 根目录 | Flask 后端主程序（最核心） |
| `settings.py` | 根目录 | 配置文件 |
| `custom.js` | `static/js/custom.js` | 前端聊天逻辑（含 session 管理、localStorage 隔离） |
| `chat.html` | `templates/chat.html` | 聊天页面模板 |
| `model-config.html` | `templates/model-config.html` | 模型接入页面（含价格配置 + 代理信息） |
| `token-usage.html` | `templates/token-usage.html` | Token 使用统计页面 |
| `billing.html` | `templates/billing.html` | 费用使用页面 |
| `admin.html` | `templates/admin.html` | 用户管理页面（含计费开关 + 充值） |
| `admin-shared-models.html` | `templates/admin-shared-models.html` | 管理员共享模型管理 |
| `layout.html` | `templates/layout.html` | 全局布局模板（含导航权限） |
| `login.html` | `templates/login.html` | 登录页面（联系管理员弹窗） |
| `home.html` | `templates/home.html` | 首页（联系作者弹窗） |

---

## 快速排查清单

如果以后「模型接入改了但聊天没生效」，按以下顺序排查：

1. **确认数据库里查出来的模型名是对的**
   ```sql
   SELECT name, api_url, is_default FROM models WHERE user_id = ?;
   SELECT name, api_url, is_default FROM shared_models;
   ```

2. **确认后端代码没有优先使用前端传来的 `model` 参数**
   ```python
   # app.py 中应该是：
   actual_model = model_config["name"]
   # 而不是：
   actual_model = model or model_config["name"]
   ```

3. **确认前端没有缓存旧模型名**
   ```javascript
   // custom.js 中聊天按钮点击事件，data 中不能有 model 字段
   // 只有 model 选择器变化时才传
   ```

4. **重启 Flask 服务**（修改 Python 代码后需要重启）

5. **浏览器强制刷新**（Ctrl+Shift+R 或 Cmd+Shift+R）

---

## 修改记录汇总

| 日期 | 问题 | 修改文件 | 修改内容 |
|------|------|---------|---------|
| 2026-04-28 | 模型接入改后不生效 | `static/js/custom.js` | 前端不再发送 `model` 参数 |
| 2026-04-28 | 后端优先使用前端旧值 | `app.py` | 固定使用 `model_config["name"]` |
| 2026-04-28 | 模型接入导航无权限控制 | `templates/layout.html` | 按 `can_chat` 条件显示/隐藏导航入口 |
| 2026-04-28 | 用户会话未按用户隔离 | `static/js/custom.js` | localStorage key 按用户ID隔离 + 退出清缓存 |
| 2026-04-28 | 用户会话未按用户隔离 | `templates/chat.html` | 注入 `window.currentUserId` |
| 2026-04-28 | 用户会话未按用户隔离 | `app.py` | `chat_page()` 渲染时传入 `user_id` |
| 2026-04-28 | 启动报错唯一键冲突 | `app.py` | `init_db()` 删除 UPDATE 同步逻辑，仅首次空表时 INSERT |
| 2026-04-28 | 新增 Token 使用统计 | `app.py` | 新增 `token_usage` 表 + `_save_token_usage()` + `MODEL_PRICING` + 接口 |
| 2026-04-28 | 新增 Token 使用统计 | `templates/token-usage.html` | 新增 Token 统计页面 |
| 2026-04-28 | 新增 Token 使用统计 | `templates/model-config.html` | 新增模型价格输入/输出配置字段 |
| 2026-04-28 | 新增 Token 使用统计 | `templates/layout.html` | 新增「Token使用统计」导航入口 |
| 2026-04-28 | Token 统计 API 未返回 usage | `app.py` | `generate()` 增加字符数估算兜底（÷1.5） |
| 2026-04-28 | Token 统计未执行 | `app.py` | 闭包前预先保存 `user_id` 和 `model_name` |
| 2026-04-28 | Token 统计因跳转丢失 | `app.py` | `generate()` 改为 `try...finally` 强制保存 |
| 2026-04-28 | Token 统计时间显示错误 | `app.py` | 后端加 8 小时时差，UTC 转北京时间 |
| 2026-04-28 | 前端 API Key 绕过 | `app.py` + `custom.js` | 彻底删除前端 apiKey 传入，强制走后端代理 |
| 2026-04-28 | 聊天页面模型选择器 | `app.py` + `custom.js` + `chat.html` | `/api/available-models` + 下拉选择 + 后端按 model 切换 |
| 2026-04-28 | 模型接入页防绕过 | `app.py` + `model-config.html` | `proxy_api_key` + `/v1/chat/completions` 代理 |
| 2026-04-28 | /api/models KeyError | `app.py` | `SELECT` 补上 `api_url` 字段（编辑弹窗需要） |
| 2026-04-28 | 计费用户模型接入页报错 | `model-config.html` | `loadSharedModels()` 展示 `proxy_url` + `proxy_api_key`，不再读 `m.api_key` |
| 2026-04-28 | 计费用户设共享默认 | `app.py` | 新增 `POST /api/shared-models/<id>/set-default` |
| 2026-04-28 | OneAPI 404 HTML | `app.py` | 新增 `GET /v1/models` + `GET /v1/chat/completions` 返回模型列表 |
| 2026-04-28 | OneAPI POST 测试失败 | `app.py` | `_proxy_chat_json()` 非流式直接返回 JSON（SSE→JSON） |
| 2026-04-28 | OneAPI API URL 路径错误 | `app.py` | `proxy_url` 只返回 Base URL（不含 `/v1/chat/completions`） |
| 2026-04-28 | 新增计费功能 | `app.py` + `admin.html` | `billing_enabled` + `balance` + 充值 + 扣费 |
| 2026-04-28 | 新增计费功能 | `app.py` | `_save_token_usage()` 中集成扣费逻辑 |
| 2026-04-28 | 新增计费功能 | `app.py` | `recharges` + `billing_records` 表 |
| 2026-04-28 | 新增费用使用页 | `templates/billing.html` | 余额 + 充值记录 + 消费明细 |
| 2026-04-28 | 新增共享模型管理 | `app.py` + `admin-shared-models.html` | `shared_models` 增删查 + 默认切换 |
| 2026-04-28 | 新增共享模型管理 | `app.py` | `api_my_shared_models()` 计费用户展示代理信息 |
| 2026-04-28 | 登录页联系管理员 | `templates/login.html` | 移除默认账号提示，改为「联系管理员」弹窗 |
| 2026-04-28 | 首页联系作者 | `templates/home.html` | 联系作者弹窗显示手机号 `19800236191` |
| 2026-04-29 | 管理员不能修改自己的密码 | `app.py` + `admin.html` | 仅禁止自己降角色，允许改密码和其他属性 |
