+++
title = "Next 学习笔记（持续更新中）"
date = "2026-04-23T18:02:16+08:00"
draft = false
+++

# 📘 Next.js App Router 核心笔记

## 一、约定式页面组件

| 文件            | 作用                                                         |
| --------------- | ------------------------------------------------------------ |
| `page.tsx`      | 页面组件                                                     |
| `layout.tsx`    | 布局组件，状态持久化                                         |
| `template.tsx`  | 布局组件，状态不持久，路由切换时重置 state 和 useEffect      |
| `loading.tsx`   | 页面加载时的占位组件（如接口请求期间）                       |
| `error.tsx`     | 页面报错时展示的组件                                         |

---

## 二、路由导航（4种方式）

### 1️⃣ `Link` 组件

```tsx
import Link from "next/link";

<Link
  href={{ pathname: "/about/me", query: { name: "张三" } }}
  prefetch={true}
  scroll={true}
  replace={true}
>
  me
</Link>
```

- `prefetch`：可视区内出现时预加载目标页面
- `scroll`：是否自动保存滚动位置
- `replace`：不保存历史记录

🔍 获取 query 参数

```tsx
import { useSearchParams } from "next/navigation";

const searchParams = useSearchParams();
const name = searchParams.get("name");      // 单个
const names = searchParams.getAll("name");  // 多个
```

### 2️⃣ useRouter Hook

```tsx
import { useRouter } from "next/navigation";

const router = useRouter();
router.push("/about/me?name=张三");
router.replace("/page");
router.back();
router.forward();
router.refresh();
router.prefetch("/about");
```

### 3️⃣ redirect / permanentRedirect

```tsx
import { redirect, permanentRedirect } from "next/navigation";

redirect("/login");              // 307
permanentRedirect("/home");      // 308
```


## 三、动态路由

### 1️⃣ 获取参数方式

```tsx
import { useParams } from "next/navigation";
const params = useParams();
console.log(params);
```

### 2️⃣ 三种定义方式

| 模式            | 路径示例                                                        |params 值       |
| ------------------ | --------------------- | ------------------------ |
| `[id]`      | 	`/shop/123`               | `{ id: "123" }` |
| `[...id]`    | `/shop/123/456/789`       | `{ id: ["123", "456", "789"] }`|
| `[[...id]]`  | `/shop 或 /shop/123`      |`{} 或 { id: ["123"] }`   |


## 四、平行路由

- 通过 `@文件夹名` 定义，自动注入到 `layout` 中，与 `children` 平级
- ⚠️ 硬刷新会 404，需定义 `default.tsx` 解决

## 五、路由组
- 文件夹命名 `(groupName)`，不影响 URL 路径
- 可为不同组独立配置 `layout`

```text
app/
├── (a)/
│   └── page.tsx
├── (b)/
│   └── page.tsx
```

##  六、路由处理程序（API Routes）

### 1️⃣ 基本结构

```text
app/api/user/route.ts
```

### 2️⃣ 请求处理示例

```ts
import { NextRequest, NextResponse } from "next/server";

// GET with query params
export async function GET(request: NextRequest) {
  const id = request.nextUrl.searchParams.get("id");
  return NextResponse.json({ id });
}

// GET with dynamic params
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  return NextResponse.json({ message: `Hello, ${id}!` });
}

// POST with body
export async function POST(request: NextRequest) {
  const body = await request.json();
  return NextResponse.json(body);
}
```

### 3️⃣ Cookies 操作
```ts
import { cookies } from "next/headers";

const cookieStore = await cookies();
cookieStore.set("token", "value", { maxAge: 86400, httpOnly: true });
const token = cookieStore.get("token");
console.log(token?.value);

```

## 七、AI 集成（DeepSeek）

### 1️⃣ 安装依赖

```bash
npm i ai @ai-sdk/deepseek @ai-sdk/react
```

### 2️⃣ 创建 API 路由 app/api/chat/route.ts

```ts
import { createDeepSeek } from "@ai-sdk/deepseek";
import { streamText, convertToModelMessages } from "ai";

const deepseek = createDeepSeek({ apiKey: process.env.DEEPSEEK_API_KEY });

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = await streamText({
    model: deepseek("deepseek-chat"),
    messages: await convertToModelMessages(messages),
    system: "你是一个超级助手",
  });
  return result.toUIMessageStreamResponse();
}
```

### 3️⃣ 客户端使用
```tsx
import { useChat } from "@ai-sdk/react";

const { messages, sendMessage } = useChat(); // 默认请求 /api/chat
```

## 八、Proxy 代理

### 1️⃣ 主要功能
- **跨域请求处理** （避免 CORS）
- **接口转发**（隐藏真实后端地址）
- **限流、鉴权、缓存** 等中间层逻辑

### 2️⃣ 使用方式

建立文件`src/proxy.ts`，通过导出一个`proxy`函数，以下是配置跨域的例子

