# VidFa — پلتفرم اشتراک‌گذاری ویدئوی فارسی

نسخه شروع (MVP) با رابط استاتیک GitHub Pages و API روی Cloudflare Workers، دیتابیس D1 و ذخیره‌سازی R2.

## ساختار
- `site/` رابط فارسی و واکنش‌گرا
- `worker/` API و کد Worker
- `worker/migrations/0001_init.sql` ساخت جدول‌های D1
- `.github/workflows/pages.yml` انتشار خودکار GitHub Pages

## پیش‌نیازها
حساب GitHub و Cloudflare. برای اجرای کامل، در Cloudflare یک D1 database با نام `vidfa-db` و یک R2 bucket با نام `vidfa-videos` بساز. فعال‌سازی R2 ممکن است نیازمند ثبت روش پرداخت باشد؛ سهمیه رایگان تضمین‌کننده هزینه صفر در همه شرایط نیست.

## ۱) انتشار رابط با GitHub Pages
1. یک مخزن عمومی GitHub با نام `vidfa` بساز.
2. محتویات این پوشه را در شاخه `main` مخزن قرار بده.
3. در Settings → Pages، گزینه Source را روی **GitHub Actions** بگذار.
4. پس از اجرای workflow، آدرس `https://USERNAME.github.io/vidfa/` ساخته می‌شود.

## ۲) ساخت D1 و R2
در Cloudflare Dashboard:
1. Workers & Pages → D1 → Create database → نام `vidfa-db`.
2. R2 Object Storage → Create bucket → نام `vidfa-videos`.
3. شناسه D1 را در `worker/wrangler.toml` جایگزین `REPLACE_WITH_D1_DATABASE_ID` کن.

یا با Wrangler از پوشه `worker`:
```bash
npm install -g wrangler
wrangler login
wrangler d1 create vidfa-db
wrangler r2 bucket create vidfa-videos
```
شناسه دیتابیس برگشتی را در `wrangler.toml` قرار بده، سپس:
```bash
wrangler d1 migrations apply vidfa-db --remote
wrangler secret put JWT_SECRET
```
برای JWT_SECRET یک رشته تصادفی طولانی انتخاب کن. آن را در کد سایت یا GitHub قرار نده.

## ۳) پیکربندی Worker
در `worker/wrangler.toml` مقدار `APP_ORIGIN` را برابر آدرس دقیق GitHub Pages قرار بده؛ مثال:
`https://USERNAME.github.io`

نکته: اگر سایت در مسیر `/vidfa/` است، مقدار Origin همچنان فقط scheme + host است و مسیر ندارد.

از پوشه `worker` اجرا کن:
```bash
wrangler deploy
```
آدرس Worker شبیه `https://vidfa-api.USERNAME.workers.dev` خواهد بود.

## ۴) اتصال سایت به API
در `site/app.js` خط اول مقدار `API` را با آدرس Worker جایگزین کن:
```js
const API = 'https://vidfa-api.YOUR-SUBDOMAIN.workers.dev';
```
سپس تغییر را به GitHub push کن تا Pages دوباره منتشر شود.

## محدودیت‌ها و نکات امنیتی
- نسخه اولیه آپلود را به ۵۰ مگابایت محدود می‌کند. پلن رایگان Cloudflare برای درخواست ورودی حداکثر ۱۰۰ مگابایت ذکر می‌کند؛ سقف واقعی و شرایط حساب را در داشبورد بررسی کن.
- فایل ویدئو از Worker به R2 منتقل و پخش می‌شود. این نسخه برای فایل‌های کوچک و آزمایش مناسب است؛ برای ویدئوهای طولانی/حجم بالا، آپلود مستقیم چندبخشی به R2 و پردازش HLS لازم است.
- پخش ویدئو در این نسخه از R2 به شکل فایل MP4/WebM است؛ تبدیل خودکار کیفیت، زیرنویس، استریم تطبیقی و CDN اختصاصی هنوز اضافه نشده‌اند.
- قبل از انتشار عمومی، محدودسازی نرخ درخواست (rate limiting)، تأیید ایمیل، بازیابی رمز، گزارش تخلف، ابزار حذف محتوا، سیاست حریم خصوصی و شرایط استفاده اضافه شود.
- برای کاربران واقعی، بررسی حقوق انتشار ویدئوها و جلوگیری از انتشار محتوای غیرمجاز ضروری است.
- GitHub Pages فقط رابط استاتیک را میزبانی می‌کند؛ منطق ورود و آپلود در Worker اجرا می‌شود.

## APIهای پیاده‌سازی‌شده
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/videos`
- `POST /api/videos` (multipart/form-data؛ فیلد `video` الزامی)
- `GET /media/:id`
- `POST /api/videos/:id/view`
- `GET|POST /api/videos/:id/comments`
- `POST /api/videos/:id/like`
- `POST /api/users/:id/follow`

## وضعیت نسخه
این یک MVP قابل استقرار است، نه جایگزین آماده‌به‌کار آپارات در مقیاس بزرگ. ابتدا با چند فایل آزمایشی و حساب تست راه‌اندازی و بررسی شود.
