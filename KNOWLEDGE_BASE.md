# Knowledge Base — Technology Intelligence Studio

> نسخه: 1.0  
> تاریخ بازبینی: 2026-11-23  
> هدف: دانش پایه عملیاتی برای مشاور فنی، با تمرکز بر AvalAI، تولید ویدئو با Veo 3.1 و معماری قابل اجرا بدون VPS.  
> قاعده: اطلاعات قیمت/مدل/endpoint متغیرند. قبل از تصمیم تولیدی، لینک رسمی و وضعیت حساب را دوباره بررسی کن.

## 1. پروفایل فنی و محدودیت‌های واقعی کاربر

### امکانات شناخته‌شده
- هاست اشتراکی با PHP 8.2
- MariaDB
- Redis (در صورت فعال بودن و دسترسی از هاست)
- Cloudflare Workers
- GitHub
- Telegram Bot API
- Python hosting به‌عنوان امکان بالقوه؛ جزئیات پلن و قابلیت‌ها باید تأیید شود.
- VPS در اختیار کاربر نیست.

### اصول معماری
1. راهکار اصلی نباید به VPS، Docker روی VPS یا worker دائمی متکی باشد.
2. هاست اشتراکی برای API سبک، مدیریت کاربران، CRUD، webhook سبک و ثبت metadata مناسب‌تر است؛ برای FFmpeg، رندر سنگین، OCR/Whisper و پردازش طولانی مناسب فرض نشود.
3. از Cloudflare Workers برای webhook، اعتبارسنجی، API proxy، هماهنگی مراحل کوتاه و ارتباط با سرویس‌ها استفاده شود.
4. Cloudflare Queues برای اجرای غیرهم‌زمان مناسب است، اگر در حساب فعال باشد. اگر فعال نیست، معماری جایگزین با Cron Trigger و جدول jobs یا Cron هاست اشتراکی بررسی شود.
5. برای ذخیره فایل‌ها، R2 فقط در صورت فعال‌سازی و بررسی سهمیه/هزینه استفاده شود.
6. MariaDB موجود می‌تواند منبع حقیقت باقی بماند؛ Worker نباید مستقیم به اتصال TCP دیتابیس اشتراکی وابسته شود. یک API PHP محدود و احرازشده روی HTTPS برای CRUD لازم است. گزینه دیگر، انتقال صرفاً metadata به Cloudflare D1 است؛ مهاجرت DB در MVP ضروری نیست.
7. Worker محیط اجرای FFmpeg نیست. اگر Python hosting دارای FFmpeg و امکان اجرای job زمان‌بر تأیید نشد، MVP باید کلیپ‌های جداگانه را تحویل دهد، نه اینکه ادعا کند MP4 نهایی چندصحنه‌ای ساخته است.
8. برای هر Job شناسه یکتا، وضعیت، مرحله، تعداد تلاش، خطا، شناسه درخواست AvalAI، هزینه واقعی، مالک و زمان ایجاد/تکمیل ذخیره شود.
9. عملیات غیرقابل تکرار مثل ایجاد ویدئو باید idempotency داشته باشد؛ retry کور می‌تواند هزینه دوباره ایجاد کند.
10. هیچ کلید API در GitHub، کد Frontend یا فایل عمومی قرار نگیرد.

## 2. AvalAI — مبانی مشترک API

### Base URL و احراز هویت
- API پایه: `https://api.avalai.ir/v1`
- User API: `https://api.avalai.ir/user/v1`
- هدر: `Authorization: Bearer $AVALAI_API_KEY`
- کلید باید در Secret/Environment Variable سمت Worker یا Backend نگهداری شود.

