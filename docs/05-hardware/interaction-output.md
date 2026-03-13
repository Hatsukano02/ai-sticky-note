---
title: 硬件输出交互规范
description: LED PWM控制、7个基础提示音、音量控制、充电灯反馈
type: hardware
scope: [hardware]
created: "2026-03-13"
updated: "2026-03-13"
dependencies:
  - ../02-architecture/hardware-architecture.md
  - interaction-input.md
related:
  - ../04-modules/ui-display.md
  - ../04-modules/state-machine.md
---

# 硬件输出交互规范

## 1. 扬声器提示音

**硬件规格**: 8Ω 0.5W迷你喇叭

**设计原则**: 纯提示音方案，无语音播报。7个基础提示音覆盖所有交互场景。

#### 基础提示音定义

| 编号 | 音效 | 场景 | 参数 | 描述 |
|------|------|------|------|------|
| S1 | "滴" | 单击、页面切换、WiFi连接 | 50ms, 1kHz | 短促确认音 |
| S2 | "滴滴" | 双击 | 50ms+50ms间隔, 1kHz | 双音确认 |
| S3 | "叮~" | 长按开始录音 | 200ms, 800→1200Hz | 上升音，提示说话 |
| S4 | "咚" | 录音结束、进入休眠 | 100ms, 1200→800Hz | 下降音，表示结束 |
| S5 | "叮叮" | 成功、充电完成 | 150ms+150ms, 1500Hz | 双高音，清脆悦耳 |
| S6 | "嘟" | 失败、错误、低电量 | 300ms, 400Hz | 长低音，沉闷警示 |
| S7 | "叮~" | 唤醒 | 300ms, 600→1200Hz | 渐强上升，温和唤醒 |

#### 场景映射表

| 场景 | 音效 | 说明 |
|------|------|------|
| **单击（页面切换）** | S1 | 短促确认 |
| **双击（反向切换）** | S2 | 双音确认 |
| **长按开始录音** | S3 | 上升音提示请说话 |
| **录音结束** | S4 | 下降音表示结束 |
| **语音识别成功** | S5 | 执行成功确认 |
| **语音识别失败** | S6 | 失败提示 |
| **语音超时** | S6 | 无结果提示 |
| **同步完成** | S5 | 成功提示 |
| **同步失败** | S6 | 失败提示 |
| **唤醒** | S7 | 温和唤醒提示 |
| **进入休眠** | S4 | 复用结束音 |
| **WiFi连接成功** | S1 | 连接确认 |
| **配网成功** | S5 | 成功提示 |
| **配网失败** | S6 | 失败提示 |
| **配对码过期** | S6 | 需要重置提示 |
| **低电量警告** | S6 | 每分钟一次 |
| **WiFi错误** | S6 | 连接失败提示 |
| **充电插入** | S1 | 触发提示 |
| **充电完成** | S5 | 充满提示 |
| **充电异常** | S6 | 异常警示 |
| **重置确认（双击+长按）** | S2+S3 | 双击后长按开始音 |
| **恢复出厂** | S6×3 | 三连失败音警示 |

**注意**: 开机、RTC唤醒、配对等待等场景保持静默。

#### 技术参数

```cpp
// 音高定义 (Hz)
#define TONE_S1        1000   // 单击/双击/WiFi连接
#define TONE_S3_START  800    // 长按开始起始
#define TONE_S3_END    1200   // 长按结束
#define TONE_S4_START  1200   // 录音结束起始
#define TONE_S4_END    800    // 录音结束结束
#define TONE_S5        1500   // 成功高音
#define TONE_S6        400    // 失败低音
#define TONE_S7_START  600    // 唤醒起始
#define TONE_S7_END    1200   // 唤醒结束

// 时长定义 (ms)
#define DUR_S1         50     // 短音
#define DUR_S2_GAP     50     // 双音间隔
#define DUR_S3         200    // 长按开始
#define DUR_S4         100    // 录音结束
#define DUR_S5_GAP     150    // 成功音间隔

---

## 2. LED 交互规范

### 8.1 硬件配置

**主状态灯（1个白光LED）**
- 型号：3mm 白光 LED，直插
- 驱动：PWM 控制，1 个 GPIO
- 用途：按钮反馈、语音识别状态、系统状态

**充电灯（1个白光LED）**
- 型号：3mm 白光 LED，直插
- 驱动：数字控制，1 个 GPIO
- 用途：充电状态指示（独立，常亮或呼吸）

**PWM 参数**
```cpp
#define LED_PWM_FREQ      1000    // 1kHz，无频闪
#define LED_PWM_RES       8       // 8-bit 分辨率 (0-255)
#define LED_MIN_BRIGHTNESS 20     // 最小亮度 8%，避免完全熄灭误判
#define LED_MAX_BRIGHTNESS 255    // 最大亮度 100%

