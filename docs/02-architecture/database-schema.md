---
title: 数据库设计规范
description: 多租户多设备架构下的PostgreSQL+Redis数据结构设计，含用户隔离、设备管理、配对流程
type: architecture
scope: [backend]
created: "2026-03-13"
updated: "2026-03-13"
dependencies: []
related:
  - system-architecture.md
  - ../03-protocols/pairing-protocol.md
  - ../03-protocols/sync-protocol.md
---

# 数据库设计规范

## 1. 概述

本规范定义多租户多设备架构下的数据库结构，支持用户隔离、设备管理、配对流程和数据同步。

**数据库选型**: PostgreSQL (主数据库) + Redis (缓存/会话)

---

## 2. 实体关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户 (users)                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • user_id (PK)                                            │  │
│  │ • username                                                │  │
│  │ • password_hash                                           │  │
│  │ • role (admin/user)                                       │  │
│  │ • created_at                                              │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ 1:N
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                       设备 (devices)                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • device_id (PK) - MAC地址                                 │  │
│  │ • user_id (FK)                                             │  │
│  │ • device_name                                              │  │
│  │ • status (online/offline/pairing)                          │  │
│  │ • last_seen                                                │  │
│  │ • registry_version                                         │  │
│  │ • created_at                                               │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ 1:1
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                  页面注册表 (registries)                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • registry_id (PK)                                         │  │
│  │ • user_id (FK) - 用户级隔离                                │  │
│  │ • pages (JSON) - 页面配置                                  │  │
│  │ • templates (JSON)                                         │  │
│  │ • version                                                  │  │
│  │ • updated_at                                               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    配对会话 (pairing_sessions)                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • session_id (PK)                                          │  │
│  │ • device_id                                                │  │
│  │ • pairing_code (6位数字)                                   │  │
│  │ • status (pending/confirmed/expired)                       │  │
│  │ • created_at                                               │  │
│  │ • expires_at (TTL 10分钟)                                  │  │
│  │ • confirmed_at                                             │  │
│  │ • confirmed_by (admin_id)                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   同步日志 (sync_logs)                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • log_id (PK)                                              │  │
│  │ • user_id (FK)                                             │  │
│  │ • device_id (FK)                                           │  │
│  │ • sync_type (push/pull/full)                               │  │
│  │ • status (success/failed)                                  │  │
│  │ • details (JSON)                                           │  │
│  │ • created_at                                               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 表结构定义

### 3.1 用户表 (users)

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user' CHECK (role IN ('admin', 'user')),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_role ON users(role);

-- 初始管理员账号
INSERT INTO users (user_id, username, password_hash, role) 
VALUES ('00000000-0000-0000-0000-000000000000', 'admin', '<hashed>', 'admin');
```

### 3.2 设备表 (devices)

```sql
CREATE TABLE devices (
    device_id VARCHAR(17) PRIMARY KEY,  -- MAC地址格式: AA:BB:CC:DD:EE:FF
    user_id UUID REFERENCES users(user_id) ON DELETE SET NULL,
    device_name VARCHAR(100),
    device_type VARCHAR(50) DEFAULT 'sticky_v1',
    
    -- 连接状态
    status VARCHAR(20) DEFAULT 'offline' 
        CHECK (status IN ('online', 'offline', 'pairing', 'error')),
    last_seen TIMESTAMP WITH TIME ZONE,
    ip_address INET,
    
    -- 配置版本
    registry_version INTEGER DEFAULT 1,
    registry_hash VARCHAR(64),  -- SHA256校验
    
    -- 配对信息
    paired_at TIMESTAMP WITH TIME ZONE,
    paired_by UUID REFERENCES users(user_id),
    
    -- 硬件信息
    firmware_version VARCHAR(20),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_devices_user_id ON devices(user_id);
CREATE INDEX idx_devices_status ON devices(status);
CREATE INDEX idx_devices_last_seen ON devices(last_seen);
```

### 3.3 页面注册表 (registries)

```sql
CREATE TABLE registries (
    registry_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- 页面配置 (JSON格式，见下方示例)
    pages JSONB NOT NULL DEFAULT '[]',
    templates JSONB NOT NULL DEFAULT '[]',
    
    -- 版本控制
    version INTEGER DEFAULT 1,
    
    -- 元数据
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_registries_user_id ON registries(user_id);
CREATE INDEX idx_registries_updated_at ON registries(updated_at);
```

**Registry JSON 结构示例**:
```json
{
  "pages": [
    {
      "page_id": "date",
      "type": "builtin",
      "visible": true,
      "order": 0,
      "template": "standard",
      "config": {
        "show_lunar": true,
        "font_size": "large"
      }
    },
    {
      "page_id": "stock_simple",
      "type": "custom",
      "visible": true,
      "order": 8,
      "created_by": "agent",
      "created_at": "2026-03-13T10:30:00Z",
      "template": "center_large",
      "config": {
        "data_source": "stock_api",
        "symbol": "AAPL"
      },
      "layout": {
        "elements": [...]
      }
    }
  ],
  "templates": [
    {
      "template_id": "center_large",
      "description": "居中大字：适合单数据展示",
      "slots": ["main"]
    }
  ]
}
```

### 3.4 配对会话表 (pairing_sessions)

```sql
CREATE TABLE pairing_sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id VARCHAR(17) NOT NULL,
    
    -- 配对码
    pairing_code VARCHAR(6) NOT NULL,  -- 6位数字
    
    -- 状态
    status VARCHAR(20) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'confirmed', 'expired', 'cancelled')),
    
    -- WiFi配置（AP模式时设备提交）
    wifi_ssid VARCHAR(100),
    wifi_password_encrypted TEXT,  -- 加密存储
    
    -- 时间控制
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE DEFAULT (NOW() + INTERVAL '10 minutes'),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    
    -- 确认信息
    confirmed_by UUID REFERENCES users(user_id),
    assigned_user_id UUID REFERENCES users(user_id),
    
    -- 设备信息
    device_name VARCHAR(100),
    firmware_version VARCHAR(20)
);

