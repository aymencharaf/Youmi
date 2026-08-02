# منصة SaaS متعددة المستأجرين لإنشاء المتاجر الإلكترونية

مرجع التصميم الكامل: [`docs/TDD-Multi-Tenant-Ecommerce-SaaS.md`](./docs/TDD-Multi-Tenant-Ecommerce-SaaS.md)

**حالة المشروع:** المرحلة 0 (Foundation) مكتملة. راجع `docs/PHASE-0-REPORT.md` لتفاصيل ما تم إنجازه.

---

## البنية (Monorepo)

```
apps/
  api/                → NestJS: الـ Backend الرئيسي (REST API)
  workers/             → NestJS: معالجات BullMQ (بريد، PDF، AI...)
  storefront/           → Next.js: واجهة المتجر العامة
  store-dashboard/       → Next.js: لوحة تحكم صاحب المتجر
  admin-dashboard/        → Next.js: لوحة تحكم Super Admin
packages/
  database/             → Prisma Schema + RLS + عميل قاعدة البيانات
  types/                  → أنواع وDTOs وZod Schemas مشتركة
  utils/                   → دوال مساعدة (money, slug, dates, tenant, logger...)
  ui/                       → مكتبة مكونات shadcn/ui مشتركة (RTL/LTR)
  sdk/                       → عميل API نمطي (Type-safe) للتطبيقات الثلاثة
  config/                     → إعدادات TypeScript/ESLint مشتركة
infra/docker/                → Dockerfiles
docs/                         → وثائق التصميم
```

---

## المتطلبات الأساسية

| الأداة | الإصدار الأدنى |
|---|---|
| Node.js | 20.x |
| pnpm | 9.x (`corepack enable` يفعّله تلقائيًا) |
| Docker + Docker Compose | أي إصدار حديث |

---

## التثبيت (Installation)

```bash
# 1) استنساخ المشروع والدخول لمجلده
git clone <repo-url> saas-platform && cd saas-platform

# 2) تفعيل pnpm عبر Corepack (يأتي مدمجًا مع Node.js 20+)
corepack enable

# 3) تثبيت كل التبعيات عبر الـ Monorepo (Turborepo + pnpm workspaces)
pnpm install

# 4) نسخ ملف متغيرات البيئة وتعديله عند الحاجة
cp .env.example .env
```

---

## التطوير المحلي (Development)

### الخطوة 1: تشغيل البنية التحتية

```bash
docker compose up -d postgres redis minio minio-init mailhog
```

يشغّل هذا: PostgreSQL (`5432`)، Redis (`6379`)، MinIO كـ S3 محلي (`9000` API / `9001` Console)، وMailhog لالتقاط رسائل البريد أثناء التطوير (`1025` SMTP / `8025` واجهة الويب).

### الخطوة 2: توليد Prisma Client وتطبيق الـ Migrations وRLS

```bash
pnpm db:generate        # توليد Prisma Client
pnpm db:migrate:dev      # ينشئ الجداول + يطبّق RLS تلقائيًا (انظر packages/database/package.json)
pnpm db:seed              # بيانات أولية: خطط الاشتراك + قالب افتراضي + متجر تجريبي
```

> **ملاحظة معمارية مهمة:** كل من `db:migrate:dev`، `db:migrate:deploy`، و`db:migrate:reset`
> تُشغّل تلقائيًا سكربتات `packages/database/prisma/rls/*.sql` بعد كل Migration
> (عبر `pnpm run rls:apply`)، بحيث لا تُنسى سياسات Row-Level Security أبدًا بعد
> أي تغيير في الـ Schema.

### الخطوة 3: تشغيل كل التطبيقات

```bash
pnpm dev
```

يشغّل Turborepo كل التطبيقات بالتوازي عبر Watch Mode:

| التطبيق | الرابط |
|---|---|
| Storefront | http://localhost:3000 |
| Store Dashboard | http://localhost:3001 |
| Admin Dashboard | http://localhost:3002 |
| API | http://localhost:4000/api/v1 |
| API Docs (Swagger) | http://localhost:4000/api/docs |
| Health Check | http://localhost:4000/health |
| MinIO Console | http://localhost:9001 |
| Mailhog UI | http://localhost:8025 |

### تشغيل تطبيق واحد فقط

```bash
pnpm --filter @platform/api dev
pnpm --filter @platform/storefront dev
```

### أوامر إضافية مفيدة

```bash
pnpm db:studio     # واجهة Prisma Studio لتصفح البيانات
pnpm lint          # ESLint عبر كل الحزم/التطبيقات
pnpm typecheck     # TypeScript Strict عبر كل الحزم/التطبيقات
pnpm test          # اختبارات وحدوية
pnpm test:e2e      # اختبارات E2E (تتطلب DB/Redis يعملان)
pnpm build         # بناء كل شيء (يستفيد من Turborepo Cache)
```

---

## التشغيل الكامل عبر Docker (بديل لـ `pnpm dev`)

```bash
docker compose up --build
```

يبني وشغّل كل الخدمات (البنية التحتية + api + workers + التطبيقات الثلاثة)
في حاويات معزولة، بنفس الإعدادات التي ستُستخدم في الإنتاج تقريبًا. **ملاحظة:**
يجب تطبيق الـ Migrations يدويًا من الجهاز المضيف أولًا (`pnpm db:migrate:deploy`
مع `DATABASE_URL` يشير إلى `localhost:5432`) قبل أو بعد إقلاع الحاويات، لأن
حاوية `api` لا تُشغّل Migrations تلقائيًا عند الإقلاع (قرار متعمد لتفادي
تعارض عدة نسخ من الـ API تحاول تطبيق Migration في نفس اللحظة عند التوسع
الأفقي في الإنتاج - انظر TDD "CI/CD Strategy").

---

## متغيرات البيئة (Environment Variables)

كل المتغيرات موثّقة في [`./.env.example`](./.env.example) مع قيم افتراضية صالحة
للتطوير المحلي. الأهم:

| المتغير | الوصف |
|---|---|
| `DATABASE_URL` | اتصال PostgreSQL (بصلاحيات كاملة - Migrations) |
| `DATABASE_URL_APP` | اتصال بصلاحيات `app_tenant_role` المقيّدة بـ RLS (إنتاج) |
| `REDIS_URL` | اتصال Redis (Cache، BullMQ، Rate Limiting) |
| `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` | أسرار توقيع JWT (32 حرفًا كحد أدنى - يُتحقق منها إلزاميًا عند الإقلاع) |
| `S3_*` | اتصال التخزين المتوافق مع S3 (MinIO محليًا) |
| `FIELD_ENCRYPTION_KEY` | مفتاح تشفير الحقول الحساسة (بيانات بوابات الدفع) |
| `PLATFORM_ROOT_DOMAIN` | الدومين الجذري لحل الـ Subdomain (`platform.local` محليًا) |

⚠️ **لا تستخدم القيم الافتراضية في `.env.example` في بيئة الإنتاج أبدًا** -
خصوصًا `JWT_*_SECRET` و`FIELD_ENCRYPTION_KEY` و`POSTGRES_PASSWORD`.

---

## الإنتاج (Production)

1. **البنية التحتية المُدارة:** استبدل `postgres`/`redis`/`minio` المحليين
   بخدمات مُدارة (مثل Render PostgreSQL، Upstash Redis، AWS S3/Cloudflare R2)
   حسب `docs/TDD-Multi-Tenant-Ecommerce-SaaS.md` § Deployment Architecture.
2. **بناء الصور:**
   ```bash
   docker build -f infra/docker/api.Dockerfile -t platform/api .
   docker build -f infra/docker/workers.Dockerfile -t platform/workers .
   docker build -f infra/docker/nextjs.Dockerfile \
     --build-arg APP_NAME=storefront --build-arg APP_PORT=3000 \
     -t platform/storefront .
   # كرّر لـ store-dashboard (3001) وadmin-dashboard (3002)
   ```
