# راهنمای رفع مشکل اتصال MySQL در Laravel Sail

## مشکل
خطای `SQLSTATE[HY000] [2002] php_network_getaddresses: getaddrinfo for mysql failed: Name or service not known` هنگام اجرای دستورات Laravel Artisan مانند `migrate`، `db:seed` و غیره.

## علل احتمالی

### 1. عدم تنظیم فایل .env
- فایل `.env` وجود ندارد یا تنظیمات پایگاه داده ناقص است
- `APP_KEY` تنظیم نشده است
- `DB_PASSWORD` خالی است
- `DB_USERNAME` نامناسب است

### 2. مشکل در MySQL Container
- MySQL container متوقف شده یا crash کرده است
- مشکل در MySQL image
- تداخل در socket files
- مشکل در volume های Docker

### 3. مشکل در تنظیمات Docker
- تنظیمات نامناسب در `docker-compose.yml`
- مشکل در network configuration
- تداخل port ها

## راه‌حل‌های مرحله به مرحله

### مرحله 1: بررسی وضعیت Container ها

```bash
# بررسی container های در حال اجرا
docker ps

# بررسی همه container ها (شامل متوقف شده ها)
docker ps -a | grep mysql

# بررسی logs MySQL container
docker logs karokia-mysql-1
```

### مرحله 2: تنظیم فایل .env

#### الف) تولید APP_KEY
```bash
# تغییر مجوز فایل .env
chmod 666 .env

# تولید APP_KEY
./vendor/bin/sail artisan key:generate
```

#### ب) تنظیم رمز عبور پایگاه داده
```bash
# ویرایش فایل .env و تنظیم رمز عبور
sed -i 's/DB_PASSWORD=/DB_PASSWORD=password/' .env

# یا به صورت دستی فایل .env را ویرایش کنید:
# DB_PASSWORD=password
```

#### ج) تنظیم نام کاربری مناسب
```bash
# تغییر نام کاربری از root به sail
sed -i 's/DB_USERNAME=root/DB_USERNAME=sail/' .env

# یا به صورت دستی:
# DB_USERNAME=sail
```

### مرحله 3: اصلاح docker-compose.yml

اگر از MySQL image مشکل‌دار استفاده می‌کنید، آن را تغییر دهید:

```yaml
# قبل (مشکل‌دار)
mysql:
    image: 'mysql/mysql-server:8.0'
    # ...

# بعد (بهتر)
mysql:
    image: 'mysql:8.0'
    ports:
        - '${FORWARD_DB_PORT:-3306}:3306'
    environment:
        MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
        MYSQL_DATABASE: '${DB_DATABASE}'
        MYSQL_USER: '${DB_USERNAME}'
        MYSQL_PASSWORD: '${DB_PASSWORD}'
        MYSQL_ALLOW_EMPTY_PASSWORD: 1
    volumes:
        - 'sail-mysql:/var/lib/mysql'
    networks:
        - sail
    healthcheck:
        test:
            - CMD
            - mysqladmin
            - ping
            - '-p${DB_PASSWORD}'
        retries: 3
        timeout: 5s
```

### مرحله 4: راه‌اندازی مجدد کامل

```bash
# متوقف کردن همه container ها و حذف volume ها
./vendor/bin/sail down -v

# راه‌اندازی مجدد
./vendor/bin/sail up -d

# صبر کردن برای آماده شدن MySQL (حداقل 30 ثانیه)
sleep 30
```

### مرحله 5: تست اتصال

```bash
# اجرای migration
./vendor/bin/sail artisan migrate

# یا تست اتصال با Tinker
./vendor/bin/sail artisan tinker
# سپس در Tinker: DB::connection()->getPdo();
```

## تنظیمات پیشنهادی فایل .env

```env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:YOUR_GENERATED_KEY_HERE
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password

# سایر تنظیمات...
```

## نکات مهم

### 1. زمان انتظار
- MySQL container نیاز به زمان دارد تا کاملاً آماده شود
- حداقل 30 ثانیه صبر کنید قبل از اجرای دستورات database

### 2. مجوز فایل‌ها
- اطمینان حاصل کنید که فایل `.env` قابل نوشتن است
- در صورت نیاز: `chmod 666 .env`

### 3. تداخل Port ها
- اطمینان حاصل کنید که port 3306 در host system آزاد است
- در صورت تداخل، port را در `docker-compose.yml` تغییر دهید

### 4. Volume های Docker
- در صورت مشکل مداوم، volume ها را پاک کنید: `./vendor/bin/sail down -v`
- این کار باعث از دست رفتن داده‌های موجود می‌شود

## دستورات مفید برای عیب‌یابی

```bash
# بررسی وضعیت همه container ها
docker ps -a

# بررسی logs همه container ها
docker logs karokia-mysql-1
docker logs karokia-laravel.test-1

# بررسی network های Docker
docker network ls
docker network inspect karokia_sail

# بررسی volume های Docker
docker volume ls
docker volume inspect karokia_sail-mysql

# تست اتصال مستقیم به MySQL
./vendor/bin/sail mysql

# بررسی تنظیمات Laravel
./vendor/bin/sail artisan config:cache
./vendor/bin/sail artisan config:clear
```