### انتخاب سطح API
- برای پروژه‌های جدید متنی/استدلالی/چندوجهی و ابزارمحور، ابتدا پشتیبانی مدل انتخابی از `/v1/responses` را بررسی کن.
- `/v1/chat/completions` همچنان برای یکپارچگی‌های موجود و مدل‌هایی که chat route را ارائه می‌کنند مناسب است.
- schema، پارامترها و خروجی Responses و Chat Completions یکسان فرض نشوند.
- در هر integration، endpoint، model ID، پارامترهای پذیرفته‌شده و نمونه پاسخ را با مستند همان route تطبیق بده.
- برای Responses، پارامتر خروجی و ساختار `output` با Chat Completions متفاوت است؛ به‌صورت کور `max_tokens` یا `choices[0]` را منتقل نکن.

### فهرست مدل‌ها و قیمت
- کاتالوگ و مدل‌های فعال را از AvalAI Model Explorer / Models API دریافت کن.
- قیمت را از کاتالوگ جاری بخوان؛ قیمت داخل کد را حقیقت دائمی فرض نکن.
- بعضی قیمت‌ها بر حسب یک میلیون توکن و برخی مدل‌ها بر اساس ثانیه/تصویر/درخواست هستند.
- `null`، نبودن قیمت یا نبودن قابلیت در کاتالوگ را به معنی رایگان بودن یا عدم پشتیبانی قطعی تفسیر نکن.
- مدل‌های deprecated/preview را برای پروژه جدید انتخاب نکن مگر اینکه دلیل روشن و وضعیت رسمی داشته باشند.
- قیمت ورودی، cached input، cache creation و خروجی را جداگانه ثبت کن، اگر مدل این موارد را دارد.

### محدودیت فعلی مشاور داخل این رابط
- درخواست مشاور در نسخه فعلی از `/v1/chat/completions` استفاده می‌کند تا با مدل‌های Chat-compatible سازگار بماند.
- بنابراین مدلی که فقط Responses API را ارائه کند نباید در فهرست انتخاب مشاور ظاهر شود؛ فهرست مدل‌های این بخش به مدل‌های chat/completion یا مدل‌های بدون metadata محدود می‌شود.
- پشتیبانی از `/v1/responses` در دانش فنی توضیح داده شده، اما برای استفاده واقعی باید مسیر درخواست و parser پاسخ به‌صورت جداگانه پیاده‌سازی و تست شود؛ صرفاً تغییر نام endpoint کافی نیست.

## 3. AvalAI User API — موجودی و هزینه واقعی

Base URL: `https://api.avalai.ir/user/v1`

### موجودی
`GET /credit`

فیلدهای مهم در پاسخ مستند:
- `remaining_irt`
- `remaining_unit`
- `total_unit`
- `exchange_rate`
- `account_tier`
- `credit_sources`

Tier روی محدودیت نرخ برخی endpointهای User API اثر دارد. تعداد درخواست‌های مجاز را از مستند جاری بخوان و بی‌دلیل polling پرتکرار انجام نده.

### تراکنش‌ها
`GET /transactions`
- تاریخچه تراکنش‌ها با فیلتر زمانی، مدل، provider، status و pagination.
- برای گزارش مالی خود، داده‌های مهم را در DB پروژه نگه دار؛ به retention بیرونی تکیه نکن.

### تطبیق دقیق هزینه
`POST /transactions/lookup`
- شناسه درخواست/تراکنش را ارسال کن.
- پاسخ می‌تواند جزئیات هزینه واقعی را برگرداند.
- برای هزینه هر عملیات، شناسه request را در لحظه ایجاد ذخیره کن و بعداً lookup کن.
- در مستند User API، دسترسی به رکوردها حداقل ۹۰ روز تضمین شده؛ برای تحلیل بلندمدت هزینه‌ها، خودت داده را ذخیره کن.

### توصیه مالی
- هزینه تخمینی بر اساس قیمت کاتالوگ فقط تخمین است.
- هزینه دقیق نهایی را از User API بگیر.
- در صورت نبود transaction در لحظه اول، با فاصله و تعداد محدود دوباره lookup کن؛ درخواست تولید را تکرار نکن.
- `safety_identifier` را برای اتصال درخواست AvalAI به user/job/project خودت یکتا و غیرحساس انتخاب کن.

