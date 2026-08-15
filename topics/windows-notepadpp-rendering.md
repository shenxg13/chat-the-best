# Windows 11 / Notepad++：窗口恢复后的 Theme / 重绘延迟

## 适用场景

Notepad++ 本身已经启动完成，Theme 也正常，但在以下操作后出现明显的视觉延迟：

```text
最小化 Notepad++
→ 切换到其他窗口
→ 再切回 Notepad++
→ 窗口先恢复，编辑区 / Theme 随后才完成绘制
```

这种现象优先按 **Scintilla 渲染 / 窗口重绘问题** 排查，而不是按 Theme XML 加载、磁盘 IO 或启动插件初始化问题排查。

## 已确认环境与现象

- 系统：Windows 11。
- 显卡：集成显卡 + NVIDIA 5080 独立显卡。
- 日常桌面默认走集显；独显主要在游戏等高性能场景才启用。
- 用户希望保持这种低功耗策略，不为了 Notepad++ 强制启用独显。
- 故障只在窗口最小化、切换焦点后恢复时明显，符合“恢复后的重绘延迟”特征，而不是首次启动 Theme 加载慢。

## Scintilla Rendering Mode A/B 结果

实际测试结果：

| Rendering Mode | 结果 |
|---|---|
| DirectWrite | 窗口恢复后有延迟 |
| DirectX 11 / DirectWrite | 仍有延迟 |
| GDI | 正常 |
| DirectWrite - Draw to GDI DC | 正常 |

这组结果非常有区分度：

```text
DirectWrite                 → 异常
DirectX 11 + DirectWrite    → 异常
GDI                         → 正常
DirectWrite → GDI DC        → 正常
```

因此可以基本排除：

- Theme XML 本身加载过慢；
- Notepad++ Theme 配置读取慢；
- 单纯的程序启动阶段问题；
- 仅由某一个 DirectX 版本造成的问题。

## 当前判断

最符合现象的范围是：

```text
Notepad++ / Scintilla
        ↓
DirectWrite / Direct2D 硬件绘制路径
        ↓
Windows 11 DWM / GPU 图形链路
        ↓
窗口恢复时重绘出现延迟
```

`Draw to GDI DC` 正常是关键证据：它说明 **DirectWrite 的文字布局 / 字体处理本身未必有问题**，更可疑的是 DirectWrite 最终直接走 GPU / Direct2D surface 的绘制、合成或窗口恢复链路。

双显卡环境有可能参与这个问题，例如集显负责日常桌面输出，而某些硬件加速绘制路径涉及 GPU 选择、surface 或 DWM 合成。但目前没有做“强制集显 vs 强制独显”的 A/B 测试，因此 **不能把双显卡直接定为根因**。

## 最终采用的配置

保持 Notepad++：

```text
Scintilla Rendering Mode
→ DirectWrite (Draw to GDI DC)
```

这是当前最合适的长期 workaround：

- 已实测窗口恢复不再出现延迟；
- 仍保留 DirectWrite 相关的字体布局 / 渲染能力；
- 避开当前环境下有问题的直接 GPU 绘制路径；
- 不需要为了 Notepad++ 强制唤醒 NVIDIA 独显；
- 不需要修改 Windows 全局 DWM、硬件加速或显卡策略。

如果字体观感没有额外问题，优先使用 `Draw to GDI DC`，无需退回纯 GDI。

## 后续若需要继续追根因

当前问题已经有稳定 workaround，没有必要为了定位而扩大系统改动。如果未来驱动、Windows 或 Notepad++ 更新后希望重新验证，可按下面顺序做最小化测试：

1. 重新测试 DirectWrite 是否仍能复现；
2. 分别强制 Notepad++ 使用集显、NVIDIA 独显，做 DirectWrite A/B；
3. 对比显卡驱动更新前后的行为；
4. 再考虑 Windows DWM / 硬件加速相关设置。

判断原则：如果只有直接硬件绘制模式异常，而 `GDI` 与 `Draw to GDI DC` 都稳定，就继续把排查重点放在 **DirectWrite/Direct2D → GPU → DWM** 这一段，而不是回头折腾 Theme 文件。