CREATE INDEX idx_pairing_code ON pairing_sessions(pairing_code) 
    WHERE status = 'pending';
CREATE INDEX idx_pairing_device ON pairing_sessions(device_id);
CREATE INDEX idx_pairing_expires ON pairing_sessions(expires_at);

-- 自动清理过期会话
CREATE OR REPLACE FUNCTION expire_old_sessions()
RETURNS void AS $$
BEGIN
    UPDATE pairing_sessions 
    SET status = 'expired' 
    WHERE status = 'pending' AND expires_at < NOW();
END;
$$ LANGUAGE plpgsql;
```

### 3.5 同步日志表 (sync_logs)

```sql
CREATE TABLE sync_logs (
    log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    device_id VARCHAR(17) REFERENCES devices(device_id) ON DELETE CASCADE,
    
    -- 同步信息
    sync_type VARCHAR(20) CHECK (sync_type IN ('push', 'pull', 'full', 'wakeup')),
    -- wakeup: RTC定时唤醒或按键唤醒时触发的同步
    status VARCHAR(20) CHECK (status IN ('success', 'failed', 'conflict')),
    
    -- 详情
    details JSONB DEFAULT '{}',
    -- 示例: {"uploaded_ops": 5, "downloaded_items": 10, "conflicts": 0, "duration_ms": 1500}
    
    -- 错误信息
    error_code VARCHAR(20),
    error_message TEXT,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_sync_logs_user ON sync_logs(user_id);
CREATE INDEX idx_sync_logs_device ON sync_logs(device_id);
CREATE INDEX idx_sync_logs_created ON sync_logs(created_at DESC);

-- 保留最近1000条日志，自动清理
CREATE OR REPLACE FUNCTION cleanup_old_sync_logs()
RETURNS void AS $$
BEGIN
    DELETE FROM sync_logs 
    WHERE log_id NOT IN (
        SELECT log_id FROM sync_logs 
        ORDER BY created_at DESC 
        LIMIT 1000
    );
END;
$$ LANGUAGE plpgsql;
```

### 3.6 离线操作队列表 (offline_operations)

```sql
CREATE TABLE offline_operations (
    op_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    device_id VARCHAR(17) REFERENCES devices(device_id) ON DELETE CASCADE,
    
    -- 操作内容
    op_type VARCHAR(20) CHECK (op_type IN ('CREATE', 'UPDATE', 'DELETE')),
    entity_type VARCHAR(20) CHECK (entity_type IN ('todo', 'countdown', 'event', 'preference')),
    entity_id VARCHAR(100),
    data JSONB,
    
    -- 状态
    status VARCHAR(20) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'synced', 'failed', 'conflict')),
    
    -- 重试
    retry_count INTEGER DEFAULT 0,
    last_retry_at TIMESTAMP WITH TIME ZONE,
    
    -- 时间戳
    device_timestamp BIGINT NOT NULL,  -- 设备端时间戳（用于排序）
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_offline_ops_user ON offline_operations(user_id);
CREATE INDEX idx_offline_ops_device ON offline_operations(device_id);
CREATE INDEX idx_offline_ops_status ON offline_operations(status) WHERE status = 'pending';
CREATE INDEX idx_offline_ops_timestamp ON offline_operations(device_timestamp);
```

---

## 4. Redis 数据结构

### 4.1 设备连接池

```
Key: device:connection:{device_id}
Type: Hash
TTL: 300s (5分钟，活跃期自动续期)
Value: {
    "ws_conn_id": "uuid",
    "connected_at": "timestamp",
    "ip": "192.168.1.100",
    "firmware": "1.0.0"
}