## 4. AvalAI Video API — Veo 3.1

### مدل‌های پایدار
| Model ID | قیمت مستند فعلی | کاربرد |
|---|---:|---|
| `veo-3.1-fast-generate-001` | $0.15 / ثانیه ویدئوی تولیدشده | نمونه‌سازی، تولید اقتصادی، شبکه اجتماعی |
| `veo-3.1-generate-001` | $0.40 / ثانیه ویدئوی تولیدشده | کیفیت بالاتر و خروجی حرفه‌ای |

مدل‌های Preview قدیمی را برای پروژه جدید پیش‌فرض نکن؛ از شناسه‌های پایدار بالا استفاده کن.

### محدودیت‌های کلیدی
- هر درخواست ۴، ۶ یا ۸ ثانیه است؛ یک درخواست ۶۰ ثانیه‌ای وجود ندارد.
- حداکثر مدت هر تولید طبق مستند جاری ۸ ثانیه است.
- نسبت‌های 16:9 و 9:16 پشتیبانی می‌شوند.
- 1080p طبق مستند جاری فقط برای 16:9 اعلام شده؛ 9:16 را 720p فرض کن مگر مستند جدیدتر خلاف آن را بگوید.
- Veo 3.1 خروجی MP4 با صدای native تولید می‌کند؛ گفتار، افکت و صدای محیط را می‌توان در prompt توصیف کرد.
- Image-to-video، reference images و video extension نیز مستند شده‌اند؛ جزئیات فرمت ورودی را با راهنمای جاری چک کن.
- برای حفظ شخصیت/استایل بین نماها، از reference images در صورت پشتیبانی route استفاده کن؛ تداوم کامل شخصیت تضمین‌شده نیست.

### Endpointها
- ایجاد: `POST https://api.avalai.ir/v1/videos`
- مشاهده وضعیت: `GET https://api.avalai.ir/v1/videos/{video_id}`
- دریافت محتوا: `GET https://api.avalai.ir/v1/videos/{video_id}/content`
- فهرست: `GET https://api.avalai.ir/v1/videos/`

نمونه درخواست متنی (الگوی رسمی، قبل از اجرا با schema جاری تطبیق بده):
```bash
curl -X POST "https://api.avalai.ir/v1/videos" \
  -H "Authorization: Bearer $AVALAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "veo-3.1-fast-generate-001",
    "prompt": "A cinematic vertical shot ...",
    "size": "720x1280",
    "seconds": "8",
    "safety_identifier": "project_job_scene_001"
  }'
```

### پردازش ناهم‌زمان
1. درخواست create را یک بار ارسال کن.
2. `video.id` و `request_id` را در DB ثبت کن.
3. پاسخ فوری به کاربر بده: «در حال تولید».
4. در اجرای بعدی/cron، `GET /videos/{id}` را صدا بزن.
5. اگر `queued` یا `processing` بود، زمان بررسی بعدی را ثبت کن و کار را تمام کن؛ sleep loop اجرا نکن.
6. اگر `completed` بود، محتوا را دانلود و به storage منتقل کن.
7. اگر `failed` بود، خطا را ثبت کن و فقط با سیاست retry محدود و بررسی هزینه اقدام کن.
8. اگر ارتباط هنگام create قطع شد، اول `GET /videos/` و وضعیت درخواست اخیر را بررسی کن. تا زمانی که وضعیت نامشخص است، create جدید نفرست؛ ممکن است درخواست اول هزینه‌ساز شده باشد.

