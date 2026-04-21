# Phase 0 — Foundation Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a reproducible CodeIgniter 4 development environment with Docker, testing harness, linting, base UI layout, and CI pipeline — so subsequent phases can focus entirely on clinical features.

**Architecture:** Docker Compose wraps Nginx + PHP-FPM 8.3 + MariaDB 11 + Redis 7 + Mailhog. The CI4 app is organized into modules under `app/Modules/` with a `Shared` module for cross-cutting services. GitHub Actions runs lint + PHPUnit on every pull request.

**Tech Stack:** CodeIgniter 4.5+, PHP 8.3, MariaDB 11, Redis 7, Nginx, Docker, PHPUnit, Playwright, Bootstrap 5.3, HTMX 2, Alpine.js 3, PHP_CodeSniffer.

**Spec reference:** `docs/superpowers/specs/2026-04-21-iHomes-hospital-system-design.md` sections 5.2, 5.4, 12.3, 12.4.

---

## Prerequisites

- Docker Desktop (Windows/Mac) or Docker Engine + Compose (Linux) installed.
- Git configured with name + email.
- Node.js 20+ (for Playwright only — not used at runtime).
- Write access to `github.com/devnetechn/iHomes`.

## File Structure

```
iHomes/
├── .editorconfig                       NEW
├── .github/
│   └── workflows/
│       └── ci.yml                      NEW
├── app/
│   ├── Config/
│   │   ├── Autoload.php                MODIFY  (register modules)
│   │   ├── Database.php                MODIFY  (MariaDB via env)
│   │   ├── Cache.php                   MODIFY  (Redis driver)
│   │   ├── Session.php                 MODIFY  (Redis handler)
│   │   └── Routes.php                  MODIFY  (module route loader)
│   ├── Modules/
│   │   ├── Shared/                     NEW  (skeleton)
│   │   │   └── .gitkeep
│   │   ├── ER/                         NEW  (skeleton)
│   │   │   └── .gitkeep
│   │   ├── OPD/                        NEW  (skeleton)
│   │   ├── IPD/                        NEW  (skeleton)
│   │   ├── LIS/                        NEW  (skeleton)
│   │   ├── RIS/                        NEW  (skeleton)
│   │   ├── Pharmacy/                   NEW  (skeleton)
│   │   ├── Billing/                    NEW  (skeleton)
│   │   ├── MRS/                        NEW  (skeleton)
│   │   ├── HR/                         NEW  (skeleton)
│   │   ├── PhilHealth/                 NEW  (skeleton)
│   │   ├── HMO/                        NEW  (skeleton)
│   │   ├── Reports/                    NEW  (skeleton)
│   │   └── Queueing/                   NEW  (skeleton)
│   └── Views/
│       ├── layouts/
│       │   ├── main.php                NEW  (sidebar + topbar)
│       │   ├── auth.php                NEW  (centered card)
│       │   └── print.php               NEW  (print-only)
│       ├── components/
│       │   ├── sidebar.php             NEW
│       │   ├── topbar.php              NEW
│       │   └── flash.php               NEW
│       └── home.php                    NEW  (smoke-test landing page)
├── docker/
│   ├── php/
│   │   └── Dockerfile                  NEW
│   ├── nginx/
│   │   └── default.conf                NEW
│   └── mariadb/
│       └── init.sql                    NEW
├── docker-compose.yml                  NEW
├── public/
│   ├── assets/
│   │   ├── css/
│   │   │   └── app.css                 NEW
│   │   ├── js/
│   │   │   └── app.js                  NEW
│   │   └── vendor/                     NEW  (Bootstrap, HTMX, Alpine)
│   └── index.php                       (CI4 default — no change)
├── tests/
│   ├── _support/
│   │   └── DatabaseTestCase.php        NEW
│   ├── Modules/
│   │   └── Shared/
│   │       └── SmokeTest.php           NEW
│   └── e2e/
│       ├── playwright.config.ts        NEW
│       └── smoke.spec.ts               NEW
├── .env.example                        NEW
├── .gitignore                          MODIFY  (add node_modules, test-results)
├── composer.json                       NEW
├── package.json                        NEW
├── phpcs.xml                           NEW
├── phpunit.xml                         NEW
├── Makefile                            NEW
└── README.md                           MODIFY  (dev setup instructions)
```

---

## Task 1: Create `.env.example` Template

**Files:**
- Create: `.env.example`

`composer.json` will be created by `composer create-project` in Task 3, then augmented there. We only stage the env template first.

- [ ] **Step 1: Create `.env.example`**

```env
#--------------------------------------------------------------------
# APPLICATION
#--------------------------------------------------------------------
CI_ENVIRONMENT=development
app.baseURL='http://localhost:8080/'
app.forceGlobalSecureRequests=false
app.sessionDriver='CodeIgniter\Session\Handlers\RedisHandler'
app.sessionCookieName='ihomes_session'
app.sessionSavePath='tcp://redis:6379'
app.sessionExpiration=1800

#--------------------------------------------------------------------
# DATABASE
#--------------------------------------------------------------------
database.default.hostname=mariadb
database.default.database=ihomes
database.default.username=ihomes
database.default.password=ihomes_dev
database.default.DBDriver=MySQLi
database.default.port=3306
database.default.charset=utf8mb4
database.default.DBCollat=utf8mb4_unicode_ci

#--------------------------------------------------------------------
# CACHE / REDIS
#--------------------------------------------------------------------
cache.handler=redis
cache.redis.host=redis
cache.redis.port=6379

#--------------------------------------------------------------------
# PHILHEALTH
#--------------------------------------------------------------------
philhealth.mode=mock
philhealth.endpoint=http://philhealth-mock:8090/eclaims
philhealth.hciCode=
philhealth.senderId=
philhealth.certPath=
philhealth.certPass=
```

