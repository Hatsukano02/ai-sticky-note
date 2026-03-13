---
title: 天气系统设计规范
description: 基于和风天气API，支持IP自动定位+WebUI手动设置，后端代理请求并提供缓存
type: module
scope: [backend, device]
created: "2026-03-13"
updated: "2026-03-13"
dependencies:
  - ../03-protocols/pairing-protocol.md
  - ../02-architecture/database-schema.md
related:
  - ../06-interface/mcp-tools.md
---

# 天气系统设计规范

## 1. 概述

基于和风天气 API 的多设备天气系统，支持 IP 自动定位 + WebUI 手动设置，后端代理请求并提供缓存。

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        WebUI 管理界面                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 天气配置面板                                              │  │
│  │ • 当前地区：上海市浦东新区（自动识别）                     │  │
│  │ • 重新定位按钮 → IP定位 → 更新地区                        │  │
│  │ • 手动输入：省/市/区下拉选择 或 文本输入                   │  │
│  │ • 坐标输入：可选（纬度/经度）                              │  │
│  │ • 和风API Key配置（JWT保护）                              │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS
┌──────────────────────────────▼──────────────────────────────────┐
│                         后端服务                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ WeatherService                                            │  │
│  │ • 接收设备天气请求（MQTT）                                │  │
│  │ • 检查缓存（Redis，TTL 15分钟）                           │  │
│  │ • 缓存未命中 → 请求和风API                                │  │
│  │ • 响应设备并缓存                                          │  │
│  │ • 支持按用户多地区（预留）                                │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ LocationService                                           │  │
│  │ • IP反查地区（公网IP → 经纬度/城市名）                    │  │
│  │ • 调用第三方IP定位API（如ipapi.co、百度地图）             │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS
┌──────────────────────────────▼──────────────────────────────────┐
│                      第三方服务                                 │
│  • 和风天气 API（实时天气）                                     │
│  • IP定位 API（可选多个供应商）                                 │
└─────────────────────────────────────────────────────────────────┘
                               ↑ MQTT
┌──────────────────────────────┼──────────────────────────────────┐
│                         设备端                                │  │
│  RTC唤醒/按键唤醒 → 请求天气 → 显示 → 休眠                    │  │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 地区配置策略（混合方案）

### 3.1 默认流程（自动定位）

```
设备首次启动/配对成功
    ↓
请求公网IP（httpbin.org/ip 或类似服务）
    ↓
IP发送给后端
    ↓
后端调用IP定位API
    ↓
获得：省、市、区、经纬度
    ↓
保存到设备配置 + Registry
    ↓
WebUI显示"当前地区：上海市浦东新区"
```

### 3.2 手动修改流程

```
用户在WebUI点击"修改地区"
    ↓
打开地区选择器
    ↓
方案A：省/市/区三级联动下拉
方案B：文本输入（支持模糊匹配）
方案C：地图选点（高级）
    ↓
选择后保存
    ↓
更新设备配置
    ↓
下次天气请求使用新区
```

### 3.3 重新定位

```
用户点击"重新定位"
    ↓
重新执行IP定位流程
    ↓
如地区变化，提示用户确认
    ↓
确认后更新配置
```

## 4. 数据模型

### 4.1 设备天气配置

```json
// /spiffs/config/weather.json
{
  "location": {
    "source": "auto",  // "auto" | "manual"
    "province": "上海市",
    "city": "上海市",
    "district": "浦东新区",
    "latitude": 31.2304,
    "longitude": 121.4737,
    "updated_at": "2026-03-13T10:00:00Z"
  },
  "display": {
    "template": "standard",  // standard | detail | minimal
    "show_forecast": true,
    "update_interval": 30  // 分钟，跟随RTC唤醒周期
  }
}
```

### 4.2 后端天气缓存（Redis）

```
Key: weather:{location_key}
TTL: 900s (15分钟)
Value: {
  "temperature": 24,
  "description": "多云",
  "icon": "cloudy",
  "humidity": 65,
  "wind": "东南风2级",
  "forecast": [...],
  "updated_at": "2026-03-13T10:00:00Z",
  "source": "qweather"
}

location_key: "上海_浦东新区" 或 "31.2304_121.4737"
```

### 4.3 数据库表

```sql
-- 用户天气配置（按用户存储）
CREATE TABLE user_weather_configs (
    config_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- 地区信息
    province VARCHAR(50),
    city VARCHAR(50),
    district VARCHAR(50),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    
    -- 配置来源
    location_source VARCHAR(20) DEFAULT 'auto',  -- auto | manual
    
    -- 显示设置
    template VARCHAR(20) DEFAULT 'standard',
    show_forecast BOOLEAN DEFAULT true,
    
    -- 更新时间
    location_updated_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_weather_config_user ON user_weather_configs(user_id);
```

## 5. API 设计

### 5.1 后端接口

