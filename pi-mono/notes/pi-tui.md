# pi-tui：终端 UI 框架

> 分析日期：2026-04-19
> 源码版本：0.67.68
> 源码路径：`packages/tui/src/`
> 路径相对于 pi-mono 仓库根目录

**定位**：无内部依赖的终端 UI 框架，核心特性是**行级差分渲染（differential rendering）**和**同步输出（synchronized update）**。被 `pi-coding-agent` 和 `pi-web-ui` 使用。

---

## 1. 目录结构

```
packages/tui/src/
├── 核心框架
│   ├── index.ts               # 公共 API 导出
│   ├── tui.ts                 # TUI 类：差分渲染、Overlay、焦点
│   └── terminal.ts            # Terminal 接口 + ProcessTerminal
│
├── 输入系统
│   ├── keys.ts                # 键盘解析：Kitty 协议 + legacy ANSI
│   ├── keybindings.ts         # 全局键盘绑定
│   └── stdin-buffer.ts        # 转义序列完整性检测
│
├── 文本处理
│   ├── utils.ts               # visibleWidth / truncate / wrap（Unicode-aware）
│   └── fuzzy.ts               # 模糊匹配
│
├── 编辑相关
│   ├── editor-component.ts    # Editor 组件接口
│   ├── autocomplete.ts        # 自动补全协议
│   ├── kill-ring.ts           # Emacs 风格剪切环
│   └── undo-stack.ts          # 撤销栈（结构化克隆）
│
├── 终端功能
│   └── terminal-image.ts      # Kitty/iTerm2 图像协议
│
└── components/                # 13 个内置组件
    ├── editor.ts              # 多行编辑器
    ├── input.ts               # 单行输入
    ├── select-list.ts         # 可过滤下拉
    ├── text.ts                # 多行只读文本
    ├── markdown.ts            # Markdown → ANSI
    ├── box.ts                 # 盒模型容器
    ├── image.ts               # 图片显示
    ├── loader.ts              # 加载动画
    ├── cancellable-loader.ts  # 可取消 Loader
    ├── truncated-text.ts      # 截断文本
    ├── spacer.ts              # 弹性间隔
    ├── settings-list.ts       # 设置列表
    └── ...
```

---

## 2. 公共 API

核心类型：

```typescript
interface Component {
  render(width: number): string[]
  handleInput?(data: string): void
  wantsKeyRelease?: boolean  // 默认过滤 key release
  invalidate(): void
}

interface Focusable { focused: boolean }

class TUI {
  addChild(c) / removeChild(c)
  showOverlay(component, options): OverlayHandle
  setFocus(component)
  requestRender(force?)
  start() / stop()
}

class ProcessTerminal implements Terminal {
  start() / stop()
  write(data)
  get columns / rows / kittyProtocolActive
}
```

常数：

```typescript
CURSOR_MARKER = "\x1b_pi:c\x07"   // APC 序列，零宽，用于 IME 光标定位
```

---

## 3. 渲染模型（核心）

### 3.1 内部表示：字符串行数组

TUI 不维护字符网格（cell grid），而是**每行一个字符串**，字符串内混合可见文本和 ANSI/OSC/APC 转义码：

```typescript
class TUI {
  previousLines: string[] = []       // 上一次渲染的行
  cursorRow: number                  // 逻辑光标行
  hardwareCursorRow: number          // 硬件光标行（IME 用）
  maxLinesRendered: number           // 历史最大行数
  previousViewportTop: number        // 上一次 viewport 顶部
}
```

一行可以包含：
- Grapheme clusters（可能是宽字符、emoji、组合字符）
- CSI SGR（颜色）：`\x1b[31m`
- OSC 8 超链接：`\x1b]8;;url\x07`
- APC（光标标记）：`\x1b_pi:c\x07`
- 图像协议（Kitty `\x1b_G...` / iTerm2 `\x1b]1337;...`）

### 3.2 行级差分渲染

核心算法（`doRender()`）：

```
1. 收集：所有组件 render(width) → newLines[]
2. 合成：compositeOverlays() 把 overlay 叠到 newLines 上
3. 提取：extractCursorPosition() 找 CURSOR_MARKER 定位光标
4. 重置：applyLineResets() 每行末尾加 \x1b[0m\x1b]8;;\x07（防样式泄漏）
5. 对比 previousLines 和 newLines：
   firstChanged = 第一个差异行
   lastChanged  = 最后一个差异行
6. 光标移到 firstChanged，输出 firstChanged..lastChanged 之间的行
7. 更新 previousLines = newLines
```

