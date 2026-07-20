```markdown
# AI 友好型组件库研究

> 研究如何设计和构建一个对 AI（尤其是 LLM）友好的前端组件库，使 AI 能更准确地理解、使用和生成组件代码。

---

## 一、背景与动机

随着 AI 辅助编程（Cursor、Copilot、Codex 等）的普及，组件库的消费方式正在发生变化：不仅是人类开发者在使用组件，AI Agent 也在生成和调用它们。传统的组件库设计以"人类可读"为首要目标，但对 AI 来说，某些设计模式会导致：

- **歧义**：同名 prop 在不同组件中语义不同
- **隐式约束**：某些组合方式不可用，但文档未明确声明
- **上下文碎片**：一个组件的完整用法分散在多个文件中
- **类型不足**：缺少精确的类型定义，AI 难以推断用法

---

## 二、AI 友好性的核心原则

### 2.1 显式优于隐式

| 原则 | 说明 | 示例 |
|------|------|------|
| 命名一致性 | 同类 prop 统一命名 | `variant` vs `type` vs `kind` 选其一 |
| 显式接口 | 避免魔法属性/隐式透传 | 用 `restProps` 明确声明 |
| 完整导出 | 所有子组件/类型均 export | 避免内部模块不可访问 |

### 2.2 类型完备

- 所有 prop 都有精确 TypeScript 类型，避免 `any`
- 使用联合类型而非 `string` 枚举
- 为回调函数提供完整类型签名
- 导出所有内部类型，方便 AI 组合引用

```tsx
// ❌ 不友好
interface ButtonProps {
  type?: string
  onClick?: (e: any) => void
}

// ✅ 友好
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void
}
```

### 2.3 结构化文档

AI 擅长从结构化数据中理解组件，而非自由文本。

- **组件级 JSDoc**：描述用途、使用场景、限制
- **Props 行内注释**：描述每个 prop 的作用和默认值
- **使用示例代码块**：完整的、可运行的示例
- **组合规则**：明确说明哪些组件可以/不可以组合

---

## 三、关键设计模式

### 3.1 原子化组件体系

遵循 Atomic Design 分层：

```
atoms/          —— 基础元素（Button, Input, Icon）
molecules/      —— 复合组件（FormField, Card）
organisms/      —— 区块组件（DataTable, Navigation）
templates/      —— 布局组件（PageLayout, SidebarLayout）
```

AI 更容易在清晰的层级中理解和组合组件。

### 3.2 组合优先（Composition First）

```tsx
// ❌ 大型配置对象模式
<Table
  columns={columns}
  data={data}
  pagination={{ pageSize: 10 }}
  sortable={true}
  filterable={true}
/>

// ✅ 组合模式 — AI 更容易推理每个子部件
<Table>
  <Table.Columns definition={columns} />
  <Table.Body data={data} />
  <Table.Pagination pageSize={10} />
  <Table.Toolbar>
    <Table.Sort />
    <Table.Filter />
  </Table.Toolbar>
</Table>
```

### 3.3 Props 扁平化

避免深层嵌套的 prop 对象，AI 在平面结构中检索和生成的成本更低。

```tsx
// ❌ 深层嵌套
<DatePicker
  input={{ placeholder: '选择日期', format: 'YYYY-MM-DD' }}
  popup={{ position: 'bottom', animation: 'slide' }}
/>

// ✅ 扁平结构
<DatePicker
  placeholder="选择日期"
  format="YYYY-MM-DD"
  popupPosition="bottom"
  popupAnimation="slide"