// 亮度等级
#define LED_OFF        0
#define LED_DIM        20         // 微亮 8%
#define LED_LOW        50         // 低亮 20%
#define LED_MEDIUM     128        // 中亮 50%
#define LED_HIGH       200        // 高亮 80%
#define LED_FULL       255        // 最亮 100%
```

### 8.2 主状态灯反馈模式

#### 按钮操作反馈

| 操作         | LED 表现                                        | 持续时间 |
| ------------ | ----------------------------------------------- | -------- |
| 单击         | 短闪 100ms（80%亮度）                           | 100ms    |
| 双击         | 双闪 100ms-间隔100ms-100ms                      | 300ms    |
| 长按开始     | 渐亮 500ms（0%→80%）                            | 500ms    |
| 长按保持     | 常亮 80%                                        | 直到松开 |

#### 语音识别状态（含 PWM 音量响应）

**录音阶段（ASR）**
```cpp
// 音量检测 → PWM 映射
void updateVoiceLED(int volumeLevel) {
    // volumeLevel: 0-100 (VAD 检测的音量百分比)
    int brightness;
    
    if (volumeLevel < 20) {
        brightness = LED_DIM;           // 20 (8%)
    } else if (volumeLevel < 60) {
        brightness = map(volumeLevel, 20, 60, LED_DIM, LED_MEDIUM);  // 20-128
    } else {
        brightness = map(volumeLevel, 60, 100, LED_MEDIUM, LED_FULL); // 128-255
    }
    
    ledcWrite(LED_CHANNEL, brightness);
}

// 响应延迟: <50ms
// 效果: 灯随声音"呼吸跳动"
```

**处理阶段（上传+识别+执行）**
```cpp
// 呼吸灯模式
void breathingLED() {
    // 周期: 1.5秒
    // 亮度范围: 30%-70%
    // 效果: 柔和等待感
    static float phase = 0;
    phase += 0.04;  // 1.5秒周期
    if (phase > TWO_PI) phase = 0;
    
    int brightness = LED_LOW + (LED_MEDIUM - LED_LOW) * (0.5 + 0.5 * sin(phase));
    ledcWrite(LED_CHANNEL, brightness);
}
```

**执行结果反馈**

| 结果 | LED 表现                                              |
| ---- | ----------------------------------------------------- |
| 成功 | 快闪两下：■■■ 200ms → 熄灭200ms → ■■■ 200ms           |
| 失败 | 慢闪三下：■■■ 500ms → 熄灭500ms → ■■■ 500ms → 熄灭500ms → ■■■ 500ms |
| 超时 | 快速渐暗：80% → 0% (1秒内)                            |

#### 系统状态指示

| 状态        | LED 表现                      | 含义               |
| ----------- | ----------------------------- | ------------------ |
| 深度休眠    | 熄灭                          | 省电模式           |
| RTC唤醒瞬间 | 渐亮200ms→熄灭                | 系统唤醒（无声）   |
| 同步中      | 慢闪 1Hz（亮500ms/灭500ms）   | 数据同步中         |
| 配网模式    | 双闪-停顿-双闪（2秒周期）     | 等待配对           |
| 低电量      | 慢呼吸 3秒周期（微弱20-40%）  | 电量<20%           |
| WiFi错误    | 快闪 2Hz（持续3次）           | 连接失败           |
| 系统错误    | 常亮 100%（3秒后自动恢复）    | 需要重启           |

### 8.3 充电灯反馈模式

充电灯为独立 LED，仅指示充电状态，与主状态灯分开。

| 充电状态 | LED 表现                    | 设计意图           |
| -------- | --------------------------- | ------------------ |
| 未充电   | 熄灭                        | 不干扰             |
| 充电中   | 慢速呼吸 2秒周期（20%-50%） | 柔和不刺眼         |
| 充满     | 常亮 50%                    | 明确指示，不过亮   |
| 充电异常 | 快闪 3Hz                    | 需检查充电器       |

```cpp
// 充电灯控制（数字 GPIO，非 PWM）
void updateChargeLED(ChargeStatus status) {
    switch(status) {
        case CHARGING:
            // 软件实现呼吸效果
            // 周期: 2秒，亮度: 20%-50%
            break;
        case CHARGED:
            digitalWrite(CHARGE_LED_PIN, HIGH);  // 常亮（外接限流电阻控制亮度）
            break;
        case NOT_CHARGING:
            digitalWrite(CHARGE_LED_PIN, LOW);
            break;
        case CHARGE_ERROR:
            // 快闪 3Hz
            break;
    }
}
```

### 8.4 典型场景示例

**场景1：语音添加待办**
```
用户长按按钮
    ↓