### هزینه ویدئوی ۶۰ ثانیه‌ای
Veo یک کلیپ ۶۰ ثانیه‌ای را در یک درخواست تولید نمی‌کند. برای تقریباً ۶۰ ثانیه خروجی:
- ۸ کلیپ × ۸ ثانیه = ۶۴ ثانیه خام؛ بعد از مونتاژ ۴ ثانیه trim شود.
- Fast: `64 × $0.15 = $9.60`
- Standard: `64 × $0.40 = $25.60`
- اگر دقیقاً ۶۰ ثانیه billable شود، تخمین نظری Fast = $9 و Standard = $24.
- این ارقام فقط هزینه تولید ویدئو هستند؛ retry، رندر/مونتاژ، storage، انتقال، تصویر مرجع، TTS مستقل و سایر هزینه‌ها جدا هستند.
- در عمل هزینه دقیق از تراکنش‌های AvalAI خوانده شود.

### صدا و TTS
- Veo 3.1 صدای native دارد؛ در MVP ابتدا همین مسیر را تست کن تا هزینه و پیچیدگی TTS حذف شود.
- برای نریشن فارسی دقیق، صدای برند ثابت، تلفظ واژه‌های حساس یا کنترل دقیق متن گفتار، یک TTS مستقل ممکن است لازم شود.
- اگر TTS مستقل استفاده شد، باید صدای تولیدشده با طول کلیپ و تدوین همگام شود؛ آن را به‌صورت پیش‌فرض برای هر صحنه اضافه نکن.

## 5. طراحی ویدئوی چندصحنه‌ای

### خط تولید پیشنهادی
1. دریافت brief کاربر: موضوع، مخاطب، پلتفرم، نسبت تصویر، زبان، سبک، مدت، لحن و محدودیت‌ها.
2. LLM یک storyboard ساختاریافته تولید کند: scene_id، هدف صحنه، prompt انگلیسی/چندزبانه مناسب مدل، duration (4/6/8)، camera، subject/action، ambience/dialogue، continuity notes.
3. اعتبارسنجی JSON و محدودیت مدت/رزولوشن انجام شود.
4. یک صحنه نمونه با Veo Fast تولید شود.
5. کاربر نمونه را تأیید کند.
6. سایر صحنه‌ها با concurrency محدود تولید شوند.
7. هر `video_id` و `request_id` جداگانه ثبت شود.
8. خروجی‌ها به ترتیب صحنه ذخیره شوند.
9. مونتاژ فقط در محیطی انجام شود که FFmpeg واقعاً در دسترس است.
10. خروجی نهایی و breakdown هزینه برای کاربر ثبت شود.

### چرا پنج صحنه ۱۲ ثانیه‌ای غلط است؟
محدودیت هر درخواست Veo 3.1 برابر ۴/۶/۸ ثانیه است. پنج صحنه ۱۲ ثانیه‌ای را نمی‌توان مستقیماً به‌عنوان پنج درخواست Veo فرستاد. یا هر صحنه باید به چند کلیپ کوتاه‌تر شکسته شود، یا تعداد sceneها و مدت‌ها مطابق سقف مدل طراحی شوند.

### راهبرد کیفیت
- هر prompt شامل subject، action، setting، camera movement، shot/composition، lens/focus، lighting/color، style و audio cues باشد.
- promptهای کلی مثل “a beautiful video” کافی نیستند.
- یک موضوع/کاراکتر/لباس/لوکیشن را در همه sceneها با continuity bible ثابت توصیف کن.
- خروجی اولین scene را قبل از تولید دسته‌ای ارزیابی کن.
- تعداد retry را محدود کن؛ prompt را قبل از تکرار اصلاح کن.
- خروجی را از نظر aspect ratio، طول، صدا، artefacts و هماهنگی صحنه بررسی کن.

## 6. معماری Orchestrator بدون VPS

### دیاگرام مفهومی
```text
Telegram / Web UI
       |
       v
Cloudflare Worker (Webhook + Auth + Rate Limit)
       |
       +----> AvalAI API (create/status/download)
       |
       v
Cloudflare Queue (اگر در حساب فعال باشد)
       |
       v
Queue Consumer Worker / Cron Trigger
       |
       +----> PHP HTTPS API ----> Existing MariaDB
       |
       +----> Cloudflare R2 (media objects)
       |
       +----> Telegram Bot API (progress/result)
```

