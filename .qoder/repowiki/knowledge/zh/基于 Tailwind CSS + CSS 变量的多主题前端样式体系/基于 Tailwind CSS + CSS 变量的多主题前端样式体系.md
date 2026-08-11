---
kind: frontend_style
name: 基于 Tailwind CSS + CSS 变量的多主题前端样式体系
category: frontend_style
scope:
    - '**'
source_files:
    - web/tailwind.config.js
    - web/app/globals.css
    - web/lib/theme.ts
    - web/components/ThemeScript.tsx
    - web/postcss.config.js
    - web/package.json
    - web/components/ui/Button.tsx
    - web/components/ui/Tooltip.tsx
    - web/components/ui/ConfirmDialog.tsx
---

## 1. 系统/方法

DeepTutor Web 前端（`web/`）采用 **Next.js App Router + Tailwind CSS v3** 作为核心样式方案，通过 PostCSS + Autoprefixer 构建。样式以原子化 `@apply` 组合为主，所有颜色、边框、阴影等视觉变量均通过 **CSS 自定义属性（CSS Variables）** 暴露，由 Tailwind 的 `theme.extend.colors` 映射到语义化 token（`background`、`foreground`、`primary`、`card`、`popover`、`muted`、`accent`、`destructive`、`border`、`input`、`ring`、`overlay`），从而实现“一套类名、多套配色”的主题切换。

- 主题切换机制：在 `<html>` 根元素上切换 class —— `.dark`（深色）、`.theme-glass`（毛玻璃紫黑）、`.theme-snow`（纯白中性，默认浅色主题）。`tailwind.config.js` 中 `darkMode: "class"` 启用基于 class 的暗色模式。
- 字体：`fontFamily.sans` 与 `fontFamily.serif` 分别绑定 CSS 变量 `--font-sans` / `--font-serif`，便于运行时替换。
- 动画：全部使用原生 `@keyframes`（如 `dt-breathing`、`turn-flash`、`session-pulse`、`dt-popup-up`、`animate-fade-in`、`animate-overlay-in`、`animate-pop-in`），配合 `will-change`、`prefers-reduced-motion` 媒体查询实现无障碍降级。
- Markdown 渲染：通过 `.prose` 与 `.md-renderer` 两套样式覆盖 `react-markdown` 输出，代码块强制使用固定深色背景并覆盖主题继承，保证代码高亮一致性。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `web/tailwind.config.js` | Tailwind 配置，定义 `darkMode: "class"`、`content` 扫描路径、`theme.extend.colors` 映射至 CSS 变量 |
| `web/app/globals.css` | 全局样式入口，集中声明四套主题调色板（`:root` 奶油暖色系、`.dark` 深暖系、`.theme-snow` 纯白中性、`.theme-glass` 毛玻璃紫黑），以及 prose、滚动条、动画、响应式抽屉挤压等通用样式 |
| `web/lib/theme.ts` | 主题状态管理：`Theme = "light" | "dark" | "glass" | "snow"`，提供 `getStoredTheme`、`saveThemeToStorage`、`initializeTheme`、`setTheme`、`subscribeToThemeChanges` 等 API |
| `web/components/ThemeScript.tsx` | 在 SSR HTML 中内联执行脚本，避免 React hydration 时的主题闪烁（FOUC） |
| `web/postcss.config.js` | PostCSS 插件链：`tailwindcss` → `autoprefixer` |
| `web/package.json` | 依赖声明：`next 16`、`react 19`、`tailwindcss ^3.4`、`tailwind-merge ^3.4`、`framer-motion`、`lucide-react`、`i18next` 等 |
| `web/components/ui/Button.tsx`、`Tooltip.tsx`、`ConfirmDialog.tsx` | 基础 UI 组件，统一使用 `clsx` + `twMerge` 合并 className |
| `web/components/common/*`、`components/chat/*`、`components/settings/*` 等 | 业务组件，全部通过 Tailwind 原子类 + CSS 变量组合样式 |

## 3. 架构与设计约定

### 设计令牌（Design Tokens）
- 所有颜色通过 CSS 变量集中声明在 `globals.css` 的 `@layer base` 中，每个主题必须完整提供一组约定的 token（注释明确列出 `--background`、`--foreground`、`--card`/`--popover`、`--primary`、`--secondary`、`--muted`、`--accent`、`--destructive`、`--border`、`--input`、`--ring`、`--overlay`）。
- Tailwind 通过 `theme.extend.colors` 将这些变量映射为语义化 token，组件只引用 `bg-background`、`text-primary`、`border-border` 等语义类，不直接写色值。

