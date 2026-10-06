# 7. 前端架构 — 样式层

> 本层只回答一件事：**长什么样**。不涉及代码组织（第 5 节）与交互行为（第 6 节）。
> 技术选型：CSS Modules + CSS Custom Properties（设计令牌）+ PostCSS，**不引入运行时 CSS-in-JS**

---

## 7.1 样式技术决策

### 7.1.1 为什么不用运行时 CSS-in-JS

| 考量 | CSS Modules + 令牌 | 运行时 CSS-in-JS |
|---|---|---|
| 运行时开销 | 零（构建期生成） | 每次渲染需序列化与注入 |
| 首屏样式 | 静态 CSS，可预加载 | 需等 JS 执行后才注入，存在 FOUC |
| 播放器场景 | 样式与 JS 解耦，播放器 chunk 加载不阻塞样式 | 样式随组件 chunk 到达，可能有样式闪烁 |
| 主题切换 | 改 CSS 变量，一处生效，零重渲染 | 通常需重渲染组件树 |
| 包体积 | 无额外运行时 | 5–15 KB |
| 类型安全 | 需额外工具（typed-css-modules） | 原生 |

**播放器的样式闪烁是决定性因素**：视频播放器 chunk 通常 200KB+，加载期间若样式未就绪，用户会看到无样式的原生 `<video>` 控件闪现。静态 CSS 可在 `<head>` 预加载，彻底避免此问题。

### 7.1.2 样式分层

```
styles/
├── tokens/            # 设计令牌：唯一真相源
│   ├── color.tokens.css
│   ├── typography.tokens.css
│   ├── spacing.tokens.css
│   ├── radius.tokens.css
│   ├── shadow.tokens.css
│   ├── motion.tokens.css
│   ├── zindex.tokens.css
│   └── breakpoint.tokens.css
├── themes/            # 主题：只重定义语义令牌
│   ├── light.css
│   ├── dark.css
│   ├── sepia.css      # 护眼阅读主题
│   └── high-contrast.css
├── base/              # 全局基础样式
│   ├── reset.css
│   ├── global.css
│   └── a11y.css       # 焦点可见、屏幕阅读器工具类
└── features/          # 特定领域的复杂样式（非组件级）
    ├── player.css
    ├── danmaku.css
    └── reader.css
```

组件样式用 CSS Modules 与组件同目录：`ui/Button/Button.module.css`。

---

## 7.2 设计令牌（Design Tokens）

### 7.2.1 令牌的三层结构

这是整个样式体系的骨架。**组件只能引用语义层令牌，绝不引用原始层**：

```
第 1 层：原始令牌 (Primitive)  —— 调色板，无语义
   --gray-100: #f8f9fa;
   --gray-900: #1a1d21;
   --blue-500: #3b82f6;

第 2 层：语义令牌 (Semantic)   —— 描述用途，指向原始令牌
   --color-bg-surface: var(--gray-100);
   --color-text-primary: var(--gray-900);
   --color-action-primary: var(--blue-500);

第 3 层：组件令牌 (Component)  —— 组件私有，默认指向语义令牌
   --button-bg: var(--color-button-bg);
   --button-radius: var(--radius-md);
```

**组件只引用第 3 层或第 2 层。** 这样：
- 改主题 → 只需重定义第 2 层
- 改某个组件外观 → 改第 3 层，不影响其他组件
- 换调色板 → 改第 1 层，全局联动

### 7.2.2 颜色令牌

```css
/* styles/tokens/color.tokens.css */
:root {
  /* ── 第 1 层：原始调色板（仅此处出现字面色值） ── */
  --gray-0:   #ffffff;
  --gray-50:  #f8f9fa;
  --gray-100: #f1f3f5;
  --gray-200: #e9ecef;
  --gray-300: #dee2e6;
  --gray-400: #ced4da;
  --gray-500: #adb5bd;
  --gray-600: #868e96;
  --gray-700: #495057;
  --gray-800: #343a40;
  --gray-900: #212529;
  --gray-950: #0d1117;

  --blue-50:  #eff6ff;
  --blue-400: #60a5fa;
  --blue-500: #3b82f6;
  --blue-600: #2563eb;
  --blue-700: #1d4ed8;

  --green-500: #22c55e;
  --green-600: #16a34a;
  --amber-400: #fbbf24;
  --amber-500: #f59e0b;
  --orange-500:#f97316;
  --red-500:   #ef4444;
  --red-600:   #dc2626;
  --red-700:   #b91c1c;

  /* 风险等级专用色（审核工作台核心视觉语言） */
  --risk-none:     var(--gray-500);
  --risk-low:      var(--green-500);
  --risk-medium:   var(--amber-500);
  --risk-high:     var(--orange-500);
  --risk-critical: var(--red-600);

  /* ── 第 2 层：语义令牌（主题切换只改这一层） ── */
  --color-bg-canvas:      var(--gray-50);    /* 页面底色 */
  --color-bg-surface:     var(--gray-0);     /* 卡片、面板 */
  --color-bg-sunken:      var(--gray-100);   /* 内嵌区域 */
  --color-bg-overlay:     rgb(0 0 0 / 0.5);  /* 弹窗遮罩 */
  --color-bg-inverse:     var(--gray-900);   /* 深色块 */

  --color-text-primary:   var(--gray-900);
  --color-text-secondary: var(--gray-700);
  --color-text-tertiary:  var(--gray-600);
  --color-text-inverse:   var(--gray-0);
  --color-text-on-accent: var(--gray-0);

  --color-border-subtle:  var(--gray-200);
  --color-border-default: var(--gray-300);
  --color-border-strong:  var(--gray-400);

  --color-action-primary:       var(--blue-600);
  --color-action-primary-hover: var(--blue-700);
  --color-action-primary-text:  var(--gray-0);

  --color-feedback-success: var(--green-600);
  --color-feedback-warning: var(--amber-500);
  --color-feedback-danger:  var(--red-600);
  --color-feedback-info:    var(--blue-500);

  --color-focus-ring: var(--blue-500);
}
```

