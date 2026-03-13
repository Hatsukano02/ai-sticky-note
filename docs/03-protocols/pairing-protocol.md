---
title: 设备配对协议
description: ESP32首次启动、AP模式配网、配对码生成、MQTT配对流程、安全机制
type: protocol
scope: [backend, device, network]
created: "2026-03-13"
updated: "2026-03-13"
dependencies:
  - ../02-architecture/system-architecture.md
  - ../02-architecture/database-schema.md
related:
  - sync-protocol.md
  - ../04-modules/weather.md
---

# 设备配对协议

## 1. 概述

定义 ESP32 设备首次启动、WiFi配置、后端绑定的完整流程。采用 **AP模式配网** 方案，无需额外输入设备。

---

## 2. 配对模式状态机

```
                    ┌─────────────┐
                    │   首次上电   │
                    └──────┬──────┘
                           │
                           ▼
              ┌────────────────────────┐
              │ 检查 /spiffs/config/   │
              │ wifi.json 是否存在     │
              └───────────┬────────────┘
                          │
           ┌──────────────┼──────────────┐
           │有WiFi配置    │              │无WiFi配置
           ▼              │              ▼
    ┌─────────────┐       │     ┌─────────────────┐
    │ 连接WiFi    │       │     │   进入AP模式    │
    │ 检查后端    │       │     │                 │
    │ 绑定状态    │       │     │ • 发射WiFi信号  │
    └──────┬──────┘       │     │ • 启动HTTP服务  │
           │              │     │ • 显示配置指引  │
    ┌──────┴──────┐       │     └────────┬────────┘
    │             │       │              │
    ▼已绑定       ▼未绑定  │              ▼
┌─────────┐  ┌─────────┐  │     ┌─────────────────┐
│正常模式  │  │配对模式  │  │     │ 用户连接WiFi    │
│启动同步  │  │显示配对码│  │     │ 访问配置页面    │
└─────────┘  └────┬────┘  │     └────────┬────────┘
                  │       │                │
                  └───────┘                ▼
                                  ┌─────────────────┐
                                  │ 提交WiFi配置    │
                                  │ • SSID          │
                                  │ • Password      │
                                  │ • Device Name   │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ 连接家庭WiFi    │
                                  │ 生成配对码      │
                                  │ MQTT上报后端    │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ 管理员Web确认   │
                                  │ 绑定到用户      │
                                  │ Registry下发    │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │    正常模式     │
                                  │ 保存Registry    │
                                  │ 开始同步        │
                                  └─────────────────┘
```

---

## 3. AP模式详细流程

### 3.1 设备端行为

**触发条件**: `/spiffs/config/wifi.json` 不存在或无效

**ESP32 AP模式配置**:
```cpp
// WiFi.softAPConfig(local_ip, gateway, subnet);
WiFi.softAPConfig(
    IPAddress(192, 168, 4, 1),    // AP IP
    IPAddress(192, 168, 4, 1),    // Gateway
    IPAddress(255, 255, 255, 0)   // Subnet
);

// SSID格式: AI-Sticky-{MAC后4位}
String ssid = "AI-Sticky-" + macAddress.substring(12, 14) + macAddress.substring(15, 17);
WiFi.softAP(ssid.c_str(), NULL);  // 无密码开放网络

// 启动DNS服务器，捕获所有域名指向192.168.4.1
// 实现"连接WiFi后自动弹窗"效果
```

**屏幕显示内容**:
```
┌──────────────────────┐
│    首次使用配置      │
│                      │
│ 1. 手机连接WiFi:     │
│    AI-Sticky-A3F2    │
│                      │
│ 2. 自动打开配置页    │
│    或访问:           │
│    192.168.4.1       │
│                      │
│ 3. 按提示完成设置    │
└──────────────────────┘
```

### 3.2 配置Web页面

**技术栈**: ESP32AsyncWebServer (轻量级)

