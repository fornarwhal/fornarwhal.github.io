---
title: 如何使用 DAPLink 调试器(保姆级教程)
date: 2026-08-03 16:39:35
updated: 2026-08-03 16:40:00
tags:
  - DAPLink
  - 调试
  - STM32
  - Keil
categories:
  - 嵌入式调试
---

# 如何使用 DAPLink 调试器

> 开发板调试太贵?ST-Link 缺货?试试 DAPLink——开源、便宜、免版权费,还自带串口。本文从接线到 Keil 配置,手把手带你跑通。

## 一、DAPLink 是什么

**DAPLink**(原 mbed CMSIS-DAP)是 ARM 官方开源的一款调试器固件方案。你可以花二三十块钱买到一块成品 DAPLink 小板,也可以自己用树莓派 Pico、STM32F103 等芯片刷固件自制。

它的核心能力:

| 功能 | 说明 |
|------|------|
| SWD 调试 | 支持下载 + 在线调试,和 ST-Link/J-Link 类似 |
| 虚拟串口(CDC) | 板载 USB 转串口,调试时直接 `printf` 输出,省一个 USB-TTL |
| HID 拖拽下载 | 部分固件支持像 U 盘一样拖 bin 文件进来自动烧录 |
| 开源免费 | 无版权费,不用担心商用授权问题 |

**常见形态**:
- 成品小扳手:几十元,带排针,接杜邦线即可
- 树莓派 Pico 刷固件:约 10 元成本,性价比之王
- 部分开发板自带(如某些国产板子板载 DAPLink)

## 二、接线(最关键的一步)

DAPLink 与目标板通过 **SWD 协议** 通信,最少只需要 **3 根线**:

| DAPLink 引脚 | 目标板引脚 | 说明 |
|:---:|:---:|---|
| SWCLK | SWCLK / TCK | 时钟线,必须接 |
| SWDIO | SWDIO / TMS | 数据线,必须接 |
| GND | GND | 地线,必须接(共地!) |
| 3.3V | 3.3V / VCC | 可选,给目标板供电(注意电流) |
| RST | NRST | 可选,复位线,连上更稳 |

```
┌──────────┐          ┌──────────┐
│ DAPLink  │          │  目标板   │
│          │  SWCLK   │          │
│  SWCLK ──┼──────────┼── SWCLK  │
│  SWDIO ──┼──────────┼── SWDIO  │
│  GND   ──┼──────────┼── GND    │
│  3.3V  ──┼───可选───┼── 3.3V   │
└──────────┘          └──────────┘
```

> ⚠️ **三个常见坑**:
> 1. **忘记共地**——GND 不接,调试器 100% 连不上目标板
> 2. **SWDIO/SWCLK 接反**——最常见的低级错误,仔细核对丝印
> 3. **目标板已上电时,3.3V 引脚别接**——两块电源打架容易烧板子;目标板自己供电时,不接 DAPLink 的 3.3V

## 三、驱动安装

**Win10/11 系统基本免驱**:DAPLink 使用 HID/USB CDC 类设备,Windows 会自动识别,插入后**设备管理器**里应出现:

- `Ports (COM & LPT)` → `mbed Serial Port (COMx)` —— 虚拟串口
- 调试器本体通常显示为 HID 设备,无需单独装驱动

如果设备管理器出现感叹号,试试:
1. 换一根**数据线**(很多线只能充电不能传数据,这是新手头号杀手)
2. 换一个 USB 口(直插主板后置口,别用前置面板)
3. 手动指定驱动:`mbedWinSerial_16466.exe` 安装串口驱动

## 四、Keil MDK 配置(核心步骤)

以 **STM32 + Keil MDK5** 为例:

### 1. 打开调试设置

`Options for Target`(魔术棒图标)→ 切到 **Debug** 标签页:

- 右上角下拉框选择:**CMSIS-DAP Debugger**(注意:不是 ST-Link,也不是 J-Link)
- 勾选 **Run to main()**(调试时自动跳到 main 入口)

### 2. 确认连接

点 Debug 页右侧的 **Settings** 按钮,弹出对话框:

- **Debug 标签页**:
  - `Port` 选 **SW**(SWD 协议)
  - `Max Clock` 建议先选低一点(如 1MHz)测试,稳定后再调高(4~10MHz)
  - 下方 `SW Device` 区域应能看到芯片 ID(如 STM32 显示 `ARM CoreSight SW-DP` + 设备 ID)
- **Flash Download 标签页**:
  - 勾选 **Reset and Run**(下载完自动复位运行)
  - 确认 Programming Algorithm 里有对应芯片型号(如 `STM32F1xx Flash`)

> 如果 Settings 里 `SW Device` 是空的,回第二步查接线。

### 3. 下载调试

- **下载**:点工具栏 `LOAD` 按钮(或 F8),状态栏出现 `Flash Load: ... OK` 即烧录成功
- **调试**:点 `DEBUG`(或 Ctrl+F5)进入调试界面,可单步、断点、查看变量
- **全速运行**:F5,此时虚拟串口应能输出 `printf` 信息

### 4. 重定向 printf 到串口(可选但强烈推荐)

在工程里加入:

```c
#include <stdio.h>

// 重定向 fputc 到 USART1,配合 DAPLink 虚拟串口实现 printf 调试
int fputc(int ch, FILE *f)
{
    // 此处填入你的串口发送函数,例如:
    // HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}
```

然后在电脑上打开串口助手,选择 DAPLink 的 COM 口,波特率与工程串口一致(通常 115200),即可实时看到调试输出——**调试体验直接起飞**。

## 五、常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| SW Device 空、报 No target connected | 接线问题或未共地 | 检查 SWDIO/SWCLK/GND 三根线,确认目标板上电 |
| 下载报 RDDI-DAP Error | 目标芯片被锁/供电不稳 | 按住复位键点下载,或先擦除整个芯片 |
| 烧录成功但程序不跑 | Reset and Run 未勾选 | Debug→Settings→Flash Download 勾选 Reset and Run |
| 识别不到串口 | 数据线问题 | 换能传数据的线,换 USB 口 |
| 高速率连接不稳定 | 线太长/杜邦线干扰 | 降 Max Clock 到 1MHz,或换短粗的线 |
| 调试时无法全速运行 | 断点太多/看门狗没关 | 调试阶段先关闭 IWDG 看门狗 |

## 六、DAPLink vs ST-Link vs J-Link

| 对比项 | DAPLink | ST-Link | J-Link |
|--------|---------|---------|--------|
| 价格 | 20~50 元 | 30~80 元 | 几百到上万 |
| 开源 | ✅ | ❌ | ❌ |
| 虚拟串口 | ✅ 自带 | 部分 | 需单独买 |
| Keil 支持 | CMSIS-DAP | 原生 | 原生 |
| 商用授权 | 免费 | 需确认 | 需付费 |
| 适合场景 | 学习/个人项目 | ST 芯片 | 量产/复杂调试 |

**结论**:个人学习、日常开发,DAPLink 完全够用;便宜、开源、还送串口,是性价比极高的入门选择。等以后需要 J-Flash、RTT 这类高级功能,再考虑上 J-Link 也不迟。

## 七、小结

跑通 DAPLink 调试,记住四步:

1. **接线**:SWCLK + SWDIO + GND,别忘了共地
2. **选型**:Keil 里 Debug 选 `CMSIS-DAP Debugger`
3. **验证**:Settings 里看到 SW Device 芯片 ID 才算连上
4. **享受**:下载、断点、printf,一气呵成

---

*如果你在配置过程中遇到问题,欢迎在评论区留言讨论。*