### 7.2.3 排版令牌

```css
/* styles/tokens/typography.tokens.css */
:root {
  /* 字体族：中英文分别指定，避免中文回退到宋体 */
  --font-sans: system-ui, -apple-system, 'Segoe UI', Roboto,
               'Helvetica Neue', 'PingFang SC', 'Hiragino Sans GB',
               'Microsoft YaHei', sans-serif;
  --font-serif: Georgia, 'Times New Roman', 'Songti SC', 'SimSun', serif;
  --font-mono: ui-monospace, 'SF Mono', Menlo, Consolas,
               'Liberation Mono', monospace;

  /* 字号阶梯：1.200 比率（小三度），兼顾紧凑与层级清晰 */
  --font-size-2xs: 0.6875rem;  /* 11px — 徽标、时间戳 */
  --font-size-xs:  0.75rem;    /* 12px — 辅助文字 */
  --font-size-sm:  0.875rem;   /* 14px — 次要正文、表单 */
  --font-size-md:  1rem;       /* 16px — 正文基准 */
  --font-size-lg:  1.125rem;   /* 18px — 强调正文 */
  --font-size-xl:  1.25rem;    /* 20px — 小标题 */
  --font-size-2xl: 1.5rem;     /* 24px — 标题 */
  --font-size-3xl: 1.875rem;   /* 30px — 页面标题 */
  --font-size-4xl: 2.25rem;    /* 36px — 大标题 */
  --font-size-5xl: 3rem;       /* 48px — 主视觉 */

  /* 行高：与字号配对使用 */
  --line-height-tight:  1.25;  /* 标题 */
  --line-height-snug:   1.4;   /* 紧凑正文 */
  --line-height-normal: 1.6;   /* 标准正文（中文阅读舒适） */
  --line-height-relaxed:1.8;   /* 宽松阅读 */
  --line-height-loose:  2.0;   /* 沉浸阅读 */

  /* 字重 */
  --font-weight-normal:   400;
  --font-weight-medium:   500;
  --font-weight-semibold: 600;
  --font-weight-bold:     700;

  /* 字间距：中文不宜过紧 */
  --letter-spacing-tight:  -0.01em;
  --letter-spacing-normal: 0;
  --letter-spacing-wide:   0.02em;
  --letter-spacing-wider:  0.05em;   /* 全大写小标签 */
}
```

**中文排版的关键决策**：
- 正文行高 `1.6`–`1.8`。西文常用 `1.5`，但中文笔画密集，需要更多呼吸空间。
- 不使用 `letter-spacing` 负值于中文正文（会挤压字形）。
- 字体栈中 `PingFang SC`（macOS）与 `Microsoft YaHei`（Windows）分别指定，避免落到默认宋体。

### 7.2.4 间距令牌

采用 **4px 基准的等比阶梯**，命名用 `size-N`（N 为 4px 的倍数）：

```css
/* styles/tokens/spacing.tokens.css */
:root {
  --space-0:  0;
  --space-1:  0.25rem;  /*  4px */
  --space-2:  0.5rem;   /*  8px */
  --space-3:  0.75rem;  /* 12px */
  --space-4:  1rem;     /* 16px — 基准间距 */
  --space-5:  1.25rem;  /* 20px */
  --space-6:  1.5rem;   /* 24px */
  --space-8:  2rem;     /* 32px */
  --space-10: 2.5rem;   /* 40px */
  --space-12: 3rem;     /* 48px */
  --space-16: 4rem;     /* 64px */
  --space-20: 5rem;     /* 80px */
  --space-24: 6rem;     /* 96px */

  /* 语义间距：用于布局节奏 */
  --gutter-page:      var(--space-4);   /* 页面左右内边距 */
  --gutter-card:      var(--space-4);   /* 卡片内边距 */
  --stack-tight:      var(--space-2);   /* 紧密元素间 */
  --stack-default:    var(--space-4);   /* 默认段落间 */
  --stack-section:    var(--space-8);   /* 区块间 */
  --stack-page:       var(--space-12);  /* 大区块间 */
}
```

**为什么用固定阶梯而非任意值**：任意值（如 `13px`、`18px`）会让界面失去韵律感。限定在阶梯内，间距自然形成视觉节奏。审查时若发现组件用了阶梯外的值，即视为设计缺陷。

### 7.2.5 圆角、阴影、动效

