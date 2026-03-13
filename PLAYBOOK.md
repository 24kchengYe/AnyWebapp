# 全栈 Web 应用开发手册 — 独立开发者实战指南

> 基于 Human Skill Tree 项目（2 周，从 0 到上线收费）的完整经验总结。
> 适用于：Next.js + Supabase + Vercel + AI 类 SaaS/工具产品。

---

## 目录

1. [Phase 0: 项目初始化](#phase-0-项目初始化)
2. [Phase 1: MVP 核心功能](#phase-1-mvp-核心功能)
3. [Phase 2: UI/UX 打磨](#phase-2-uiux-打磨)
4. [Phase 3: 数据持久化](#phase-3-数据持久化)
5. [Phase 4: 认证系统](#phase-4-认证系统)
6. [Phase 5: 付费系统](#phase-5-付费系统)
7. [Phase 6: 国际化 (i18n)](#phase-6-国际化-i18n)
8. [Phase 7: 部署与域名](#phase-7-部署与域名)
9. [Phase 8: 国内可访问](#phase-8-国内可访问)
10. [Phase 9: 推广上线](#phase-9-推广上线)
11. [踩坑记录](#踩坑记录)
12. [技术选型速查表](#技术选型速查表)
13. [检查清单](#检查清单)

---

## Phase 0: 项目初始化

### 技术栈选择

```
框架:     Next.js 16 (App Router) + TypeScript
样式:     Tailwind CSS v4 + shadcn/ui
数据库:   Supabase (PostgreSQL + Auth + Realtime)
AI:       Vercel AI SDK v6 + OpenRouter
部署:     Vercel
CDN:      Cloudflare (可选, 国内加速用)
支付:     LemonSqueezy (海外) + 爱发电 (国内)
i18n:     next-intl
```

### 初始化命令

```bash
npx create-next-app@latest my-app --typescript --tailwind --eslint --app --src-dir
cd my-app

# 核心依赖
npm install ai @ai-sdk/openai          # Vercel AI SDK
npm install next-intl                   # 国际化
npm install next-themes                 # 主题切换
npm install @supabase/supabase-js @supabase/ssr  # Supabase

# UI 组件 (按需)
npx shadcn@latest init
npm install @xyflow/react               # 如果需要节点图/流程图

# 开发工具
npm install -D @types/node
```

### 项目结构规划

```
src/
├── app/
│   ├── [locale]/              # i18n 路由 (如不需要多语言则不用)
│   │   ├── page.tsx           # 首页/Landing
│   │   ├── dashboard/         # 主功能页
│   │   └── layout.tsx         # 导航 + Provider
│   └── api/
│       ├── chat/route.ts      # AI 对话 API
│       ├── auth/callback/     # OAuth 回调
│       └── webhooks/          # 支付 webhook
├── components/
│   ├── ui/                    # shadcn/ui 基础组件
│   ├── landing/               # 首页区块
│   └── [feature]/             # 按功能分组
├── lib/
│   ├── supabase/
│   │   ├── client.ts          # 浏览器端 client
│   │   ├── server.ts          # 服务端 client
│   │   └── middleware.ts      # Auth session 刷新
│   ├── models.ts              # AI 模型配置
│   └── constants.ts           # 全局常量
├── i18n/                      # next-intl 配置
│   ├── routing.ts
│   ├── request.ts
│   └── navigation.ts
└── middleware.ts               # 全局中间件 (i18n + auth)
```

### 环境变量模板

```bash
# .env.local.example

# AI (必须)
OPENAI_API_KEY=sk-or-v1-xxxx           # OpenRouter API Key
OPENAI_BASE_URL=https://openrouter.ai/api/v1

# Supabase (认证/数据库)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
SUPABASE_SERVICE_ROLE_KEY=eyJxxx       # 仅服务端, 不暴露到前端

# 支付
NEXT_PUBLIC_LS_BASIC_CHECKOUT=https://xxx.lemonsqueezy.com/buy/xxx
NEXT_PUBLIC_LS_PRO_CHECKOUT=https://xxx.lemonsqueezy.com/buy/xxx
LS_WEBHOOK_SECRET=whsec_xxx
NEXT_PUBLIC_AFDIAN_URL=https://afdian.com/a/xxx

# 管理
NEXT_PUBLIC_ADMIN_EMAILS=your@email.com
```

### Git 策略

```bash
# 初始化
git init
git remote add origin https://github.com/you/repo.git

# 如果代码含付费逻辑 → 仓库设 private
# 如果纯开源 → public

# 部署不依赖 Git, 用 Vercel CLI
npm i -g vercel
```

---

## Phase 1: MVP 核心功能

### 原则

> **先跑通核心链路, 再迭代。** MVP 阶段不做: 认证、支付、i18n、主题、动画。

### AI 对话 API 搭建

```typescript
// src/app/api/chat/route.ts
import { streamText } from "ai";
import { createOpenAI } from "@ai-sdk/openai";

const openai = createOpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: process.env.OPENAI_BASE_URL,
  compatibility: "compatible",  // 关键: OpenRouter 兼容模式
});

export async function POST(request: Request) {
  const { messages, model } = await request.json();

  const result = streamText({
    model: openai.chatModel(model || "deepseek/deepseek-chat-v3-0324"),
    messages,
    system: "你的系统提示词",
  });

  return result.toDataStreamResponse();
}
```

**关键点:**
- `compatibility: "compatible"` — OpenRouter 不完全支持 OpenAI Responses API, 必须用 Chat Completions
- Vercel AI SDK v6 默认用 Responses API, 要显式切换
- 流式响应用 `toDataStreamResponse()`

### 前端调用

```typescript
// 使用 Vercel AI SDK 的 useChat hook
import { useChat } from "ai/react";

const { messages, input, handleInputChange, handleSubmit, isLoading, stop } = useChat({
  api: "/api/chat",
  body: { model: selectedModel },
});
```

### 本地数据存储 (MVP 阶段)

```typescript
// 先用 localStorage, 后续再迁移到 Supabase
function saveData(key: string, data: unknown) {
  localStorage.setItem(key, JSON.stringify(data));
}

function loadData<T>(key: string, defaultValue: T): T {
  const stored = localStorage.getItem(key);
  return stored ? JSON.parse(stored) : defaultValue;
}
```

**经验: localStorage 先行, Supabase 后补。** 这样 MVP 阶段不需要任何后端, 也不需要注册登录。等产品验证后再加认证和云同步。

---

## Phase 2: UI/UX 打磨

### 暗色主题基础

```css
/* Tailwind CSS v4 - 全局变量 */
@theme {
  --color-background: oklch(0.145 0 0);
  --color-foreground: oklch(0.985 0 0);
  --color-muted-foreground: oklch(0.708 0 0);
  --color-border: oklch(0.269 0 0);
  --color-accent: oklch(0.269 0 0);
  --color-popover: oklch(0.205 0 0);
}
```

### 氛围光效 (Ambient Glow)

```html
<!-- 背景光晕, 放在容器最前面 -->
<div class="pointer-events-none absolute top-[-20%] left-1/2 -translate-x-1/2
  h-[500px] w-[800px] rounded-full bg-purple-600/10 blur-[120px]" />
```

### 毛玻璃导航栏

```html
<nav class="sticky top-0 z-50 flex h-16 items-center justify-between
  border-b border-border/50 bg-background/70 px-4 backdrop-blur-xl">
```

### z-index 层级管理

```
z-10   正常浮动元素
z-20   导航栏下拉菜单 (需要 CSS `isolate` 在父容器)
z-50   模态框 overlay
z-[200] 全屏弹窗 (如定价弹窗)
```

**踩坑:** 如果 header 里的下拉菜单被页面内容挡住, 在 header 父容器加 `isolate` class 创建新的 stacking context。

### 移动端适配关键点

- 侧边栏用 `fixed inset-0 z-50` 全屏覆盖, 不用 transform 动画 (性能差)
- 输入框底部固定: `sticky bottom-0`
- 长列表: `overflow-y-auto` + `max-h-[calc(100vh-xxx)]`
- 触控区域: 按钮最小 44×44px

---

## Phase 3: 数据持久化

### localStorage → Supabase 迁移策略

```
阶段1 (MVP):     全部 localStorage
阶段2 (上线):    localStorage + Supabase 双写
阶段3 (成熟):    Supabase 为主, localStorage 做缓存
```

### Supabase Client 配置

```typescript
// src/lib/supabase/client.ts (浏览器端)
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}

// src/lib/supabase/server.ts (服务端)
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          );
        },
      },
    }
  );
}
```

### Middleware Session 刷新

```typescript
// src/lib/supabase/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest, response: NextResponse) {
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) => {
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );
  await supabase.auth.getUser();
  return response;
}
```

### 数据库表设计 (典型 SaaS)

```sql
-- 用户资料 + 计划
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE,
  avatar_url TEXT,
  plan TEXT DEFAULT 'free' CHECK (plan IN ('free', 'basic', 'pro', 'admin')),
  plan_expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 用量追踪
CREATE TABLE usage_logs (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID REFERENCES profiles(id),
  action TEXT NOT NULL,  -- 'message', 'export', etc.
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 云同步 (通用 KV 存储)
CREATE TABLE user_data (
  user_id UUID REFERENCES profiles(id),
  data_key TEXT NOT NULL,
  data_value JSONB NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (user_id, data_key)
);

-- RLS 策略: 用户只能访问自己的数据
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can read own profile" ON profiles
  FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile" ON profiles
  FOR UPDATE USING (auth.uid() = id);

-- 自动创建 profile
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id) VALUES (NEW.id);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

### 双向同步方案

```typescript
// 上传: localStorage → Supabase
async function uploadToCloud(userId: string) {
  const keys = ["chat-history", "learning-data", "settings"];
  for (const key of keys) {
    const data = localStorage.getItem(key);
    if (data) {
      await supabase.from("user_data").upsert({
        user_id: userId,
        data_key: key,
        data_value: JSON.parse(data),
        updated_at: new Date().toISOString(),
      });
    }
  }
}

// 下载: Supabase → localStorage
async function downloadFromCloud(userId: string) {
  const { data } = await supabase
    .from("user_data")
    .select("data_key, data_value")
    .eq("user_id", userId);

  data?.forEach(({ data_key, data_value }) => {
    localStorage.setItem(data_key, JSON.stringify(data_value));
  });
}
```

**同步时机:**
- 登录成功 → `downloadFromCloud()` (云端覆盖本地)
- 登出前 → `uploadToCloud()` (本地上传到云端)
- 定时同步 → 每 5 分钟 `uploadToCloud()` (后台静默)

---

## Phase 4: 认证系统

### Supabase Auth 配置步骤

#### 1. Supabase Dashboard 设置

```
Authentication → Providers:
  ✅ Email (关闭 Confirm Email, 除非你有自定义域名的邮件服务)
  ✅ Google  (填 Client ID + Secret)
  ✅ GitHub  (填 Client ID + Secret)

Authentication → URL Configuration:
  Site URL: https://your-domain.com
  Redirect URLs:
    https://your-domain.com/**
    https://your-custom-domain.com/**
    http://localhost:3000/**
```

#### 2. Google OAuth 配置

```
1. https://console.cloud.google.com/apis/credentials
2. 创建 OAuth 2.0 Client ID → Web application
3. Authorized redirect URIs: https://xxx.supabase.co/auth/v1/callback
4. 把 Client ID + Secret 填到 Supabase → Providers → Google
```

#### 3. GitHub OAuth 配置

```
1. https://github.com/settings/developers → New OAuth App
2. Authorization callback URL: https://xxx.supabase.co/auth/v1/callback
3. 把 Client ID + Secret 填到 Supabase → Providers → GitHub
```

### 前端 Auth Provider

```typescript
// src/components/auth/auth-provider.tsx
"use client";
import { createContext, useContext, useEffect, useState } from "react";

type AuthContextType = {
  user: User | null;
  loading: boolean;
  planInfo: PlanInfo | null;
  signOut: () => Promise<void>;
};

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const supabase = createClient();

    // 初始 session
    supabase.auth.getSession().then(({ data: { session } }) => {
      setUser(session?.user ?? null);
      setLoading(false);
    });

    // 监听变化
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      async (event, session) => {
        setUser(session?.user ?? null);
        if (event === "SIGNED_IN") {
          await downloadFromCloud(session.user.id);
        } else if (event === "SIGNED_OUT") {
          // 清理
        }
      }
    );

    return () => subscription.unsubscribe();
  }, []);

  // ...
}
```

### Auth 回调路由

```typescript
// src/app/api/auth/callback/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) {
      return NextResponse.redirect(`${origin}/?auth=confirmed`);
    }
  }

  return NextResponse.redirect(`${origin}/?auth=error`);
}
```

### 邮件服务 (Resend)

```
注意: Resend 免费版 (无自定义域名) 只能发给绑定邮箱。
建议: 初期关闭邮箱确认, 等有自定义域名后再开启。

Supabase SMTP 配置 (有自定义域名后):
  Host: smtp.resend.com
  Port: 465
  Username: resend
  Password: re_xxxx (Resend API Key)
  Sender: noreply@yourdomain.com
```

---

## Phase 5: 付费系统

### 方案选择

| 方案 | 适用 | 特点 |
|------|------|------|
| LemonSqueezy | 海外用户 | 信用卡/PayPal, Webhook 自动化, 个人可用 |
| Stripe | 海外用户 | 功能最强, 但需要公司主体 |
| 爱发电 | 国内用户 | 微信/支付宝, 无 Webhook, 需手动核实 |

### LemonSqueezy 配置

```
1. lemonsqueezy.com → 创建 Store → 创建 Products (Basic/Pro)
2. 获取 Checkout URL → 填入 NEXT_PUBLIC_LS_BASIC_CHECKOUT
3. Settings → Webhooks → 添加 Webhook:
   URL: https://your-domain.com/api/webhooks/lemonsqueezy
   Events: order_created, subscription_created, subscription_updated
   Secret: 填入 LS_WEBHOOK_SECRET
```

### Webhook 处理

```typescript
// src/app/api/webhooks/lemonsqueezy/route.ts
import crypto from "crypto";

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get("x-signature");

  // 验签
  const hmac = crypto.createHmac("sha256", process.env.LS_WEBHOOK_SECRET!);
  const digest = hmac.update(body).digest("hex");
  if (digest !== signature) {
    return new Response("Invalid signature", { status: 401 });
  }

  const event = JSON.parse(body);
  const { meta, data } = event;

  if (meta.event_name === "order_created") {
    const email = data.attributes.user_email;
    const productName = data.attributes.first_order_item.product_name;

    // 根据产品名判断计划
    const plan = productName.includes("Pro") ? "pro" : "basic";
    const expiresAt = new Date();
    expiresAt.setMonth(expiresAt.getMonth() + 1);

    // 更新数据库
    await supabase
      .from("profiles")
      .update({
        plan,
        plan_expires_at: expiresAt.toISOString(),
      })
      .eq("id", userId);  // 通过 email 查找 userId
  }

  return new Response("OK");
}
```

### 前端计划刷新

```typescript
// 付费后前端不会立即更新, 需要主动刷新机制

useEffect(() => {
  if (!user) return;

  // 每 60 秒刷新一次计划信息
  const interval = setInterval(() => fetchPlanInfo(), 60_000);

  // Tab 可见时刷新 (用户从支付页面回来)
  const handleVisibility = () => {
    if (document.visibilityState === "visible") fetchPlanInfo();
  };
  document.addEventListener("visibilitychange", handleVisibility);

  return () => {
    clearInterval(interval);
    document.removeEventListener("visibilitychange", handleVisibility);
  };
}, [user]);
```

### 模型分级限制

```typescript
// API 端校验 (不能只靠前端)
export async function POST(request: Request) {
  const { model, userId } = await parseRequest(request);

  // 查询用户计划
  const { data: profile } = await supabase
    .from("profiles")
    .select("plan, plan_expires_at")
    .eq("id", userId)
    .single();

  let plan = profile?.plan || "free";

  // 检查是否过期
  if (plan !== "free" && profile?.plan_expires_at &&
      new Date(profile.plan_expires_at) < new Date()) {
    plan = "free";
  }

  // 检查模型权限
  if (!canAccessModel(model, plan)) {
    return new Response("Upgrade required", { status: 403 });
  }

  // 检查日用量
  const todayUsage = await getTodayUsage(userId);
  const limit = PLAN_LIMITS[plan]; // { free: 10, basic: 100, pro: Infinity }
  if (todayUsage >= limit) {
    return new Response("Daily limit reached", { status: 429 });
  }

  // 记录用量
  await supabase.from("usage_logs").insert({
    user_id: userId, action: "message"
  });

  // 正常处理...
}
```

---

## Phase 6: 国际化 (i18n)

### next-intl 配置

```typescript
// src/i18n/routing.ts
import { defineRouting } from "next-intl/routing";

export const routing = defineRouting({
  locales: ["en", "zh", "ja"],
  defaultLocale: "en",
  localeDetection: false,  // 不自动检测浏览器语言
});
```

```typescript
// src/i18n/request.ts
import { getRequestConfig } from "next-intl/server";
import { routing } from "./routing";

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;
  if (!locale || !routing.locales.includes(locale)) {
    locale = routing.defaultLocale;
  }
  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default,
  };
});
```

### 翻译文件结构

```json
// messages/zh.json
{
  "nav": {
    "home": "首页",
    "dashboard": "控制台"
  },
  "auth": {
    "login": "登录",
    "logout": "退出"
  },
  "pricing": {
    "title": "升级计划",
    "month": "月"
  }
}
```

### 使用

```typescript
"use client";
import { useTranslations } from "next-intl";

function MyComponent() {
  const t = useTranslations("auth");
  return <button>{t("login")}</button>;
}
```

### 翻译工作量估算

一般一个中等复杂度 App, 每种语言约 200-300 个翻译 key。3 种语言 = 600-900 个 key。用 Claude 批量翻译, 约 30 分钟搞定。

---

## Phase 7: 部署与域名

### Vercel 部署

```bash
# 首次
npx vercel          # 选择项目设置
npx vercel --prod   # 部署到生产

# 之后每次
npx next build && npx vercel --prod
```

**为什么用 CLI 而不是 Git 集成:**
- Git 集成要求关联 Git 用户, 多账号时容易混乱
- CLI 更可控, 不会意外触发自动部署
- 可以在本地构建验证后再部署

### 环境变量

```bash
# 在 Vercel 设置环境变量 (不要在代码里写)
vercel env add OPENAI_API_KEY
vercel env add SUPABASE_SERVICE_ROLE_KEY
# 或在 Vercel Dashboard → Settings → Environment Variables 手动添加
```

### 自定义域名 (Vercel)

```
1. Vercel Dashboard → Project → Settings → Domains
2. 输入域名 → Add
3. 按提示在 DNS 服务商添加记录 (CNAME 或 A)
4. 等待验证 (1-5 分钟)
```

---

## Phase 8: 国内可访问

> 核心问题: Vercel 域名 (*.vercel.app) 在国内部分地区被墙。

### 方案: Cloudflare CNAME 代理

**优点:** 免费, 无需备案, 10 分钟搞定。

```
1. 你需要一个已接入 Cloudflare 的域名 (如 yechengzhang.com)

2. Vercel: Settings → Domains → 添加子域名
   例: app.yourdomain.com

3. Cloudflare: DNS → Add Record
   Type:   CNAME
   Name:   app  (你的子域名前缀)
   Target: cname.vercel-dns.com  (或 Vercel 给你的具体值)
   Proxy:  ✅ Proxied (橙色云朵, 必须开启!)

4. Cloudflare: SSL/TLS → Overview
   模式: Full (strict)  ← 重要! 否则会无限重定向

5. 等 1-2 分钟, 刷新 Vercel Domains 页面确认绿色 ✓
```

**Vercel 会提示 "Proxy Detected" — 忽略它, 不要点 "1-click fix"。**

### Middleware 地理检测

```typescript
// 根据 IP 自动切换语言
function getCountry(request: NextRequest): string {
  return (
    request.headers.get("cf-ipcountry") ||      // Cloudflare
    request.headers.get("x-vercel-ip-country") || // Vercel
    ""
  ).toUpperCase();
}

export async function middleware(request: NextRequest) {
  const country = getCountry(request);
  const isCN = country === "CN";
  const hasLocalePrefix = /^\/(en|zh|ja)(\/|$)/.test(request.nextUrl.pathname);

  // 中国 IP + 未选语言 → 自动跳转中文
  if (!hasLocalePrefix && isCN) {
    const url = request.nextUrl.clone();
    url.pathname = `/zh${request.nextUrl.pathname}`;
    const response = NextResponse.redirect(url);
    response.cookies.set("geo-cn", "1", { path: "/", maxAge: 86400 });
    return response;
  }

  // 正常处理...
}
```

### 费用

```
Cloudflare 免费套餐:
  ✅ 无限带宽 CDN
  ✅ DDoS 防护
  ✅ SSL 自动续期
  ✅ 无请求数限制

Vercel Hobby (免费):
  - 带宽: 100 GB/月
  - Serverless: 100,000 次/月
  - 构建: 6,000 分钟/月
  (几千用户内完全够用)
```

---

## Phase 9: 推广上线

### 推广渠道 (按优先级)

| 渠道 | 适合 | 时机 |
|------|------|------|
| 知乎专栏 | 技术深度文章 | 第 1 天 |
| 小红书 | 短平快, 破圈 | 第 1-7 天 (系列) |
| Hacker News | 海外开发者 | 第 4 天 |
| Reddit | 海外细分社区 | 第 5 天 |
| Product Hunt | 产品发布 | 第 2 周 |
| 即刻 | 独立开发者圈 | 持续 |

### README 最佳实践

```markdown
# 项目名

一句话介绍。

**Live Demo**: [链接](url)

## 功能演示 (GIF/截图)
## 核心功能列表
## 技术栈
## 快速开始 (3 步以内)
## 环境变量说明
## License
```

**必须有 GIF demo** — 没有 demo 的 README 转化率极低。

---

## 踩坑记录

### 1. OpenRouter + Vercel AI SDK

```
问题: Vercel AI SDK v6 默认使用 OpenAI Responses API, OpenRouter 不完全支持
解决: createOpenAI({ compatibility: "compatible" })
```

### 2. Turbopack 中文引号

```
问题: .ts 文件中的中文引号 "..." (U+201C/U+201D) 被 Turbopack 当作 JS 字符串解析
解决: 用 「...」 替换, 或用普通引号
```

### 3. Cloudflare + Vercel 无限重定向

```
问题: ERR_TOO_MANY_REDIRECTS
原因: Cloudflare SSL 设置为 Flexible (HTTP→HTTPS 循环)
解决: Cloudflare SSL/TLS → Full (strict)
```

### 4. Supabase Auth Session 丢失

```
问题: 刷新页面后登录状态丢失
原因: 没有在 middleware 中刷新 session cookie
解决: middleware.ts 中调用 supabase.auth.getUser() 刷新 cookie
```

### 5. z-index 层级混乱

```
问题: Header 下拉菜单被页面内容遮挡
原因: 父容器没有创建新的 stacking context
解决: 在 header 父容器加 `isolate` class (CSS isolation: isolate)
```

### 6. Resend 免费版限制

```
问题: 发送确认邮件报错 "Email rate limit exceeded"
原因: Resend 无自定义域名时只能发给绑定邮箱
解决: 暂时关闭 Supabase 邮箱确认, 等有自定义域名再开启
```

### 7. 付费后前端不更新

```
问题: Webhook 已更新数据库, 但前端仍显示 Free
原因: planInfo 只在登录时获取, 没有刷新机制
解决: 60s 定时轮询 + visibilitychange 事件刷新
```

### 8. 移动端虚拟键盘

```
问题: 输入框获得焦点时, 虚拟键盘挤压布局
解决: 输入框容器用 sticky bottom-0, 不用 fixed
```

---

## 技术选型速查表

### 框架

| 需求 | 选择 | 原因 |
|------|------|------|
| SSR + SSG + API Routes | Next.js | 全栈一体, Vercel 原生 |
| 纯 SPA | Vite + React | 更轻量, 不需要 SSR |
| 极简 | Astro | 内容站, 几乎零 JS |

### 数据库

| 需求 | 选择 | 原因 |
|------|------|------|
| Auth + DB + Realtime | Supabase | 开箱即用, 免费额度够 |
| 纯 DB | PlanetScale / Neon | MySQL / PostgreSQL |
| 无后端 | localStorage + IndexedDB | MVP 阶段 |

### AI

| 需求 | 选择 | 原因 |
|------|------|------|
| 多模型切换 | OpenRouter | 一个 key 18+ 模型 |
| 单模型 | 直连 API | Anthropic/OpenAI SDK |
| 流式 UI | Vercel AI SDK | useChat hook 开箱即用 |

### 支付

| 需求 | 选择 | 原因 |
|------|------|------|
| 海外个人 | LemonSqueezy | 无需公司主体, Webhook |
| 海外公司 | Stripe | 功能最强 |
| 国内 | 爱发电 | 微信/支付宝, 无 Webhook |

### 部署

| 需求 | 选择 | 原因 |
|------|------|------|
| Next.js | Vercel | 原生支持, 免费 |
| 国内加速 | Cloudflare CNAME 代理 | 免费, 无需备案 |
| Docker | Railway / Fly.io | 需要自定义运行时 |

---

## 检查清单

### 上线前

- [ ] `npx next build` 本地构建通过
- [ ] 环境变量全部配置到 Vercel
- [ ] Supabase RLS 策略已启用
- [ ] API 路由有认证 + 用量检查
- [ ] Webhook 有签名验证
- [ ] 敏感信息不在前端暴露 (SUPABASE_SERVICE_ROLE_KEY 等)
- [ ] 移动端测试通过
- [ ] 错误处理: API 超时、网络断连、模型不可用

### 域名配置后

- [ ] Supabase Redirect URLs 添加新域名
- [ ] OAuth Provider 回调 URL 确认 (Google/GitHub → Supabase)
- [ ] Cloudflare SSL → Full (strict)
- [ ] HTTPS 访问正常
- [ ] 国内网络访问测试 (关掉 VPN)

### 推广前

- [ ] README 有 demo GIF
- [ ] Landing page 有明确 CTA
- [ ] 移动端体验正常
- [ ] 访客计数徽章
- [ ] Star History 图表

---

*基于 Human Skill Tree 项目实战经验 · 2026-03*
