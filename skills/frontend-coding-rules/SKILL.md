---
name: frontend-coding-rules
description: 前端编码规范。当编写前端代码时，Claude 必须遵循这些规则以保持项目一致性。
disable-model-invocation: false
user-invocable: false
---

# 项目编码规范

## 1. UI 组件：优先使用 shadcn-ui

编写任何 UI 代码前，**必须先检查 `@/components/ui/` 目录中是否已有对应组件**。

### shadcn-ui 完整组件清单（未安装的可按需添加）

**Form & Input**: Form, Field, Button, Button Group, Input, Input Group, Input OTP, Textarea, Checkbox, Radio Group, Select, Switch, Slider, Calendar, Date Picker, Combobox, Label

**Layout & Navigation**: Accordion, Breadcrumb, Navigation Menu, Sidebar, Tabs, Separator, Scroll Area, Resizable

**Overlays & Dialogs**: Dialog, Alert Dialog, Sheet, Drawer, Popover, Tooltip, Hover Card, Context Menu, Dropdown Menu, Menubar, Command

**Feedback & Status**: Alert, Toast (Sonner), Progress, Spinner, Skeleton, Badge, Empty

**Display & Media**: Avatar, Card, Table, Data Table, Chart, Carousel, Aspect Ratio, Typography, Item, Kbd

**Misc**: Collapsible, Toggle, Toggle Group, Pagination

### 规则

- 如果需要的组件已安装 → 直接从 `@/components/ui/xxx` 导入使用
- 如果需要的组件在 shadcn-ui 清单中但未安装 → 通过 `npx shadcn@latest add <component>` 安装
- 禁止手写已有 shadcn-ui 组件的替代实现

## 2. 样式：Tailwind CSS + cn()

- 所有样式使用 Tailwind CSS class
- **颜色必须使用项目已定义的语义化颜色**（如 `text-primary`、`bg-secondary`、`border-muted` 等），不要使用原始颜色值（如 `text-red-500`），除非用户特别指定
- 需要条件合并 class 时，使用 `@/lib/utils` 中的 `cn()` 函数：

```tsx
import { cn } from "@/lib/utils"

<div className={cn("base-classes", isActive && "active-classes")} />
```

- `cn()` 基于 `clsx` + `tailwind-merge`，能智能处理 Tailwind class 冲突

### 自定义样式：Tailwind CSS v4 @utility

当需要自定义复杂样式（如渐变背景、复合状态）时，使用 Tailwind CSS v4 的 `@utility` 指令定义在 `globals.css` 中，而非内联写大量 class：

```css
@utility bg-primary-button {
  background: linear-gradient(0deg, var(--color-brand) 0%, var(--color-brand-accent) 139.13%);
  cursor: pointer;

  &:disabled {
    cursor: not-allowed;
    background: linear-gradient(0deg, #4C4033 0%, #DEC9B4 139.13%);
    opacity: 1;
  }

  &:hover:not(:disabled) {
    background: linear-gradient(0deg, var(--color-brand-dark) 0%, var(--color-brand-accent) 139.13%);
  }
}
```

使用时直接当作 Tailwind class：

```tsx
<button className="bg-primary-button px-4 py-2">提交</button>
```

## 3. 静态常量提升到组件外部

不依赖 props 或 state 的静态数据（数组、对象、配置映射等）**必须定义在组件函数外部**，避免每次渲染都重新创建引用：

```tsx
// ✅ 正确：组件外部定义，使用 UPPER_SNAKE_CASE 命名
const INITIAL_FORM = { name: "", email: "" };
const STATUS_OPTIONS = ["active", "inactive", "pending"];
const TAB_CONFIG = [
  { key: "overview", label: "Overview" },
  { key: "settings", label: "Settings" },
];

function MyComponent() {
  const [form, setForm] = useState(INITIAL_FORM);
  // ...
}

// ❌ 错误：组件内部定义，每次渲染都重新创建
function MyComponent() {
  const initialForm = { name: "", email: "" };
  const statusOptions = ["active", "inactive", "pending"];
  // ...
}
```

### 判断标准
- 值在整个组件生命周期内**不会变化** → 移到外部
- 同一个对象字面量在组件内**出现多次** → 提取为外部常量并复用
- 值依赖 props / state / context → 保留在组件内部

## 4. 表单：react-hook-form + zod + Field

所有表单必须遵循以下流程：

1. **先定义 zod schema**
2. **用 `useForm` + `zodResolver` 绑定 schema**
3. **用 `Controller` + shadcn-ui 的 `Field` 系列组件渲染每个字段**