```css
/* styles/tokens/radius.tokens.css */
:root {
  --radius-none: 0;
  --radius-sm:   0.25rem;   /*  4px — 标签、小按钮 */
  --radius-md:   0.5rem;    /*  8px — 按钮、输入框 */
  --radius-lg:   0.75rem;   /* 12px — 卡片 */
  --radius-xl:   1rem;      /* 16px — 大面板 */
  --radius-2xl:  1.5rem;    /* 24px — 浮层 */
  --radius-full: 9999px;    /* 胶囊、头像 */
}

/* styles/tokens/shadow.tokens.css */
:root {
  /* 阴影用多层叠加，比单层大模糊更自然 */
  --shadow-xs: 0 1px 2px rgb(0 0 0 / 0.04);
  --shadow-sm: 0 1px 3px rgb(0 0 0 / 0.06), 0 1px 2px rgb(0 0 0 / 0.04);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.07), 0 2px 4px -2px rgb(0 0 0 / 0.05);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.08), 0 4px 6px -4px rgb(0 0 0 / 0.05);
  --shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.10), 0 8px 10px -6px rgb(0 0 0 / 0.06);
  --shadow-focus: 0 0 0 3px rgb(59 130 246 / 0.35);   /* 焦点环 */
}

/* styles/tokens/motion.tokens.css */
:root {
  --duration-instant: 80ms;    /* 悬停反馈 */
  --duration-fast:    150ms;   /* 按钮、开关 */
  --duration-normal:  250ms;   /* 抽屉、折叠 */
  --duration-slow:    400ms;   /* 页面过渡、模态 */

  /* 缓动：入场用 ease-out（先快后慢），出场用 ease-in */
  --ease-standard: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-decelerate: cubic-bezier(0, 0, 0.2, 1);
  --ease-accelerate: cubic-bezier(0.4, 0, 1, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);   /* 轻微回弹，用于弹窗 */
}

/* styles/tokens/zindex.tokens.css */
:root {
  /* 集中管理层级，杜绝 z-index: 9999 军备竞赛 */
  --z-base:       0;
  --z-sticky:     100;    /* 吸顶头部 */
  --z-player-ui:  200;    /* 播放器控件 */
  --z-danmaku:    300;    /* 弹幕层（位于控件**之上**，见下方说明） */
  --z-dropdown:   400;
  --z-drawer:     500;
  --z-modal:      600;
  --z-toast:      700;
  --z-tooltip:    800;
  --z-devtools:   900;
}
```

**弹幕层级与播放器控件的关系**：弹幕在 `--z-danmaku`(300)，播放控件在 `--z-player-ui`(200)。弹幕**覆盖在控件之上**是有意的——但这也意味着弹幕会遮挡进度条。解法：控件可见时给弹幕层加 `opacity: 0.3` 或限定弹幕区在画面顶部 80% 内，避开底部控件区。

---

## 7.3 主题系统

### 7.3.1 主题实现机制

主题**只重定义第 2 层语义令牌**，不触碰原始调色板。切换通过 `data-theme` 属性：

```css
/* styles/themes/dark.css */
[data-theme='dark'] {
  --color-bg-canvas:      var(--gray-950);
  --color-bg-surface:     var(--gray-900);
  --color-bg-sunken:      #161b22;
  --color-bg-overlay:     rgb(0 0 0 / 0.7);
  --color-bg-inverse:     var(--gray-100);

  --color-text-primary:   #e6edf3;
  --color-text-secondary: #adbac7;
  --color-text-tertiary:  #768390;
  --color-text-inverse:   var(--gray-900);

  --color-border-subtle:  #21262d;
  --color-border-default: #30363d;
  --color-border-strong:  #444c56;

  --color-action-primary:       var(--blue-500);   /* 深色下提亮 */
  --color-action-primary-hover: var(--blue-400);

  /* ⚠️ 按钮底色必须单独校准：
     白字压在 --blue-500 (#3b82f6) 上只有 3.7:1，**不满足 4.5:1**。
     所以按钮（含 hover 态）一律用 --blue-600，而链接、图标等
     "文字本身是彩色"的场景才用提亮后的 --blue-500。
     两者目标相反：按钮底色要够暗以衬托白字，文字色要够亮以衬托深底。 */
  --color-button-bg:       var(--blue-600);
  --color-button-bg-hover: var(--blue-600);   /* 深色下不再提亮，保对比度 */
  --color-button-bg-active: #1d4ed8;          /* blue-700，按下态更深 */

  /* 深色下阴影几乎不可见，改用边框强化层级 */
  --shadow-sm: 0 0 0 1px var(--color-border-subtle);
  --shadow-md: 0 0 0 1px var(--color-border-default);
  --shadow-lg: 0 0 0 1px var(--color-border-default), 0 8px 24px rgb(0 0 0 / 0.4);
}
```

浅色主题下两个按钮令牌同样需要定义（浅色下 `--blue-600` 已满足对比度，无需分化，但为保证令牌在两个主题中都存在，显式声明）：

```css
:root {
  --color-button-bg:       var(--blue-600);
  --color-button-bg-hover: var(--blue-700);
}
```

**深色模式的关键调整**：
- **不用纯黑**（`#000`）。纯黑与白色文字对比过强，长时间阅读产生光晕感。用 `#0d1117`。
- **不用纯白文字**。用 `#e6edf3`，降低对比至舒适区间。
- **蓝色需按角色分别校准**。这里仅针对 `--color-action-primary`（文字、图标、链接等"彩色前景"）：`--blue-600` 在深色背景上偏沉闷，改用 `--blue-500` 提升可读性。**注意：此规则不适用于按钮底色**——按钮上是白字，需要的是更暗的底色，见下方 `--color-button-bg` 的单独说明。
- **阴影改为边框**。深色背景下阴影不可见，层级靠边框表达。
- **风险色需重新校准**。深色背景下 `--red-600` 对比度不足，改用 `--red-500`。

### 7.3.2 护眼阅读主题

```css
/* styles/themes/sepia.css */
[data-theme='sepia'] {
  --color-bg-canvas:  #f4ecd8;
  --color-bg-surface: #faf3e3;
  --color-text-primary:   #3d3229;
  --color-text-secondary: #6b5d4f;
  --color-border-subtle:  #e0d5bd;
  --color-action-primary: #8b5e34;
}
```
米黄底 + 深棕字，降低蓝光与对比度，适合长时间阅读。

### 7.3.3 主题切换的交互与持久化