### 主题策略
- 四种主题并存：
  - **Cream (`:root`)**：近白暖色画布，搭配赤陶色（terracotta）主色，灵感来自 Claude 的 chrome。
  - **Dark (`.dark`)**：同属暖色家族的深色变体。
  - **Snow (`.theme-snow`)**：纯白中性画布，蓝色主色，对标 Codex/ChatGPT 风格。
  - **Glass (`.theme-glass`)**：半透明紫色面板 + near-black 背景，带 `backdrop-filter: blur()` 毛玻璃效果与径向渐变背景。
- 主题持久化：`localStorage` key 为 `deeptutor-theme`；无存储时按系统 `prefers-color-scheme` 回退（亮→`snow`，暗→`dark`）。
- 应用启动优先级：`localStorage` > 系统偏好；`ThemeScript` 在 SSR 阶段提前注入以避免 FOUC。

### 组件与样式组织
- 组件按功能域分目录：`components/chat/`、`components/knowledge/`、`components/memory/`、`components/settings/`、`components/space/`、`components/mcp/`、`components/partners/`、`components/quiz/`、`components/research/`、`components/visualize/`、`components/common/`、`components/layout/`、`components/sidebar/`、`components/ui/`。
- 共享基础组件集中在 `components/ui/`（Button、Tooltip、ConfirmDialog），业务组件复用这些基础组件。
- 样式几乎全部通过 Tailwind 原子类 + `@apply` 组合，不使用独立 SCSS/CSS 模块；全局样式集中在 `globals.css`。

### 响应式策略
- 基于 Tailwind 断点（如 `sm:`、`md:`、`lg:`、`xl:`）实现响应式布局。
- 聊天容器与文件预览抽屉通过 `chat-preview-shell` 的 `padding-right` 动态挤压，在大屏（`min-width: 1024px`）下根据 `data-preview-open`、`data-viewer-open` 与 CSS 变量 `--viewer-width` 控制挤压宽度。

### 可访问性
- 全站遵循 `prefers-reduced-motion`：循环动画（呼吸、脉冲、旋转）被移除，交互过渡缩短至不可感知时长。
- 模态打开时通过 `body[data-picker-open]` 暂停后台动画，避免 backdrop-filter 重采样导致的闪烁。

## 4. 约定与约束

- **颜色必须走 CSS 变量**：新增主题或修改配色时，需在 `globals.css` 对应主题块中补齐所有约定 token，并在 `tailwind.config.js` 的 `theme.extend.colors` 中映射，禁止在组件中硬编码十六进制颜色。
- **主题 class 命名规范**：仅允许在 `<html>` 上添加 `dark`、`theme-glass`、`theme-snow` 三类 class；`light` 主题即不添加任何主题 class（使用 `:root` 默认奶油主题）。
- **主题切换 API**：组件应调用 `lib/theme.ts` 中的 `setTheme` / `initializeTheme` / `subscribeToThemeChanges`，而非直接操作 DOM class。
- **Markdown 代码块样式隔离**：代码块强制使用固定深色背景与浅色文字，不受主题 `--foreground` 影响；新增 markdown 相关样式需放在 `.md-renderer` 作用域下。
- **动画命名空间**：自定义动画统一使用 `dt-` 前缀（如 `dt-breathing`、`dt-popup-up`、`dt-tooltip-fade-in`），避免与第三方库冲突。
- **Tailwind 内容扫描范围**：`tailwind.config.js` 的 `content` 已限定为 `pages/`、`components/`、`app/` 下的 `*.js,*.ts,*.jsx,*.tsx,*.mdx`，新增样式目录需同步更新扫描路径。
- **PostCSS 插件顺序**：必须保持 `tailwindcss` → `autoprefixer` 的顺序，不得插入其他会改变 CSS 结构的插件。
- **滚动行为契约**：所有包裹 LLM 流式输出的容器需携带 `data-chat-scroll-root="true"`，以启用 `overflow-anchor: none` 与 `scroll-behavior: auto`，防止浏览器自动锚定导致滚动抖动。

## 5. 工具链与生态

- **框架**：Next.js 16（App Router）、React 19。
- **样式引擎**：Tailwind CSS 3.4 + PostCSS + Autoprefixer。
- **类名合并**：`clsx` + `tailwind-merge`（用于安全合并多个 className 字符串）。
- **图标**：`lucide-react` 作为统一图标库。
- **动效**：`framer-motion` 用于复杂交互动画，其余轻量动画使用原生 CSS `@keyframes`。
- **国际化**：`i18next` + `react-i18next`，与主题系统解耦。
- **测试**：Playwright 提供 UI 审计用例（`playwright test --project=ui-audit`）。

总体而言，该仓库的前端样式体系以 **Tailwind 原子类 + CSS 变量主题** 为核心，通过严格的 token 约定与主题 class 管理机制，实现了在同一套组件代码上无缝切换 Cream/Dark/Snow/Glass 四种视觉风格，同时兼顾了代码块渲染一致性、流式滚动稳定性与无障碍体验。