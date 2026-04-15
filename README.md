# web_mediadownloader

```
/opt/telegram-bots/ytd_web
│
├── web_ytd.py                # основной backend (FastAPI + логика очереди и скачивания)
├── .env                      # реальные настройки (секреты, пути, лимиты)
├── .env.example              # пример конфигурации (без секретов)
│
├── venv/                     # виртуальное окружение Python
│   └── ...                   # зависимости (fastapi, yt-dlp и т.д.)
│
├── static/                   # фронтенд (JS + CSS)
│   ├── app.js                # логика UI (запросы, статус задач, очередь)
│   └── style.css             # стили (тёмная/светлая тема, UI)
│
├── templates/                # HTML-шаблоны (Jinja2)
│   └── index.html            # основная страница сервиса
│
├── cookies/                  # cookies пользователей (yt-dlp)
│   ├── cookies_u_xxxxx.txt
│   ├── cookies_u_yyyyy.txt
│   └── ...
│
├── data/                     # служебные данные
│   ├── users.json            # пользователи (id, last_seen, cookies и т.д.)
│   ├── sessions.json         # сессии (cookie → user_id)
│   │
│   └── history/              # история скачиваний
│       ├── history_u_xxxxx.json
│       ├── history_u_yyyyy.json
│       └── ...
│
├── logs/                     # логи сервиса
│   └── web_ytd.log
│
└── (внешняя папка)
    /download/                # скачанные файлы (через Angie отдаются)
    ├── a1b2c3d4_1712345678.mp4
    ├── e5f6g7h8_1712345680.mp3
    └── ...
```
