# 离线模式与数据同步规范

## 1. 离线模式定义

### 1.1 离线判定条件
设备处于以下任一状态时，判定为离线模式：
- WiFi未连接
- 已连接WiFi但无法访问互联网
- 服务器连接超时（5秒内无响应）

### 1.2 离线模式能力矩阵

| 功能 | 在线状态 | 离线状态 | 离线缓存 |
|------|---------|---------|---------|
| 页面切换 | ✓ | ✓ | 当前页面状态 |
| 日期显示 | ✓ | ✓ | RTC本地计算 |
| 农历显示 | ✓ | ✓ | 本地算法 |
| 天气显示 | ✓ | ✗ | 上次天气缓存 |
| 待办查看 | ✓ | ✓ | 本地全量缓存 |
| 待办添加 | ✓ | ✓ | 本地队列，联网后同步 |
| 待办完成 | ✓ | ✓ | 本地队列，联网后同步 |
| 待办删除 | ✓ | ✓ | 本地队列，联网后同步 |
| 倒计时查看 | ✓ | ✓ | 本地全量缓存 |
| 倒计时添加 | ✓ | ✓ | 本地队列，联网后同步 |
| 日历查看 | ✓ | ✓ | 本地日程缓存 |
| 语音控制 | ✓ | ✗ | 需联网识别 |
| 数据同步 | ✓ | ✗ | 离线时暂停 |
| 手动同步 | ✓ | ✗ | 提示离线不可用 |

### 1.3 离线缓存数据范围

在多设备架构下，离线缓存需要支持：
1. **用户级Registry缓存** - 所有设备共享同一Registry
2. **设备级操作队列** - 每个设备独立维护操作队列
3. **同步冲突检测** - 设备离线期间的修改与服务器最新版本对比

**缓存层级**:
```
Device A (User 1)          Device B (User 1)          Device C (User 2)
├── Registry Cache         ├── Registry Cache         ├── Registry Cache
│   (User 1 的 Registry)   │   (User 1 的 Registry)   │   (User 2 的 Registry)
│   version: 5             │   version: 5             │   version: 3
│   synced_at: 12:00       │   synced_at: 12:00       │   synced_at: 11:30
│                          │                          │
└── Operation Queue        └── Operation Queue        └── Operation Queue
    (Device A独有)            (Device B独有)            (Device C独有)
    [op1, op2, op3]           []                         [op1]

注意：Device A 和 B 共享 Registry，但操作队列独立
```

#### 必须本地存储的数据（离线可用）
```json
{
  "todos": {
    "max_count": 50,
    "fields": ["id", "content", "completed", "priority", "due_date", "created_at"],
    "sync_mode": "全量缓存"
  },
  "countdowns": {
    "max_count": 10,
    "fields": ["id", "name", "target_date", "display_style", "created_at"],
    "sync_mode": "全量缓存"
  },
  "calendar_events": {
    "max_count": 100,
    "days_range": 30,
    "fields": ["id", "title", "date", "type", "reminder"],
    "sync_mode": "近30天缓存"
  },
  "weather": {
    "fields": ["temp", "desc", "icon", "humidity", "wind", "update_time"],
    "ttl": 3600,
    "sync_mode": "单条缓存，过期显示旧数据"
  },
  "user_preferences": {
    "fields": ["default_page", "templates", "voice_settings", "sync_interval"],
    "sync_mode": "全量缓存"
  }
}
```

#### 仅在线获取的数据
- 语音识别结果
- 天气实时更新
- 农历/节气精确数据（首次）
- 服务器推送通知

---

## 2. 离线操作队列

### 2.1 队列设计

```cpp
struct PendingOperation {
    string id;              // 操作唯一ID (UUID)
    string type;            // 操作类型: CREATE/UPDATE/DELETE
    string entity;          // 实体类型: todo/countdown/event
    string entity_id;       // 实体ID
    json data;              // 操作数据
    uint64_t timestamp;     // 操作时间戳
    int retry_count;        // 重试次数
    uint64_t last_retry;    // 上次重试时间
};

// 队列存储在SPIFFS: /spiffs/pending_ops.json
vector<PendingOperation> pending_queue;
```

### 2.2 支持离线操作类型