- [ ] **Step 2: Commit**

```bash
git add .env.example
git commit -m "chore: add .env.example template"
```

---

## Task 2: Build Docker Compose Development Stack

**Files:**
- Create: `docker/php/Dockerfile`
- Create: `docker/nginx/default.conf`
- Create: `docker/mariadb/init.sql`
- Create: `docker-compose.yml`
- Modify: `.gitignore`

- [ ] **Step 1: Create `docker/php/Dockerfile`**

```dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache \
    bash \
    git \
    unzip \
    libzip-dev \
    icu-dev \
    oniguruma-dev \
    libxml2-dev \
    $PHPIZE_DEPS \
 && docker-php-ext-install \
    intl \
    mbstring \
    zip \
    pdo \
    pdo_mysql \
    mysqli \
    xml \
    opcache \
 && pecl install redis \
 && docker-php-ext-enable redis \
 && apk del $PHPIZE_DEPS \
 && rm -rf /tmp/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

RUN addgroup -g 1000 app && adduser -u 1000 -G app -S app
USER app

EXPOSE 9000
```

- [ ] **Step 2: Create `docker/nginx/default.conf`**

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php index.html;

    client_max_body_size 50M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 60s;
    }

    location ~ /\.ht { deny all; }

    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 7d;
        add_header Cache-Control "public, no-transform";
    }
}
```

- [ ] **Step 3: Create `docker/mariadb/init.sql`**

```sql
CREATE DATABASE IF NOT EXISTS ihomes
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_unicode_ci;

CREATE DATABASE IF NOT EXISTS ihomes_test
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_unicode_ci;

GRANT ALL PRIVILEGES ON ihomes.*      TO 'ihomes'@'%';
GRANT ALL PRIVILEGES ON ihomes_test.* TO 'ihomes'@'%';
FLUSH PRIVILEGES;
```

- [ ] **Step 4: Create `docker-compose.yml`**

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    ports: ["8080:80"]
    volumes:
      - .:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on: [php]

  php:
    build: ./docker/php
    volumes:
      - .:/var/www/html
    environment:
      PHP_IDE_CONFIG: "serverName=ihomes"
    depends_on: [mariadb, redis]

  mariadb:
    image: mariadb:11
    ports: ["3306:3306"]
    environment:
      MARIADB_ROOT_PASSWORD: root_dev
      MARIADB_USER: ihomes
      MARIADB_PASSWORD: ihomes_dev
      MARIADB_DATABASE: ihomes
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./docker/mariadb/init.sql:/docker-entrypoint-initdb.d/init.sql:ro

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI

volumes:
  mariadb_data:
```

- [ ] **Step 5: Add `node_modules/` and `test-results/` to `.gitignore`**

Modify `.gitignore`, append at the end:

```gitignore
# Playwright
test-results/
playwright-report/
/e2e-results/
```

Check `/node_modules/` is already present (it is).

- [ ] **Step 6: Verify stack boots**

```bash
docker compose up -d
docker compose ps
```

Expected: `nginx`, `php`, `mariadb`, `redis`, `mailhog` all showing `Up` or `running`.

- [ ] **Step 7: Commit**

```bash
git add docker/ docker-compose.yml .gitignore
git commit -m "feat: add Docker Compose dev stack (nginx, php 8.3, mariadb 11, redis 7, mailhog)"
```

---

## Task 3: Install CodeIgniter 4 Appstarter + Extra Dependencies

**Files:**
- Create: `app/`, `public/`, `writable/`, `spark`, `phpunit.xml.dist`, `composer.json` (all from CI4 appstarter)
- Modify: generated `composer.json` — add extra packages
- Create: `.env` (copy from `.env.example`, gitignored)

- [ ] **Step 1: Scaffold CI4 appstarter into the repo root**

`composer create-project` requires the target dir to be empty of composer metadata. Our repo already contains `README.md`, `.gitignore`, `.env.example`, `docker/`, `docker-compose.yml`, `docs/`, `.git/`. The appstarter install tolerates extra non-composer files.

Run from the host (not the container, because create-project expects to manage the vendor dir directly; Docker exec would not have composer binary available before install):

```bash
docker compose exec php composer create-project codeigniter4/appstarter . --no-install --remove-vcs
```

Expected: `app/`, `public/`, `writable/`, `spark`, `composer.json`, `phpunit.xml.dist`, `preload.php` created. If it errors that the dir is not empty, ensure no `composer.json` or `composer.lock` already exists, then retry.

- [ ] **Step 2: Augment the generated `composer.json`**

Open the freshly generated `composer.json` and merge these additions:

Into `require` (keep the CI4 entries the appstarter added; add these):

```json
"predis/predis": "^2.2",
"dompdf/dompdf": "^3.0",
"ramsey/uuid": "^4.7"
```

Into `require-dev`:

```json
"squizlabs/php_codesniffer": "^3.10",
"mockery/mockery": "^1.6",
"fakerphp/faker": "^1.23"
```

Replace or add the `scripts` block:

```json
"scripts": {
    "lint": "phpcs",
    "test": "phpunit",
    "serve": "php spark serve --host 0.0.0.0 --port 8080"
}
```

Add autoload-dev (keep existing autoload untouched):

```json
"autoload-dev": {
    "psr-4": {
        "Tests\\": "tests/_support/"
    }
}
```

- [ ] **Step 3: Install composer dependencies**

```bash
docker compose exec php composer install --no-interaction
```

Expected: `vendor/` created. No errors.

- [ ] **Step 4: Copy `.env.example` to `.env`**

```bash
cp .env.example .env
```

- [ ] **Step 5: Verify CI4 responds**

```bash
curl -s http://localhost:8080/ | head -20
```

Expected: HTML containing "Welcome to CodeIgniter 4!".

- [ ] **Step 6: Commit**

```bash
git add composer.json composer.lock app/ public/ writable/ spark phpunit.xml.dist preload.php
git status --short
git commit -m "feat: install CodeIgniter 4 appstarter + extra dependencies (predis, dompdf, uuid, phpcs, mockery, faker)"
```

If `.env` appears in status, it is already gitignored — confirm with `git check-ignore .env`.

---

## Task 4: Configure CI4 for Modular Architecture

**Files:**
- Modify: `app/Config/Autoload.php`
- Modify: `app/Config/Routes.php`
- Modify: `app/Config/Database.php`
- Modify: `app/Config/Cache.php`
- Modify: `app/Config/Session.php`
- Create: `app/Modules/Shared/.gitkeep` (and one per module)

- [ ] **Step 1: Create module skeleton folders**

```bash
for m in Shared ER OPD IPD LIS RIS Pharmacy Billing MRS HR PhilHealth HMO Reports Queueing; do
  mkdir -p "app/Modules/$m/Controllers" \
           "app/Modules/$m/Models" \
           "app/Modules/$m/Services" \
           "app/Modules/$m/Views" \
           "app/Modules/$m/Database/Migrations" \
           "app/Modules/$m/Database/Seeds" \
           "app/Modules/$m/Config"
  touch "app/Modules/$m/.gitkeep"
done
```

- [ ] **Step 2: Register modules in `app/Config/Autoload.php`**

Locate the `$psr4` array (around line 40-55) and add module namespaces. Replace the existing `$psr4` array with:

```php
public $psr4 = [
    APP_NAMESPACE                      => APPPATH,
    'App\\Modules\\Shared'             => APPPATH . 'Modules/Shared',
    'App\\Modules\\ER'                 => APPPATH . 'Modules/ER',
    'App\\Modules\\OPD'                => APPPATH . 'Modules/OPD',
    'App\\Modules\\IPD'                => APPPATH . 'Modules/IPD',
    'App\\Modules\\LIS'                => APPPATH . 'Modules/LIS',
    'App\\Modules\\RIS'                => APPPATH . 'Modules/RIS',
    'App\\Modules\\Pharmacy'           => APPPATH . 'Modules/Pharmacy',
    'App\\Modules\\Billing'            => APPPATH . 'Modules/Billing',
    'App\\Modules\\MRS'                => APPPATH . 'Modules/MRS',
    'App\\Modules\\HR'                 => APPPATH . 'Modules/HR',
    'App\\Modules\\PhilHealth'         => APPPATH . 'Modules/PhilHealth',
    'App\\Modules\\HMO'                => APPPATH . 'Modules/HMO',
    'App\\Modules\\Reports'            => APPPATH . 'Modules/Reports',
    'App\\Modules\\Queueing'           => APPPATH . 'Modules/Queueing',
];
```

- [ ] **Step 3: Add module route loader to `app/Config/Routes.php`**

At the end of the `$routes` block (before the last closing `}`), add:

```php
// Auto-load per-module route files
$moduleNames = [
    'Shared','ER','OPD','IPD','LIS','RIS','Pharmacy','Billing',
    'MRS','HR','PhilHealth','HMO','Reports','Queueing',
];
foreach ($moduleNames as $mod) {
    $modRoutes = APPPATH . "Modules/{$mod}/Config/Routes.php";
    if (is_file($modRoutes)) {
        require $modRoutes;
    }
}
```

- [ ] **Step 4: Configure Redis session handler**

Edit `app/Config/Session.php`, ensure the `public string $driver` is set to `'CodeIgniter\\Session\\Handlers\\RedisHandler'` and `public string $savePath` supports env override. Replace the `$savePath` default:

```php
public string $savePath = 'tcp://redis:6379';
```

- [ ] **Step 5: Verify routing still works**

```bash
docker compose exec php php spark routes
```

Expected: command runs without error, lists at least the default `/` route.

- [ ] **Step 6: Verify homepage still works**

```bash
curl -sI http://localhost:8080/ | head -1
```

Expected: `HTTP/1.1 200 OK`.

- [ ] **Step 7: Commit**

```bash
git add app/Config/ app/Modules/
git commit -m "feat: register 14 modules in CI4 autoload + routes; scaffold module folders"
```

---

## Task 5: Set Up EditorConfig + PHP_CodeSniffer

**Files:**
- Create: `.editorconfig`
- Create: `phpcs.xml`