Key: device:connections (Set)
Value: [device_id_1, device_id_2, ...]
```

### 4.2 在线用户设备映射

```
Key: user:devices:{user_id}
Type: Set
Value: [device_id_1, device_id_2, ...]
TTL: 无（持久化）
```

### 4.3 配对会话缓存

```
Key: pairing:code:{pairing_code}
Type: String
TTL: 600s (10分钟)
Value: {session_id}

Key: pairing:session:{session_id}
Type: Hash
TTL: 600s
Value: {
    "device_id": "AA:BB:CC:DD:EE:FF",
    "status": "pending",
    "created_at": "timestamp"
}
```

### 4.4 Registry 缓存

```
Key: registry:{user_id}
Type: String (JSON)
TTL: 3600s (1小时，变更时更新)
Value: 完整的 registry JSON

Key: registry:version:{user_id}
Type: String
Value: "42" (版本号，每次变更+1)
```

### 4.5 同步锁（防止并发冲突）

```
Key: sync:lock:{user_id}
Type: String
TTL: 30s
Value: device_id (获取锁的设备)
```

---

## 5. 数据隔离策略

### 5.1 用户级隔离

所有用户相关数据通过 `user_id` 外键关联：

```sql
-- 查询某用户的所有设备
SELECT * FROM devices WHERE user_id = ?;

-- 查询某用户的Registry
SELECT * FROM registries WHERE user_id = ?;

-- 查询某用户的同步日志
SELECT * FROM sync_logs WHERE user_id = ? ORDER BY created_at DESC LIMIT 100;
```

### 5.2 设备级隔离

设备只能访问自己的数据：

```python
# 设备认证时验证
def authenticate_device(device_id, token):
    user_id = get_user_by_device(device_id)
    if not verify_token(token, user_id):
        raise Unauthorized()
    return user_id
```

### 5.3 同步隔离

同一用户的多个设备共享 Registry，但各自有独立的连接状态和同步日志：

```
User A
├── Device 1 (在线) ──→ Registry A (共享)
├── Device 2 (离线) ──→ Registry A (共享)
└── Device 3 (在线) ──→ Registry A (共享)

User B
└── Device 4 ──→ Registry B (与User A完全隔离)
```

---

## 6. 索引优化建议

### 6.1 常用查询优化

```sql
-- 查询用户下所有在线设备
SELECT * FROM devices 
WHERE user_id = ? AND status = 'online';
-- 索引: (user_id, status)

-- 查询待处理的配对会话
SELECT * FROM pairing_sessions 
WHERE pairing_code = ? AND status = 'pending' AND expires_at > NOW();
-- 索引: (pairing_code, status, expires_at)

-- 查询用户最新的Registry版本
SELECT version FROM registries 
WHERE user_id = ?;
-- 索引: (user_id) [已有唯一索引]

-- 查询设备最近的同步记录
SELECT * FROM sync_logs 
WHERE device_id = ? ORDER BY created_at DESC LIMIT 10;
-- 索引: (device_id, created_at DESC)
```

### 6.2 分区建议（数据量大时）

```sql
-- sync_logs 按时间分区
CREATE TABLE sync_logs_partitioned (
    LIKE sync_logs INCLUDING ALL
) PARTITION BY RANGE (created_at);

-- 按月分区
CREATE TABLE sync_logs_y2026m03 PARTITION OF sync_logs_partitioned
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

---

## 7. 数据备份策略

| 数据类型 | 备份频率 | 保留周期 | 方式 |
|---------|---------|---------|------|
| users | 每日 | 30天 | pg_dump |
| devices | 每日 | 30天 | pg_dump |
| registries | 实时 | 7个版本 | 变更时触发 + 每日全量 |
| pairing_sessions | 不备份 | - | 临时数据 |
| sync_logs | 每日 | 7天 | pg_dump |
| offline_operations | 每日 | 7天 | pg_dump |

---

**数据库设计完成**。这套设计支持：
- ✅ 多用户隔离
- ✅ 用户下多设备管理
- ✅ 设备配对流程
- ✅ 用户级Registry同步
- ✅ 离线操作队列
- ✅ 完整的同步日志