> 注意：使用 shadcn-ui 新版 `Field` 组件（`@/components/ui/field`），**不要**使用旧版 `Form` + `FormField` 模式。

```tsx
import { z } from "zod"
import { useForm, Controller } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import {
  Field,
  FieldDescription,
  FieldError,
  FieldGroup,
  FieldLabel,
  FieldContent,
} from "@/components/ui/field"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

// 1. 定义 schema
const formSchema = z.object({
  title: z.string().min(5, "标题至少 5 个字符").max(32, "标题最多 32 个字符"),
  description: z.string().min(20, "描述至少 20 个字符").max(100, "描述最多 100 个字符"),
})

// 2. useForm 绑定
const form = useForm<z.infer<typeof formSchema>>({
  resolver: zodResolver(formSchema),
  defaultValues: { title: "", description: "" },
})

// 3. Controller + Field 渲染
<form onSubmit={form.handleSubmit(onSubmit)}>
  <FieldGroup>
    <Controller
      name="title"
      control={form.control}
      render={({ field, fieldState }) => (
        <Field data-invalid={fieldState.invalid}>
          <FieldLabel htmlFor="title">标题</FieldLabel>
          <FieldContent>
             <Input
               {...field}
               id="title"
               aria-invalid={fieldState.invalid}
               placeholder="请输入标题"
             />
          </FieldContent>
          {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
        </Field>
      )}
    />
    <Controller
      name="description"
      control={form.control}
      render={({ field, fieldState }) => (
        <Field data-invalid={fieldState.invalid}>
          <FieldLabel htmlFor="description">描述</FieldLabel>
          <FieldContent>
             <Input
               {...field}
               id="description"
               aria-invalid={fieldState.invalid}
               placeholder="请输入描述"
             />
          </FieldContent>
          <FieldDescription>详细描述问题和重现步骤</FieldDescription>
          {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
        </Field>
      )}
    />
  </FieldGroup>
  <Button type="submit">提交</Button>
</form>
```

### Field 系列组件说明

| 组件 | 用途 |
|------|------|
| `Field` | 字段容器，通过 `data-invalid` 控制错误状态样式 |
| `FieldLabel` | 字段标签 |
| `FieldDescription` | 字段辅助说明 |
| `FieldError` | 错误信息，接收 `errors={[fieldState.error]}` |
| `FieldGroup` | 多字段分组容器 |
| `FieldContent` | 包裹表单控件 |

### 简单表单的替代方案：useActionState + Server Action

当表单非常简单（如只有一两个字段、无复杂交互）时，可以使用 `useActionState` + Server Action 替代 react-hook-form：

- Server Action 统一放在 `@/actions/` 目录下
- 一个业务一个文件（如 `@/actions/blog.ts`、`@/actions/prayer.ts`）
- Server Action 文件顶部必须声明 `"use server"`

### 需要安装的组件

如果 `@/components/ui/field.tsx` 不存在，需先执行：
```bash
npx shadcn@latest add field
```

如果表单中需要 InputGroup（带前缀/后缀的输入框），还需安装：
```bash
npx shadcn@latest add input-group
```

## 5. Toast 通知：Sonner

- 直接使用 `sonner` 导出的 `toast`
- **不要**使用 `@/components/ui/use-toast` 或其他 toast 方案

```tsx
import { toast } from "sonner"

toast.success("操作成功")
toast.error("操作失败")
```

## 6. 危险操作确认：AlertDialog

执行删除、重置等不可逆操作前，必须使用 `AlertDialog` 让用户确认：

```tsx
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel,
  AlertDialogContent, AlertDialogDescription, AlertDialogFooter,
  AlertDialogHeader, AlertDialogTitle, AlertDialogTrigger,
} from "@/components/ui/alert-dialog"
```

## 7. 服务端代码隔离

纯服务端模块（如数据库操作、密钥处理等）必须在文件顶部导入 `server-only`，防止意外在客户端使用：

```tsx
import "server-only"
```

## 8. 包管理器：bun

项目使用 **bun** 作为包管理器，所有依赖安装、脚本执行统一使用 bun：

```bash
bun add <package>        # 安装依赖
bun add -d <package>     # 安装开发依赖
bun run <script>         # 执行脚本
```

**不要**使用 npm、yarn 或 pnpm。

## 9. 复杂状态管理：useImmer

当组件内的 state 结构复杂（嵌套对象、多属性）时，使用 `useImmer` 替代 `useState`：