| 操作 | 实体 | 离线支持 | 队列行为 |
|------|------|---------|---------|
| 添加待办 | todo | ✓ | 本地创建，加入队列 |
| 完成待办 | todo | ✓ | 本地更新，加入队列 |
| 删除待办 | todo | ✓ | 本地标记删除，加入队列 |
| 修改待办 | todo | ✓ | 本地更新，加入队列 |
| 添加倒计时 | countdown | ✓ | 本地创建，加入队列 |
| 删除倒计时 | countdown | ✓ | 本地标记删除，加入队列 |
| 修改倒计时 | countdown | ✓ | 本地更新，加入队列 |
| 切换模板 | preference | ✓ | 本地保存，加入队列 |
| 修改设置 | preference | ✓ | 本地保存，加入队列 |

### 2.3 队列处理流程

```
设备上线
    │
    ▼
检测网络恢复
    │
    ▼
检查队列长度
    │
    ├─ 空队列 ──▶ 正常同步（下载服务器最新）
    │
    ▼
上传队列操作（按时间顺序）
    │
    ├─ 全部成功 ──▶ 清空队列 ──▶ 正常同步
    │
    ▼
部分失败
    │
    ├─ 可重试错误 ──▶ 保留队列 ──▶ 下次重试
    │
    ▼
不可重试错误
    │
    ▼
冲突解决
    │
    ▼
清空队列/保留失败项
```

### 2.4 队列持久化

- **存储位置**: `/spiffs/pending_ops.json`
- **最大长度**: 100条操作
- **队列满处理**: 拒绝新操作，提示"操作过多，请先联网同步"
- **持久化策略**: 每次操作立即写入，避免掉电丢失

---

## 3. 数据同步机制

### 3.1 同步触发条件

| 触发方式 | 条件 | 优先级 | 说明 |
|---------|------|--------|------|
| 服务器推送 | Registry变更时 | 最高 | 多设备场景：同一用户的其他设备修改后，服务器主动推送 |
| 离线操作后恢复 | 网络恢复且队列非空 | 高 | 设备重新上线，优先处理本地队列 |
| 手动同步 | 用户语音"立即同步" | 高 | 强制全量同步 |
| 设备上线 | 从离线状态恢复 | 中 | 检查版本号，落后则拉取 |
| 启动同步 | 设备启动后 | 中 | 首次启动从服务器加载 |
| RTC定时唤醒 | 深度休眠设备周期性唤醒 | 中 | 白天30分钟/夜间1小时 |
| 按键唤醒 | 用户按键触发 | 高 | 立即同步最新数据 |

**深度休眠与同步**:
- 设备平时处于深度休眠，WiFi断开，功耗极低
- **RTC定时唤醒**：周期性检查消息（白天30分钟，夜间1小时）
- **按键唤醒**：用户操作触发，立即连接WiFi并同步
- **活跃期**：语音操作后保持活跃5分钟，期间可实时接收推送
- 唤醒后自动比较版本号，落后则拉取更新

### 3.2 同步流程

#### 场景1：Registry变更后多设备推送

```
设备A（活跃）        设备B（活跃）        设备C（休眠）        服务器
    │                  │                  │                  │
    │                  │                  │                  │
    │                  │                  │       [用户语音修改Registry]
    │                  │                  │                  │ v5 → v6
    │←────MQTT推送─────│←────MQTT推送─────│                  │
    │  Registry v6     │  Registry v6     │                  │
    │  [即时更新]      │  [即时更新]      │                  │
    │                  │                  │                  │
    │                  │                  │  [RTC唤醒]       │
    │                  │                  │                  │
    │                  │                  │──连接──────────→│
    │                  │                  │  version=5       │
    │                  │                  │                  │ [检测到落后]
    │                  │                  │←──推送Registry──│
    │                  │                  │  v6              │
    │                  │                  │  [更新后休眠]    │
```

#### 场景2：设备离线操作后同步（冲突解决）

```
设备A（离线）                           设备B（在线）        服务器
    │                                        │               │
    │  [离线期间添加待办]                     │  [添加待办]   │
    │  操作入队                               │               │
    │  [op1: add_todo "买牛奶"]              │               │
    │                                        │               │
    │  [添加倒计时]                           │  [隐藏天气页] │
    │  操作入队                               │  Registry v5  │
    │  [op2: add_countdown "考试"]           │       ↓       │
    │                                        │  Registry v6  │
    │                                        │               │
    │  [设备A上线]                            │               │
    │                                        │               │
    │──上传队列────────────────────────────→│               │
    │  [op1, op2]                            │               │
    │                                        │               │
    │                                        │               │ [应用op1, op2]
    │                                        │               │ [生成v7]
    │                                        │               │
    │←──同步成功─────────────────────────────│               │
    │  new_version: 7                        │               │
    │                                        │               │
    │                                        │←─MQTT推送v7───│
    │                                        │  [设备B同步]  │
```

#### 标准同步流程