```
优先级（从高到低）
  ① 用户显式选择（localStorage / 服务端 readingPreferences）
  ② 系统偏好 prefers-color-scheme
  ③ 默认浅色
  ↓
切换逻辑
  ├─ 立即设置 document.documentElement.dataset.theme（无重渲染，纯 CSS 变量切换）
  ├─ 更新 <meta name="theme-color">（影响移动端浏览器地址栏颜色）
  └─ 持久化到 localStorage
  ↓
防止闪烁（FOUC）
  └─ 在 <head> 内联一段阻塞脚本，在 React 挂载前设置 data-theme
```

```html
<!-- index.html — 必须在任何样式表之前 -->
<script>
  (function () {
    try {
      var saved = localStorage.getItem('theme');
      var theme = saved === 'system' || !saved
        ? (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
        : saved;
      document.documentElement.dataset.theme = theme;
    } catch (e) {
      document.documentElement.dataset.theme = 'light';
    }
  })();
</script>
```

> 这段脚本必须内联且同步执行。若等 React 挂载后再设置主题，用户会看到浅色闪一下变深色——这是最常见的主题实现缺陷。

### 7.3.4 高对比度主题（无障碍）

```css
[data-theme='high-contrast'] {
  --color-text-primary:   #000000;
  --color-bg-surface:     #ffffff;
  --color-border-default: #000000;
  --color-action-primary: #0000ee;
  --color-focus-ring:     #000000;
}
[data-theme='high-contrast'] * {
  border-width: 2px;   /* 加粗边框提升可辨识度 */
}
```

---

## 7.4 布局系统

### 7.4.1 断点定义

```css
/* styles/tokens/breakpoint.tokens.css — 供文档参考；实际用 PostCSS 变量 */
:root {
  --bp-sm:  480px;    /* 大屏手机横屏 */
  --bp-md:  768px;    /* 平板竖屏 */
  --bp-lg:  1024px;   /* 平板横屏 / 小笔记本 */
  --bp-xl:  1280px;   /* 桌面 */
  --bp-2xl: 1536px;   /* 大桌面 */
}
```

PostCSS 自定义媒体查询：

```css
/* 使用语义化断点，不写裸像素值 */
@custom-media --md-down (max-width: 767px);
@custom-media --md-up   (min-width: 768px);
@custom-media --lg-up   (min-width: 1024px);
@custom-media --xl-up   (min-width: 1280px);
```

**移动优先**：默认样式针对移动端，用 `min-width` 向上叠加。理由：移动端约束更强，向上扩展比向下裁剪容易。

### 7.4.2 页面容器

```css
.container {
  width: 100%;
  max-width: 1280px;              /* 常规内容最大宽度 */
  margin-inline: auto;
  padding-inline: var(--gutter-page);
}

@media (--lg-up) {
  .container { padding-inline: var(--space-8); }
}

/* 内容浏览页可以更宽（多列网格） */
.container--wide  { max-width: 1536px; }

/* 阅读页需要窄容器（正文可读性优先） */
.container--prose { max-width: 72ch; }
```

> **`72ch` 而非固定像素**：`ch` 单位基于当前字体的"0"字符宽度。无论用户把字号调成 14px 还是 24px，`72ch` 始终保持约 72 个字符的行宽——这正是最佳可读性区间（45–75 字符）。用 `800px` 则在字号放大后行变得过长。

### 7.4.3 网格与响应式列数

```css
/* 内容卡片网格：自适应列数 */
.content-grid {
  display: grid;
  gap: var(--space-4);
  /* auto-fill + minmax 让列数随容器宽度自动计算，无需断点 */
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
}

@media (--md-down) {
  .content-grid {
    /* 移动端双列，卡片偏小但仍可辨识缩略图 */
    grid-template-columns: repeat(2, 1fr);
    gap: var(--space-3);
  }
}
```

**`auto-fill + minmax` 优于断点硬编码**：断点方案在 1023px→1024px 时列数突变，而 `auto-fill` 是平滑连续的。仅在移动端需要特殊处理（列数少于自动计算结果）。

### 7.4.4 审核工作台布局

```css
.audit-workspace {
  display: grid;
  /* 三栏：队列 280px | 预览 自适应 | 决策面板 360px */
  grid-template-columns: 280px minmax(0, 1fr) 360px;
  gap: var(--space-4);
  height: calc(100dvh - var(--header-height));
  overflow: hidden;
}

/* 队列可折叠，把空间让给预览 */
.audit-workspace[data-queue-collapsed='true'] {
  grid-template-columns: 48px minmax(0, 1fr) 360px;
}

/* 窄屏：决策面板变为底部抽屉 */
@media (max-width: 1279px) {
  .audit-workspace {
    grid-template-columns: 240px minmax(0, 1fr);
  }
  .audit-decision-panel {
    position: fixed;
    inset-block-end: 0;
    inset-inline: 0;
    max-height: 60dvh;
    transform: translateY(calc(100% - 56px));   /* 默认只露出标题栏 */
    transition: transform var(--duration-normal) var(--ease-standard);
  }
  .audit-decision-panel[data-expanded='true'] {
    transform: translateY(0);
  }
}

/* 移动端：明确不支持审核 */
@media (max-width: 767px) {
  .audit-workspace { display: none; }
  .audit-mobile-notice { display: block; }
}
```

**`minmax(0, 1fr)` 的原因**：`1fr` 的默认最小值是 `auto`，当预览区含长标题或宽视频时会把网格撑爆。`minmax(0, 1fr)` 强制允许收缩到 0，配合 `overflow` 处理内容溢出。

**`100dvh` 而非 `100vh`**：移动端浏览器地址栏收起/展开时 `vh` 不变化，导致底部内容被裁切。`dvh`（dynamic viewport height）会动态调整。

### 7.4.5 播放器样式

