---
title: 设备状态机与性能指标
description: 设备状态定义、状态转换规则、性能指标要求
type: module
scope: [device]
created: "2026-03-13"
updated: "2026-03-13"
dependencies:
  - ../05-hardware/interaction-input.md
  - ../03-protocols/sync-protocol.md
related:
  - ui-display.md
  - ../05-hardware/interaction-output.md
---

# 设备状态机与性能指标

## 1. 性能指标

### 1.1 响应时间要求
| 操作 | 目标 | 最大容忍 |
|------|------|---------|
| 按钮响应 | 50ms | 100ms |
| 页面切换 | 300ms | 500ms |
| 墨水屏刷新 | 1.5s | 2s |
| 语音触发 | 800ms | 1s |
| 录音上传 | 1s | 3s |
| ASR识别 | 2s | 5s |
| 整体语音流程 | 5s | 10s |
| 同步完成 | 3s | 5s |

### 6.2 流畅度要求
- 页面切换动画: 无（墨水屏不支持）
- 状态刷新: 15分钟自动刷新，或手动触发
- 连续操作: 支持每秒1次按钮操作

---

## 2.: 状态机完整定义

```cpp
enum DeviceState {
    // 待机状态
    STATE_IDLE,              // 空闲，显示当前页面
    
    // 按钮操作
    STATE_BUTTON_PRESS,      // 按钮按下，检测中
    STATE_BUTTON_LONG_PRESS, // 长按确认，开始语音
    
    // 语音流程
    STATE_VOICE_RECORDING,   // 录音中
    STATE_VOICE_UPLOADING,   // 上传音频
    STATE_VOICE_RECOGNIZING, // ASR识别中
    STATE_VOICE_EXECUTING,   // 执行Tool
    
    // 同步流程
    STATE_SYNCING,           // 同步数据中
    
    // 特殊状态
    STATE_ERROR,             // 错误状态
    STATE_POWER_ON,          // 开机中
    STATE_SHUTDOWN,          // 关机中
    
    // 电源管理
    STATE_DEEP_SLEEP         // 深度休眠
};

// 状态转换表
StateTransition transitions[] = {
    // 从IDLE状态
    {STATE_IDLE, EVENT_BUTTON_PRESS, STATE_BUTTON_PRESS},
    {STATE_IDLE, EVENT_SYNC_TRIGGER, STATE_SYNCING},
    {STATE_IDLE, EVENT_ERROR, STATE_ERROR},
    
    // 从BUTTON_PRESS状态
    {STATE_BUTTON_PRESS, EVENT_BUTTON_RELEASE_QUICK, STATE_IDLE},  // 单击执行页面切换
    {STATE_BUTTON_PRESS, EVENT_BUTTON_RELEASE_DOUBLE, STATE_IDLE}, // 双击执行反向切换
    {STATE_BUTTON_PRESS, EVENT_BUTTON_HOLD_800MS, STATE_BUTTON_LONG_PRESS},
    
    // 从BUTTON_LONG_PRESS状态
    {STATE_BUTTON_LONG_PRESS, EVENT_VOICE_START, STATE_VOICE_RECORDING},
    
    // 从VOICE_RECORDING状态
    {STATE_VOICE_RECORDING, EVENT_BUTTON_RELEASE, STATE_VOICE_UPLOADING},
    {STATE_VOICE_RECORDING, EVENT_TIMEOUT_10S, STATE_ERROR},
    
    // 从VOICE_UPLOADING状态
    {STATE_VOICE_UPLOADING, EVENT_UPLOAD_COMPLETE, STATE_VOICE_RECOGNIZING},
    {STATE_VOICE_UPLOADING, EVENT_UPLOAD_FAIL, STATE_ERROR},
    {STATE_VOICE_UPLOADING, EVENT_BUTTON_CLICK, STATE_IDLE}, // 取消
    
    // 从VOICE_RECOGNIZING状态
    {STATE_VOICE_RECOGNIZING, EVENT_RECOGNIZE_COMPLETE, STATE_VOICE_EXECUTING},
    {STATE_VOICE_RECOGNIZING, EVENT_RECOGNIZE_FAIL, STATE_ERROR},
    {STATE_VOICE_RECOGNIZING, EVENT_BUTTON_CLICK, STATE_IDLE}, // 取消
    
    // 从VOICE_EXECUTING状态
    {STATE_VOICE_EXECUTING, EVENT_EXECUTE_SUCCESS, STATE_IDLE},
    {STATE_VOICE_EXECUTING, EVENT_EXECUTE_FAIL, STATE_ERROR},
    
    // 从SYNCING状态
    {STATE_SYNCING, EVENT_SYNC_COMPLETE, STATE_IDLE},
    {STATE_SYNCING, EVENT_SYNC_FAIL, STATE_ERROR},
    
    // 从ERROR状态
    {STATE_ERROR, EVENT_BUTTON_CLICK, STATE_IDLE},
    {STATE_ERROR, EVENT_BUTTON_LONG_PRESS, STATE_IDLE}, // 重启
    
    // 深度休眠状态
    {STATE_IDLE, EVENT_TIMEOUT_5MIN, STATE_DEEP_SLEEP},         // 活跃期超时，进入休眠
    {STATE_DEEP_SLEEP, EVENT_BUTTON_PRESS, STATE_IDLE},         // 按键唤醒
    {STATE_DEEP_SLEEP, EVENT_RTC_WAKEUP, STATE_IDLE},           // RTC定时唤醒
};
```

---