```
开始同步
    │
    ▼
检查网络状态
    │
    ├─ 离线 ──▶ 标记失败，结束
    │
    ▼
设备认证
    │
    ▼
1. 上传阶段
    │
    ├─ 检查待上传队列
    │
    ├─ 空 ──▶ 跳过
    │
    ▼
    按时间顺序上传操作
    │
    ├─ 全部成功 ──▶ 清空队列
    │
    ▼
    部分失败
    │
    ├─ 可重试 ──▶ 保留队列
    │
    ▼
    冲突 ──▶ 冲突解决流程
    │
    ▼
2. 下载阶段
    │
    ├─ 请求服务器最新数据
    │
    ├─ 比较版本号/时间戳
    │
    ▼
    合并数据
    │
    ▼
3. 更新本地
    │
    ├─ 更新缓存文件
    │
    ├─ 刷新当前页面（如果需要）
    │
    ▼
同步完成
    │
    ├─ 更新"上次同步时间"
    ├─ 更新同步状态为"已同步"
    └─ 记录同步日志
```

### 3.3 同步数据包格式

#### 上传请求
```json
{
  "device_id": "esp32_abc123",
  "timestamp": 1710420000,
  "last_sync": 1710416400,
  "operations": [
    {
      "id": "op_001",
      "type": "CREATE",
      "entity": "todo",
      "timestamp": 1710417000,
      "data": {
        "id": "todo_001",
        "content": "买牛奶",
        "completed": false
      }
    },
    {
      "id": "op_002",
      "type": "UPDATE",
      "entity": "todo",
      "entity_id": "todo_002",
      "timestamp": 1710418000,
      "data": {
        "completed": true
      }
    }
  ]
}
```

#### 下载响应
```json
{
  "status": "success",
  "server_timestamp": 1710420001,
  "todos": [
    {"id": "todo_001", "content": "买牛奶", "completed": false, "updated_at": 1710417000},
    {"id": "todo_002", "content": "提交报告", "completed": true, "updated_at": 1710418000}
  ],
  "countdowns": [
    {"id": "cd_001", "name": "考试", "target_date": "2026-06-07"}
  ],
  "preferences": {
    "default_page": "date",
    "template": "standard"
  },
  "conflicts": [] // 冲突列表
}
```

#### 多设备Registry推送格式（服务器→设备）

**方式1：全量推送（默认）**
```json
{
  "type": "full",
  "version": 6,
  "registry": {
    "pages": [...],
    "templates": [...]
  },
  "timestamp": 1710420001
}
```

**方式2：增量推送（大Registry时优化）**
```json
{
  "type": "delta",
  "version": 6,
  "base_version": 5,
  "changes": [
    {"op": "add", "path": "/pages/-", "value": {"page_id": "stock", ...}},
    {"op": "replace", "path": "/pages/1/visible", "value": false},
    {"op": "remove", "path": "/pages/3"}
  ],
  "timestamp": 1710420001
}
```

**设备唤醒与版本检测**
```json
// 设备→服务器（RTC唤醒或按键唤醒）
{
  "device_id": "AA:BB:CC:DD:EE:FF",
  "timestamp": 1710420000,
  "registry_version": 5,
  "battery": 85,
  "wakeup_reason": "rtc",  // "rtc" 或 "button"
  "last_sleep_duration": 1800  // 上次休眠时长（秒）
}

// 服务器→设备（唤醒响应，如版本落后则推送）
{
  "status": "ok",
  "server_time": 1710420001,
  "registry_update": true,  // 提示有新版本
  "latest_version": 6,
  "next_wakeup": 1800  // 建议下次唤醒间隔（秒）
}
```

---

## 4. 冲突解决策略

### 4.1 冲突场景定义

| 场景 | 描述 | 示例 |
|------|------|------|
| **双向修改** | 本地和服务器都修改了同一实体 | 手机和设备都改了同一个待办 |
| **离线删除** | 本地删除，服务器还有 | 设备离线删除了待办，手机还在 |
| **离线新增** | 两端都新增了相同ID（UUID冲突） | 极端情况，概率极低 |
| **依赖缺失** | 本地修改依赖已删除的实体 | 修改一个已删除的倒计时 |

### 4.2 冲突解决策略