```tsx
import { NextRequest, NextResponse } from "next/server";
import { ProxyConfig } from "next/server";
export async function proxy(request: NextRequest) {
    const response = NextResponse.next();
    Object.entries(corsHeaders).forEach(([key, value]) => {
        response.headers.set(key, value);
    })
    return response;
}

const corsHeaders = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
}

export const config: ProxyConfig = {
   matcher:'/api/:path*',
}
```

同时，可通过 `next.config.js` 的 `rewrites` 实现轻量代理：
```tsx
module.exports = {
  async rewrites() {
    return [
      {
        source: "/api/proxy/:path*",
        destination: "https://backend.example.com/:path*",
      },
    ];
  },
};
```

## 九、客户端组件 vs 服务端组件

| 特性            | 客户端组件                                                     | 服务端组件                |
| ---------------------- | ------------------------------------------------------------ | ------------------------- |
| 指令      | `"use client"`                                                   | 	默认（或 `"use server"`） |
| 可使用 Hook          | ✅                                                         | ❌ |
| 可使用浏览器 API          | ✅                                                       | ❌ |
| 嵌套限制            | 不可嵌套服务端组件                       | 可以嵌套客户端组件 |
| 访问敏感数据		    | ❌                                        | ✅  |

> 若希望某个模块只在服务端使用，可以 import "server-only"。

## 十、缓存组件（实验性）
> 需要 `next.config.ts` 配置 `experimental.cacheComponents = true`

### 1️⃣ 静态内容（预渲染）
以下情况组件会变成完全静态：
- 读取本地文件：`fs.readFileSync(...)`
- 动态导入静态数据：`await import('data.json')`
- 纯计算（不含动态请求）`const names = impData.list.map(item=>item.name).join(',')`

### 2️⃣ 动态内容 + Suspense
必须搭配 `Suspense` 使用，Next.js 通过 PPR（Partial Prerendering）提供静态外壳 + 动态流式填充。
```tsx
import { Suspense } from "react";

export default function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <DynamicComponent />
    </Suspense>
  );
}
```
> ⚠️ 非确定性函数如 random()、Date.now() 单独使用会报错，需结合 Suspense 和 connection() 标记。

### 3️⃣ 'use cache' 与 cacheLife
```tsx
"use cache";
cacheLife("seconds"); // 预设：stale 5s, revalidate 10s, expire 60s

// 或自定义
cacheLife({
  stale: 5, // 客户端在此时间内直接使用缓存，不向服务器发请求(单位:秒)
  revalidate: 10,  // 超过此时间后，服务器收到请求时会在后台重新生成内容(单位:秒)
  expire: 60, // 超过此时间无访问，缓存完全失效，下次请求需要等待重新计算(单位:秒)
});
```

## 十一、缓存策略（较旧版本）

**未启用 `cacheComponents`**
默认情况下 `fetch` 会被缓存（静态内容）。退出缓存的方法：
```tsx
// 1. 页面级别
export const revalidate = 0;
export const dynamic = "force-dynamic";

// 2. fetch 选项
fetch(url, { cache: "no-store" });

// 3. 使用 cookies / headers / searchParams 自动变为动态
```

## 十二、Image 组件

### 1️⃣ 基础用法
```tsx
import Image from "next/image";

// 本地图片
import logo from "./logo.png";
<Image src={logo} alt="logo" />; // 自动宽高

// 或者直接路径 + 宽高
<Image src="/logo.png" width={100} height={100} alt="logo" />;
```

### 2️⃣ 远程图片配置

```js
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "cdn.example.com",
        pathname: "/images/**",
      },
    ],
  },
};
```

### 3️⃣ 优点
- 自动懒加载、WebP/AVIF 优化
- 防止 CLS（布局偏移）
- 支持 placeholder="blur" 模糊占位

### 4️⃣ 全局配置
```tsx
images: {
  formats: ['image/avif', 'image/webp'],
  deviceSizes: [640, 750, 828, 1080, 1200, 1920],
  imageSizes: [16, 32, 48, 64, 96],
}
```

## 十三、Script 组件

### 1️⃣ 基础用法
```tsx
import Script from "next/script";

<Script src="https://example.com/script.js" strategy="afterInteractive" />;
```
`strategy` 选项：
- `beforeInteractive`：阻塞页面渲染，在页面代码前加载
- `afterInteractive`：页面变为可交互后加载（默认）
- `lazyOnload`：浏览器空闲时加载

### 2️⃣ 事件监听
```tsx
<Script
  src="..."
  onLoad={() => console.log("loaded")}
  onReady={() => console.log("ready, each mount")}
  onError={(e) => console.error(e)}
/>
```
> `onReady` 在每次组件挂载时都会执行（例如路由切换后重新挂载）。

✅ 小提示
- 平行路由 + 硬刷新 → 记得添加 `default.tsx`
- API 路由中设置 Cookie 必须用 `NextResponse`，不能直接 `cookies().set`
- `redirect` 和 `permanentRedirect` 在非 Server Action 环境需结合 `try/catch` 使用
- 图片远程域名需配置 `remotePatterns`，避免使用 `dangerouslyAllowSVG` 等风险配置