- [ ] **Step 1: Create `.editorconfig`**

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false

[*.{yml,yaml,json}]
indent_size = 2

[Makefile]
indent_style = tab
```

- [ ] **Step 2: Create `phpcs.xml`**

```xml
<?xml version="1.0"?>
<ruleset name="iHomes">
    <description>PHP coding standard for iHomes</description>

    <file>app</file>
    <file>tests</file>

    <exclude-pattern>*/vendor/*</exclude-pattern>
    <exclude-pattern>*/Views/*</exclude-pattern>

    <arg name="colors"/>
    <arg value="p"/>
    <arg name="extensions" value="php"/>

    <rule ref="PSR12"/>

    <rule ref="Generic.Files.LineLength">
        <properties>
            <property name="lineLimit" value="120"/>
            <property name="absoluteLineLimit" value="160"/>
        </properties>
    </rule>
</ruleset>
```

- [ ] **Step 3: Verify linter runs**

```bash
docker compose exec php vendor/bin/phpcs
```

Expected: runs without crashing. May report violations on stock CI4 files — that is OK for now, don't fix stock code. The command must exit (0 clean or non-zero with report).

- [ ] **Step 4: Commit**

```bash
git add .editorconfig phpcs.xml
git commit -m "chore: add .editorconfig and PSR-12 phpcs ruleset"
```

---

## Task 6: Set Up PHPUnit + Shared Test Case

**Files:**
- Create: `phpunit.xml`
- Create: `tests/_support/DatabaseTestCase.php`
- Create: `tests/Modules/Shared/SmokeTest.php`

- [ ] **Step 1: Create `phpunit.xml`** (overrides the CI4 `phpunit.xml.dist`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.5/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
         cacheResultFile=".phpunit.cache/test-results">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Modules</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <directory>app/Modules</directory>
        </include>
        <exclude>
            <directory>app/Modules/*/Views</directory>
        </exclude>
    </source>
    <php>
        <env name="CI_ENVIRONMENT" value="testing"/>
        <env name="database.default.database" value="ihomes_test"/>
    </php>
</phpunit>
```

- [ ] **Step 2: Create `tests/_support/DatabaseTestCase.php`**

```php
<?php

namespace Tests;

use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;
use CodeIgniter\Test\FeatureTestTrait;

abstract class DatabaseTestCase extends CIUnitTestCase
{
    use DatabaseTestTrait;
    use FeatureTestTrait;

    protected $refresh  = true;
    protected $namespace = null;

    protected function setUp(): void
    {
        parent::setUp();
    }
}
```

- [ ] **Step 3: Write the smoke test `tests/Modules/Shared/SmokeTest.php`**

```php
<?php

namespace Tests\Modules\Shared;

use PHPUnit\Framework\TestCase;

final class SmokeTest extends TestCase
{
    public function test_php_version_is_supported(): void
    {
        $this->assertTrue(
            version_compare(PHP_VERSION, '8.3.0', '>='),
            'PHP 8.3 or higher is required; got ' . PHP_VERSION
        );
    }

    public function test_redis_extension_is_loaded(): void
    {
        $this->assertTrue(extension_loaded('redis'), 'redis PHP extension must be loaded');
    }

    public function test_pdo_mysql_extension_is_loaded(): void
    {
        $this->assertTrue(extension_loaded('pdo_mysql'), 'pdo_mysql extension must be loaded');
    }
}
```

- [ ] **Step 4: Run the smoke test**

```bash
docker compose exec php vendor/bin/phpunit tests/Modules/Shared/SmokeTest.php
```

Expected: `OK (3 tests, 3 assertions)`.

- [ ] **Step 5: Commit**

```bash
git add phpunit.xml tests/
git commit -m "test: add PHPUnit config and environment smoke tests"
```

---

## Task 7: Add Frontend Assets — Bootstrap 5, HTMX, Alpine.js

**Files:**
- Create: `package.json`
- Create: `public/assets/css/app.css`
- Create: `public/assets/js/app.js`
- Create: `public/assets/vendor/` (downloaded vendor files)

- [ ] **Step 1: Create `package.json`** (used only for vendor asset management + Playwright)

```json
{
  "name": "ihomes-frontend",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "vendor:install": "node scripts/copy-vendor.mjs",
    "e2e": "playwright test",
    "e2e:ui": "playwright test --ui"
  },
  "devDependencies": {
    "@playwright/test": "^1.47.0",
    "bootstrap": "5.3.3",
    "bootstrap-icons": "^1.11.3",
    "htmx.org": "2.0.2",
    "alpinejs": "3.14.1"
  }
}
```

- [ ] **Step 2: Create `scripts/copy-vendor.mjs`**

```javascript
import { mkdir, copyFile } from 'node:fs/promises';
import { existsSync } from 'node:fs';

const VENDOR = 'public/assets/vendor';

const COPIES = [
  ['node_modules/bootstrap/dist/css/bootstrap.min.css',   `${VENDOR}/bootstrap/bootstrap.min.css`],
  ['node_modules/bootstrap/dist/js/bootstrap.bundle.min.js', `${VENDOR}/bootstrap/bootstrap.bundle.min.js`],
  ['node_modules/bootstrap-icons/font/bootstrap-icons.min.css', `${VENDOR}/bootstrap-icons/bootstrap-icons.min.css`],
  ['node_modules/htmx.org/dist/htmx.min.js',              `${VENDOR}/htmx/htmx.min.js`],
  ['node_modules/alpinejs/dist/cdn.min.js',               `${VENDOR}/alpine/alpine.min.js`],
];