#### 策略0: 服务器版本优先
```
规则：服务器版本总是权威，设备离线期间的修改作为增量操作处理

场景：
- 设备A离线期间，设备B修改了Registry（v5 → v6）
- 设备A重新上线，携带离线操作队列
- 服务器：
  1. 接受设备A的操作队列
  2. 基于v6应用这些操作
  3. 生成新版本v7
  4. 推送给所有设备（A、B、...）

优点：
- 实现简单，无需复杂的三向合并
- 最终一致性：所有设备最终都会同步到v7
- 用户意图保留：离线操作不会丢失，而是追加到最新版本

缺点：
- 如果v6的修改与离线操作冲突，可能产生意外结果
- 例如：v6删除了页面X，离线操作修改页面X → 操作无效
```

#### 策略1: 时间戳优先（默认）
```
规则：以最后修改时间为准

场景：
- 设备修改时间：T1
- 服务器修改时间：T2
- T1 > T2：采用设备版本
- T2 > T1：采用服务器版本，覆盖本地
- T1 == T2：设备优先（避免循环）
```

#### 策略2: 操作类型优先
```
优先级（从高到低）：
1. DELETE（删除最高优先级）
2. UPDATE
3. CREATE

场景：
- 本地DELETE，服务器UPDATE → 删除（DELETE优先）
- 本地UPDATE，服务器DELETE → 删除（DELETE优先）
```

#### 策略3: 特定实体规则

**待办事项**:
- 完成状态：取并集（任一完成即完成）
- 内容修改：时间戳优先
- 删除：DELETE优先

**倒计时**:
- 目标日期修改：时间戳优先
- 删除：DELETE优先

**设置偏好**:
- 总是设备优先（用户当前设备的设置为准）

### 4.3 冲突解决流程

```
检测到冲突
    │
    ▼
识别冲突类型
    │
    ├─ 双向修改 ──▶ 应用策略1（时间戳）
    │
    ├─ 离线删除 ──▶ 应用策略2（DELETE优先）
    │
    ├─ UUID冲突 ──▶ 服务器重新分配ID
    │
    └─ 依赖缺失 ──▶ 丢弃无效操作
    │
    ▼
记录冲突日志
    │
    ▼
应用解决方案
    │
    ├─ 采用服务器版本 ──▶ 更新本地
    │
    ├─ 采用设备版本 ──▶ 保留，下次同步
    │
    └─ 合并 ──▶ 生成新版本
```

### 4.4 冲突通知

- **策略**: 静默解决，不打扰用户
- **记录**: 记录到同步日志，可在"同步页面"查看
- **极端情况**: 如果冲突导致数据丢失，显示提示"部分数据已同步"

---

## 5. 版本管理与数据迁移

### 5.1 版本号设计

```json
{
  "data_version": "1.0.0",
  "schema_version": 1,
  "last_modified": 1710420000,
  "entities": {
    "todos": {"version": 5, "count": 10},
    "countdowns": {"version": 2, "count": 3},
    "preferences": {"version": 1, "count": 1}
  }
}
```

### 5.2 版本升级策略

| 升级类型 | 处理方式 |
|---------|---------|
| 小版本(1.0.0→1.0.1) | 自动兼容，无需处理 |
| 中版本(1.0.0→1.1.0) | 自动迁移，保留旧字段 |
| 大版本(1.0.0→2.0.0) | 需要重置数据，提示用户 |

### 5.3 数据迁移示例

```python
def migrate_v1_to_v2(data):
    """v1到v2的数据迁移"""
    if data['version'] == 1:
        # 新增字段设置默认值
        for todo in data['todos']:
            todo['priority'] = todo.get('priority', 1)
            todo['tags'] = todo.get('tags', [])
        
        data['version'] = 2
    
    return data
```

---

## 6. 存储管理

### 6.1 SPIFFS分区规划

```
总空间: 2MB (ESP32-S3默认)

分区分配:
├── config/ (256KB)
│   ├── device.json (4KB)
│   ├── wifi.json (2KB)
│   └── user.json (10KB)
├── cache/ (512KB)
│   ├── weather.json (8KB)
│   ├── current_page.json (1KB)
│   └── sync_status.json (2KB)
├── data/ (1024KB)
│   ├── todos.json (max 200KB, 50条)
│   ├── countdowns.json (max 50KB, 10条)
│   ├── events.json (max 300KB, 100条)
│   └── preferences.json (10KB)
├── queue/ (128KB)
│   └── pending_ops.json (max 100条操作)
└── logs/ (128KB)
    └── sync.log (滚动存储，保留最近100条)
```

### 6.2 存储清理策略

| 场景 | 清理策略 |
|------|---------|
| 空间不足(<10%) | 删除最旧的日志文件 |
| 待办超过50条 | 拒绝新增，提示同步 |
| 操作队列超过100条 | 拒绝新操作，提示同步 |
| 日志超过100条 | 自动删除最早的50条 |

### 6.3 数据备份

