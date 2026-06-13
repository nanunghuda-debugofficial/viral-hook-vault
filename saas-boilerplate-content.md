# SaaS BOILERPLATE PREMIUM
## Starter Kit Next.js + Supabase untuk Bangun Aplikasi SaaS
### Multi-Tenant, RBAC, Payment Gateway Lokal

---

## 📖 DOCUMENTATION

### TECH STACK:
- **Frontend:** Next.js 14 (App Router, Server Components, Server Actions)
- **Language:** TypeScript
- **Styling:** Tailwind CSS + Shadcn/UI
- **Database:** Supabase (PostgreSQL)
- **ORM:** Prisma
- **Auth:** Supabase Auth
- **Payment:** Midtrans & Xendit
- **Deployment:** Vercel / Docker / Railway

---

## 🗂️ PROJECT STRUCTURE

```
saas-boilerplate/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   ├── forgot-password/page.tsx
│   │   │   └── reset-password/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── dashboard/page.tsx
│   │   │   ├── settings/page.tsx
│   │   │   ├── billing/page.tsx
│   │   │   └── team/page.tsx
│   │   ├── (admin)/
│   │   │   ├── admin/users/page.tsx
│   │   │   ├── admin/organizations/page.tsx
│   │   │   └── admin/settings/page.tsx
│   │   ├── api/
│   │   │   ├── auth/callback/route.ts
│   │   │   ├── webhook/midtrans/route.ts
│   │   │   ├── webhook/xendit/route.ts
│   │   │   ├── organizations/route.ts
│   │   │   └── subscriptions/route.ts
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── ui/           # Shadcn components
│   │   ├── layout/       # Sidebar, Header, Footer
│   │   ├── auth/         # Auth forms
│   │   ├── dashboard/    # Dashboard widgets
│   │   └── payment/      # Payment forms
│   ├── lib/
│   │   ├── supabase/     # Supabase client (server + client)
│   │   ├── prisma/       # Prisma client
│   │   ├── auth/         # Auth helpers
│   │   ├── payment/      # Midtrans & Xendit helpers
│   │   └── utils.ts      # Utility functions
│   ├── middleware.ts      # Auth middleware + tenant resolution
│   └── types/
│       └── index.ts      # TypeScript types
├── prisma/
│   └── schema.prisma     # Database schema
├── emails/               # Email templates
├── public/               # Static assets
├── .env.example          # Environment variables template
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
├── docker-compose.yml
├── Dockerfile
└── README.md
```

---

## 🗄️ DATABASE SCHEMA (Prisma)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// === MULTI-TENANT ===
model Organization {
  id        String   @id @default(cuid())
  name      String
  slug      String   @unique
  logo      String?
  plan      Plan     @default(FREE)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  members       Member[]
  subscriptions Subscription[]
  invitations   Invitation[]

  @@map("organizations")
}

enum Plan {
  FREE
  STARTER
  PRO
  ENTERPRISE
}

// === USERS & ROLES ===
model User {
  id            String   @id @default(cuid())
  email         String   @unique
  name          String?
  avatar        String?
  emailVerified DateTime?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  members       Member[]
  accounts      Account[]
  sessions      Session[]

  @@map("users")
}

model Member {
  id             String       @id @default(cuid())
  userId         String
  organizationId String
  role           Role         @default(MEMBER)
  createdAt      DateTime     @default(now())

  user         User         @relation(fields: [userId], references: [id])
  organization Organization @relation(fields: [organizationId], references: [id])

  @@unique([userId, organizationId])
  @@map("members")
}

enum Role {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

// === SUBSCRIPTIONS ===
model Subscription {
  id                   String             @id @default(cuid())
  organizationId       String
  status               SubscriptionStatus @default(INACTIVE)
  plan                 Plan               @default(FREE)
  currentPeriodStart   DateTime?
  currentPeriodEnd     DateTime?
  paymentProvider      String?            // "midtrans" | "xendit"
  paymentSubscriptionId String?
  createdAt            DateTime           @default(now())
  updatedAt            DateTime           @updatedAt

  organization Organization @relation(fields: [organizationId], references: [id])

  @@map("subscriptions")
}

enum SubscriptionStatus {
  ACTIVE
  PAST_DUE
  CANCELED
  INACTIVE
  TRIALING
}

// === INVITATIONS ===
model Invitation {
  id             String       @id @default(cuid())
  email          String
  organizationId String
  role           Role         @default(MEMBER)
  token          String       @unique
  expiresAt      DateTime
  acceptedAt     DateTime?
  createdAt      DateTime     @default(now())

  organization Organization @relation(fields: [organizationId], references: [id])

  @@map("invitations")
}

// === AUTH (Supabase adapter) ===
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?