```css
/* styles/features/player.css */
.player-shell {
  position: relative;
  width: 100%;
  /* 16:9 容器：现代方案，避免 padding-top 技巧 */
  aspect-ratio: 16 / 9;
  background: #000;                  /* 视频底色恒为黑，不跟随主题 */
  border-radius: var(--radius-lg);
  overflow: hidden;
}

/* 全屏时圆角与容器约束移除 */
.player-shell:fullscreen {
  border-radius: 0;
  aspect-ratio: auto;
  width: 100vw;
  height: 100vh;
}

/* 竖屏视频：限制高度，避免超宽留白 */
.player-shell[data-orientation='portrait'] {
  aspect-ratio: 9 / 16;
  max-height: 80dvh;
  width: auto;
  margin-inline: auto;
}

/* ── 控件层 ── */
.player-controls {
  position: absolute;
  inset-block-end: 0;
  inset-inline: 0;
  z-index: var(--z-player-ui);
  padding: var(--space-4);
  /* 从底部渐变的遮罩，保证白色控件在任何画面上可读 */
  background: linear-gradient(to top, rgb(0 0 0 / 0.8), transparent);
  opacity: 0;
  transition: opacity var(--duration-fast) var(--ease-standard);
}

.player-shell[data-controls-visible='true'] .player-controls,
.player-shell:focus-within .player-controls {
  opacity: 1;
}

/* 键盘用户：焦点进入控件时永久可见，不因超时隐藏 */
.player-controls:focus-within { opacity: 1 !important; }

/* 大触屏设备：控件区域扩大，符合最小点击尺寸 */
@media (pointer: coarse) {
  .player-control-button {
    min-width: 44px;
    min-height: 44px;
  }
}
```

**`aspect-ratio` 而非 padding 技巧**：`padding-top: 56.25%` 是旧方案，需要额外绝对定位子元素且无法处理内容撑高。`aspect-ratio` 原生支持，且与 `object-fit` 配合自然。

### 7.4.6 弹幕层样式

```css
/* styles/features/danmaku.css */
.danmaku-layer {
  position: absolute;
  inset: 0;
  z-index: var(--z-danmaku);
  overflow: hidden;
  pointer-events: none;         /* 默认穿透，不阻挡播放器点击 */
  /* 底部留出控件区高度，避免弹幕被进度条遮挡 */
  padding-block-end: 15%;
}

.danmaku-item {
  position: absolute;
  white-space: nowrap;
  pointer-events: auto;         /* 单项可交互（悬停暂停、点击菜单） */
  font-weight: var(--font-weight-semibold);
  /* 描边保证在任何画面上可读——比 text-shadow 更清晰 */
  paint-order: stroke fill;
  -webkit-text-stroke: 1.5px rgb(0 0 0 / 0.75);
  will-change: transform;       /* 提示浏览器走合成层 */

  /* 移动动画：线性匀速，符合弹幕直觉 */
  animation: danmaku-scroll linear forwards;
}

/* 关键：用 CSS 变量传时长，避免为每条弹幕生成不同 keyframes */
@keyframes danmaku-scroll {
  from { transform: translateX(var(--danmaku-start-x)); }
  to   { transform: translateX(var(--danmaku-end-x)); }
}

/* 暂停时冻结动画（不暂停视频时冻结单条弹幕） */
.danmaku-item[data-paused='true'],
.danmaku-layer[data-frozen='true'] .danmaku-item {
  animation-play-state: paused;
}

/* 乐观插入的弹幕：降低不透明度直到服务端确认 */
.danmaku-item[data-pending='true'] { opacity: 0.6; }

/* 减少动效偏好：弹幕改为静态显示（出现在固定位置，不滚动） */
@media (prefers-reduced-motion: reduce) {
  .danmaku-item {
    animation: none;
    /* 退化方案：固定位置淡入淡出 */
    inset-inline-start: var(--space-4);
  }
}
```

**`paint-order: stroke fill` + `-webkit-text-stroke`** 是弹幕可读性的关键：文字先描边后填充，保证在亮色画面上白色文字仍有黑色轮廓。这比多层 `text-shadow` 性能更好且边缘更干净。

**通过 CSS 变量传动画起止位置**，而非为每条弹幕动态生成 `@keyframes`。后者会导致样式表无限增长，是弹幕实现的常见性能陷阱。

### 7.4.7 阅读器样式

```css
/* styles/features/reader.css */
.reader-prose {
  /* 阅读宽度由用户偏好控制，默认 72ch */
  max-width: var(--reader-content-width, 72ch);
  margin-inline: auto;
  padding-block: var(--space-12);
  padding-inline: var(--gutter-page);

  font-family: var(--reader-font-family, var(--font-sans));
  font-size: var(--reader-font-size, var(--font-size-md));
  line-height: var(--reader-line-height, var(--line-height-relaxed));
  letter-spacing: var(--letter-spacing-normal);
  color: var(--color-text-primary);
  text-align: var(--reader-text-align, start);
}

.reader-prose p {
  margin-block: 0 var(--reader-paragraph-spacing, 1em);
  /* 中文首行缩进：2 字符 */
  text-indent: var(--reader-text-indent, 2em);
}

/* 首段不缩进（常见排版惯例） */
.reader-prose > p:first-of-type { text-indent: 0; }

.reader-prose h2 {
  font-size: var(--font-size-2xl);
  line-height: var(--line-height-tight);
  margin-block: var(--space-12) var(--space-4);
  text-indent: 0;
  /* 章节标题上方加分隔线，强化章节感 */
  padding-block-start: var(--space-8);
  border-block-start: 1px solid var(--color-border-subtle);
}

.reader-prose img {
  max-width: 100%;
  height: auto;
  border-radius: var(--radius-md);
  margin-block: var(--space-6);
  /* 图片居中，与缩进正文区分 */
  display: block;
  margin-inline: auto;
}

.reader-prose blockquote {
  margin-block: var(--space-6);
  padding-inline-start: var(--space-4);
  border-inline-start: 3px solid var(--color-action-primary);
  color: var(--color-text-secondary);
  text-indent: 0;
}

.reader-prose :where(a) {   /* :where 保持优先级为 0，便于覆盖 */
  color: var(--color-action-primary);
  text-decoration: underline;
  text-underline-offset: 0.2em;   /* 下划线与文字拉开距离，中文更易读 */
}

/* 沉浸式模式：进一步收紧周边元素 */
[data-immersive='true'] .reader-prose {
  padding-block: var(--space-20);
}
[data-immersive='true'] .reader-chrome { display: none; }

/* 打印：去掉一切装饰 */
@media print {
  .reader-prose { max-width: none; font-size: 11pt; }
  .reader-chrome, .reader-progress-ring { display: none; }
  .reader-prose a::after { content: ' (' attr(href) ')'; font-size: 0.8em; }
}
```