```tsx
import { useImmer } from "use-immer"

const [state, updateState] = useImmer({
  user: { name: "", settings: { theme: "light", notifications: true } },
})

// 直接修改 draft，无需手动展开
updateState(draft => {
  draft.user.settings.theme = "dark"
})
```

需要安装：
```bash
bun add immer use-immer
```

## 10. 代码组织：合理拆分文件

**禁止将所有代码堆在一个文件中**。按以下规则拆分：

### 页面级组件拆分

每个页面路由下可建立 `components/` 子目录存放该页面专属组件：

```
app/blog/
├── page.tsx              # 页面入口，组合各子组件
├── components/
│   ├── BlogList.tsx      # 博客列表
│   ├── BlogCard.tsx      # 博客卡片
│   └── BlogFilter.tsx    # 筛选器
└── loading.tsx
```

### 提取规则

| 场景 | 放置位置 |
|------|---------|
| 仅当前页面使用的组件 | `@/app/<page>/components/` |
| 多个页面复用的组件 | `@/components/` |
| 全局通用的工具函数 | `@/lib/` |
| 全局通用的自定义 Hook | `@/hooks/` （如 `use-window-resize`、`use-local-storage`） |
| 自定义图标 | `@/components/icons/` |
| Server Action | `@/actions/<业务名>.ts` |

### 自定义 Hook 提取原则

当一段逻辑可以被抽象为 hook 时（如监听窗口变化、操作 localStorage、管理 interval 等），应提取到 `@/hooks/` 目录，文件命名使用 kebab-case：

```
hooks/
├── use-window-resize.ts
├── use-local-storage.ts
├── use-debounce.ts
└── use-media-query.ts
```

## 11. 图标：lucide-react

使用图标时优先从 `lucide-react` 导入：

```tsx
import { Search, ChevronRight, X } from "lucide-react"
```

如果 `lucide-react` 中没有需要的图标，自行设计 SVG 图标组件，放到 `@/components/icons/` 目录：

```tsx
// @/components/icons/CustomIcon.tsx
export function CustomIcon(props: React.SVGProps<SVGSVGElement>) {
  return (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" {...props}>
      {/* SVG path */}
    </svg>
  )
}
```

## 12. 动画：framer-motion

需要动画效果时，优先使用 `framer-motion`：

```tsx
import { motion } from "framer-motion"

<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  exit={{ opacity: 0 }}
  transition={{ duration: 0.3 }}
>
  内容
</motion.div>
```

如果动画需求是纯 CSS 层面的自定义属性（如复杂渐变、伪类状态），则使用 Tailwind CSS v4 的 `@utility` 定义（见第 2 条）。

## 13. 时间处理：dayjs

格式化、解析、操作时间统一使用 `dayjs`，**不要**使用 `date-fns` 或原生 `Date` 格式化：

```tsx
import dayjs from "dayjs"

dayjs(date).format("YYYY-MM-DD HH:mm")
dayjs().subtract(7, "day")
```

如果项目中未安装，需先执行：
```bash
bun add dayjs
```

## 14. 唯一标识：uuid

需要生成唯一标识时，统一使用 `uuid` 库的 v4：

```tsx
import { v4 as uuid } from "uuid"

const id = uuid()
```

如果项目中未安装，需先执行：
```bash
bun add uuid
bun add -d @types/uuid
```

## 15. 工具函数：es-toolkit

需要工具函数时，使用 `es-toolkit` 替代 lodash（更小体积、tree-shakeable）。

> **注意**：如果 JS 原生已有对应方法（如 `String.prototype.trim()`、`Array.prototype.map()` 等），直接使用原生方法，不要强行用 es-toolkit。es-toolkit 只用于原生不具备或实现较复杂的场景（如 `debounce`、`groupBy`、`cloneDeep` 等）。

**导入规则**：从子路径按分类导入，不要从顶层导入：

```tsx
// ✅ 正确：从子路径导入
import { debounce } from "es-toolkit/function"
import { groupBy } from "es-toolkit/array"
import { pick, omit } from "es-toolkit/object"
import { clamp, random } from "es-toolkit/math"
import { isNil, isEqual } from "es-toolkit/predicate"
import { camelCase } from "es-toolkit/string"

// ❌ 错误：从顶层导入
import { debounce } from "es-toolkit"
```

### 可用方法分类