3. **Migrations في الإنتاج:** تُشغَّل مرة واحدة فقط (خطوة منفصلة في CI/CD،
   وليس عند إقلاع كل نسخة من الـ API):
   ```bash
   pnpm db:migrate:deploy
   ```
4. **الواجهات الأمامية:** يُنصح بنشر تطبيقات Next.js الثلاثة على Vercel
   (تكامل Edge Middleware وISR أفضل)، والـ `api`/`workers` على Render أو
   أي منصة تدعم Docker، كما هو موضّح في TDD § Deployment Architecture.

---

## استكشاف الأخطاء وإصلاحها (Troubleshooting)

**`pnpm install` يفشل بخطأ `ERR_PNPM_NO_MATCHING_VERSION` أو مشابه**
تأكد من Node.js 20+ وأن `corepack enable` نُفِّذ قبل `pnpm install`.

**`prisma generate` أو `migrate` يفشل باتصال مرفوض (`ECONNREFUSED`)**
تأكد أن حاوية `postgres` تعمل وصحية: `docker compose ps` يجب أن يُظهر
`postgres` بحالة `healthy`. جرّب `docker compose logs postgres`.

**التطبيقات تُقلع لكن `/health` يُرجع 503**
هذا متوقَّع إذا لم تُشغَّل `docker compose up -d postgres redis` بعد، أو
إذا كانت `DATABASE_URL`/`REDIS_URL` في `.env` لا تطابق المنافذ الفعلية.

**خطأ `relation "xxx" does not exist` عند تشغيل أي استعلام**
لم تُطبَّق الـ Migrations بعد: شغّل `pnpm db:migrate:dev` (تطوير) أو
`pnpm db:migrate:deploy` (إنتاج/CI).

**بيانات متجر تظهر فارغة رغم وجودها في الجدول عبر Prisma Studio**
هذا **سلوك متوقَّع ومقصود** لو استُعلم عبر `prisma` مباشرة بدل
`withTenantContext()` - انظر تعليق `packages/database/src/client.ts`: RLS
يرفض إرجاع أي صف بدون `app.current_store_id` مضبوط في نفس الـ transaction
(Fail-Closed، وليس تسريب بيانات). تأكد أن كل استعلام على جدول Tenant-scoped
يمر عبر `withTenantContext(storeId, ...)`.

**تعارض منافذ (`port is already allocated`) عند `docker compose up`**
منفذ من `5432/6379/9000/9001/1025/8025/3000/3001/3002/4000` مستخدَم مسبقًا
على جهازك. عدّل الـ mapping في `docker-compose.yml` (الجزء الأيسر من `:`) أو
أوقف الخدمة المتعارضة.

**Next.js apps: خطأ `Module not found: @platform/ui`**
تأكد من تشغيل `pnpm install` من **جذر الـ Monorepo** (وليس من داخل مجلد
التطبيق)، لأن pnpm workspaces يربط الحزم الداخلية عبر symlinks تُبنى فقط
عند التثبيت من الجذر.

---

## الأمان - تذكير سريع

راجع `docs/TDD-Multi-Tenant-Ecommerce-SaaS.md` § Security Design للتفاصيل
الكاملة. النقاط الحرجة للمطورين الجدد على المشروع:

- **لا تستعلم أبدًا** من جدول Tenant-scoped (Product, Order, Customer...)
  بدون المرور عبر `withTenantContext()` من `@platform/database`.
- **لا تعتمد أبدًا** على `storeId` القادم من الـ URL للتفويض - استخدم فقط
  القيمة المستخرجة من JWT عبر `@CurrentStore()`.
- أي جدول جديد يحمل بيانات متجر يجب أن يحمل عمود `storeId` + تُضاف له
  سياسة RLS في `packages/database/prisma/rls/02_enable_rls.sql`.
#   y o u m i  
 