**用户偏好通过 CSS 变量注入**，而非动态生成样式表：

```typescript
// 偏好变更时只更新变量，不产生新样式规则
useEffect(() => {
  const root = document.documentElement;
  root.style.setProperty('--reader-font-size', `${prefs.fontSize}px`);
  root.style.setProperty('--reader-line-height', String(prefs.lineHeight));
  root.style.setProperty('--reader-content-width', `${prefs.contentWidth}ch`);
  root.style.setProperty('--reader-font-family', FONT_FAMILY_MAP[prefs.fontFamily]);
  root.style.setProperty('--reader-paragraph-spacing', `${prefs.paragraphSpacing}em`);
}, [prefs]);
```

这样调整字号是**零重渲染**的纯样式变更，滑动调节时如丝般顺滑。

---

## 7.5 组件样式规范

### 7.5.1 变体用数据属性，不用类名组合

```css
/* Button.module.css */
.button {
  /* 第 3 层组件令牌，默认指向语义令牌 */
  --button-bg:       var(--color-button-bg);
  --button-bg-hover: var(--color-button-bg-hover);
  --button-fg:       var(--color-action-primary-text);
  --button-padding:  var(--space-2) var(--space-4);
  --button-radius:   var(--radius-md);

  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);
  padding: var(--button-padding);
  border: 1px solid transparent;
  border-radius: var(--button-radius);
  background: var(--button-bg);
  color: var(--button-fg);
  font-family: inherit;
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  line-height: 1.4;
  cursor: pointer;
  transition: background-color var(--duration-fast) var(--ease-standard),
              border-color     var(--duration-fast) var(--ease-standard);

  /* 最小点击尺寸（无障碍要求） */
  min-height: 36px;
}

.button:hover:not(:disabled) { background: var(--button-bg-hover); }

.button:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 2px;
}

.button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* ── 变体：data-variant ── */
.button[data-variant='secondary'] {
  --button-bg: var(--color-bg-surface);
  --button-bg-hover: var(--color-bg-sunken);
  --button-fg: var(--color-text-primary);
  border-color: var(--color-border-default);
}

.button[data-variant='ghost'] {
  --button-bg: transparent;
  --button-bg-hover: var(--color-bg-sunken);
  --button-fg: var(--color-text-secondary);
}

.button[data-variant='danger'] {
  --button-bg: var(--color-feedback-danger);
  --button-fg: var(--color-text-on-accent);
}

/* ── 尺寸：data-size ── */
.button[data-size='sm'] { --button-padding: var(--space-1) var(--space-3); min-height: 28px; font-size: var(--font-size-xs); }
.button[data-size='lg'] { --button-padding: var(--space-3) var(--space-6); min-height: 44px; font-size: var(--font-size-md); }

/* ── 全宽：data-block ── */
.button[data-block='true'] { width: 100%; }
```

**为什么用数据属性而非类名**：`className={clsx(styles.button, styles.danger, styles.large)}` 在 CSS Modules 下需要组合多个类，选择器优先级互相干扰。使用 `data-variant="danger"` 后，所有变体规则优先级相同（都是 `[data-variant]` 单属性选择器），后定义者覆盖，行为可预测。

### 7.5.2 风险徽标（审核工作台核心组件）

```css
/* features/audit/components/RiskBadge.module.css */
.badge {
  display: inline-flex;
  align-items: center;
  gap: var(--space-1);
  padding: var(--space-1) var(--space-2);
  border-radius: var(--radius-sm);
  font-size: var(--font-size-xs);
  font-weight: var(--font-weight-semibold);
  /* 用 color-mix 生成半透明底色，无需为每个等级定义两套色值 */
  background: color-mix(in srgb, var(--badge-color) 15%, transparent);
  color: var(--badge-color);
  border: 1px solid color-mix(in srgb, var(--badge-color) 30%, transparent);
}

.badge[data-level='NONE']     { --badge-color: var(--risk-none); }
.badge[data-level='LOW']      { --badge-color: var(--risk-low); }
.badge[data-level='MEDIUM']   { --badge-color: var(--risk-medium); }
.badge[data-level='HIGH']     { --badge-color: var(--risk-high); }
.badge[data-level='CRITICAL'] { --badge-color: var(--risk-critical); }

/* 高危等级：加脉冲动画吸引注意（尊重减少动效偏好） */
.badge[data-level='CRITICAL'] {
  animation: risk-pulse 2s var(--ease-standard) infinite;
}
@keyframes risk-pulse {
  0%, 100% { box-shadow: 0 0 0 0 color-mix(in srgb, var(--risk-critical) 40%, transparent); }
  50%      { box-shadow: 0 0 0 4px color-mix(in srgb, var(--risk-critical) 0%, transparent); }
}
@media (prefers-reduced-motion: reduce) {
  .badge[data-level='CRITICAL'] { animation: none; }
}

/* 风险等级不能只靠颜色区分（色盲用户）——必须带文字与图标 */
.badge-icon { flex-shrink: 0; }
.badge-label { /* 文字始终显示，不省略 */ }
```

