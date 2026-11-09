---
kind: frontend_style
name: Next.js + Tailwind CSS 主题与样式体系
category: frontend_style
scope:
    - '**'
source_files:
    - web/tailwind.config.js
    - web/postcss.config.js
    - web/package.json
    - web/app/layout.tsx
    - web/app/globals.css
    - web/features/ui-layout/classic/classic.css
    - web/features/ui-layout/modern/modern.css
    - web/features/ui-layout/classic/tokens.ts
    - web/features/ui-layout/modern/tokens.ts
---

## 1. 系统概览

DeepTutor Web 前端（`web/`）基于 **Next.js App Router**（React 19 / Next 16），样式体系以 **Tailwind CSS v3** 为核心，配合 **PostCSS + Autoprefixer**。没有引入 shadcn/ui、Ant Design 等组件库；自定义 UI 通过 `features/ui-layout/{classic,modern}` 两套布局主题提供，并通过 CSS 变量实现多主题切换。

关键依赖：
- `tailwindcss ^3.4.17`、`autoprefixer`、`postcss`
- `clsx`、`tailwind-merge`（用于类名合并）
- `framer-motion`（动画）、`lucide-react`（图标）
- `katex`、`mermaid`、`react-syntax-highlighter`（内容渲染）

## 2. 关键文件

- `web/tailwind.config.js` — Tailwind 配置，定义暗色模式、字体族、颜色映射
- `web/postcss.config.js` — PostCSS 管线（tailwindcss → autoprefixer）
- `web/app/globals.css` — 全局样式入口，声明所有 CSS 设计令牌（design tokens）
- `web/app/layout.tsx` — 根布局，导入 `globals.css` 以及两套 UI 布局的 CSS
- `web/features/ui-layout/classic/classic.css`、`modern/modern.css` — 两套布局主题的样式与 token 覆盖
- `web/features/ui-layout/{classic,modern}/tokens.ts` — 各主题的 JS 侧 token 常量
- `web/components/**/*.module.css` — 组件级局部样式（如 `ChatComposer.module.css`）

## 3. 架构与约定

### 3.1 主题系统（CSS 变量驱动）

`app/globals.css` 在 `@layer base` 中声明四套主题家族，统一发布以下必需 token：

```
--background, --foreground, --card/--card-foreground,
--popover/--popover-foreground, --primary/--primary-foreground,
--secondary/--secondary-foreground, --muted/--muted-foreground,
--accent/--accent-foreground, --destructive/--destructive-foreground,
--border, --input, --ring, --overlay,
--success/--success-foreground/--success-surface,
--warning/--warning-foreground/--warning-surface,
--info/--info-foreground/--info-surface,
--surface-raised, --shadow-raised
```

四套主题：
- `:root`（Cream）— 暖色系，赤陶土 accent，默认 Light
- `.dark` — 同家族的深色变体
- `.theme-snow` — 纯白中性画布，蓝色主色（Codex/ChatGPT 风格）
- `.theme-glass` — 半透明紫色 + 近黑背景

Tailwind 通过 `alpha(name)` 辅助函数将 CSS 变量包装为 `color-mix(in srgb, var(--name) calc(<alpha-value> * 100%), transparent)`，使 `bg-primary/70` 等带透明度的工具类能正确工作（见 `tailwind.config.js` 顶部注释解释为何必须用 `color-mix` 而非直接引用完整值变量）。

### 3.2 字体策略

`tailwind.config.js` 扩展 `fontFamily.sans` 与 `fontFamily.serif`，显式列出 CJK 回退字体链（PingFang SC、Hiragino Sans GB、Microsoft YaHei、Noto Sans SC、Songti SC、STSong、Noto Serif SC、Source Han Serif SC、SimSun），避免中文落入浏览器默认的 serif/sans 导致中英文混排不一致。Latin 部分使用 `var(--font-sans)` / `var(--font-serif)` 供主题覆盖。

### 3.3 响应式策略：容器查询优先

项目大量使用 CSS Container Queries 而非媒体查询：
- `.dt-page-container { container: feature-page / inline-size; }` 作为页面容器
- `@container feature-page (min-width: 520px|640px|800px|1040px)` 控制网格列数与标题字号
- 组件级容器如 `chat-composer`、`reader-bar` 同样通过 `container-type: inline-size` 暴露给子组件

### 3.4 布局主题双轨制

`features/ui-layout/` 下有两套并列布局：
- `classic/` — ClassicLearningDashboard + ClassicSidebarShell
- `modern/` — 现代布局

每套布局提供独立的 `*.css` 与 `tokens.ts`，由 `app/layout.tsx` 同时导入两者，运行时通过 `[data-ui-layout="modern"|"classic"]` 属性选择激活哪一套。布局主题可覆盖 `--dt-page-title-font` 等页面级 token。

### 3.5 样式组织层级

遵循 Tailwind 的 `@layer` 约定：
- `@layer base` — 全局 token、基础元素样式
- `@layer components` — 复用组件类（如卡片、标签、按钮基类）
- `@layer utilities` — 工具类增强

组件内样式使用 CSS Modules（`.module.css`），共享样式放在 `components/` 或 `features/` 下的独立 `.css` 文件中。

## 4. 约定与约束

- **暗色模式开关**：`darkMode: 'class'`，通过 `<html>` 上的 `.dark` 类切换（非 `media`）。
- **颜色使用规范**：组件应通过 Tailwind 语义化类名（`bg-primary`、`text-muted-foreground`）访问颜色，禁止硬编码十六进制色值；透明度通过 `/70` 等修饰符叠加，底层由 `color-mix` 适配。
- **CJK 字体链不得省略**：Sans/Serif 的 CJK 回退列表是强制性的，目的是保证中英文混排在不同 OS 上表现一致（见 `tailwind.config.js` 注释）。
- **主题 token 契约**：新增主题时必须发布 `globals.css` 中列出的全部 token，否则 Tailwind 生成的工具类会失效。
- **响应式优先使用容器查询**：页面与组件通过 `container: <name> / inline-size` 暴露断点，样式写在 `@container <name> (...)` 块中，而非 `@media`。
- **布局主题隔离**：新增布局主题需在 `features/ui-layout/<name>/` 下提供 `index.ts`、`*.css`、`tokens.ts`，并在 `app/layout.tsx` 中导入其 CSS。
- **构建产物扫描范围**：Tailwind `content` 仅扫描 `pages/`、`components/`、`features/`、`shared/`、`app/` 及 `lib/reading-age-presentation.ts`，未在此范围内的文件不会被生成样式。