- **自动备份**: 每次同步成功后备份关键配置
- **备份位置**: `/spiffs/backup/config_backup.json`
- **恢复机制**: 启动时检测主配置损坏，自动从备份恢复

---

## 7. 错误处理与重试

### 7.1 错误分类

| 错误码 | 描述 | 可重试 | 处理策略 |
|--------|------|--------|----------|
| NET_001 | 网络未连接 | 是 | 等待网络恢复 |
| NET_002 | 服务器无响应 | 是 | 指数退避重试 |
| NET_003 | DNS解析失败 | 是 | 3次后提示用户 |
| AUTH_001 | 设备认证失败 | 否 | 重置设备配对 |
| AUTH_002 | Token过期 | 是 | 自动刷新Token |
| DATA_001 | 数据格式错误 | 否 | 重置本地数据 |
| DATA_002 | 存储空间不足 | 否 | 清理空间 |
| SRV_001 | 服务器内部错误 | 是 | 5分钟后重试 |

### 7.2 重试策略

```python
retry_schedule = {
    1: 5,      # 第1次重试：5秒后
    2: 30,     # 第2次重试：30秒后
    3: 300,    # 第3次重试：5分钟后
    4: 1800,   # 第4次重试：30分钟后
}
# 第4次后不再自动重试，等待下次触发
```

### 7.3 失败记录

```json
{
  "failures": [
    {
      "timestamp": 1710420000,
      "error_code": "NET_002",
      "operation": "sync_upload",
      "retry_count": 3,
      "resolved": false
    }
  ]
}
```

---

## 8. 同步日志

### 8.1 日志格式

```json
{
  "logs": [
    {
      "timestamp": 1710420000,
      "type": "sync",
      "status": "success",
      "duration_ms": 1500,
      "uploaded_ops": 5,
      "downloaded_items": 10,
      "conflicts": 0
    },
    {
      "timestamp": 1710416400,
      "type": "sync",
      "status": "failed",
      "error_code": "NET_002",
      "message": "服务器无响应"
    }
  ]
}
```

### 8.2 日志保留策略

- **最大条目**: 100条
- **保留时间**: 7天
- **滚动方式**: FIFO，新日志覆盖最旧

---

## 9. 性能指标

### 9.1 同步性能要求

| 指标 | 目标值 | 最大容忍 |
|------|--------|----------|
| 单次同步时间 | <3秒 | 5秒 |
| 上传操作数/秒 | >10条 | - |
| 下载数据大小 | <100KB | 500KB |
| 队列处理时间 | <1秒 | - |
| 冲突解决时间 | <100ms | - |

### 9.2 存储性能要求

| 操作 | 目标时间 |
|------|----------|
| 读取本地缓存 | <50ms |
| 写入操作队列 | <100ms |
| 全量数据保存 | <500ms |

---

## 10. 附录: 离线模式状态机

```cpp
enum NetworkState {
    NET_ONLINE,      // 在线状态
    NET_OFFLINE,     // 离线状态（WiFi未连接）
    NET_WEAK,        // 弱网状态（WiFi连接但不通）
    NET_RECOVERING   // 恢复中（正在重连）
};

// 状态转换
NetworkState transitions[] = {
    // 在线 -> 离线
    {NET_ONLINE, EVENT_WIFI_DISCONNECT, NET_OFFLINE},
    {NET_ONLINE, EVENT_SERVER_TIMEOUT, NET_WEAK},
    
    // 离线 -> 恢复中
    {NET_OFFLINE, EVENT_WIFI_CONNECTING, NET_RECOVERING},
    
    // 恢复中 -> 在线/离线
    {NET_RECOVERING, EVENT_SERVER_CONNECTED, NET_ONLINE},
    {NET_RECOVERING, EVENT_SERVER_TIMEOUT, NET_OFFLINE},
    
    // 弱网 -> 在线/离线
    {NET_WEAK, EVENT_SERVER_CONNECTED, NET_ONLINE},
    {NET_WEAK, EVENT_WIFI_DISCONNECT, NET_OFFLINE},
};

// 各状态下的行为
void on_state_online() {
    // 正常同步
    // 语音功能可用
    // 显示绿色WiFi图标
}

void on_state_offline() {
    // 停止自动同步
    // 禁用语音功能
    // 显示灰色WiFi图标
    // 操作进入队列
}

void on_state_weak() {
    // 减少同步频率
    // 延长超时时间
    // 显示弱信号图标
}

void on_state_recovering() {
    // 显示"连接中..."
    // 尝试重连服务器
    // 3次失败后转为离线
}
```
