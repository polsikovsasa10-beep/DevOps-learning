```markdown
# Деплой FastAPI + Nginx + Systemd: полный гайд

**Дата:** 2026-02-28  
**Репозиторий приложения:** https://github.com/polsikovsasa10-beep/App-for-test  
**Результат:** приложение доступно по http://IP_СЕРВЕРА/

Этот файл описывает **полный путь деплоя** FastAPI приложения на Ubuntu через systemd + nginx reverse proxy.

---

## Структура проекта

```
App-for-test/
├── app.py              # FastAPI приложение
├── requirements.txt    # зависимости
├── static/
│   └── index.html      # фронтенд страница
└── README.md
```

---

## 1. Подготовка сервера

```bash
# обновляем систему
sudo apt update && sudo apt upgrade -y

# ставим зависимости
sudo apt install python3 python3-venv python3-pip nginx git ufw -y
```

---

## 2. Деплой приложения

### 2.1 Клонируем репозиторий на сервер

```bash
sudo mkdir -p /srv/myapp
sudo chown $USER:$USER /srv/myapp
cd /srv/myapp

# клонируем твой репозиторий
git clone https://github.com/polsikovsasa10-beep/App-for-test.git .
```

### 2.2 Настраиваем Python окружение

```bash
# создаём venv
python3 -m venv venv
source venv/bin/activate

# ставим зависимости
pip install -r requirements.txt

# проверяем, что всё работает
venv/bin/uvicorn app:app --host 127.0.0.1 --port 8000
```

**Ctrl+C** после проверки.

### 2.3 Права доступа

```bash
sudo chown -R www-data:www-data /srv/myapp
sudo chmod 755 /srv/myapp
```

---

## 3. Systemd сервис

Создаём `/etc/systemd/system/myapp.service`:

```bash
sudo tee /etc/systemd/system/myapp.service > /dev/null << 'EOF'
[Unit]
Description=My FastAPI app
After=network.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/srv/myapp
Environment="PATH=/srv/myapp/venv/bin"
ExecStart=/srv/myapp/venv/bin/uvicorn app:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Запуск:

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service

# проверка
sudo systemctl status myapp.service
curl http://127.0.0.1:8000/
```

---

## 4. Nginx reverse proxy

Создаём `/etc/nginx/sites-available/myapp`:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Активация:

```bash
sudo ln -sf /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5. Файрвол

```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
```

---

## 6. Финальная проверка

1. **На сервере:**
   ```bash
   curl http://127.0.0.1/           # FastAPI напрямую
   curl http://127.0.0.1:80/        # через nginx
   sudo systemctl status myapp.service
   sudo journalctl -u myapp.service -f
   ```

2. **В браузере:** `http://IP_СЕРВЕРА/`

---

## 7. Обновление приложения

```bash
cd /srv/myapp
git pull origin main
/srv/myapp/venv/bin/pip install -r requirements.txt --upgrade
sudo systemctl restart myapp.service
```

---

## 8. Логи и мониторинг

```bash
# логи приложения
sudo journalctl -u myapp.service -f

# логи nginx
sudo tail -f /var/log/nginx/error.log

# процессы
ps aux | grep uvicorn
ss -tulpn | grep 8000
```

---

## 9. Возможные проблемы и решения

| Ошибка | Решение |
|--------|---------|
| `203/EXEC` | Проверить путь в `ExecStart`, права `chmod +x` |
| `200/CHDIR` | Проверить `WorkingDirectory`, права на каталог |
| Nginx дефолтка | `rm /etc/nginx/sites-enabled/default` |
| Не заходит извне | `ufw allow 'Nginx Full'` |

---

## 10. Что изучено

✅ **Git** — клонирование, обновление репозитория  
✅ **Python venv** — изолированное окружение  
✅ **Systemd** — сервисы, автозапуск, рестарт  
✅ **Nginx** — reverse proxy, виртуальные хосты  
✅ **UFW** — файрвол  
✅ **Логирование** — `journalctl`, nginx логи  
✅ **Диагностика** — `ps`, `ss`, `curl`

**Время на выполнение:** ~2 часа  
**Результат:** production-ready деплой FastAPI приложения
```