**触发全重绘的场景**：
- 首次渲染
- 终端宽度变化（wrap 会变）
- 终端高度变化（Termux 例外——软键盘弹出）
- 内容大幅收缩 + `clearOnShrink=true`
- 首个变更行在旧 viewport 上方（无法差分修复）

### 3.3 ANSI 序列清单

光标：

```
\x1b[?25h         显示光标
\x1b[?25l         隐藏光标
\x1b[${n}A/B/C/D  上/下/右/左 N
\x1b[${n}G        绝对列 n
\r                回到列 0
```

清屏：

```
\x1b[2J\x1b[H     清屏 + 回原点
\x1b[3J           清 scrollback（xterm 扩展）
\x1b[2K            清当前行
```

样式重置：

```
\x1b[0m           重置 SGR
\x1b]8;;\x07      关闭 OSC 8 超链接
```

每行末尾都带这套重置，防 padding 继承颜色。

### 3.4 同步输出（关键技巧）

整个 buffer 包装：

```
\x1b[?2026h      # BEGIN synchronized update
  [完整渲染内容]
\x1b[?2026l      # END synchronized update
```

然后一次 `terminal.write(buffer)`。效果：
- 部分渲染不会被 SIGWINCH、网络延迟之类打断
- 终端收到完整帧后再提交显示
- **视觉上原子性**，消除闪烁

这不是驱动级同步（DECRQM 2026 是应用层协议），但足够好。

### 3.5 Viewport 和 Scrollback

```typescript
workingHeight = max(renderedLines, termHeight, minLinesForOverlays)
viewportTop = max(0, workingHeight - termHeight)
```

内容增长时自动滚动到底部；缩小时若开启 `clearOnShrink` 全重绘，否则残留行不清理（符合终端常规行为）。

### 3.6 光标标记（IME 支持）

组件在光标位置插入 `CURSOR_MARKER = \x1b_pi:c\x07`。TUI 扫描行，找到后：
1. 从行中去除标记
2. 发出光标定位序列，把硬件光标移到那里

APC 序列（`\x1b_...\x07`）是零宽，终端会忽略未知 APC。这样编辑器组件能告诉 TUI 光标在哪，不污染渲染。

---

## 4. 组件模型

### 4.1 无 layout 引擎

没有 flex / grid。布局靠：
- **Container 纵向拼接**：children 的 `render(width)` 结果按顺序拼接行
- **宽度约束**：父组件传 `width`，子组件必须产出每行 ≤ width
- **Overlay 绝对定位**：`OverlayOptions.anchor + offset`（支持百分比）

```typescript
// 典型 Container 实现
render(width: number): string[] {
  const lines: string[] = []
  for (const child of this.children) lines.push(...child.render(width))
  return lines
}
```

### 4.2 宽度处理（坑）

Unicode 宽度计算是这个框架最细致的部分：

- **ASCII 快速路径**：`isPrintableAscii` 直接用 `.length`
- **CJK 宽字符**：`east-asian-width` 库判定
- **Emoji**：grapheme segmenter 处理 ZWJ sequences、skin tones
- **区域指示符**（国旗 emoji）：保守按宽度 2 处理
- **ANSI 跳过**：单遍扫描时遇到 ESC `[` / `]` / `_` 跳整段
- **LRU 缓存 512 项**：避免重复计算

每个组件在返回行之前要 padding 到精确宽度：

```typescript
const visLen = visibleWidth(line)
line + " ".repeat(maxWidth - visLen)
```

### 4.3 缓存失效（无 dirty flag）

没有显式的 dirty 传播，组件自行缓存：

```typescript
class Text {
  render(width) {
    if (this.cachedLines && this.cachedWidth === width && this.cachedText === this.text) {
      return this.cachedLines
    }
    // 重算
    this.cachedLines = result
    return result
  }

  invalidate() {
    this.cachedLines = undefined  // 主题变更时调用
  }
}
```

### 4.4 焦点和输入分发

单焦点模型：

```typescript
class TUI {
  focusedComponent: Component | null

  setFocus(component) {
    if (isFocusable(this.focusedComponent)) this.focusedComponent.focused = false
    this.focusedComponent = component
    if (isFocusable(component)) component.focused = true
  }

  handleInput(data: string) {
    if (this.focusedComponent?.handleInput) {
      if (!isKeyRelease(data) || this.focusedComponent.wantsKeyRelease) {
        this.focusedComponent.handleInput(data)
        this.requestRender()
      }
    }
  }
}
```

---

## 5. 内置组件（13 个）

### 关键组件

