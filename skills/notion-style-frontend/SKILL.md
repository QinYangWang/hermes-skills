---
name: notion-style-frontend
description: 当编写前端页面或 UI 组件时，必须采用 Notion 风格的设计规范。
---

# Notion 风格前端开发规范

## 触发条件
在撰写以下任何前端代码时，必须将本 skill 作为设计约束：
- 网页、Web App、管理后台、文档工具
- React/Vue/Svelte 等前端框架的组件
- 任何需要 HTML/CSS 的界面

## 核心设计原则
1. **极简主义 (Minimalist)** — 减少装饰，内容为王。
2. **无干扰界面** — 避免浓烈颜色、深色边框、强烈阴影。
3. **块级思维 (Block-based)** — 页面由可拖拽的块/卡片组成，每个块独立且通透。
4. **呼吸感 (Breathing Room)** — 大量留白，元素之间有明确的视觉层次。

## 具体规范

### 1. 配色系统 (Color Tokens)
Notion 不用纯黑纯白，采用"灰色阶梯"与"柔和色" 。

| 用途 | 颜色值 | 说明 |
|------|--------|------|
| 主文字 | `#37352f` | 接近黑的深灰，不刺眼 |
| 次文字 | `#9b9a97` | 中性灰，用于描述、时间戳 |
| 背景色 | `#ffffff` | 纯白主内容区 |
| 侧边栏背景 | `#f7f6f3` | 微暖的米灰，赋予温感 |
| Hover 背景 | `rgba(55, 53, 47, 0.08)` | 极淡的灰色遮罩 |
| 边框色 | `rgba(55, 53, 47, 0.16)` | 几乎看不见的细边框 |
| 分隔线 | `rgba(55, 53, 47, 0.09)` | 更淡的分隔 |

**Dark Mode 变体**：
- 主文字：`rgba(255, 255, 255, 0.9)`
- 背景：`#191919`
- 侧边栏：`#202020`
- Hover：`rgba(255, 255, 255, 0.1)`

### 2. 字体与排版 (Typography)
- **字体族**：`幼圆 / -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif`
- **字重**：主体 `400`，标题 `600`，不使用过细 (`300`)
- **行高 (Line Height)**：正文 `1.5`，标题 `1.2`
- **字号阶梯**：
  - H1: `2.5rem` (40px)
  - H2: `1.875rem` (30px)
  - H3: `1.5rem` (24px)
  - 正文: `1rem` (16px)
  - 辅助文字: `0.875rem` (14px)
  - 很小（时间戳）: `0.75rem` (12px)

### 3. 间距与布局 (Spacing & Layout)
- **页面最大宽度**：`900px` — `960px`，居中对齐。
- **页面内边距 (Padding)**：水平方向 `min 24px`。
- **块与块之间 (Block Gap)**：`8px`（紧凑）。
- **大段落间距 (Section Gap)**：`24px`。
- **边距应用“整数倍数法”**：以 `4px` 为基础单位。

### 4. 布局与页面结构 (Layout & Architecture)
Notion 的界面不是组件的堆叠，而是一套有严格组织规律的空间系统。

#### 整体框架
- **两栏式（或三栏式）：左侧固定/可折叠边栏 + 右侧主内容区**
- **边栏宽度**：标准 `240px`，支持用户拖拽调整（约 `200px` — `320px`）
- **顶部无传统导航栏**：页面标题和面包屑直接放在内容区顶部
- **主内容区**：`flex: 1`，独立滚动，不带顶部固定导航

#### 侧边栏 (Sidebar)
- **背景色**：`#f7f6f3` (或 dark `#202020`)
- **顶部区域**：工作区名称 + 用户头像，配很小的 `chevron-down` (`12px`)
- **页面树 (Page Tree)**：无限层级的可折叠树状结构
  - 每个节点：展开三角 + emoji 图标 + 页面标题
  - 子页面缩进：`padding-left: 20px`
  - hover 整行高亮，右侧出现 `"..."` 操作按钮
  - 当前选中页：背景略深 (`rgba(55,53,47,0.14)`)
- **可拖拽排序**：支持拖拽节点重新排序或放入其他父页面
- **底部工具**：新建页面、垃圾筒（小字文字链接）

#### 页面头部 (Page Header)
- **面包屑 (Breadcrumbs)**：位于内容区最顶，字号 `14px`
  - 用 `>` 或 `/` 分隔，颜色非常淡（`#9b9a97`）
  - 每个节点可点击，hover 时可能出现下划线
- **页面封面图**（可选）：居中显示在内容区顶部，宽 `100%`，高 `200px` — `300px`，无圆角，与边缘无间距
- **页面图标 (Emoji)**：位于标题正上方或左侧，大小 `40px` — `48px`
- **页面标题**：H1 级别，`无边框输入框`，可直接点击编辑

#### 内容画布 (Content Canvas)
- **块级布局**：每个内容块是独立行级元素
  - hover 时左侧出现 `⋮⋮` 拖拽手柄（淡灰，`14px`）
  - 块与块之间 `gap: 8px`
  - 缩进层级用 `padding-left: 24px` 表示