### مسئولیت‌ها
- Frontend: گرفتن درخواست و نمایش وضعیت؛ کلید AvalAI در آن قرار نگیرد.
- Worker ingress: اعتبارسنجی کاربر، محدودیت نرخ، کنترل ورودی، ایجاد job، enqueue.
- Queue consumer: اجرای یک transition کوتاه؛ create/poll/download را از هم جدا کند.
- Cron Trigger: بررسی jobهایی که `next_check_at` آن‌ها رسیده است.
- PHP API: دسترسی محدود و احرازشده به MariaDB؛ فقط endpointهای لازم، نه دسترسی مستقیم و عمومی SQL.
- R2: ذخیره فایل MP4 و تصاویر مرجع در صورت استفاده.
- Telegram adapter: ارسال job id، progress و لینک خروجی.

### State machine پیشنهادی
`queued -> planning -> awaiting_approval -> generating -> polling -> downloading -> rendering (optional) -> completed`

و وضعیت‌های جانبی:
`failed`, `cancelled`, `needs_review`, `retry_scheduled`

### جدول jobs پیشنهادی
- `id` UUID/ULID
- `user_id`
- `status`
- `stage`
- `input_json`
- `plan_json`
- `provider`
- `model_id`
- `provider_video_id`
- `provider_request_id`
- `attempt_count`
- `next_check_at`
- `output_key`
- `estimated_cost_usd`
- `actual_cost_usd`
- `error_code`
- `error_message`
- `created_at`, `updated_at`, `completed_at`

### جدول job_scenes پیشنهادی
- `id`
- `job_id`
- `scene_index`
- `prompt`
- `seconds`
- `model_id`
- `video_id`
- `request_id`
- `status`
- `storage_key`
- `actual_cost_usd`
- `error_message`

### نکات پایداری
- Queue message را پس از ثبت تغییر وضعیت ACK کن.
- از lock/lease و unique key برای جلوگیری از اجرای هم‌زمان یک مرحله استفاده کن.
- retry برای 429/5xx با backoff و jitter؛ خطاهای validation/401/403 را retry نکن.
- `max_attempts` و dead-letter مشخص داشته باش.
- هر create ویدئو قبل از فراخوانی، یک رکورد intent ذخیره کند.
- اگر نتیجه create به دلیل قطع ارتباط نامشخص شد، وضعیت provider را بازیابی کن؛ درخواست جدید را کورکورانه نساز.
- هزینه واقعی پس از دریافت request_id با User API reconcile شود.

## 7. مونتاژ نهایی MP4 — محدودیت کلیدی بدون VPS

Cloudflare Workers برای orchestrating API مناسب‌اند، اما نباید آن‌ها را محیط عمومی اجرای FFmpeg یا پردازش سنگین ویدئو فرض کرد.

گزینه‌ها:
1. **بدون مونتاژ در MVP:** تحویل sceneها به‌صورت کلیپ‌های مجزا با شماره ترتیب؛ سریع‌ترین و کم‌ریسک‌ترین شروع.
2. **Python hosting موجود:** فقط پس از تأیید اینکه FFmpeg نصب است، اجرای background task مجاز است، زمان/حافظه/فضای ذخیره‌سازی کافی است و فایل‌های حجیم قابل دریافت/ارسال‌اند.
3. **رندر خارجی:** اگر خروجی یک MP4 یکپارچه الزام محصول است و Python hosting مناسب نیست، سرویس رندر API اضافه شود؛ هزینه و محدودیت provider باید جداگانه محاسبه شود.
4. **مرورگر کاربر:** برای ابزار داخلی و فایل‌های محدود می‌توان prototype مبتنی بر WebCodecs/ffmpeg.wasm را آزمایش کرد، اما برای pipeline خودکار Telegram/server-side راهکار تولیدی قابل اتکا فرض نشود.