**Editor**（components/editor.ts，最复杂）：
- 多行文本编辑、行号、可选语法高亮
- 光标导航（上下左右、词跳、行首/行尾）
- Emacs 快捷键（Ctrl+A/E/K/W/Y、Alt+F/B/D 等）
- Kill-ring 循环粘贴（yank-pop）
- UndoStack（structuredClone 克隆）
- 自动补全（AutocompleteProvider 协议）
- Paste 段落标记（防止 word wrap 分裂粘贴块）
- Grapheme-aware 光标移动

**SelectList**：模糊过滤、键盘导航、自动滚动保持选中可见、两列（label + description）

**Input**：单行版 Editor，同样支持 kill-ring / undo / bracketed paste

**Markdown**：基于 `marked` 解析器，主题函数控制样式，支持代码块语法高亮、OSC 8 链接、Kitty/iTerm2 图片内嵌

**Image**：自动缩放、保持宽高比、管理 Kitty 图像 ID

**Loader**：80ms 帧率的 Braille spinner（10 帧循环）

**Box**：padding + background 函数，缓存内容和背景样本

### 完整列表

| 组件 | 功能 |
|------|------|
| Editor | 多行编辑器（主角） |
| Input | 单行输入 |
| SelectList | 下拉选择 |
| Text | 多行只读文本 |
| Markdown | Markdown 渲染 |
| Box | Padding + 背景容器 |
| Image | Kitty/iTerm2 图片 |
| Loader | 加载动画 |
| CancellableLoader | 可按 Esc 取消的 Loader |
| TruncatedText | 省略号截断 |
| Spacer | 弹性间隔 |
| SettingsList | key-value 显示 |

---

## 6. 输入处理

### 6.1 Raw mode + bracketed paste

```typescript
// ProcessTerminal.start()
process.stdin.setRawMode(true)          // 关闭行缓冲、信号
process.stdin.setEncoding("utf8")
process.stdout.write("\x1b[?2004h")     // 启用 bracketed paste
// 查询 Kitty keyboard protocol 能力
```

### 6.2 Kitty Keyboard Protocol

格式：`\x1b[<codepoint>;<modifiers>:<event-type>u`

示例：
- `\x1b[99;5u` = Ctrl+C（99='c', 5=Ctrl）
- `\x1b[9;2u` = Shift+Tab
- `\x1b[c;m:3u` = c 的 release 事件

优先级：**Kitty 协议 > xterm modifyOtherKeys（tmux 用）> legacy**

### 6.3 类型安全的 KeyId

```typescript
type KeyId =
  | Letter | Digit | SymbolKey | SpecialKey
  | `${Modifier}+${KeyId}`  // 递归组合

// 例子
"c" | "ctrl+c" | "alt+shift+a" | "enter" | "up" | "f1"

matchesKey("\x1b[1;5A", "ctrl+up")  // true，适配多种编码
```

### 6.4 StdinBuffer：转义序列完整性

stdin 事件可能切片：`\x1b[2` → `0~`。StdinBuffer 检测：

```
isCompleteSequence(data) → "complete" | "incomplete" | "not-escape"

CSI: \x1b[...终止符（m|G|K|H|J 等）
OSC: \x1b]...\x07 或 ...\x1b\\
DCS: \x1b P...\x1b\\
APC: \x1b _...\x1b\\（Kitty 图形）
```

发出事件：
- `data` → 完整序列
- `paste` → 脱括号的 paste 内容（原子）
- `kitty-query-response` → 协议能力应答

### 6.5 Key Release 过滤

Kitty 协议上报按键释放事件，默认过滤。只有 `wantsKeyRelease: true` 的组件（编辑器常用于 repeat 去重）才收到。

---

## 7. 生命周期

```typescript
const terminal = new ProcessTerminal()
const tui = new TUI(terminal)

// 启动
tui.start()
  → terminal.start()        // raw mode + bracketed paste + Kitty query + resize listener
  → terminal.hideCursor()
  → tui.queryCellSize()     // 查询像素尺寸（给图片用）
  → tui.requestRender()

// 停止
await tui.stop()
  → terminal.stop()         // 关闭 bracketed paste / Kitty / StdinBuffer / raw mode
  → 移动光标到内容末尾
  → showCursor()
  → drainInput(1000, 50)    // 排空残留按键释放事件
```

**SIGINT / SIGTERM**：需要应用层注册：

```typescript
process.on("SIGINT", async () => { await tui.stop(); process.exit(0) })
```

**并发 stdout 的限制**：TUI 运行时其他 `console.log` 会搞乱差分状态。约定是——要输出就先 `tui.stop()`，输出完再重建。没有"暂停/恢复"原语。

---

## 8. 主题

主题不是对象结构，而是**函数集合**：