## مشکلات رایج و راه‌حل‌ها

### مشکل: "Another process with pid X is using unix socket file"
**راه‌حل**: پاک کردن volume ها و راه‌اندازی مجدد
```bash
./vendor/bin/sail down -v
./vendor/bin/sail up -d
```

### مشکل: "MYSQL_USER and MYSQL_PASSWORD are for configuring a regular user"
**راه‌حل**: تغییر DB_USERNAME از `root` به `sail`

### مشکل: "Permission denied" در فایل .env
**راه‌حل**: تغییر مجوز فایل
```bash
chmod 666 .env
```

## منابع مفید

- [Laravel Sail Documentation](https://laravel.com/docs/sail)
- [Docker MySQL Official Image](https://hub.docker.com/_/mysql)
- [Laravel Database Configuration](https://laravel.com/docs/database)

## نتیجه‌گیری

این مشکل معمولاً به دلیل تنظیمات ناقص یا نامناسب در فایل `.env` و `docker-compose.yml` رخ می‌دهد. با پیروی از مراحل بالا، مشکل باید حل شود. در صورت ادامه مشکل، بررسی logs container ها و تنظیمات network Docker می‌تواند کمک‌کننده باشد.

---

**نکته**: این راهنما برای Laravel Sail و Docker طراحی شده است. اگر از محیط‌های دیگر استفاده می‌کنید، ممکن است نیاز به تنظیمات متفاوتی داشته باشید. 

---

## تاریخچه کامل عیب‌یابی و تحلیل اولیه مشکل

*این بخش به تشریح کامل مشکل اولیه، تمام راه حل‌هایی که تست شدند اما ناموفق بودند، و نتیجه‌گیری اولیه قبل از پیدا شدن راه‌حل نهایی می‌پردازد. این اطلاعات برای درک عمیق‌تر مشکل بسیار مفید است.*

### ۱. خلاصه‌ی دقیق مشکل
مشکل اصلی یک خطای اتصال به پایگاه داده با پیغام `Name or service not known` یا `Temporary failure in name resolution` است.

**معنی خطا:** این یعنی پروسه PHP که دستور artisan را اجرا می‌کند، نمی‌تواند نام هاست `mysql` را در شبکه داخلی داکر پیدا کرده و به آدرس IP ترجمه کند.

**نکته کلیدی و تضاد بزرگ:** ما با اجرای دستور `curl mysql:3306` از داخل کانتینر، ثابت کردیم که شبکه داخلی داکر سالم است و نام `mysql` برای ابزارهای دیگر قابل شناسایی است.

**نتیجه‌گیری:** مشکل به طور خاص مربوط به محیط اجرای PHP-CLI (خط فرمان PHP) در داخل کانتینر داکر است و به نظر می‌رسد این محیط به درستی از سیستم DNS داکر استفاده نمی‌کند.

### ۲. کارهایی که تا به حال انجام داده‌ایم (و ناموفق بوده‌اند)
ما به صورت سیستماتیک تمام راه حل‌های استاندارد برای مشکلات محیطی و کش را امتحان کردیم:

* **راه حل‌های محیطی:**
    * خاموش و روشن کردن Sail (دستورات `sail down` و `sail up`).
    * خاموش و روشن کردن کامل WSL (دستور `wsl --shutdown`).
    * ری‌استارت کردن کامل نرم‌افزار Docker Desktop.
    * بازسازی کامل ایمیج‌های داکر (دستور `sail build --no-cache`).
    * ریست کردن داکر به تنظیمات کارخانه و نصب مجدد کامل WSL و داکر.
* **بررسی فایل‌های پیکربندی:**
    * فایل `.env` بررسی شد و مقادیر `DB_HOST=mysql` و سایر تنظیمات صحیح بودند.
    * فایل `docker-compose.yml` بررسی شد و نام سرویس `mysql` و شبکه `sail` صحیح بودند.
* **پاکسازی کش لاراول:**
    * کش تنظیمات پاک شد (`config:clear`).
    * تمام کش‌های لاراول به صورت کامل پاک شدند (`optimize:clear`).
* **عیب‌یابی شبکه:**
    * وضعیت کانتینرها بررسی شد و همگی سالم و در حال اجرا بودند (`sail ps`).
    * اتصال مستقیم از کانتینر لاراول به کانتینر MySQL با `curl` موفقیت‌آمیز بود.
* **استفاده از IP به جای نام هاست:** ما آدرس IP کانتینر MySQL را مستقیماً در فایل `.env` قرار دادیم و خطا از `Name resolution` به `No route to host` تغییر کرد که نشان‌دهنده یک مشکل عمیق‌تر در مسیریابی شبکه است.

### ۳. نتیجه‌گیری نهایی از تحلیل اولیه
با توجه به اینکه تمام موارد بالا بررسی شده و به نتیجه نرسیده‌اند، ما با یک مشکل بسیار نادر و خاص در لایه شبکه مجازی ویندوز (Hyper-V Virtual Switch) که توسط WSL و داکر استفاده می‌شود، روبرو هستیم. این مشکل فراتر از لاراول و داکر است.