## 8. امنیت و حریم خصوصی
- API key در GitHub/HTML public/JS source ممنوع.
- کلیدها فقط در Cloudflare Worker Secrets یا environment secrets سمت سرور.
- CORS جایگزین احراز هویت نیست.
- Webhook تلگرام باید secret path/token و اعتبارسنجی update داشته باشد.
- ورودی prompt، URL و فایل باید محدودیت اندازه/نوع داشته باشد.
- لینک خروجی باید موقت/امضاشده یا کنترل‌دسترسی‌شده باشد.
- لاگ‌ها نباید API key، token تلگرام یا محتوای حساس را افشا کنند.
- `safety_identifier` نباید شامل اطلاعات شخصی قابل شناسایی باشد.

## 9. هزینه واقعی و سودآوری
هزینه هر job را جزءبه‌جزء ذخیره کن:
- LLM: input/output/cached token cost
- Veo: billable seconds × model rate
- TTS: کاراکتر/ثانیه/درخواست بر اساس مدل واقعی
- image: قیمت مدل واقعی و تعداد تولید
- storage: GB-month و عملیات/egress طبق پلن
- rendering: زمان/دقیقه یا هزینه API
- retry: هزینه اضافی جدا
- پیام‌رسانی و سایر سرویس‌ها

فرمول:
`gross_margin = customer_revenue - (llm + video + image + tts + render + storage + egress + retries + platform_cost_share)`

برای هر job `estimated_cost` و `actual_cost` جدا نگهداری شود. قیمت تخمینی نباید جایگزین صورت‌حساب واقعی AvalAI شود.

## 10. تست پذیرش MVP

### تست AvalAI
- کلید معتبر: credit endpoint پاسخ دهد.
- کلید نامعتبر: 401/403 درست نمایش داده شود.
- create یک ویدئوی ۴ ثانیه‌ای Fast.
- video id و request id ذخیره شوند.
- poll تا completed/failed بدون sleep loop و بدون create تکراری.
- دانلود MP4 و بررسی Content-Type/حجم.
- transaction lookup و ذخیره هزینه واقعی.
- شبیه‌سازی قطع اتصال پس از create و اطمینان از عدم تولید تکراری.

### تست زیرساخت بدون VPS
- webhook از Telegram/Web به Worker برسد.
- job به Queue برسد یا fallback زمان‌بندی‌شده اجرا شود.
- DB فقط از API امن قابل دسترسی باشد.
- Worker در صورت timeout مرحله را دوباره از state ذخیره‌شده ادامه دهد.
- در صورت نبود FFmpeg، سیستم صادقانه کلیپ‌های جدا را تحویل دهد.
- secrets در bundle عمومی/DevTools/log ظاهر نشوند.

### معیار انتشار
- حداقل ۳ نمونه با موضوع متفاوت.
- اندازه‌گیری زمان از ثبت تا تکمیل.
- هزینه دقیق حداقل ۵ درخواست آزمایشی از User API.
- بررسی کیفیت بصری/صوتی توسط انسان.
- محاسبه قیمت فروش با حاشیه سود و retry budget.

## 11. منابع رسمی AvalAI

- API overview: https://docs.avalai.org/en/api-reference/introduction
- Quick start: https://docs.avalai.org/en/index
- User API: https://docs.avalai.org/en/api-reference/user
- Veo 3.1 video generation: https://docs.avalai.org/en/guides/generate-videos-using-veo
- Google model catalog: https://docs.avalai.org/en/providers/gemini
- Deployment checklist: https://docs.avalai.org/en/guides/deployment-checklist
- Provider-specific parameters: https://docs.avalai.org/en/guides/provider-specific-params
- Pricing: https://docs.avalai.org/en/pricing

