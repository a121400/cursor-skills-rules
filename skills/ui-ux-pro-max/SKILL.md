---
name: ui-ux-pro-max
description: UI/UX 设计智能搜索数据库，提供 UI 风格、配色方案、字体配对、图表类型、产品推荐、UX 最佳实践和技术栈指南。当用户需要设计 UI、构建前端界面、选择配色、选择字体、创建着陆页、或需要 UX 建议时使用。
---

# UI/UX Pro Max - 设计智能

可搜索的 UI 样式、配色方案、字体配对、图表类型、产品推荐、UX 指南和技术栈最佳实践数据库。

## 使用方法

当用户请求 UI/UX 相关工作时（设计、构建、创建、实现、审查、修复、改进），执行以下搜索。

### 搜索命令

```bash
python scripts/search.py "<关键词>" --domain <域名> [-n <最大结果数>]
```

脚本位置：`C:/Users/Administrator/.cursor/skills/ui-ux-pro-max/scripts/search.py`

**注意**：运行前需切换到 skill 目录：

```powershell
cd C:\Users\Administrator\.cursor\skills\ui-ux-pro-max
python scripts/search.py "关键词" --domain style
```

## 可用搜索域

| 域名 | 用途 | 示例关键词 |
|------|------|-----------|
| `product` | 产品类型推荐 | SaaS, e-commerce, portfolio, healthcare, beauty |
| `style` | UI 风格、颜色、效果 | glassmorphism, minimalism, dark mode, brutalism |
| `typography` | 字体配对、Google Fonts | elegant, playful, professional, modern |
| `color` | 按产品类型的配色方案 | saas, ecommerce, healthcare, beauty, fintech |
| `landing` | 页面结构、CTA 策略 | hero, hero-centric, testimonial, pricing |
| `chart` | 图表类型、库推荐 | trend, comparison, timeline, funnel, pie |
| `ux` | 最佳实践、反模式 | animation, accessibility, z-index, loading |
| `prompt` | AI 提示词、CSS 关键词 | (风格名称) |

## 技术栈指南

```bash
python scripts/search.py "<关键词>" --stack <技术栈>
```

| 技术栈 | 专注领域 |
|--------|----------|
| `html-tailwind` | Tailwind 工具类、响应式、无障碍 (默认) |
| `react` | 状态管理、hooks、性能、模式 |
| `nextjs` | SSR、路由、图片、API 路由 |
| `vue` | Composition API、Pinia、Vue Router |
| `svelte` | Runes、stores、SvelteKit |
| `swiftui` | Views、State、Navigation、Animation |
| `react-native` | 组件、导航、列表 |
| `flutter` | Widgets、State、Layout、主题 |

## 推荐工作流

1. **分析产品类型**：`python scripts/search.py "beauty spa" --domain product`
2. **获取风格指南**：`python scripts/search.py "elegant minimal" --domain style`
3. **选择字体配对**：`python scripts/search.py "elegant luxury" --domain typography`
4. **获取配色方案**：`python scripts/search.py "beauty spa" --domain color`
5. **着陆页结构**：`python scripts/search.py "hero-centric" --domain landing`
6. **UX 最佳实践**：`python scripts/search.py "animation" --domain ux`
7. **技术栈指南**：`python scripts/search.py "layout" --stack html-tailwind`

## 专业 UI 通用规则

### 图标与视觉元素

| 规则 | 正确做法 | 错误做法 |
|------|----------|----------|
| 禁止 emoji 图标 | 使用 SVG 图标 (Heroicons, Lucide) | 使用 emoji 作为 UI 图标 |
| 稳定的悬停状态 | 使用颜色/透明度过渡 | 使用导致布局偏移的缩放 |
| 正确的品牌 Logo | 从 Simple Icons 获取官方 SVG | 猜测或使用错误的 logo |

### 交互与光标

| 规则 | 正确做法 | 错误做法 |
|------|----------|----------|
| 光标指针 | 所有可点击元素添加 `cursor-pointer` | 交互元素使用默认光标 |
| 悬停反馈 | 提供视觉反馈 (颜色、阴影、边框) | 无交互指示 |
| 平滑过渡 | 使用 `transition-colors duration-200` | 瞬间变化或过慢 (>500ms) |

### 明暗模式对比度

| 规则 | 正确做法 | 错误做法 |
|------|----------|----------|
| 玻璃卡片亮色模式 | 使用 `bg-white/80` 或更高 | 使用 `bg-white/10` |
| 文字对比度 | 使用 `#0F172A` (slate-900) | 使用 `#94A3B8` (slate-400) |
| 边框可见性 | 亮色模式使用 `border-gray-200` | 使用 `border-white/10` |

### 布局与间距

| 规则 | 正确做法 | 错误做法 |
|------|----------|----------|
| 浮动导航栏 | 添加 `top-4 left-4 right-4` 间距 | 固定到 `top-0 left-0 right-0` |
| 内容内边距 | 考虑固定导航栏高度 | 内容被固定元素遮挡 |
| 一致的最大宽度 | 使用相同的 `max-w-6xl` | 混用不同容器宽度 |

## 交付前检查清单

### 视觉质量
- [ ] 不使用 emoji 作为图标
- [ ] 所有图标来自一致的图标集
- [ ] 品牌 Logo 正确
- [ ] 悬停状态不导致布局偏移

### 交互
- [ ] 所有可点击元素有 `cursor-pointer`
- [ ] 悬停状态提供清晰的视觉反馈
- [ ] 过渡平滑 (150-300ms)
- [ ] 键盘导航的焦点状态可见

### 明暗模式
- [ ] 亮色模式文字对比度足够 (最低 4.5:1)
- [ ] 玻璃/透明元素在亮色模式可见
- [ ] 边框在两种模式下可见

### 布局
- [ ] 浮动元素与边缘有适当间距
- [ ] 无内容被固定导航栏遮挡
- [ ] 在 320px、768px、1024px、1440px 响应式
- [ ] 移动端无水平滚动

### 无障碍
- [ ] 所有图片有 alt 文本
- [ ] 表单输入有标签
- [ ] 颜色不是唯一指示器
- [ ] 尊重 `prefers-reduced-motion`
