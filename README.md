# Web YTD — закрытый веб‑сервис для скачивания медиа

Web YTD — это приватный веб‑сервис на FastAPI для скачивания видео и аудио по ссылке через `yt-dlp`, с очередью задач, пользовательскими и общими `cookies.txt`, одноразовыми ссылками доступа, SQLite и обратным прокси через Angie.

## Возможности

- анализ ссылки перед скачиванием;
- скачивание в нескольких режимах: лучшее качество, любой формат, MP3, конкретный формат;
- очередь задач и ограничение количества одновременных скачиваний;
- история скачиваний по пользователям;
- личные `cookies.txt` пользователей и общий `cookies.txt` администратора;
- отдельный вход по постоянной ссылке и по одноразовым ссылкам;
- ручные метки/имена для одноразовых ссылок доступа;
- отзыв доступа администратором;
- отслеживание активности пользователей за последние 5 и 10 минут;
- хранение пользователей, сессий, истории и доступов в SQLite;
- отдача готовых файлов через Angie из `/download`;
- автоматическая очистка старых файлов из `/download`;
- автоматическое ночное обновление `yt-dlp` и `yt-dlp-ejs`.

## Что есть в репозитории

- `web_ytd.py` — основной backend;
- `templates/` — HTML‑шаблоны;
- `static/` — JS и CSS;
- `.env.example` — понятный пример конфигурации;
- `requirements.txt` — Python‑зависимости с фиксированными версиями;
- `install-versions.env` — переменные для автоустановки;
- `scripts/install.sh` — готовый bash‑скрипт автоустановки;
- `scripts/ytd_db.sh` — готовый bash‑скрипт для backup / restore / проверки SQLite;
- `deploy/systemd/ytd_web.service` — пример systemd‑службы;
- `deploy/angie/site.conf.example` — пример конфига сайта для Angie;
- `deploy/angie/download_filename_map.conf` — `map` для корректного имени скачиваемого файла;
- `deploy/cron/crontab.example` — готовые строки для cron.

---

## Минимальные требования

- Ubuntu Server 24.04 LTS;
- домен с A‑записью на IP сервера;
- доступ по SSH с `sudo`;
- открытые порты `22`, `80`, `443`;
- отдельный пользователь для запуска сервиса, например `botrunner`.

Ubuntu 24.04 по умолчанию использует Python 3.12. Angie рекомендуется ставить из официального репозитория Angie для Ubuntu. Для `yt-dlp-ejs` нужен доступный JS runtime, и в проекте для этого используется `node`.

---

## Быстрый вариант установки через автоустановщик

### 1. Клонируй репозиторий

```bash
git clone https://github.com/USERNAME/REPO.git /opt/telegram-bots/ytd_web-src
cd /opt/telegram-bots/ytd_web-src
```

### 2. Запусти автоустановщик

```bash
sudo bash scripts/install.sh
```

Скрипт:

- проверит Ubuntu и права `root`;
- установит пакеты ОС;
- добавит официальный репозиторий Angie и установит Angie;
- создаст пользователя `botrunner`, нужные каталоги и `/download`;
- скопирует проект в `/opt/telegram-bots/ytd_web`;
- создаст виртуальное окружение;
- установит Python‑зависимости из `requirements.txt`;
- создаст `.env`, если его ещё нет;
- подготовит конфиг Angie и `map` для выдачи файлов;
- создаст и включит systemd‑службу `ytd_web`;
- включит UFW и откроет только `22`, `80`, `443`;
- поставит cron‑задания:
  - обновление `yt-dlp` и `yt-dlp-ejs` в `04:00` по времени сервера;
  - удаление файлов из `/download` старше 30 минут каждые 5 минут.

`systemd` поддерживает автоматический перезапуск сервисов, в том числе режим `Restart=on-failure`, который и используется в примере службы.

### 3. После завершения

Проверь:

```bash
sudo systemctl status ytd_web --no-pager
sudo systemctl status angie --no-pager
sudo ufw status verbose
```

Открой в браузере:

- постоянную ссылку пользователя: `https://ТВОЙ_ДОМЕН/СКРЫТЫЙ_ПУТЬ/login?key=...`
- постоянную ссылку администратора: `https://ТВОЙ_ДОМЕН/СКРЫТЫЙ_ПУТЬ/login?key=...`

> Сам сервис живёт не в корне домена, а под `WEB_BASE_PATH`, поэтому открытие просто `https://example.com/` может не показывать интерфейс — это нормально.

---

## Полная ручная установка с нуля

Ниже тот же путь, но вручную, пошагово и без магии.

## 1. Подготовка DNS и сервера

Сделай A‑запись домена на IP сервера и дождись, пока домен начнёт открываться на этот IP.

Обнови пакеты:

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 2. Установка пакетов ОС

```bash
sudo apt update
sudo apt install -y ca-certificates curl unzip ffmpeg nodejs python3 python3-venv python3-pip sqlite3 cron ufw rsync
```

Что это даёт:

- `python3`, `python3-venv`, `python3-pip` — Python и виртуальное окружение;
- `ffmpeg` — нужен `yt-dlp` для постобработки медиа;
- `nodejs` — JS runtime для `yt-dlp-ejs` и YouTube;
- `sqlite3` — работа с SQLite и обслуживание БД;
- `cron` — фоновые задания по расписанию;
- `ufw` — firewall;
- `rsync` — безопасное копирование проекта при автоустановке.

---

## 3. Установка Angie из официального репозитория

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo curl -o /etc/apt/trusted.gpg.d/angie-signing.gpg https://angie.software/keys/angie-signing.gpg

echo "deb https://download.angie.software/angie/$(. /etc/os-release && echo "$ID/$VERSION_ID $VERSION_CODENAME") main" \
| sudo tee /etc/apt/sources.list.d/angie.list > /dev/null

sudo apt-get update
sudo apt-get install -y angie
sudo systemctl enable --now angie
sudo systemctl status angie --no-pager
```

Это официальный способ установки Angie OSS через их репозиторий для Debian/Ubuntu.

---

## 4. Настройка UFW

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

Порт `8093` наружу открывать не нужно: FastAPI слушает только `127.0.0.1`.

---

## 5. Создание пользователя и каталогов

```bash
sudo adduser --home /opt/telegram-bots --shell /bin/bash --disabled-password --gecos "" botrunner
sudo mkdir -p /opt/telegram-bots/ytd_web
sudo mkdir -p /download
sudo mkdir -p /opt/telegram-bots/ytd_web/{static,templates,cookies,data,logs,deploy,backups,scripts}
sudo chown -R botrunner:botrunner /opt/telegram-bots
sudo chown -R botrunner:botrunner /download
```

---

## 6. Копирование файлов проекта

Скопируй в `/opt/telegram-bots/ytd_web`:

- `web_ytd.py`
- `requirements.txt`
- `.env.example`
- `templates/index.html`
- `templates/logged_out.html`
- `static/app.js`
- `static/style.css`
- `deploy/systemd/ytd_web.service`
- `deploy/angie/site.conf.example`
- `deploy/angie/download_filename_map.conf`
- `deploy/cron/crontab.example`
- `scripts/ytd_db.sh`

Проверь владельца:

```bash
sudo chown -R botrunner:botrunner /opt/telegram-bots/ytd_web
```

---

## 7. Создание виртуального окружения и установка Python‑зависимостей

```bash
sudo -u botrunner -H python3 -m venv /opt/telegram-bots/venv
sudo -u botrunner -H bash -lc 'source /opt/telegram-bots/venv/bin/activate && pip install --upgrade pip setuptools wheel && pip install -r /opt/telegram-bots/ytd_web/requirements.txt'
```

---

## 8. Создание `.env`

Используй корневой `.env.example`. Именно такой формат выбран специально, потому что он проще для новичков: параметры сгруппированы по смыслу, важные вещи отмечены комментариями, и сразу видно, что надо менять вручную.

```bash
sudo -u botrunner cp /opt/telegram-bots/ytd_web/.env.example /opt/telegram-bots/ytd_web/.env
sudo -u botrunner nano /opt/telegram-bots/ytd_web/.env
```

Минимально нужно изменить:

- `WEB_SECRET_KEY`
- `WEB_LOGIN_KEY`
- `WEB_ADMIN_LOGIN_KEY`
- `WEB_BASE_PATH`
- `WEB_PUBLIC_BASE_URL`
- `DOWNLOAD_BASE_URL`

Пример:

```env
WEB_HOST=127.0.0.1
WEB_PORT=8093
WEB_BASE_PATH=/d5f4sdv21sdg4
WEB_PUBLIC_BASE_URL=https://example.com
DOWNLOAD_BASE_URL=https://example.com/files
```

---

## 9. Настройка Angie под сервис

### 9.1. `map` для красивого имени файла

Добавь `download_filename_map.conf` в общий `http {}` контекст Angie:

```bash
sudo cp /opt/telegram-bots/ytd_web/deploy/angie/download_filename_map.conf /etc/angie/conf.d/download_filename_map.conf
```

### 9.2. Конфиг сайта

```bash
sudo cp /opt/telegram-bots/ytd_web/deploy/angie/site.conf.example /etc/angie/http.d/example.com.conf
sudo nano /etc/angie/http.d/example.com.conf
```

Замени:

- `__APP_DOMAIN__` → свой домен;
- `__WEB_BASE_PATH__` → тот же путь, что в `.env`, например `/d5f4sdv21sdg4`.

### 9.3. Проверка и перезагрузка Angie

```bash
sudo angie -t
sudo systemctl reload angie
```

---

## 10. TLS‑сертификат через Angie ACME

В примере конфига уже используется встроенный ACME‑механизм Angie:

```nginx
acme le;
ssl_certificate      $acme_cert_le;
ssl_certificate_key  $acme_cert_key_le;
```

Для этого нужно:

- чтобы домен уже смотрел на IP сервера;
- чтобы порт `80` был доступен снаружи;
- чтобы конфиг сайта был корректным и успешно проходил `angie -t`.

После `systemctl reload angie` Angie сможет получить и дальше продлевать сертификат автоматически, если домен и HTTP доступны. Подробные инструкции по установке и пакетам Angie есть в официальной документации.

---

## 11. Создание службы systemd

Скопируй пример службы:

```bash
sudo cp /opt/telegram-bots/ytd_web/deploy/systemd/ytd_web.service /etc/systemd/system/ytd_web.service
sudo systemctl daemon-reload
sudo systemctl enable --now ytd_web
sudo systemctl status ytd_web --no-pager
```

Логи:

```bash
sudo journalctl -u ytd_web -n 200 --no-pager
```

---

## 12. Cron: автообновление `yt-dlp` и очистка `/download`

### 12.1. Удаление файлов из `/download` старше 30 минут

```cron
*/5 * * * * find /download -type f -mmin +30 -delete
```

### 12.2. Ночное обновление `yt-dlp` и `yt-dlp-ejs` в 04:00 по времени сервера

```cron
0 4 * * * before=$(/usr/bin/sudo -u botrunner -H bash -lc 'source /opt/telegram-bots/venv/bin/activate && python -c "import importlib.metadata as m; print(\"yt-dlp=\"+m.version(\"yt-dlp\") if \"yt-dlp\" in m.packages_distributions() else \"yt-dlp=NOT_INSTALLED\"); print(\"yt-dlp-ejs=\"+m.version(\"yt-dlp-ejs\") if \"yt-dlp-ejs\" in m.packages_distributions() else \"yt-dlp-ejs=NOT_INSTALLED\")"'); /usr/bin/sudo -u botrunner -H bash -lc 'source /opt/telegram-bots/venv/bin/activate && pip install -U --no-deps yt-dlp yt-dlp-ejs'; after=$(/usr/bin/sudo -u botrunner -H bash -lc 'source /opt/telegram-bots/venv/bin/activate && python -c "import importlib.metadata as m; print(\"yt-dlp=\"+m.version(\"yt-dlp\") if \"yt-dlp\" in m.packages_distributions() else \"yt-dlp=NOT_INSTALLED\"); print(\"yt-dlp-ejs=\"+m.version(\"yt-dlp-ejs\") if \"yt-dlp-ejs\" in m.packages_distributions() else \"yt-dlp-ejs=NOT_INSTALLED\")"'); [ "$before" != "$after" ] && /usr/bin/systemctl restart ytd_web || true
```

Для `root` это можно добавить так:

```bash
sudo crontab -e
```

или использовать готовый файл `deploy/cron/crontab.example`.

---

## 13. Первый запуск и проверка

Проверь, что сервис поднялся:

```bash
curl -I http://127.0.0.1:8093
sudo systemctl status ytd_web --no-pager
sudo systemctl status angie --no-pager
```

Проверь, что приложение отдаётся через внешний URL:

```bash
curl -I https://example.com/ТВОЙ_WEB_BASE_PATH/
```

---

## 14. Резервное копирование, восстановление и проверка SQLite

Используй готовый скрипт:

```bash
sudo bash /opt/telegram-bots/ytd_web/scripts/ytd_db.sh status
sudo bash /opt/telegram-bots/ytd_web/scripts/ytd_db.sh backup
sudo bash /opt/telegram-bots/ytd_web/scripts/ytd_db.sh quick-check
sudo bash /opt/telegram-bots/ytd_web/scripts/ytd_db.sh integrity-check
```

Восстановление из backup:

```bash
sudo bash /opt/telegram-bots/ytd_web/scripts/ytd_db.sh restore /opt/telegram-bots/ytd_web/backups/db/backup_YYYY-MM-DD_HH-MM-SS.sqlite3
```

Скрипт перед каждой операцией показывает:

- путь к БД;
- размер БД;
- статус службы;
- сколько неадминских пользователей было активно за последние 5 и 10 минут;
- сколько неистёкших сессий сейчас есть в БД.

Для live‑backup SQLite лучше использовать штатные механизмы вроде backup API или `VACUUM INTO`, а не просто копировать файл работающей БД произвольным `cp`. Именно поэтому в скрипте backup используется штатная команда `.backup` через `sqlite3`.

---

## 15. Важные замечания

- `.env` не коммить в Git.
- `WEB_SECRET_KEY`, `WEB_LOGIN_KEY`, `WEB_ADMIN_LOGIN_KEY` обязательно меняй на свои.
- `WEB_BASE_PATH` должен совпадать в `.env` и в конфиге Angie символ в символ.
- наружу открываются только `80/443`, а не `8093`.
- если сервис работает не в корне домена, открывать нужно именно путь `WEB_BASE_PATH`.
- если используешь Cloudflare или другой прокси перед сервером, сначала добейся нормального прямого HTTPS до сервера, а потом уже включай проксирование.

---

## 16. Что ещё можно добавить позже

- Telegram‑уведомления о сбоях службы;
- отдельный скрипт ротации логов;
- отдельный админский backup‑скрипт не только для БД, но и для `cookies/`, `.env` и Angie‑конфигов;
- fail2ban поверх SSH/Angie;
- отдельный test‑скрипт после деплоя.

