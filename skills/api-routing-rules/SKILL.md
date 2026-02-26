---
name: api-routing-rules
description: Next.js API 路由编写规范。当需要编写 API 路由时，Claude 必须遵循这些规则。
disable-model-invocation: false
user-invocable: false
---

# Next.js API 路由编写规范

## 优先使用 Server Action

大部分场景**不需要 API 路由**，优先使用 Server Action 解决：

| 场景 | 方案 |
|------|------|
| 表单提交、数据变更 | Server Action（`@/actions/<业务名>.ts`） |
| 需要给第三方/外部调用的接口 | API 路由 |
| Webhook 回调 | API 路由 |
| 需要流式响应（SSE/Streaming） | API 路由 |
| 文件上传（需要特殊处理） | API 路由 |

## 路由命名规范

- 使用 **kebab-case**，语义清晰
- 合理使用动态路由 `[param]`
- RESTful 风格组织资源

```
app/api/
├── prayers/
│   ├── route.ts              # GET /api/prayers（列表）、POST /api/prayers（创建）
│   └── [id]/
│       └── route.ts          # GET /api/prayers/:id、PUT、DELETE
├── auth/
│   ├── callback/
│   │   └── route.ts          # GET /api/auth/callback
│   └── verify-code/
│       └── route.ts          # POST /api/auth/verify-code
└── webhooks/
    └── stripe/
        └── route.ts          # POST /api/webhooks/stripe
```

## API 路由模板

### 基本结构

每个 handler 必须用 `try/catch` 整体包裹，统一返回格式：

```tsx
import { NextRequest, NextResponse } from "next/server"

export const runtime = "edge"

export const GET = async (req: NextRequest) => {
  try {
    const { searchParams } = req.nextUrl
    const category = searchParams.get("category") || ""

    return NextResponse.json({
      code: 0,
      message: "success",
      data: category,
    })
  } catch (error: any) {
    return NextResponse.json({
      code: 500,
      message: error?.message || error?.toString(),
      data: null,
    })
  }
}
```

### 带动态路由参数

```tsx
import { NextRequest, NextResponse } from "next/server"

export const runtime = "edge"

export const GET = async (
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) => {
  try {
    const { id } = await params

    return NextResponse.json({
      code: 0,
      message: "success",
      data: { id },
    })
  } catch (error: any) {
    return NextResponse.json({
      code: 500,
      message: error?.message || error?.toString(),
      data: null,
    })
  }
}
```

### POST 请求

```tsx
import { NextRequest, NextResponse } from "next/server"

export const runtime = "edge"

export const POST = async (req: NextRequest) => {
  try {
    const body = await req.json()

    // 业务逻辑...

    return NextResponse.json({
      code: 0,
      message: "success",
      data: body,
    })
  } catch (error: any) {
    return NextResponse.json({
      code: 500,
      message: error?.message || error?.toString(),
      data: null,
    })
  }
}
```

## 响应格式

**统一使用以下 JSON 结构**：

```ts
{
  code: number,    // 0 = 成功，非 0 = 失败
  message: string, // 描述信息
  data: T | null,  // 业务数据
}
```

### 规则

- `code: 0` → 操作成功
- `code: 非 0` → 操作失败（如 400、401、403、500 等，用于区分错误类型）
- HTTP 状态码**始终返回 200**，不要返回真实的 HTTP 错误状态码（如 500）
- 错误信息通过 `code` 和 `message` 字段传达，而非 HTTP 状态码

### 常用 code 值

| code | 含义 |
|------|------|
| 0 | 成功 |
| 400 | 参数错误 |
| 401 | 未认证 |
| 403 | 无权限 |
| 500 | 服务器内部错误 |

## 其他约定

- 默认使用 `export const runtime = "edge"` 开启 Edge Runtime
- 纯服务端逻辑文件加 `import "server-only"`（非 route.ts 的辅助模块）
- 需要 Supabase 时，使用 `@/lib/supabase/server.ts` 中的服务端实例