- **编辑状态**：
  - 空块 placeholder："输入内容，或按下 '/' 查看命令"，颜色非常淡
  - 选中块时整行淡蓝色高亮
  - 拖拽时出现蓝色细线占位符
- **Slash Command 菜单**：在光标位置浮动弹出，宽度 `320px`

#### 响应式布局
- **桌面（≥1280px）**：侧边栏常驻
- **平板（768px — 1279px）**：侧边栏折叠成约 `60px` 图标栏，或隐藏由汉堡菜单触发
- **移动端（<768px）**：侧边栏变为全屏抽屉，主内容全宽，顶部只保留汉堡菜单按钮

#### 空白状态与加载
- **空白页面**：居中显示大 emoji 和淡灰色提示文字
- **加载状态**：使用骨架屏或内容淡入，避免旋转 loading spinner

### 5. 组件样式 (Components)
#### 按钮
- **主按钮**：背景 `#37352f`，文字 `#fff`，圆角 `6px`，内边距 `6px 12px`
- **次按钮**：背景透明，边框 `1px solid rgba(55,53,47,0.16)`，hover 时背景变 `#f7f6f3`
- **Ghost 按钮**：无边框，hover 时出现淡灰背景

#### 输入框 / 编辑器
- 无边框 (borderless)，聚焦时显示淡蓝色外发光 (`box-shadow: 0 0 0 1px rgba(35,131,226,0.57)`)
- placeholder 颜色: `rgba(55, 53, 47, 0.35)`
- 编辑器内容区使用紧凑的 `prose`。

#### 卡片 / Callout
- 背景色比侧边栏更淡一点（如 `#f7f6f3` / `#f1f1ef` 等）
- 左边颜色条 (`border-left: 4px solid`) 作为视觉标识
- 圆角 `3px`

#### Dropdown / Menu / Popover
- 背景 `#fff`
- 边框极细 (`1px solid rgba(55,53,47,0.09)`)
- 圆角 `8px`
- 项目 hover 背景 `rgba(55,53,47,0.06)`

### 6. 圆角与阴影 (Radius & Shadow)
- **内容块 (Block)**：`border-radius: 3px`
- **UI 控件 (Button/Input)**：`border-radius: 6px`
- **弹窗/下拉菜单**：`border-radius: 8px`
- **阴影调性**：不使用浓烈的 black shadow。可参考 Notion 级影：
  ```css
  box-shadow: rgba(15, 15, 15, 0.05) 0px 0px 0px 1px,
              rgba(15, 15, 15, 0.1) 0px 3px 6px,
              rgba(15, 15, 15, 0.2) 0px 9px 24px;
  ```

### 7. 交互与动效 (Interactions)
- **动效时长**：`150ms` — `200ms`
- **过渡函数**：`ease-in-out` 或 `cubic-bezier(0.165, 0.84, 0.44, 1)`
- **Hover 反馈**：颜色变化优先，避免大幅移动或缩放。
- **Focus 状态**：蓝色发光边框 (`rgb(35, 131, 226)`)，不使用默认的 `outline`。

## 技术实现指引

### Tailwind CSS 配置示例
```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        notion: {
          text: '#37352f',
          'text-light': '#9b9a97',
          bg: '#ffffff',
          sidebar: '#f7f6f3',
          border: 'rgba(55, 53, 47, 0.16)',
          hover: 'rgba(55, 53, 47, 0.08)',
          focus: 'rgb(35, 131, 226)',
        },
      },
      borderRadius: {
        'block': '3px',
        'ui': '6px',
        'menu': '8px',
      },
      maxWidth: {
        'page': '900px',
      },
      fontFamily: {
        sans: ['-apple-system', 'BlinkMacSystemFont', 'Segoe UI', 'Helvetica', 'Arial', 'sans-serif'],
      },
    },
  },
}
```

### CSS 变量方案
建议在 `:root` 中定义 CSS 变量，以便支持 Dark Mode：
```css
:root {
  --notion-bg: #ffffff;
  --notion-text: #37352f;
  --notion-sidebar: #f7f6f3;
  --notion-hover: rgba(55, 53, 47, 0.08);
  --notion-border: rgba(55, 53, 47, 0.16);
  --notion-focus: rgb(35, 131, 226);
}

[data-theme="dark"] {
  --notion-bg: #191919;
  --notion-text: rgba(255, 255, 255, 0.9);
  --notion-sidebar: #202020;
  --notion-hover: rgba(255, 255, 255, 0.1);
  --notion-border: rgba(255, 255, 255, 0.14);
  --notion-focus: rgb(35, 131, 226);
}
```

### 推荐组件库
- **shadcn/ui**：基础组件层首选，可复制修改源码匹配 Notion 风格。
- **Novel / BlockNote**：如需编辑器功能，建议直接集成这些 Notion-style 编辑器组件库。

## 检查清单
提交前确认：
- [ ] 背景是否过于花哨？ — 应以白/浅灰为主。
- [ ] 文字是否用了纯黑 `#000000`？ — 应改为 `#37352f`。
- [ ] 边框和阴影是否浓烈？ — 应追求"几乎看不见"。
- [ ] 主内容宽度是否过宽？ — 建议不超过 `960px`。
- [ ] 动效是否过于炫酷？ — 应短 (≠200ms) 且淡雅。