for (const [src, dst] of COPIES) {
  if (!existsSync(src)) {
    console.error(`Missing ${src} — run npm install first`);
    process.exit(1);
  }
  const dir = dst.slice(0, dst.lastIndexOf('/'));
  await mkdir(dir, { recursive: true });
  await copyFile(src, dst);
  console.log(`copied ${dst}`);
}

// bootstrap-icons fonts
await mkdir(`${VENDOR}/bootstrap-icons/fonts`, { recursive: true });
for (const ext of ['woff', 'woff2']) {
  await copyFile(
    `node_modules/bootstrap-icons/font/fonts/bootstrap-icons.${ext}`,
    `${VENDOR}/bootstrap-icons/fonts/bootstrap-icons.${ext}`
  );
}
console.log('done');
```

- [ ] **Step 3: Install npm packages**

```bash
npm install
npm run vendor:install
```

Expected: `public/assets/vendor/bootstrap/`, `.../htmx/`, `.../alpine/`, `.../bootstrap-icons/` populated.

- [ ] **Step 4: Create `public/assets/css/app.css`**

```css
:root {
  --ih-sidebar-width: 240px;
  --ih-topbar-height: 56px;
  --ih-surface: #ffffff;
  --ih-muted: #6b7280;
}

html, body { height: 100%; }
body { background: #f5f6f8; font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif; }

.ih-topbar {
  position: fixed; top: 0; left: 0; right: 0;
  height: var(--ih-topbar-height);
  background: var(--ih-surface);
  border-bottom: 1px solid #e5e7eb;
  display: flex; align-items: center; padding: 0 1rem;
  z-index: 1020;
}

.ih-sidebar {
  position: fixed; top: var(--ih-topbar-height); left: 0; bottom: 0;
  width: var(--ih-sidebar-width);
  background: var(--ih-surface);
  border-right: 1px solid #e5e7eb;
  overflow-y: auto;
}
.ih-sidebar a { display: block; padding: 0.6rem 1rem; color: #374151; text-decoration: none; }
.ih-sidebar a:hover, .ih-sidebar a.active { background: #f3f4f6; color: #111827; }

.ih-main {
  margin-top: var(--ih-topbar-height);
  margin-left: var(--ih-sidebar-width);
  padding: 1.25rem;
}

@media print {
  .ih-sidebar, .ih-topbar { display: none !important; }
  .ih-main { margin: 0; padding: 0; }
}
```

- [ ] **Step 5: Create `public/assets/js/app.js`**

```javascript
document.addEventListener('DOMContentLoaded', () => {
  // Mark current sidebar link
  const current = window.location.pathname;
  document.querySelectorAll('.ih-sidebar a').forEach((a) => {
    if (current === a.getAttribute('href')) a.classList.add('active');
  });
});
```

- [ ] **Step 6: Add node_modules to .gitignore (already present)** and commit

```bash
git add package.json scripts/ public/assets/css/ public/assets/js/ public/assets/vendor/
git commit -m "feat: add Bootstrap 5, HTMX 2, Alpine 3 frontend vendor assets"
```

**Note:** committing `public/assets/vendor/` keeps production deploys reproducible without `npm install` on the hospital server.

---

## Task 8: Build Base Layout + Sidebar + Topbar

**Files:**
- Create: `app/Views/layouts/main.php`
- Create: `app/Views/layouts/auth.php`
- Create: `app/Views/layouts/print.php`
- Create: `app/Views/components/sidebar.php`
- Create: `app/Views/components/topbar.php`
- Create: `app/Views/components/flash.php`
- Create: `app/Views/home.php`
- Modify: `app/Controllers/Home.php`

- [ ] **Step 1: Create `app/Views/layouts/main.php`**

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?= esc($title ?? 'iHomes') ?></title>
    <link rel="stylesheet" href="/assets/vendor/bootstrap/bootstrap.min.css">
    <link rel="stylesheet" href="/assets/vendor/bootstrap-icons/bootstrap-icons.min.css">
    <link rel="stylesheet" href="/assets/css/app.css">
</head>
<body>
    <?= view('components/topbar') ?>
    <?= view('components/sidebar') ?>
    <main class="ih-main">
        <?= view('components/flash') ?>
        <?= $this->renderSection('content') ?>
    </main>

    <script src="/assets/vendor/bootstrap/bootstrap.bundle.min.js"></script>
    <script src="/assets/vendor/htmx/htmx.min.js"></script>
    <script defer src="/assets/vendor/alpine/alpine.min.js"></script>
    <script src="/assets/js/app.js"></script>
</body>
</html>
```

- [ ] **Step 2: Create `app/Views/layouts/auth.php`**

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?= esc($title ?? 'iHomes — Sign in') ?></title>
    <link rel="stylesheet" href="/assets/vendor/bootstrap/bootstrap.min.css">
    <link rel="stylesheet" href="/assets/css/app.css">
    <style>
        body { display:flex; align-items:center; justify-content:center; min-height:100vh; background:#0f172a; }
        .auth-card { width: 380px; background: #fff; padding: 2rem; border-radius: .5rem; }
    </style>
</head>
<body>
    <div class="auth-card">
        <?= $this->renderSection('content') ?>
    </div>
</body>
</html>
```

- [ ] **Step 3: Create `app/Views/layouts/print.php`**

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title><?= esc($title ?? 'iHomes — Print') ?></title>
    <link rel="stylesheet" href="/assets/vendor/bootstrap/bootstrap.min.css">
    <style>
        @page { size: A4; margin: 1.2cm; }
        body { font-family: Arial, sans-serif; font-size: 11pt; }
        .no-print { display: none !important; }
    </style>
</head>
<body>
    <?= $this->renderSection('content') ?>
</body>
</html>
```

- [ ] **Step 4: Create `app/Views/components/topbar.php`**

```php
<header class="ih-topbar">
    <div class="fw-bold me-3">
        <i class="bi bi-hospital text-danger"></i> iHomes
    </div>
    <form class="d-flex flex-grow-1 me-3" onsubmit="event.preventDefault()">
        <input type="text"
               class="form-control form-control-sm"
               placeholder="Search patient… (Ctrl+K)"
               aria-label="Search">
    </form>
    <div class="ms-auto d-flex align-items-center gap-2">
        <button class="btn btn-sm btn-outline-secondary" type="button" aria-label="Notifications">
            <i class="bi bi-bell"></i>
        </button>
        <div class="dropdown" x-data="{ open: false }">
            <button class="btn btn-sm btn-outline-primary" @click="open = !open">
                <i class="bi bi-person-circle"></i>
                <span><?= esc(session('user.full_name') ?? 'Guest') ?></span>
            </button>
        </div>
    </div>
</header>
```

- [ ] **Step 5: Create `app/Views/components/sidebar.php`**

```php
<nav class="ih-sidebar">
    <a href="/"><i class="bi bi-speedometer2"></i> Dashboard</a>
    <a href="/er"><i class="bi bi-bandaid"></i> ER</a>
    <a href="/opd"><i class="bi bi-clipboard-pulse"></i> OPD</a>
    <a href="/ipd"><i class="bi bi-hospital"></i> IPD</a>
    <a href="/lis"><i class="bi bi-droplet-half"></i> Laboratory</a>
    <a href="/ris"><i class="bi bi-camera"></i> Radiology</a>
    <a href="/pharmacy"><i class="bi bi-capsule"></i> Pharmacy</a>
    <a href="/billing"><i class="bi bi-receipt"></i> Billing</a>
    <a href="/mrs"><i class="bi bi-folder2-open"></i> Medical Records</a>
    <a href="/philhealth"><i class="bi bi-shield-check"></i> PhilHealth</a>
    <a href="/hmo"><i class="bi bi-credit-card"></i> HMO</a>
    <a href="/reports"><i class="bi bi-bar-chart"></i> Reports</a>
    <a href="/hr"><i class="bi bi-people"></i> HR</a>
    <a href="/queueing"><i class="bi bi-list-ol"></i> Queue</a>
    <hr>
    <a href="/admin"><i class="bi bi-gear"></i> Admin</a>
</nav>
```

- [ ] **Step 6: Create `app/Views/components/flash.php`**

```php
<?php $session = session(); ?>
<?php foreach (['success', 'info', 'warning', 'danger'] as $level): ?>
    <?php if ($msg = $session->getFlashdata($level)): ?>
        <div class="alert alert-<?= $level ?> alert-dismissible fade show" role="alert">
            <?= esc($msg) ?>
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
    <?php endif; ?>
<?php endforeach; ?>
```

- [ ] **Step 7: Create `app/Views/home.php`**

```php
<?= $this->extend('layouts/main') ?>

<?= $this->section('content') ?>
<div class="card">
    <div class="card-body">
        <h1 class="h3">iHomes Hospital Information System</h1>
        <p class="text-muted mb-0">Foundation infrastructure running. Sign in to continue.</p>
        <p class="mt-3 small">
            Build marker: <code id="build-marker">phase-0-foundation</code>
            &middot; PHP <?= PHP_VERSION ?>
            &middot; CI4 <?= \CodeIgniter\CodeIgniter::CI_VERSION ?>
        </p>
    </div>
</div>
<?= $this->endSection() ?>
```

- [ ] **Step 8: Update `app/Controllers/Home.php` to render the new view**

Replace the entire file with:

```php
<?php

namespace App\Controllers;

class Home extends BaseController
{
    public function index(): string
    {
        return view('home', ['title' => 'iHomes']);
    }
}
```

- [ ] **Step 9: Verify the page renders**

```bash
curl -s http://localhost:8080/ | grep -o 'phase-0-foundation'
```

Expected: `phase-0-foundation` is printed once.

- [ ] **Step 10: Commit**

```bash
git add app/Views/ app/Controllers/Home.php
git commit -m "feat: add base layout, sidebar, topbar, and Phase 0 home page"
```

---

## Task 9: Add Playwright E2E Smoke Test

**Files:**
- Create: `tests/e2e/playwright.config.ts`
- Create: `tests/e2e/smoke.spec.ts`

- [ ] **Step 1: Install Playwright browsers**

```bash
npx playwright install --with-deps chromium
```

Expected: Chromium downloaded and system deps installed.

- [ ] **Step 2: Create `tests/e2e/playwright.config.ts`**

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: '.',
  timeout: 30_000,
  fullyParallel: true,
  retries: 0,
  reporter: [['list'], ['html', { open: 'never' }]],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:8080',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
});
```

- [ ] **Step 3: Create `tests/e2e/smoke.spec.ts`**

```typescript
import { test, expect } from '@playwright/test';

test.describe('foundation smoke', () => {
  test('home page renders iHomes brand and build marker', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveTitle(/iHomes/);
    await expect(page.locator('.ih-topbar')).toContainText('iHomes');
    await expect(page.locator('#build-marker')).toHaveText('phase-0-foundation');
  });

  test('sidebar exposes all 14 modules plus admin', async ({ page }) => {
    await page.goto('/');
    const links = await page.locator('.ih-sidebar a').allTextContents();
    for (const label of [
      'Dashboard', 'ER', 'OPD', 'IPD', 'Laboratory', 'Radiology',
      'Pharmacy', 'Billing', 'Medical Records', 'PhilHealth', 'HMO',
      'Reports', 'HR', 'Queue', 'Admin',
    ]) {
      expect(links.join(' ')).toContain(label);
    }
  });
});
```

- [ ] **Step 4: Run the Playwright smoke test**

```bash
npx playwright test --config=tests/e2e/playwright.config.ts
```

Expected: `2 passed` in the summary.

- [ ] **Step 5: Commit**

```bash
git add tests/e2e/ package.json package-lock.json
git commit -m "test: add Playwright E2E smoke tests for home page and sidebar"
```

---

## Task 10: Add GitHub Actions CI Pipeline

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Create `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    name: Lint + PHPUnit
    runs-on: ubuntu-latest
    services:
      mariadb:
        image: mariadb:11
        env:
          MARIADB_ROOT_PASSWORD: root_dev
          MARIADB_DATABASE: ihomes_test
          MARIADB_USER: ihomes
          MARIADB_PASSWORD: ihomes_dev
        ports: ["3306:3306"]
        options: >-
          --health-cmd="mariadb-admin ping -uroot -proot_dev"
          --health-interval=10s --health-timeout=5s --health-retries=5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4

      - name: Set up PHP 8.3
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: intl, mbstring, zip, pdo_mysql, redis, xml, mysqli
          tools: composer:v2
          coverage: none

      - name: Install composer dependencies
        run: composer install --no-interaction --prefer-dist --no-progress

      - name: Lint (phpcs)
        run: vendor/bin/phpcs

      - name: Run PHPUnit
        run: vendor/bin/phpunit
        env:
          database.default.hostname: 127.0.0.1
          database.default.database: ihomes_test
          database.default.username: ihomes
          database.default.password: ihomes_dev
          cache.redis.host: 127.0.0.1

  e2e:
    name: Playwright E2E
    runs-on: ubuntu-latest
    needs: lint-and-test
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node 20
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install npm dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Build Docker stack
        run: docker compose up -d

      - name: Wait for MariaDB
        run: |
          for i in {1..30}; do
            if docker compose exec -T mariadb mariadb-admin ping -uroot -proot_dev 2>/dev/null; then
              echo "MariaDB ready"; exit 0
            fi
            sleep 2
          done
          echo "MariaDB did not become ready"; exit 1

      - name: Install composer deps in container
        run: docker compose exec -T php composer install --no-interaction

      - name: Copy env
        run: cp .env.example .env

      - name: Copy vendor assets
        run: npm run vendor:install

      - name: Run Playwright smoke tests
        run: npx playwright test --config=tests/e2e/playwright.config.ts
        env:
          BASE_URL: http://localhost:8080

      - name: Upload Playwright report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add GitHub Actions pipeline (lint, PHPUnit, Playwright E2E)"
```

- [ ] **Step 3: Push and verify CI runs green**

```bash
git push
```

Then open `https://github.com/devnetechn/iHomes/actions` in a browser. Expected: workflow triggered, both jobs succeed.

---

## Task 11: Add Makefile + README Dev Setup Guide

**Files:**
- Create: `Makefile`
- Modify: `README.md`

- [ ] **Step 1: Create `Makefile`**

```makefile
.PHONY: up down restart logs shell composer test lint e2e migrate fresh

up:
	docker compose up -d

down:
	docker compose down

restart:
	docker compose restart

logs:
	docker compose logs -f

shell:
	docker compose exec php sh

composer:
	docker compose exec php composer $(cmd)

test:
	docker compose exec php vendor/bin/phpunit

lint:
	docker compose exec php vendor/bin/phpcs

e2e:
	npx playwright test --config=tests/e2e/playwright.config.ts

migrate:
	docker compose exec php php spark migrate

fresh:
	docker compose down -v
	docker compose up -d
	sleep 5
	docker compose exec php composer install
	cp -n .env.example .env || true
	docker compose exec php php spark migrate
```

- [ ] **Step 2: Rewrite `README.md`**

```markdown
# iHomes

Hospital Information System for a PhilHealth-accredited Level 3 hospital in the Philippines, centered on the PhilHealth Outpatient Emergency Care Benefit (OECB) workflow.

**Status:** Phase 0 — foundation infrastructure.

**Design spec:** `docs/superpowers/specs/2026-04-21-iHomes-hospital-system-design.md`

## Stack

- PHP 8.3 / CodeIgniter 4
- MariaDB 11
- Redis 7
- Nginx + PHP-FPM
- Bootstrap 5.3 + HTMX 2 + Alpine.js 3
- Docker Compose for dev

## Local Development

### Prerequisites

- Docker Desktop (Windows/Mac) or Docker Engine + Compose (Linux)
- Node.js 20+ (for Playwright E2E only)
- Git

### First-Time Setup

```bash
git clone https://github.com/devnetechn/iHomes.git
cd iHomes

# Start the stack
docker compose up -d

# Install PHP dependencies
docker compose exec php composer install

# Copy env
cp .env.example .env

# Install npm dependencies + vendor assets
npm install
npm run vendor:install

# Verify
curl http://localhost:8080/
```

Open http://localhost:8080 — you should see the iHomes landing page.

### Common Commands

| Command | Description |
|---|---|
| `make up` | Start Docker stack |
| `make down` | Stop Docker stack |
| `make shell` | Open shell inside PHP container |
| `make test` | Run PHPUnit |
| `make lint` | Run PHP_CodeSniffer |
| `make e2e` | Run Playwright E2E tests |
| `make migrate` | Run database migrations |
| `make fresh` | Nuke DB, reinstall, remigrate |

### Ports

| Service | URL |
|---|---|
| App (Nginx) | http://localhost:8080 |
| MariaDB | localhost:3306 (user `ihomes`, pass `ihomes_dev`) |
| Redis | localhost:6379 |
| Mailhog UI | http://localhost:8025 |

## Project Structure

```
app/Modules/<Name>/         Each module (14 total)
docs/superpowers/specs/     Design specs
docs/superpowers/plans/     Implementation plans
tests/Modules/              PHPUnit tests
tests/e2e/                  Playwright E2E tests
docker/                     Dockerfile + config
```

## Module List

Clinical: ER, OPD, IPD, LIS, RIS, Pharmacy, MRS, Queueing.
Financial: Billing, PhilHealth, HMO.
Admin: HR, Reports.
Core: Shared.

## Contributing

1. Branch off `main`: `git checkout -b feat/<module>-<task>`
2. Make changes, keep commits small.
3. `make lint && make test` must pass.
4. Open a pull request — CI must be green before review.
5. Follow PSR-12 coding standard (enforced by phpcs).

## License

Proprietary — devnetechn.
```

- [ ] **Step 3: Commit**

```bash
git add Makefile README.md
git commit -m "docs: add Makefile and developer README with setup + commands"
```

---

## Task 12: Phase 0 Smoke Verification

**Goal:** Prove the foundation is complete and ready for Phase 1 by running the full local verification suite.

- [ ] **Step 1: Nuke and rebuild from scratch**

```bash
make fresh
```

Expected: Docker stack rebuilt, composer installed, no errors.

- [ ] **Step 2: Run PHP lint**

```bash
make lint
```

Expected: exits 0 or reports only stock CI4 violations (document which, if any, in the final commit).

- [ ] **Step 3: Run PHPUnit**

```bash
make test
```

Expected: `OK (3 tests, 3 assertions)` at minimum.

- [ ] **Step 4: Run Playwright E2E**

```bash
make e2e
```

Expected: `2 passed` in Playwright summary.

- [ ] **Step 5: Verify homepage in browser**

Open `http://localhost:8080` manually in a browser. Confirm visually:
- Topbar shows "iHomes" and a search box.
- Sidebar shows all 14 module links + Admin.
- Main area shows the Phase 0 landing card with build marker `phase-0-foundation`.

- [ ] **Step 6: Verify Mailhog works**

Open `http://localhost:8025`. Expected: Mailhog UI loads (empty inbox is fine).

- [ ] **Step 7: Tag Phase 0 release**

```bash
git tag -a phase-0-complete -m "Phase 0: foundation infrastructure complete"
git push origin phase-0-complete
```

- [ ] **Step 8: Final commit (if the verification surfaced any fix)**

If any step needed a correction, commit it. Otherwise skip.

---

## Acceptance Criteria

Phase 0 is done when **all** of these are true:

1. `docker compose up -d` brings up nginx, php, mariadb, redis, mailhog, all healthy.
2. `http://localhost:8080/` returns the Phase 0 home page with build marker `phase-0-foundation`.
3. Sidebar shows links for all 14 modules + Admin.
4. `vendor/bin/phpcs` runs without crashing.
5. `vendor/bin/phpunit` passes the 3 environment smoke tests.
6. `npx playwright test` passes the 2 Playwright smoke tests.
7. GitHub Actions CI is green on `main`.
8. Tag `phase-0-complete` pushed to origin.
9. README explains one-command local setup.
10. 14 module folders exist under `app/Modules/` and are registered in `Autoload.php` and `Routes.php`.

## What Phase 0 Does NOT Include

Explicitly deferred to later phases:

- Authentication / RBAC (Phase 1).
- Database schema / migrations for patient, encounter, etc. (Phase 1).
- Any clinical screens beyond the placeholder home page.
- PhilHealth mock server (Phase 2 — added when ER + OECB begins).
- Production server provisioning — this is a separate IT ops track.
- Backup automation — added in Phase 6 before go-live.

## Next Plan

After `phase-0-complete` is tagged:

→ `docs/superpowers/plans/YYYY-MM-DD-phase-1-core-foundation-modules.md` (Weeks 3–6: Shared, Auth, RBAC, Patient MPI, Encounter core).