**页面内容** (HTML内嵌在固件中):
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI便利贴 - 网络配置</title>
    <style>
        body { font-family: -apple-system, sans-serif; padding: 20px; max-width: 400px; margin: 0 auto; }
        h1 { color: #333; font-size: 24px; }
        .form-group { margin-bottom: 20px; }
        label { display: block; margin-bottom: 5px; color: #666; }
        input, select { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px; font-size: 16px; }
        button { width: 100%; padding: 15px; background: #007AFF; color: white; border: none; border-radius: 8px; font-size: 18px; }
        .info { background: #f0f0f0; padding: 15px; border-radius: 8px; margin-bottom: 20px; }
    </style>
</head>
<body>
    <h1>📝 AI便利贴配置</h1>
    
    <div class="info">
        <strong>设备ID:</strong> <span id="deviceId">AA:BB:CC:DD:EE:FF</span><br>
        <strong>固件版本:</strong> <span id="firmware">1.0.0</span>
    </div>
    
    <form id="configForm">
        <div class="form-group">
            <label>WiFi名称</label>
            <select id="ssid" name="ssid" required>
                <option value="">扫描中...</option>
            </select>
        </div>
        
        <div class="form-group">
            <label>WiFi密码</label>
            <input type="password" id="password" name="password" placeholder="请输入WiFi密码" required>
        </div>
        
        <div class="form-group">
            <label>设备名称（可选）</label>
            <input type="text" id="deviceName" name="device_name" placeholder="如：厨房便利贴">
        </div>
        
        <button type="submit">保存并连接</button>
    </form>
    
    <script>
        // 1. 加载可用WiFi列表
        fetch('/scan')
            .then(r => r.json())
            .then(networks => {
                const select = document.getElementById('ssid');
                select.innerHTML = networks.map(n => 
                    `<option value="${n.ssid}">${n.ssid} (${n.rssi}dBm)</option>`
                ).join('');
            });
        
        // 2. 提交配置
        document.getElementById('configForm').onsubmit = async (e) => {
            e.preventDefault();
            const formData = new FormData(e.target);
            
            const res = await fetch('/configure', {
                method: 'POST',
                body: JSON.stringify(Object.fromEntries(formData)),
                headers: {'Content-Type': 'application/json'}
            });
            
            if (res.ok) {
                alert('配置成功！设备正在连接网络...');
                document.body.innerHTML = '<h1>✅ 配置成功</h1><p>设备正在连接网络，请稍候...</p>';
            } else {
                alert('配置失败：' + await res.text());
            }
        };
    </script>
</body>
</html>
```

### 3.3 ESP32端点实现

```cpp
// GET /scan - 扫描周围WiFi
server.on("/scan", HTTP_GET, [](AsyncWebServerRequest *request) {
    int n = WiFi.scanNetworks();
    String json = "[";
    for (int i = 0; i < n; i++) {
        if (i > 0) json += ",";
        json += "{\"ssid\":\"" + WiFi.SSID(i) + "\",";
        json += "\"rssi\":" + String(WiFi.RSSI(i)) + "}";
    }
    json += "]";
    request->send(200, "application/json", json);
});

// POST /configure - 接收配置并连接
server.on("/configure", HTTP_POST, 
    [](AsyncWebServerRequest *request) {},  // 空的onRequest
    NULL,  // 空的onUpload
    [](AsyncWebServerRequest *request, uint8_t *data, size_t len, size_t index, size_t total) {
        // 解析JSON配置
        StaticJsonDocument<512> doc;
        deserializeJson(doc, data, len);
        
        String ssid = doc["ssid"];
        String password = doc["password"];
        String deviceName = doc["device_name"] | "便利贴";
        
        // 保存配置到SPIFFS
        saveWiFiConfig(ssid, password, deviceName);
        
        // 回复成功
        request->send(200, "text/plain", "OK");
        
        // 延迟2秒后重启，进入正常模式
        delay(2000);
        ESP.restart();
    }
);

// 捕获所有请求，重定向到配置页（Captive Portal）
server.onNotFound([](AsyncWebServerRequest *request) {
    request->redirect("http://192.168.4.1/");
});
```

---

## 4. 配对码生成与验证

### 4.1 配对流程时序

```
┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
│  Device  │         │  MQTT    │         │  Backend │         │  WebUI   │
└────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │                    │
     │ [WiFi连接成功]      │                    │                    │
     │                    │                    │                    │
     │──pairing/request──>│                    │                    │
     │  {                 │                    │                    │
     │    device_id,      │                    │                    │
     │    device_name,    │                    │                    │
     │    firmware_ver    │                    │                    │
     │  }                 │                    │                    │
     │                    │───────────────────>│                    │
     │                    │                    │ [生成6位配对码]     │
     │                    │                    │ [创建session]      │
     │                    │                    │                    │
     │<─pairing/code──────│                    │                    │
     │  {code: "123456"}  │                    │                    │
     │                    │                    │                    │
     │ [屏幕显示配对码]     │                    │                    │
     │                    │                    │                    │
     │                    │                    │<──管理员查看待配对设备──│
     │                    │                    │                    │
     │                    │                    │<──POST /api/pairing/confirm
     │                    │                    │   {code, user_id}  │
     │                    │                    │                    │
     │                    │                    │ [验证code]         │
     │                    │                    │ [绑定device→user]  │
     │                    │                    │                    │
     │<─pairing/confirmed──│                    │                    │
     │  {                 │                    │                    │
     │    status: "ok",   │                    │                    │
     │    user_id,        │                    │                    │
     │    registry        │                    │                    │
     │  }                 │                    │                    │
     │                    │                    │                    │
     │ [保存Registry]     │                    │                    │
     │ [进入正常模式]      │                    │                    │
     │                    │                    │                    │
```

### 4.2 配对码生成算法

```python
import random
import string
from datetime import datetime, timedelta

def generate_pairing_code(device_id: str) -> str:
    """
    生成6位数字配对码
    - 避免容易混淆的数字（0, 1, 可能混淆？不需要，数字而已）
    - 确保唯一性（检查Redis是否存在）
    - 有效期10分钟
    """
    # 生成6位随机数字
    while True:
        code = ''.join(random.choices(string.digits, k=6))
        
        # 检查是否已存在
        if not redis.exists(f"pairing:code:{code}"):
            break
    
    return code

# 后端处理配对请求
async def handle_pairing_request(device_id: str, device_name: str, firmware: str):
    # 生成配对码
    code = generate_pairing_code(device_id)
    
    # 创建session
    session_id = uuid4()
    expires_at = datetime.now() + timedelta(minutes=10)
    
    # 存入PostgreSQL
    await db.execute("""
        INSERT INTO pairing_sessions 
        (session_id, device_id, pairing_code, status, expires_at, device_name, firmware_version)
        VALUES ($1, $2, $3, 'pending', $4, $5, $6)
    """, session_id, device_id, code, expires_at, device_name, firmware)
    
    # 缓存到Redis
    redis.setex(f"pairing:code:{code}", 600, str(session_id))
    redis.hset(f"pairing:session:{session_id}", mapping={
        "device_id": device_id,
        "status": "pending",
        "created_at": datetime.now().isoformat()
    })
    redis.expire(f"pairing:session:{session_id}", 600)
    
    # 下发配对码到设备
    await mqtt.publish(f"pairing/code/{device_id}", json.dumps({
        "code": code,
        "expires_in": 600
    }))
    
    return code

# 管理员确认配对
async def confirm_pairing(code: str, admin_id: str, assign_user_id: str = None):
    # 验证配对码
    session_id = redis.get(f"pairing:code:{code}")
    if not session_id:
        raise ValueError("配对码无效或已过期")
    
    session_data = redis.hgetall(f"pairing:session:{session_id}")
    device_id = session_data["device_id"]
    
    # 如果没有指定用户，默认分配给管理员
    target_user_id = assign_user_id or admin_id
    
    # 更新数据库
    await db.execute("""
        UPDATE pairing_sessions 
        SET status = 'confirmed', confirmed_by = $1, confirmed_at = NOW(), assigned_user_id = $2
        WHERE session_id = $3
    """, admin_id, target_user_id, session_id)
    
    # 创建设备记录
    await db.execute("""
        INSERT INTO devices (device_id, user_id, device_name, status, paired_at, paired_by)
        VALUES ($1, $2, $3, 'online', NOW(), $4)
        ON CONFLICT (device_id) DO UPDATE 
        SET user_id = $2, paired_at = NOW(), paired_by = $4
    """, device_id, target_user_id, session_data.get("device_name", "便利贴"), admin_id)
    
    # 确保用户有Registry
    await ensure_user_registry(target_user_id)
    
    # 获取Registry
    registry = await get_registry(target_user_id)
    
    # 通知设备配对成功
    await mqtt.publish(f"pairing/confirmed/{device_id}", json.dumps({
        "status": "ok",
        "user_id": str(target_user_id),
        "registry": registry
    }))
    
    # 清理Redis
    redis.delete(f"pairing:code:{code}")
    redis.delete(f"pairing:session:{session_id}")
    
    return device_id
```

---

## 5. 设备重置流程

### 5.1 组合键定义

**双击 + 长按** (3秒内完成):
- 第一次单击：切换页面（正常行为）
- 在1秒内第二次单击：检测到"双击"
- 双击后立即长按超过2秒：触发重置

```cpp
// 按钮状态机
enum ButtonState {
    IDLE,
    FIRST_CLICK,
    DOUBLE_CLICKED,
    LONG_PRESS_RESET
};

void handleButton() {
    static unsigned long firstClickTime = 0;
    static unsigned long doubleClickTime = 0;
    static bool waitingSecondClick = false;
    
    if (digitalRead(BUTTON_PIN) == LOW) {
        // 按钮按下
        unsigned long pressTime = millis();
        
        if (waitingSecondClick && (pressTime - firstClickTime < 1000)) {
            // 双击检测成功，开始检测长按
            while (digitalRead(BUTTON_PIN) == LOW) {
                if (millis() - pressTime > 2000) {
                    // 长按超过2秒，触发重置
                    triggerReset();
                    return;
                }
                delay(10);
            }
            // 没有长按，正常双击行为（切换页面）
            switchPage();
            waitingSecondClick = false;
        } else {
            // 第一次单击
            firstClickTime = pressTime;
            waitingSecondClick = true;
        }
    }
}

void triggerReset() {
    // 显示重置确认
    display.clear();
    display.println("正在重置设备...");
    display.display();
    
    // 删除配置
    SPIFFS.remove("/config/wifi.json");
    SPIFFS.remove("/config/device.json");
    SPIFFS.remove("/data/registry.json");
    
    delay(1000);
    
    // 重启
    ESP.restart();
}
```

### 5.2 重置后流程

```
[双击+长按触发重置]
    ↓
[屏幕显示: "设备已重置，即将重启"]
    ↓
[删除所有配置文件]
    ↓
[重启]
    ↓
[进入AP模式] ←── 等同于首次启动
```

---

## 6. MQTT 主题定义

### 6.1 配对相关主题

| 主题 | 方向 | 描述 | Payload |
|------|------|------|---------|
| `pairing/request/{device_id}` | Device → Backend | 设备请求配对 | `{device_id, device_name, firmware}` |
| `pairing/code/{device_id}` | Backend → Device | 下发配对码 | `{code, expires_in}` |
| `pairing/confirmed/{device_id}` | Backend → Device | 配对确认 | `{status, user_id, registry}` |
| `pairing/cancelled/{device_id}` | Backend → Device | 配对取消 | `{reason}` |

### 6.2 设备管理主题

| 主题 | 方向 | 描述 | Payload |
|------|------|------|---------|
| `device/{device_id}/command` | Backend → Device | 远程命令 | `{cmd: "reboot/refresh/update"}` |
| `device/{device_id}/status` | Device → Backend | 状态上报 | `{online, battery, page}` |
| `device/{device_id}/wakeup` | Device → Backend | RTC/按键唤醒 | `{timestamp, reason: "rtc/button"}` |

---

## 7. 错误处理

### 7.1 配网失败

```cpp
// WiFi连接失败（配置错误）
if (WiFi.status() != WL_CONNECTED) {
    // 显示错误
    displayError("WiFi连接失败，请重新配置");
    
    // 删除错误配置
    SPIFFS.remove("/config/wifi.json");
    
    // 3秒后重启回AP模式
    delay(3000);
    ESP.restart();
}
```

### 7.2 配对码过期

```
[屏幕显示配对码]
    ↓
[10分钟内未配对]
    ↓
[后端自动过期]
    ↓
[屏幕显示: "配对码已过期，重启获取新码"]
    ↓
[用户双击+长按重置 / 自动重启]
```

### 7.3 后端无响应

```cpp
// 连接WiFi后，尝试连接后端
if (!connectToBackend()) {
    // 显示离线模式
    displayOfflineMode();
    
    // 使用本地缓存（如果有）
    if (loadLocalRegistry()) {
        enterNormalModeOffline();
    } else {
        // 没有缓存，只能显示默认页面
        showDefaultPage();
        // 每5分钟重试连接
    }
}
```

---

## 8. 安全考虑

### 8.1 AP模式安全

- **无密码开放网络**: 简化配网流程，但有被中间人攻击风险
- **缓解措施**:
  1. AP模式仅首次启动，时间窗口短
  2. 配置完成后立即关闭AP
  3. WiFi密码仅传输一次，不存储明文
  4. 建议: 在配置页面使用HTTPS（需要证书）或加密传输

### 8.2 配对码安全

- **短有效期**: 10分钟过期
- **一次性使用**: 确认后立即失效
- **随机生成**: 6位数字，防暴力破解
- **频率限制**: 单个设备每小时最多请求3次配对

### 8.3 设备认证

配对完成后，设备与后端通信需要:
1. **Device ID**: MAC地址（唯一标识）
2. **Token**: 配对时后端下发的JWT，存储在 `/spiffs/config/device.json`
3. **Token续期**: 定期刷新（7天有效期）

```json
// /spiffs/config/device.json
{
  "device_id": "AA:BB:CC:DD:EE:FF",
  "user_id": "uuid",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "token_expires": 1713000000,
  "paired_at": "2026-03-13T12:00:00Z"
}
```

---

**设备配对规范完成**。这套流程实现了：
- ✅ AP模式配网（无需输入设备）
- ✅ 首次自动进入配对模式
- ✅ 6位配对码+管理员确认
- ✅ 用户级Registry下发
- ✅ 组合键重置机制
- ✅ 完整的错误处理