```python
# WeatherService API

@router.get("/api/weather")
async def get_weather(
    user_id: str = Depends(get_current_user),
    force_refresh: bool = False
) -> WeatherData:
    """
    获取当前天气（WebUI调用）
    优先返回缓存，force_refresh=True时强制刷新
    """

@router.post("/api/weather/location/auto")
async def auto_detect_location(
    ip: str = Body(...),
    user_id: str = Depends(get_current_user)
) -> LocationInfo:
    """
    IP自动定位
    """

@router.post("/api/weather/location/manual")
async def set_location_manual(
    province: str = Body(...),
    city: str = Body(...),
    district: str = Body(None),
    user_id: str = Depends(get_current_user)
) -> LocationInfo:
    """
    手动设置地区
    """

@router.get("/api/weather/config")
async def get_weather_config(
    user_id: str = Depends(get_current_user)
) -> WeatherConfig:
    """
    获取天气配置（WebUI展示）
    """
```

### 5.2 MCP Tools

```python
@router.get("/api/weather/device/{device_id}")
async def get_weather_for_device(
    device_id: str,
    user_id: str = Depends(verify_device)
) -> WeatherData:
    """
    设备请求天气（MQTT代理调用）
    根据device_id找到user_id，再获取该用户的天气配置
    """
```

## 6. MQTT 消息设计

### 6.1 设备请求天气

**主题**: `device/{device_id}/weather/request`

```json
{
  "timestamp": 1710420000,
  "device_version": 5
}
```

**响应主题**: `device/{device_id}/weather/response`

```json
{
  "status": "success",
  "data": {
    "temperature": 24,
    "description": "多云",
    "icon": "cloudy",
    "humidity": 65,
    "wind": "东南风2级",
    "update_time": "10:00"
  },
  "forecast": [
    {"day": "今天", "high": 26, "low": 18, "icon": "cloudy"},
    {"day": "明天", "high": 28, "low": 20, "icon": "sunny"}
  ],
  "location": "上海 浦东",
  "cached": false
}
```

### 6.2 天气更新推送

当后端主动刷新天气后，推送给在线设备：

**主题**: `device/{device_id}/weather/update`

```json
{
  "temperature": 25,
  "description": "晴",
  "icon": "sunny",
  "update_reason": "scheduled"
}
```

## 7. 和风 API 集成

### 7.1 API 选择

**实时天气**：
- 接口：`GET https://devapi.qweather.com/v7/weather/now`
- 参数：`location=121.4737,31.2304`（经纬度）或 `location=101020100`（城市ID）
- 免费额度：1000次/天（实时天气）
- 更新频率：15分钟

**天气预报**：
- 接口：`GET https://devapi.qweather.com/v7/weather/3d`
- 参数：同上
- 免费额度：1000次/天

### 7.2 请求频率控制

```
计算：
- 和风实时天气每15分钟更新一次
- 后端每15分钟请求一次，缓存15分钟
- 1000次/天 ÷ 24小时 ÷ 4次/小时 = 约10个用户

扩展方案：
- 用户量少：15分钟刷新
- 用户量增长：延长到30分钟或1小时
- 仅设备请求时才刷新（懒加载）
```

### 7.3 API Key 管理

```yaml
# docker-compose.yml
environment:
  - QWEATHER_KEY=${QWEATHER_KEY}
```

WebUI 管理员可配置（JWT保护）。

## 8. IP 定位服务

### 8.1 可选方案

| 服务 | 免费额度 | 精度 | 国内可用 |
|------|----------|------|----------|
| ipapi.co | 1000次/天 | 城市级 | ✓ |
| 百度地图 IP定位 | 需AK | 城市级 | ✓ |
| 高德地图 IP定位 | 需Key | 城市级 | ✓ |

### 8.2 实现示例（ipapi.co）

```python
async def get_location_by_ip(ip: str) -> LocationInfo:
    """通过IP获取地理位置"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://ipapi.co/{ip}/json/")
        data = response.json()
        
        return LocationInfo(
            city=data.get("city"),
            region=data.get("region"),
            latitude=data.get("latitude"),
            longitude=data.get("longitude")
        )
```

## 9. 天气页面配置（Registry）

```json
{
  "page_id": "weather",
  "type": "builtin",
  "visible": true,
  "order": 1,
  "template": "weather_card",
  "config": {
    "show_forecast": true,
    "forecast_days": 2,
    "update_interval": 30
  }
}
```

## 10. 错误处理

| 场景 | 处理策略 |
|------|----------|
| IP定位失败 | 提示用户手动设置地区 |
| 和风API限流 | 返回缓存数据，延长下次请求时间 |
| 和风API不可用 | 显示上次缓存 + "数据可能过期"提示 |
| 设备未配置地区 | 默认显示"请配置地区"提示页 |

## 11. WebUI 界面设计

```
┌─────────────────────────────────────────────┐
│ 天气设置                                    │
├─────────────────────────────────────────────┤
│                                             │
│ 当前地区：上海市浦东新区                    │
│ [重新定位]  [修改地区]                      │
│                                             │
│ ─────────────────────────────────────────   │
│                                             │
│ 显示设置                                    │
│ [✓] 显示天气预报                            │
│ 预报天数：[2天 ▼]                           │
│ 模板样式：[标准 ▼]                          │
│                                             │
│ ─────────────────────────────────────────   │
│                                             │
│ API配置（管理员）                           │
│ 和风API Key：[************************]     │
│ [保存]                                      │
│                                             │
└─────────────────────────────────────────────┘
```

---

**设计完成**。支持：
- ✅ IP自动定位 + 手动修改
- ✅ 后端代理 + 缓存
- ✅ 按用户存储配置
- ✅ WebUI完整管理
- ✅ 预留多地区扩展