  user User @relation(fields: [userId], references: [id])

  @@unique([provider, providerAccountId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id])

  @@map("sessions")
}
```

---

## 🔐 AUTH IMPLEMENTATION

### Supabase Client Setup:
```typescript
// lib/supabase/server.ts
import { createServerClient, type CookieOptions } from "@supabase/ssr";
import { cookies } from "next/headers";

export function createClient() {
  const cookieStore = cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) { return cookieStore.get(name)?.value; },
        set(name: string, value: string, options: CookieOptions) {
          try { cookieStore.set({ name, value, ...options }); }
          catch (error) { /* Handle error */ }
        },
        remove(name: string, options: CookieOptions) {
          try { cookieStore.set({ name, value: "", ...options }); }
          catch (error) { /* Handle error */ }
        },
      },
    }
  );
}

// lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### Middleware (Auth + Tenant):
```typescript
// middleware.ts
import { createServerClient, type CookieOptions } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({
    request: { headers: request.headers },
  });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) { return request.cookies.get(name)?.value; },
        set(name: string, value: string, options: CookieOptions) {
          request.cookies.set({ name, value, ...options });
          response = NextResponse.next({ request: { headers: request.headers } });
          response.cookies.set({ name, value, ...options });
        },
        remove(name: string, options: CookieOptions) {
          request.cookies.set({ name, value: "", ...options });
          response = NextResponse.next({ request: { headers: request.headers } });
          response.cookies.set({ name, value: "", ...options });
        },
      },
    }
  );

  const { data: { user } } = await supabase.auth.getUser();

  // Protect dashboard routes
  if (!user && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  // Protect admin routes
  if (user && request.nextUrl.pathname.startsWith("/admin")) {
    // Check if user has admin role
    const { data: member } = await supabase
      .from("members")
      .select("role")
      .eq("user_id", user.id)
      .single();

    if (!member || !["OWNER", "ADMIN"].includes(member.role)) {
      return NextResponse.redirect(new URL("/dashboard", request.url));
    }
  }

  // Redirect authenticated users away from auth pages
  if (user && (request.nextUrl.pathname === "/login" || request.nextUrl.pathname === "/register")) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }

  return response;
}

export const config = {
  matcher: ["/dashboard/:path*", "/admin/:path*", "/login", "/register", "/settings/:path*"],
};
```

---

## 💳 PAYMENT IMPLEMENTATION

### Midtrans Integration:
```typescript
// lib/payment/midtrans.ts
import Midtrans from "midtrans-client";

const snap = new Midtrans.Snap({
  isProduction: process.env.NODE_ENV === "production",
  serverKey: process.env.MIDTRANS_SERVER_KEY!,
  clientKey: process.env.MIDTRANS_CLIENT_KEY!,
});

const core = new Midtrans.CoreApi({
  isProduction: process.env.NODE_ENV === "production",
  serverKey: process.env.MIDTRANS_SERVER_KEY!,
  clientKey: process.env.MIDTRANS_CLIENT_KEY!,
});

interface CreateTransactionParams {
  orderId: string;
  amount: number;
  customerName: string;
  customerEmail: string;
  plan: string;
}

export async function createMidtransTransaction({
  orderId,
  amount,
  customerName,
  customerEmail,
  plan,
}: CreateTransactionParams) {
  const parameter = {
    transaction_details: {
      order_id: orderId,
      gross_amount: amount,
    },
    customer_details: {
      first_name: customerName,
      email: customerEmail,
    },
    item_details: [
      {
        id: plan,
        price: amount,
        quantity: 1,
        name: `SaaS Plan - ${plan}`,
      },
    ],
    credit_card: { secure: true },
    callbacks: {
      finish: `${process.env.NEXT_PUBLIC_APP_URL}/billing?status=success`,
      error: `${process.env.NEXT_PUBLIC_APP_URL}/billing?status=error`,
      pending: `${process.env.NEXT_PUBLIC_APP_URL}/billing?status=pending`,
    },
  };

  return snap.createTransaction(parameter);
}

export async function checkMidtransStatus(orderId: string) {
  return core.transaction.status(orderId);
}
```