```typescript
interface MarkdownTheme {
  heading: (text) => string        // 返回带 ANSI 的字符串
  link: (text) => string
  code: (text) => string
  codeBlock: (text) => string
  quote: (text) => string
  bold: (text) => string
  italic: (text) => string
  strikethrough: (text) => string
  highlightCode?: (code, lang) => string[]  // 语法高亮
}
```

典型实现用 chalk 包装：

```typescript
{ selectedText: (t) => chalk.bold.blue(t), ... }
```

### AnsiCodeTracker

跟踪活跃 SGR 和 OSC 8 状态，用于：
- `wrapTextWithAnsi`：跨换行保留样式
- `breakLongWord`：字符级换行仍保留样式
- `compositeOverlays`：overlay 能继承背景样式

---

## 9. 推荐阅读顺序

| 顺序 | 文件 | 重点 |
|------|------|------|
| 1 | [index.ts](packages/tui/src/index.ts) | 公共 API 视图 |
| 2 | [tui.ts](packages/tui/src/tui.ts) | **核心**：差分渲染、overlay 合成、光标标记 |
| 3 | [terminal.ts](packages/tui/src/terminal.ts) | ProcessTerminal 生命周期、raw mode、Kitty 查询 |
| 4 | [keys.ts](packages/tui/src/keys.ts) | KeyId 类型系统 + 三种协议适配 |
| 5 | [utils.ts](packages/tui/src/utils.ts) | Unicode 宽度、截断、wrap、ANSI 处理 |
| 6 | [stdin-buffer.ts](packages/tui/src/stdin-buffer.ts) | 转义序列完整性检测 |
| 7 | [components/editor.ts](packages/tui/src/components/editor.ts) | 复杂组件的完整示例 |
| 8 | [terminal-image.ts](packages/tui/src/terminal-image.ts) | 可选：Kitty/iTerm2 图像协议 |

---

## 10. 设计亮点

| 亮点 | 说明 |
|------|------|
| **行级差分 + Synchronized Update** | 差分定位变更区间，整帧包在 `\x1b[?2026h/l` 里一次性写入，无闪烁 |
| **字符串行 + ANSI 内嵌** | 不用 cell grid，字符串行数组，ANSI 和文本共存，对比整行即可 |
| **APC CURSOR_MARKER** | 零宽 APC 序列嵌在行里定位光标，TUI 扫描后去除——支持 IME |
| **Unicode 单遍扫描 + 512 LRU** | isPrintableAscii 快速路径 + east-asian-width + grapheme segmenter，加缓存 |
| **无 dirty flag，组件自缓存** | 组件比较 `(width, content)` 和上次缓存，不匹配才重算；`invalidate()` 主动清缓存 |
| **类型安全 KeyId 递归** | `` `${Modifier}+${KeyId}` `` 模板字面量类型枚举所有修饰符组合 |
| **StdinBuffer 等待完整序列** | stdin 可能切片，缓冲直到识别出完整 CSI/OSC/DCS/APC 才派发 |
| **Bracketed paste 段落标记** | 编辑器把 paste 视为原子段落，word wrap / 光标移动不穿透 |
| **Kill-ring 和 Undo 分离** | kill-ring 累积（yank-pop 循环），undo 不累积——Emacs 语义 |
| **Pooled Tracker 复用** | `extractSegments` 用全局 pooledStyleTracker，清空重用避免分配 |
| **Termux 特例** | 软键盘改高度时不全重绘，只精确移动光标防历史重放 |
| **段落级 Overlay 合成** | 保留 base 行前后段落的样式继承，overlay 不割裂颜色/链接 |
| **Kitty protocol > xterm modifyOtherKeys > legacy** | 三级降级兼容，tmux / Ghostty / iTerm2 都能工作 |

---

## 11. 构建自己的 TUI 应用（最小示例）

```typescript
import { TUI, ProcessTerminal, Text, Editor } from "@mariozechner/pi-tui"

const terminal = new ProcessTerminal()
const tui = new TUI(terminal)

const title = new Text("Hello pi-tui")
const editor = new Editor()
editor.setValue("// start typing")

tui.addChild(title)
tui.addChild(editor)
tui.setFocus(editor)

tui.start()

process.on("SIGINT", async () => {
  await tui.stop()
  process.exit(0)
})
```

要做更复杂的东西：
- **模态对话框** → `tui.showOverlay(component, { anchor, offset })`
- **自定义组件** → 实现 `render(width): string[]` + 可选 `handleInput`
- **富文本** → 用 chalk/ansi-escapes 构造带样式的字符串，保证 `visibleWidth()` 正确
- **异步加载** → 起个 Loader 组件，异步完成后 `tui.removeChild(loader)` + `requestRender()`