## 12. قواعد نگهداری این دانش
- قبل از release، لینک‌های رسمی را بررسی کن.
- هر تغییر model ID، price، endpoint، parameter یا limit را با تاریخ ثبت کن.
- قیمت‌ها را به‌صورت ثابت داخل prompt هاردکد نکن؛ این سند baseline مستند است و runtime/catalog منبع تازه‌تر است.
- هر توصیه باید با یکی از برچسب‌های «مستند رسمی»، «استنباط مهندسی» یا «نیازمند تست» همراه باشد.

## Agent marketplace and technology-library evaluation rules (v6)

- Separate complete ready-to-run agents from frameworks/templates and individual skills/tools/MCP servers. Do not label an SDK or library as a complete agent.
- “Free/open source” must be qualified: inspect SPDX license, commercial-use permission, model/API fees, search/API quotas, hosting, GPU, storage, and premium feature boundaries.
- For every repository, verify recent commits/releases, archived status, issue activity, CI/tests, dependency health, security posture, install instructions, and license before recommending production use. Stars are not a quality score.
- No VPS is assumed. Cloudflare Workers can handle webhooks, API proxying, short orchestration and supported queues/cron. They are not a general Python runtime and do not replace FFmpeg, Playwright browser binaries, local Whisper inference, or persistent worker processes.
- Python Hosting is conditional until background jobs, process lifetime, RAM/CPU, disk, outbound network, FFmpeg availability, and deployment method are confirmed.
- Social-network integrations should prefer official APIs and OAuth with minimum scopes. API eligibility, account type, quotas, review requirements and pricing vary by platform and can change. Browser automation is a brittle fallback and must not bypass access controls, CAPTCHA, or platform terms. Human approval is recommended before publishing.
- Media workflows should check rights and permissions for source content, avoid assuming arbitrary video downloading is allowed, and preserve source attribution when required.
- Flex service tier is a separate request-level service tier, not an agent framework or model family. Use only for models/endpoints that currently document support; it trades lower cost for potentially higher latency/less predictable capacity. Keep time-sensitive user-facing operations on the default tier unless measured otherwise.
- AvalAI API families must remain distinct: Chat Completions, Responses, Decisions, System One, Extraction, image/video/audio endpoints. A model must be routed only to its documented endpoint and schema.

## Agent marketplace and technology-library evaluation rules (v6)

- Separate complete ready-to-run agents from frameworks/templates and individual skills/tools/MCP servers. Do not label an SDK or library as a complete agent.
- “Free/open source” must be qualified: inspect SPDX license, commercial-use permission, model/API fees, search/API quotas, hosting, GPU, storage, and premium feature boundaries.
- For every repository, verify recent commits/releases, archived status, issue activity, CI/tests, dependency health, security posture, install instructions, and license before recommending production use. Stars are not a quality score.
- No VPS is assumed. Cloudflare Workers can handle webhooks, API proxying, short orchestration and supported queues/cron. They are not a general Python runtime and do not replace FFmpeg, Playwright browser binaries, local Whisper inference, or persistent worker processes.
- Python Hosting is conditional until background jobs, process lifetime, RAM/CPU, disk, outbound network, FFmpeg availability, and deployment method are confirmed.
- Social-network integrations should prefer official APIs and OAuth with minimum scopes. API eligibility, account type, quotas, review requirements and pricing vary by platform and can change. Browser automation is a brittle fallback and must not bypass access controls, CAPTCHA, or platform terms. Human approval is recommended before publishing.
- Media workflows should check rights and permissions for source content, avoid assuming arbitrary video downloading is allowed, and preserve source attribution when required.
- Flex service tier is a separate request-level service tier, not an agent framework or model family. Use only for models/endpoints that currently document support; it trades lower cost for potentially higher latency/less predictable capacity. Keep time-sensitive user-facing operations on the default tier unless measured otherwise.
- AvalAI API families must remain distinct: Chat Completions, Responses, Decisions, System One, Extraction, image/video/audio endpoints. A model must be routed only to its documented endpoint and schema.