**`color-mix()` 的价值**：一次定义 `--badge-color`，背景与边框自动派生。新增风险等级只需一行，且深浅主题下都自动协调。

**无障碍红线**：风险等级**必须同时用颜色、文字、图标三重编码**。约 8% 的男性有红绿色觉异常，若只用颜色表示高风险，这部分审核员会漏判。

### 7.5.3 上传进度条

```css
/* features/upload/components/UploadProgress.module.css */
.progress {
  position: relative;
  height: 6px;
  background: var(--color-bg-sunken);
  border-radius: var(--radius-full);
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  /* 用 transform: scaleX 而非 width —— 走合成层，不触发重排 */
  transform-origin: left center;
  transform: scaleX(var(--progress, 0));
  background: var(--color-action-primary);
  transition: transform var(--duration-fast) linear;
}

.progress[data-status='failed'] .progress-bar { background: var(--color-feedback-danger); }
.progress[data-status='success'] .progress-bar { background: var(--color-feedback-success); }

/* 不确定进度（转码中无百分比）：滑动条动画 */
.progress[data-indeterminate='true'] .progress-bar {
  transform: none;
  width: 40%;
  animation: indeterminate 1.4s var(--ease-standard) infinite;
}
@keyframes indeterminate {
  0%   { translate: -100% 0; }
  100% { translate: 350% 0; }
}

/* 条纹动画表示"正在传输" */
.progress[data-active='true'] .progress-bar::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image: linear-gradient(
    45deg,
    rgb(255 255 255 / 0.2) 25%, transparent 25%,
    transparent 50%, rgb(255 255 255 / 0.2) 50%,
    rgb(255 255 255 / 0.2) 75%, transparent 75%
  );
  background-size: 1rem 1rem;
  animation: stripes 0.8s linear infinite;
}
@keyframes stripes {
  to { background-position: 1rem 0; }
}
```

**`transform: scaleX()` 而非 `width`**：进度条每秒更新多次，改 `width` 会触发每帧布局重算（layout thrashing）。`transform` 只在合成层做，性能差异显著——在批量上传（多个进度条同时更新）时尤其明显。

### 7.5.4 弹幕输入框样式

```css
.danmaku-input {
  position: absolute;
  inset-block-end: var(--space-4);
  inset-inline: var(--space-4);
  z-index: calc(var(--z-player-ui) + 1);   /* 输入框在控件之上 */
  display: flex;
  gap: var(--space-2);
}

.danmaku-input input {
  flex: 1;
  padding: var(--space-2) var(--space-3);
  background: rgb(0 0 0 / 0.6);
  border: 1px solid rgb(255 255 255 / 0.2);
  border-radius: var(--radius-md);
  color: #fff;
  /* 输入框在深色遮罩上，不跟随主题的浅色文字 */
  backdrop-filter: blur(8px);
}

.danmaku-input input::placeholder { color: rgb(255 255 255 / 0.5); }

.danmaku-input input[aria-invalid='true'] {
  border-color: var(--color-feedback-danger);
}
```

---

## 7.6 无障碍样式

```css
/* styles/base/a11y.css */

/* 屏幕阅读器专用文本：视觉隐藏但可被读出 */
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
  border: 0;
}

/* 跳转链接：Tab 首次按下时出现 */
.skip-link {
  position: absolute;
  inset-block-start: -100%;
  inset-inline-start: var(--space-4);
  z-index: var(--z-toast);
  padding: var(--space-2) var(--space-4);
  background: var(--color-bg-surface);
  border: 2px solid var(--color-action-primary);
  border-radius: var(--radius-md);
  transition: inset-block-start var(--duration-fast) var(--ease-decelerate);
}
.skip-link:focus { inset-block-start: var(--space-4); }

/* 全局焦点样式：只在键盘导航时显示（:focus-visible） */
:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 2px;
}

/* 移除默认 outline 的元素必须提供替代焦点样式 */
*:focus:not(:focus-visible) { outline: none; }

/* 尊重减少动效偏好 */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
  /* 但保留必要的状态指示（如进度条）——用即时切换代替动画 */
  .progress-bar { transition: none; }
}

/* 强制颜色模式（Windows 高对比度） */
@media (forced-colors: active) {
  .button { border: 1px solid ButtonText; }
  .badge   { border: 1px solid CanvasText; }
}
```

**`prefers-reduced-motion` 的正确处理**：不是简单地"关掉所有动画"，而是把**表达状态的动画**退化为瞬时切换。进度条从 30% 跳到 60% 仍需视觉更新，只是不需要过渡补间。滥用 `!important` 全局禁用会让进度反馈消失。

---

## 7.7 性能规范

### 7.7.1 避免布局抖动

| 禁止 | 替代 | 原因 |
|---|---|---|
| `width` / `height` 动画 | `transform: scale()` | 触发 layout |
| `top` / `left` 定位动画 | `translate` | 触发 layout |
| `margin` 动画 | `translate` | 触发 layout |
| 频繁读 `offsetHeight` | 缓存或 `ResizeObserver` | 强制同步布局 |
| `box-shadow` 动画 | 用伪元素 + `opacity` | 阴影重绘成本高 |

### 7.7.2 内容可见性优化

```css
/* 首屏以下的卡片：跳过渲染，节省 CPU */
.content-card {
  content-visibility: auto;
  /* 提供尺寸占位，避免滚动条跳动 */
  contain-intrinsic-size: auto 320px;
}

/* 长列表项：隔离渲染 */
.audit-queue-item {
  contain: layout style paint;
}
```