### Xendit Integration:
```typescript
// lib/payment/xendit.ts
import { Xendit } from "xendit-node";

const xendit = new Xendit({
  secretKey: process.env.XENDIT_SECRET_KEY!,
});

const { Invoice } = xendit;

interface CreateInvoiceParams {
  externalId: string;
  amount: number;
  payerEmail: string;
  description: string;
  successRedirectUrl: string;
  failureRedirectUrl: string;
}

export async function createXenditInvoice({
  externalId,
  amount,
  payerEmail,
  description,
  successRedirectUrl,
  failureRedirectUrl,
}: CreateInvoiceParams) {
  return Invoice.createInvoice({
    externalId,
    amount,
    payerEmail,
    description,
    successRedirectUrl,
    failureRedirectUrl,
    currency: "IDR",
    invoiceDuration: 86400, // 24 hours
    customer: { email: payerEmail },
    customerNotificationPreference: {
      invoiceCreated: ["email"],
      invoiceReminder: ["email"],
      invoicePaid: ["email"],
      invoiceExpired: ["email"],
    },
  });
}

export async function checkXenditStatus(invoiceId: string) {
  return Invoice.getInvoiceById({ invoiceId });
}
```

### Webhook Handlers:
```typescript
// app/api/webhook/midtrans/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/server";
import crypto from "crypto";

export async function POST(request: NextRequest) {
  const body = await request.json();
  
  // Verify signature
  const hash = crypto
    .createHash("sha512")
    .update(body.order_id + body.status_code + body.gross_amount + process.env.MIDTRANS_SERVER_KEY)
    .digest("hex");

  if (hash !== body.signature_key) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const supabase = createClient();

  // Update subscription status based on transaction status
  if (body.transaction_status === "settlement" || body.transaction_status === "capture") {
    await supabase
      .from("subscriptions")
      .update({ status: "ACTIVE" })
      .eq("id", body.order_id);
  } else if (["deny", "cancel", "expire"].includes(body.transaction_status)) {
    await supabase
      .from("subscriptions")
      .update({ status: "INACTIVE" })
      .eq("id", body.order_id);
  }

  return NextResponse.json({ received: true });
}

// app/api/webhook/xendit/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextRequest, NextResponse } from "next/request";

export async function POST(request: NextRequest) {
  const body = await request.json();
  const callbackToken = request.headers.get("x-callback-token");

  if (callbackToken !== process.env.XENDIT_CALLBACK_TOKEN) {
    return NextResponse.json({ error: "Invalid token" }, { status: 401 });
  }

  const supabase = createClient();

  if (body.status === "PAID") {
    await supabase
      .from("subscriptions")
      .update({ status: "ACTIVE" })
      .eq("id", body.external_id);
  } else if (body.status === "EXPIRED") {
    await supabase
      .from("subscriptions")
      .update({ status: "INACTIVE" })
      .eq("id", body.external_id);
  }

  return NextResponse.json({ received: true });
}
```

---

## 🎨 UI COMPONENTS (Key Examples)

### Auth Layout:
```typescript
// app/(auth)/layout.tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export default async function AuthLayout({ children }: { children: React.ReactNode }) {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (user) redirect("/dashboard");

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="max-w-md w-full space-y-8 p-8">
        {children}
      </div>
    </div>
  );
}
```

### Dashboard with RBAC:
```typescript
// app/(dashboard)/dashboard/page.tsx
import { createClient } from "@/lib/supabase/server";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default async function DashboardPage() {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: member } = await supabase
    .from("members")
    .select("role, organization:organizations(*)")
    .eq("user_id", user!.id)
    .single();

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold">Dashboard</h1>
      <p>Welcome back, {user?.email}</p>
      <p>Organization: {member?.organization?.name}</p>
      <p>Role: {member?.role}</p>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
        <Card>
          <CardHeader><CardTitle>Plan</CardTitle></CardHeader>
          <CardContent><p>{member?.organization?.plan || "FREE"}</p></CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle>Members</CardTitle></CardHeader>
          <CardContent><p>-</p></CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle>Subscription</CardTitle></CardHeader>
          <CardContent><p>-</p></CardContent>
        </Card>
      </div>
    </div>
  );
}
```