/>
```

### 3.4 一致的语义命名

建立组件库的词汇表规范：

| 概念 | 统一命名 | 避免的变体 |
|------|---------|-----------|
| 样式变体 | `variant` | `type`, `kind`, `style`, `appearance` |
| 大小 | `size` | `dimension`, `scale`, `sizing` |
| 禁用态 | `disabled` | `isDisabled`, `disable`, `readonly` |
| 加载态 | `loading` | `isLoading`, `busy`, `pending` |
| 空态 | `empty` | `isEmpty`, `noData`, `blank` |
| 选中值 | `value` / `selectedValue` | `val`, `current`, `activeKey` |
| 变化回调 | `onChange` | `onValueChange`, `onUpdate`, `changeHandler` |

---

## 四、Codex / AI 工具适配

### 4.1 为 AI 提供索引文件

```tsx
// index.ts — 集中导出，AI 一目了然
export { Button, type ButtonProps } from './atoms/Button'
export { Input, type InputProps } from './atoms/Input'
export { Card, type CardProps } from './molecules/Card'
export type { Theme, Size, Variant } from './types'
```

### 4.2 为组件添加 Usage Metadata

```tsx
/**
 * 主操作按钮，用于表单提交和关键操作。
 *
 * @example
 * ```tsx
 * <Button variant="primary" size="md" onClick={handleSubmit}>
 *   提交
 * </Button>
 * ```
 *
 * @remarks
 * - `primary` variant 应每页最多使用一次
 * - 不要在 `ghost` 按钮中使用图标作为唯一内容
 * - 与 `Tooltip` 组合时，将 Tooltip 包裹在 Button 外部
 *
 * @ai-metadata
 * category: form
 * complexity: simple
 * dependencies: []
 * related: [ButtonGroup, IconButton, LinkButton]
 */
```

### 4.3 规则文件

在组件库根目录添加 `.ai-rules` 或 `.cursorrules`，帮助 AI 理解使用约束：

```
# AI 使用规则
- 所有表单组件必须配合 Form 使用，不可单独管理状态
- Modal 组件必须搭配 Modal.Header + Modal.Body + Modal.Footer 使用
- 表格列的排序和过滤功能需通过 Table.Column 的 sortable/filterable prop 开启
- 配色方案请引用 theme.tokens，不要硬编码颜色值
```

---

## 五、推荐的参考项目

| 项目 | 特点 |
|------|------|
| [Radix UI](https://www.radix-ui.com/) | 无样式 + 组合式 API，类型极佳 |
| [shadcn/ui](https://ui.shadcn.com/) | 源码复制模式，AI 可直接修改 |
| [Ark UI](https://ark-ui.com/) | 框架无关 + 状态机驱动 |
| [Park UI](https://park-ui.com/) | 基于 Ark UI + Panda CSS |
| [React Aria](https://react-spectrum.adobe.com/react-aria/) | 全无障碍 + 类型完备 |

### shadcn/ui 模式特别分析

shadcn/ui 是目前对 AI 最友好的模式之一，原因：

1. **源码即依赖**：组件代码直接进入项目，AI 可读可改
2. **无抽象层**：不需要查文档，组件代码本身就是文档
3. **简单 Props**：每个组件 props 较少且语义明确
4. **Tailwind 类名**：修改样式对 AI 来说很直接
5. **广泛认知**：训练数据中覆盖量大

---

## 六、评估维度

构建完成后，可以通过以下维度评估组件库的 AI 友好性：

- **🧩 可发现性**：AI 能否轻松找到需要的组件？
- **📖 可理解性**：AI 能否根据类型和注释理解组件用途？
- **🔧 可生成性**：AI 能否生成正确的使用代码？
- **🔄 可组合性**：AI 能否正确组合多个组件？
- **🧪 可测试性**：AI 能否为组件生成测试？

评分示例：

```
可发现性:    ★★★★☆  (完整导出 + 良好目录结构)
可理解性:    ★★★☆☆  (类型完备，但缺少 JSDoc 示例)
可生成性:    ★★★☆☆  (组合模式优于配置模式)
可组合性:    ★★★★☆  (原子化体系)
可测试性:    ★★★☆☆  (Props 扁平化有助于生成测试)
```

---

## 七、下一步行动

- [ ] 确定组件的 API 设计规范（命名、类型、行为一致性）
- [ ] 搭建原子化目录结构
- [ ] 实现基础原子组件（Button, Input, Icon, Typography）
- [ ] 为每个组件添加完整的 JSDoc + 示例
- [ ] 建立 AI 规则文件
- [ ] 添加自动化类型导出
- [ ] 评估并迭代 AI 友好性评分

---

> **结论**：AI 友好型组件库不是重新发明轮子，而是在传统组件库设计之上增加**结构化、显式化、一致性**三个维度。Radix + shadcn/ui 的组合模式是目前最值得借鉴的方向。
```