**`content-visibility: auto` 在长列表场景收益巨大**——审核队列 200 条、内容列表 500 条时，浏览器跳过视口外元素的布局与绘制。配合 `contain-intrinsic-size` 提供占位高度，避免滚动条长度突变。

### 7.7.3 字体加载

```css
@font-face {
  font-family: 'BrandSans';
  src: url('/fonts/brand-sans.woff2') format('woff2');
  font-display: swap;          /* 先用系统字体渲染，字体到达后替换 */
  font-weight: 400 700;        /* 可变字体：一份文件覆盖多个字重 */
  unicode-range: U+0000-00FF;  /* 只加载拉丁字符，中文用系统字体 */
}
```

**关键的工程决策：中文不加载 Web 字体。** 中文字体文件通常 5–20 MB，加载成本远超收益。中文一律使用系统字体栈（`PingFang SC` / `Microsoft YaHei`），视觉上已经足够好。仅品牌标题等少量场景按需加载子集化字体。

**`font-display: swap`** 避免文字长时间不可见（FOIT）。代价是字体切换时的闪烁（FOUT），但文字立即可读——对内容平台更重要。

### 7.7.4 CSS 体积控制

| 措施 | 效果 |
|---|---|
| 按路由拆分 CSS | 审核工作台样式不进首页加载 |
| 关键 CSS 内联 | 首屏所需样式内联到 `<head>`，其余异步加载 |
| PurgeCSS 清理 | 移除未使用的工具类（若使用工具类库） |
| 避免深层嵌套 | 减少选择器体积与优先级复杂度 |
| 不用大型 UI 框架 | 自建设计系统，只写用到的样式 |

**目标**：首屏关键 CSS < 14 KB（gzip 后），全量 CSS < 60 KB。

---

## 7.8 样式规范速查

### 7.8.1 命名约定

| 场景 | 约定 | 示例 |
|---|---|---|
| CSS Module 类名 | camelCase | `styles.decisionPanel` |
| 设计令牌 | kebab-case，前缀分层 | `--color-text-primary` |
| 数据属性变体 | kebab-case 值 | `[data-variant='danger']` |
| 主题 | kebab-case | `[data-theme='dark']` |
| 全局工具类 | kebab-case | `.sr-only`, `.skip-link` |

### 7.8.2 禁止事项

| 禁止 | 原因 |
|---|---|
| 组件内写字面量色值 | 破坏主题系统，换主题时残留 |
| `!important` | 优先级失控的信号，应重构选择器 |
| 裸像素值（非令牌） | 破坏间距/圆角节奏 |
| `z-index: 9999` | 用 `--z-*` 令牌集中管理 |
| 内联 `style={{}}` 传业务样式 | 仅允许传 CSS 变量与动态数值 |
| 全局元素选择器（除 base 层） | 意外影响范围不可控 |
| 固定 `px` 字号 | 用户浏览器字号设置失效，无障碍缺陷 |

### 7.8.3 组件样式检查清单

在代码评审中逐项确认：

- [ ] 所有颜色引用语义令牌，无字面量色值
- [ ] 所有间距来自 `--space-*` 阶梯
- [ ] 交互状态齐全：`:hover` `:focus-visible` `:active` `:disabled`
- [ ] 焦点样式可见（使用 `--color-focus-ring`）
- [ ] 浅色、深色、护眼三主题下均验证过
- [ ] 尊重 `prefers-reduced-motion`
- [ ] 在 `forced-colors` 模式下可辨识
- [ ] 触屏设备下点击区域 ≥ 44×44px
- [ ] 文字与背景对比度 ≥ 4.5:1（正文）/ 3:1（大字号）
- [ ] 信息不单靠颜色传达（配文字或图标）
- [ ] 长文本溢出有 `text-overflow` 或换行处理
- [ ] 100% / 150% / 200% 缩放下列表不破版

### 7.8.4 对比度校准表

| 组合 | 浅色主题 | 深色主题 | 最低要求 |
|---|---|---|---|
| 正文 / 背景 | `#212529` / `#f8f9fa` = 14.8:1 | `#e6edf3` / `#0d1117` = 15.2:1 | 4.5:1 |
| 次要文字 / 背景 | `#495057` / `#f8f9fa` = 8.1:1 | `#adbac7` / `#0d1117` = 8.9:1 | 4.5:1 |
| 三级文字 / 背景 | `#868e96` / `#f8f9fa` = 4.6:1 | `#768390` / `#0d1117` = 4.7:1 | 4.5:1 |
| 危险色 / 背景 | `#dc2626` / `#ffffff` = 4.8:1 | `#ef4444` / `#0d1117` = 5.1:1 | 4.5:1 |
| 按钮白字 / 按钮底 | `#ffffff` / `#2563eb` = 5.2:1 | `#ffffff` / `#2563eb` = 5.2:1 | 4.5:1 |
| ~~按钮白字 / `--blue-500`~~ | 3.7:1 ⚠️ | 3.7:1 ⚠️ | **不达标，禁止用于按钮** |

> **深色主题下按钮对比度的陷阱**：`--blue-500` (#3b82f6) 配白字只有 3.7:1，不满足。因此引入了独立的 `--color-button-bg` 令牌——按钮底色固定用 `--blue-600`，而 `--color-action-primary` 在深色主题下仍可提亮为 `--blue-500` 供文字、图标、链接使用（那里是浅色文字压在深色底上，需要的是提亮而非压暗）。
>
> **同一个蓝色在两个角色下需求相反**，这是必须拆分令牌的原因。**每个按钮配色都必须在浅色与深色两套主题下单独验证对比度**，不能假设一套配色通用。
