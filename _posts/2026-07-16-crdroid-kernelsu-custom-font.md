---
layout: post
title: "在 crDroid + KernelSU Next 上刷入全局自定义字体：一次踩坑实录"
date: 2026-07-16 15:00:00 +0800
categories: 教程 Android
---

给 Android 手机换一套自定义 TTF 全局字体，听起来是个很基础的需求——找到字体文件、塞进系统、重启完事。但在 Android 16 + crDroid 12 + KernelSU Next 这套组合上，这件事比想象中复杂得多。这篇文章记录一次完整的排查过程，包括踩过的坑、走过的弯路，以及最终能用的方案。

## 目标与环境

- 系统：crDroid 12（基于 Android 16）
- Root 方案：KernelSU Next 3.0.0
- 需求：把系统里中文、英文、数字、emoji 全部换成同一个自定义 TTF
- 前提：不改真实系统分区文件，只用模块（overlay 挂载）实现，保证可逆

## 第一次尝试：直接替换 Roboto 文件

最直觉的做法：把 TTF 改名成 `Roboto-Regular.ttf`，塞进一个 KernelSU 模块，覆盖挂载到 `/system/fonts/`。

```
FontMod/
├── module.prop
└── system/
    └── fonts/
        └── Roboto-Regular.ttf
```

装上重启，效果是：**只有个别"素"App（比如终端模拟器）变了字体，系统UI和绝大多数App没反应。**

## 坑1：静态字体撞上可变字体轴

Android 的字体配置文件里，`Roboto-Regular.ttf` 实际上是个**可变字体（variable font）**，靠 `<axis tag="wght" stylevalue="700"/>` 这类声明，从一个文件里抽取不同粗细：

```xml
<font weight="700" style="normal">Roboto-Regular.ttf
    <axis tag="wght" stylevalue="700"/>
</font>
```

普通设计师做的 TTF 基本都是**静态字体**，没有 `fvar` 表，系统请求"从这个文件抽取 700 粗细"时会失败，进而回退到别的字体，导致大量文字显示不出自定义效果。

验证方法：

```bash
grep -a fvar /system/fonts/Roboto-Regular.ttf
```

没有输出，说明确实是静态字体。

**修复思路**：把这类 `<font>` 条目里的 `axis` 声明去掉，改成简单的静态映射：

```xml
<!-- 之前 -->
<font weight="700" style="normal">Roboto-Regular.ttf
    <axis tag="wght" stylevalue="700"/>
</font>

<!-- 之后 -->
<font weight="400" style="normal">Roboto-Regular.ttf</font>
```

## 坑2：改错了文件——fonts.xml 已经过时

一开始改的是 `/system/etc/fonts.xml`，改完发现毫无效果。往上翻这个文件的注释才发现：

```xml
<!-- DEPRECATED: This XML file is no longer a source of the font files installed in the system... -->
```

新版本 Android 真正生效的配置文件其实是 `/system/etc/font_fallback.xml`，格式完全不同（单个 `<font>` 标签靠 `supportedAxes` 属性代表可变字体，而不是多个 weight 条目）。`fonts.xml` 只是留给极少数直接解析该文件的第三方 App 用的历史遗留文件。

**教训**：改系统字体配置前，先确认目标文件到底是不是真正被系统读取的那份，可以用以下方式验证：

```bash
find / -name "fonts.xml" 2>/dev/null
find / -name "font_fallback.xml" 2>/dev/null
```

## 坑3：系统UI走的是另一套配置

改对 `font_fallback.xml` 之后，大部分App文字变了，但**系统设置菜单、通知栏这些系统UI依然纹丝不动**。

排查过程：
1. 确认覆盖挂载确实生效（`mount | grep overlay`）
2. 确认 `system_server` 进程也能看到挂载（`cat /proc/<pid>/mountinfo`）
3. 都正常，但系统UI还是没变——说明问题不在挂载机制上

继续深挖分区结构，发现关键文件：

```bash
find /product /system_ext /vendor -iname "*font*" 2>/dev/null
```

结果里有个 `/product/etc/fonts_customization.xml`——这才是**系统UI真正在用的字体配置**，走的是 Android 新版本 Material Design Expressive 排版体系，用的是完全独立的 `google-sans` / `google-sans-flex` 字体家族，跟 `font_fallback.xml` 里的 `sans-serif` 毫无关系。

```xml
<family customizationType="new-named-family" name="google-sans">
    <font weight="400" style="normal">GoogleSans-Regular.ttf
        <axis tag="wght" stylevalue="400"/>
    </font>
    ...
</family>
```

对应的字体文件放在 `/product/fonts/`，不是 `/system/fonts/`。

**最终结论**：这套系统里，字体配置分成三层：

| 文件 | 位置 | 作用 |
|---|---|---|
| `fonts.xml` | `/system/etc/` | 已过时，基本没用 |
| `font_fallback.xml` | `/system/etc/` | 普通文本渲染的核心配置 |
| `fonts_customization.xml` | `/product/etc/` | 系统UI（Material Expressive排版）专用 |