[LED] 渐亮至80%（提示：请说话）
    ↓
用户说"添加待办买牛奶"
    ↓
[LED] 随声音跳动（PWM 音量响应）
    ↓
用户松手
    ↓
[LED] 进入呼吸模式（处理中）
    ↓
识别成功
    ↓
[LED] 快闪两下（成功确认）
    ↓
回到页面显示，[LED] 熄灭
```

**场景2：夜间单击切换页面**
```
用户单击按钮
    ↓
[LED] 短闪100ms（确认）
    ↓
墨水屏刷新
    ↓
设备进入休眠，[LED] 熄灭
```

**场景3：充电过程**
```
插上充电器
    ↓
[充电灯] 开始呼吸（20%-50%，2秒周期）
    ↓
...充电中...
    ↓
充满
    ↓
[充电灯] 常亮50%
    ↓
拔下充电器
    ↓
[充电灯] 熄灭
```

### 8.5 实现建议

**硬件连接**
```cpp
// 主状态灯（PWM）
#define MAIN_LED_PIN    25      // GPIO25，支持 PWM
#define MAIN_LED_CHANNEL 0      // LEDC 通道 0

// 充电灯（数字）
#define CHARGE_LED_PIN  26      // GPIO26，数字输出
```

**LED 控制类**
```cpp
class LEDController {
public:
    void init();
    void update();  // 主循环中调用
    
    // 按钮反馈
    void onClick();
    void onDoubleClick();
    void onLongPressStart();
    void onLongPressEnd();
    
    // 语音状态
    void setVoiceRecording(int volume);
    void setVoiceProcessing();
    void onVoiceSuccess();
    void onVoiceFailed();
    void onVoiceTimeout();
    
    // 系统状态
    void setSystemState(SystemState state);
    
    // 充电状态
    void setChargeStatus(ChargeStatus status);
    
private:
    void setBrightness(uint8_t brightness);
    void blink(int count, int onTime, int offTime);
    void fade(uint8_t from, uint8_t to, int durationMs);
    void breathing(int periodMs, uint8_t minBrightness, uint8_t maxBrightness);
};
```

**注意事项**
1. **最低亮度限制**：设置为 20/255（8%），避免完全熄灭时用户误以为设备关机
2. **夜间友好**：最大亮度控制在 80%（200/255），充电灯充满时仅 50%
3. **响应延迟**：音量响应延迟 <50ms，确保与声音同步
4. **异常恢复**：系统错误状态 3 秒后自动熄灭，避免常亮耗电

---