| 分类 | 导入路径 | 常用方法 |
|------|---------|---------|
| 数组 | `es-toolkit/array` | chunk, compact, countBy, difference, flatten, groupBy, keyBy, orderBy, partition, sample, shuffle, sortBy, uniq, uniqBy, zip |
| 函数 | `es-toolkit/function` | debounce, throttle, once, memoize, curry, noop, retry, flow |
| 对象 | `es-toolkit/object` | clone, cloneDeep, merge, omit, omitBy, pick, pickBy, invert, flattenObject |
| 数学 | `es-toolkit/math` | clamp, inRange, mean, median, random, randomInt, range, round, sum, sumBy |
| 谓词 | `es-toolkit/predicate` | isNil, isNotNil, isEqual, isPlainObject, isEmpty, isString, isNumber, isFunction |
| 字符串 | `es-toolkit/string` | camelCase, kebabCase, snakeCase, pascalCase, capitalize, escape, trim |
| Promise | `es-toolkit/promise` | delay, timeout, withTimeout, Mutex, Semaphore |
| 错误 | `es-toolkit/error` | AbortError, TimeoutError |

如果项目中未安装，需先执行：
```bash
bun add es-toolkit
```

## 16. 可维护性

编写业务逻辑时，关注代码的可维护性，但不要过度设计：

- 复杂的业务逻辑加上必要的注释，说明**为什么**这样做，而不是**做了什么**
- 存在多个分支/状态切换的逻辑，考虑用常量或枚举替代魔法值
- 涉及特殊业务规则（如定价策略、权限判断、状态机）的代码，将规则抽离为独立函数或配置，便于后续修改
- 不要为了"可维护"强行抽象简单逻辑，三行代码能解决的事不需要拆成一个 util

## 17. 性能优化：useCallback / useMemo

合理使用 `useCallback` 和 `useMemo`，但**不要无脑包裹**：

### 应该使用的场景

- 作为 props 传递给被 `React.memo` 包裹的子组件的函数或计算值
- 计算量较大的派生数据（如对长列表的过滤、排序、聚合）
- 作为其他 Hook（`useEffect`、`useMemo`）依赖项的函数引用

### 不需要使用的场景

- 组件内部使用的简单函数，不传给子组件
- 计算量极小的表达式（如字符串拼接、简单条件判断）
- 原始值（string、number、boolean）— 它们本身就是稳定引用

```tsx
// ✅ 有意义：传给 memo 子组件的回调
const handleDelete = useCallback((id: string) => {
  // ...
}, [])

// ✅ 有意义：大列表过滤
const filteredItems = useMemo(
  () => items.filter(item => item.category === activeCategory),
  [items, activeCategory]
)

// ❌ 没必要：组件内部直接用的简单函数
const handleClick = useCallback(() => {
  setOpen(true)
}, []) // 多此一举
```

## 18. 渲染策略：优先服务端渲染

编写页面时，**优先使用 Server Component**（即默认不加 `"use client"`），不强求但尽量遵循：

- 页面组件（`page.tsx`）、布局组件（`layout.tsx`）尽量保持为 Server Component
- 数据获取放在服务端完成，减少客户端请求
- 只有需要交互（事件绑定、useState、useEffect 等）的部分才拆为 Client Component

```
app/blog/
├── page.tsx              # Server Component：获取数据、组合布局
└── components/
    ├── BlogList.tsx      # Server Component：纯展示
    └── BlogFilter.tsx    # "use client"：有交互逻辑
```

> 原则：能在服务端完成的就不推到客户端，但如果业务场景确实需要全客户端渲染，不必强行拆分。

## 19. Next.js 内置组件：Image 和 Link

在 Next.js 项目中，**必须使用内置组件替代原生 HTML 标签**：

- `<img>` → 使用 `next/image` 的 `<Image>` 组件
- `<a>` → 使用 `next/link` 的 `<Link>` 组件

```tsx
import Image from "next/image"
import Link from "next/link"

// ✅ 正确
<Image src="/logo.png" alt="Logo" width={120} height={40} />
<Link href="/about">About</Link>

// ❌ 错误
<img src="/logo.png" alt="Logo" />
<a href="/about">About</a>
```

### Image 使用要点
- 必须提供 `width` 和 `height`，或使用 `fill` 属性
- 外部图片域名需在 `next.config.ts` 的 `images.remotePatterns` 中配置

### Link 使用要点
- 内部导航一律使用 `<Link>`，自动支持客户端路由和预加载
- 外部链接可以使用 `<a>` 标签（需要 `target="_blank"` 和 `rel="noopener noreferrer"`）