要做到真正的"全局"换字体，这三层（主要是后两层）都得处理，而且字体文件要分别放在 `/system/fonts/` 和 `/product/fonts/` 两个不同分区里。

## 弯路：激进精简配置引发新问题

中途借助 AI 工具（DeepSeek）辅助处理配置文件，得到一版能大部分生效的 `font_fallback.xml`，但代价是**把原文件里除了 sans-serif 之外的所有语言家族（中日韩、emoji、阿拉伯文等几十种）全部删空了**。

后果：
- 中文常用字能显示（因为主字体本身带中文）
- 生僻字变成空白（没有中文兜底字体可退）
- emoji 变成空心方块（emoji字体家族被删了）

**教训**：验证"改哪个文件是对的"和"给出一份不留后患的最终方案"是两件事。前者可以简化测试，但简化后的版本不能直接作为最终交付版本使用。

## 修复思路：保留原文件，只做精准替换

正确做法是：导出**完整的原始 `font_fallback.xml`**（连同其中几百个语言家族一起），只用脚本精准替换需要改的那几个家族（`sans-serif`、`sans-serif-condensed`、`roboto`），其余全部原样保留：

```python
import re

pattern = re.compile(
    r'(<family name="(sans-serif|sans-serif-condensed|roboto)">\s*)'
    r'<font supportedAxes="[^"]*">\s*Roboto-Regular\.ttf\s*(?:<axis[^/]*/>\s*)+</font>'
    r'(\s*</family>)'
)

def replacer(m):
    return f'{m.group(1)}<font weight="400" style="normal">Roboto-Regular.ttf</font>{m.group(3)}'

new_content = pattern.sub(replacer, content)
```

这样中日韩、emoji、其他语言的兜底字体都不受影响，只有目标家族被替换成自定义字体。`fonts_customization.xml` 里的 `google-sans` 系列也用同样思路处理。

## 遗漏的家族：roboto-flex

即使精准替换后，桌面 Launcher 的应用名称依然没变。原因是 `roboto-flex` 这个独立家族（专门给部分UI组件用的可变字体）没有一并处理：

```xml
<!-- 修复前 -->
<family name="roboto-flex">
    <font supportedAxes="wght">RobotoFlex-Regular.ttf
        <axis tag="wdth" stylevalue="100.0"/>
    </font>
</family>

<!-- 修复后：直接指向已替换好的Roboto-Regular.ttf，不用额外新增文件 -->
<family name="roboto-flex">
    <font weight="400" style="normal">Roboto-Regular.ttf</font>
</family>
```

## 两个无法解决的边缘限制

折腾到这一步，绝大部分场景都正常了，但剩下两个问题没能彻底解决，记录一下原因，避免以后重复排查：

**1. 个别 Unicode 符号（如 🉐 🆙）显示空白**

用 `fontTools` 检查发现，这两个字符在 `NotoColorEmoji.ttf` 里其实**有完整的彩色图层数据**（COLRv1格式），文件本身没问题。真正原因是 Android 的文字排版引擎（minikin）在判断"这段文字属于什么 script"这一步，没有把这两个字符归类到 `Zsye`（emoji专用script），导致它们没有走到 `lang="und-Zsye"` 声明的 emoji 家族上——这是系统框架层面写死的判断逻辑，没法通过修改 XML 配置解决。

**2. 第三方桌面 Launcher 的应用名称不受影响**

换回 crDroid 自带的原生 Launcher 测试后完全正常，说明问题出在第三方 Launcher 本身——很多第三方启动器会把字体资源直接打包在自己的 APK 内部，代码里写死引用，根本不读系统字体配置。这跟不少国产 App（比如微信）不吃系统字体是同一类问题，只能二选一：换回原生 Launcher，或者接受这个局限。

## 最终方案结构

```
FontMod/
├── module.prop
├── system/
│   ├── fonts/
│   │   └── Roboto-Regular.ttf          ← 自定义字体
│   └── etc/
│       └── font_fallback.xml           ← 保留原文件全部内容，只精准替换4个家族
└── product/
    ├── fonts/
    │   ├── GoogleSans-Regular.ttf      ← 自定义字体
    │   ├── GoogleSans-Italic.ttf       ← 自定义字体
    │   └── GoogleSansFlex-Regular.ttf  ← 自定义字体
    └── etc/
        └── fonts_customization.xml     ← 剥离GoogleSans系列的axis声明
```

另外需要提前装好 KernelSU 的 **meta-overlayfs** 元模块（新版本 KernelSU 引入的架构，模块要修改系统文件必须先有这个前提模块，否则挂载不会生效）。

## 写在最后

这次折腾最大的收获，其实不是字体本身，而是搞清楚了 Android 字体系统的分层逻辑：`/system/etc/fonts.xml`（过时）→ `/system/etc/font_fallback.xml`（文本渲染核心）→ `/product/etc/fonts_customization.xml`（系统UI专属排版）。如果一开始就知道要往 `/product` 分区查，能省下至少一半的排查时间。

以后再遇到"改了配置为什么不生效"这类问题，第一反应应该是先确认"这份文件到底是不是系统真正在读的那份"，而不是急着改内容。