### Organization Switcher:
```typescript
// components/layout/org-switcher.tsx
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";

export function OrganizationSwitcher({ organizations }: { organizations: any[] }) {
  const [activeOrg, setActiveOrg] = useState(organizations[0]);
  const router = useRouter();

  return (
    <select
      value={activeOrg?.id}
      onChange={(e) => {
        const org = organizations.find(o => o.id === e.target.value);
        setActiveOrg(org);
        router.refresh();
      }}
      className="border rounded px-3 py-1"
    >
      {organizations.map((org) => (
        <option key={org.id} value={org.id}>{org.name}</option>
      ))}
    </select>
  );
}
```

---

## 🚀 DEPLOYMENT

### Vercel:
```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Set environment variables
vercel env add NEXT_PUBLIC_SUPABASE_URL
vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY
vercel env add DATABASE_URL
vercel env add MIDTRANS_SERVER_KEY
vercel env add MIDTRANS_CLIENT_KEY
vercel env add XENDIT_SECRET_KEY
vercel env add XENDIT_CALLBACK_TOKEN
vercel env add NEXT_PUBLIC_APP_URL
```

### Docker:
```dockerfile
# Dockerfile
FROM node:20-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN yarn build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

---

## 📧 EMAIL TEMPLATES

### Welcome Email:
```html
<!-- emails/welcome.html -->
<!DOCTYPE html>
<html>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
  <div style="background: #0D1117; color: white; padding: 30px; text-align: center;">
    <h1>Welcome to [App Name]! 🚀</h1>
  </div>
  <div style="padding: 30px; background: #f8f9fa;">
    <p>Hi {{name}},</p>
    <p>Welcome aboard! Your account is ready.</p>
    <a href="{{dashboardUrl}}" style="display: inline-block; background: #3B82F6; color: white; padding: 12px 24px; text-decoration: none; border-radius: 6px;">
      Go to Dashboard
    </a>
  </div>
</body>
</html>
```

### Invoice Email:
```html
<!-- emails/invoice.html -->
<div style="font-family: Arial; max-width: 600px; margin: 0 auto;">
  <h2>Invoice #{{invoiceNumber}}</h2>
  <p>Dear {{customerName}},</p>
  <p>Your payment of <strong>Rp {{amount}}</strong> for {{plan}} plan has been received.</p>
  <table style="width: 100%; border-collapse: collapse;">
    <tr><td style="padding: 10px; border: 1px solid #ddd;">Plan</td><td>{{plan}}</td></tr>
    <tr><td style="padding: 10px; border: 1px solid #ddd;">Period</td><td>{{period}}</td></tr>
    <tr><td style="padding: 10px; border: 1px solid #ddd;">Amount</td><td>Rp {{amount}}</td></tr>
    <tr><td style="padding: 10px; border: 1px solid #ddd;">Status</td><td>LUNAS</td></tr>
  </table>
</div>
```

---

## 📋 ENVIRONMENT VARIABLES

```env
# .env.example

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
NODE_ENV=development

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
DATABASE_URL=postgresql://postgres:password@db.xxx.supabase.co:5432/postgres

# Midtrans
MIDTRANS_CLIENT_KEY=your-client-key
MIDTRANS_SERVER_KEY=your-server-key

# Xendit
XENDIT_SECRET_KEY=your-secret-key
XENDIT_CALLBACK_TOKEN=your-callback-token

# Email (Resend)
RESEND_API_KEY=your-resend-api-key
```

---

## 🎯 CUSTOMIZATION GUIDE

### Changing Brand/Colors:
1. Edit `tailwind.config.ts` — change primary color
2. Edit `app/globals.css` — change CSS variables
3. Replace `public/logo.svg` — your logo
4. Edit email templates — update branding

### Adding Features:
1. Create new page in `app/(dashboard)/`
2. Add Prisma model in `prisma/schema.prisma`
3. Run `npx prisma migrate dev`
4. Create API route in `app/api/`
5. Build UI components in `components/`

### Adding Subscription Tiers:
1. Add plan names in `enum Plan` (Prisma schema)
2. Add pricing in billing page
3. Configure Midtrans/Xendit for each tier
4. Add feature gating based on plan

---

## 📞 SUPPORT

- **Discord/Telegram:** [Link member group]
- **Email:** support@[yourdomain].com
- **Documentation:** Full docs included in README.md
- **Updates:** Lifetime updates via GitHub releases

---

**© 2024 SaaS Boilerplate Premium. All rights reserved.**
**Single Developer License — Unlimited projects.**
