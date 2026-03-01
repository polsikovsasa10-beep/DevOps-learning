```markdown
# Linux: практика

Самый полезный вариант практики — поставить Linux второй системой или в виртуалку и работать в нём регулярно. Все задания ниже предполагают, что у тебя есть доступ по SSH к своему серверу.

---

## 1. Первичная настройка сервера

Цель: превратить «голый» сервер в более‑менее безопасную рабочую машину.

### Шаг 1. Создать отдельного пользователя

```bash
# создать пользователя
sudo adduser devops

# добавить его в группу sudo
sudo usermod -aG sudo devops
```

Проверка:

```bash
su - devops
sudo whoami    # должен вывести root
```

### Шаг 2. Настроить SSH по ключам

На своей машине:

```bash
ssh-keygen -t ed25519 -C "devops-key"
ssh-copy-id devops@server
```

После этого:

```bash
ssh devops@server
```

должно заходить без пароля.

Опционально в `/etc/ssh/sshd_config` на сервере:

```text
PasswordAuthentication no
PermitRootLogin no
```

и затем:

```bash
sudo systemctl reload sshd
```

### Шаг 3. Базовый файрвол (ufw)

```bash
sudo apt update
sudo apt install ufw
```

Правила и включение [web:42][web:45][web:48][web:51]:

```bash
# по умолчанию запрещаем входящие, разрешаем исходящие
sudo ufw default deny incoming
sudo ufw default allow outgoing

# разрешаем SSH (или свой порт, например 2222/tcp)
sudo ufw allow ssh
# sudo ufw allow 2222/tcp

# включаем ufw
sudo ufw enable

# смотрим статус
sudo ufw status verbose
```

После этого на сервер можно зайти по SSH, другие порты по умолчанию закрыты [web:42][web:45][web:48][web:51].

---

## 2. Написать свой systemd‑юнит

Цель: любой «долго живущий» скрипт оформить как сервис, который стартует сам и перезапускается при падении.

### Шаг 1. Скрипт‑сервис

Например, простой HTTP‑сервер на Python:

```bash
mkdir -p /opt/my-http
cat << 'EOF' | sudo tee /opt/my-http/server.sh
#!/bin/bash
set -euo pipefail

cd /opt/my-http
python3 -m http.server 8080
EOF

sudo chmod +x /opt/my-http/server.sh
```

### Шаг 2. Юнит‑файл systemd

```bash
sudo tee /etc/systemd/system/my-http.service > /dev/null << 'EOF'
[Unit]
Description=Simple HTTP server for practice
After=network.target

[Service]
Type=simple
ExecStart=/opt/my-http/server.sh
Restart=on-failure
RestartSec=5
User=www-data
Group=www-data
WorkingDirectory=/opt/my-http

[Install]
WantedBy=multi-user.target
EOF
```

Здесь:

- `Restart=on-failure` и `RestartSec=5` заставляют сервис подниматься после краша [web:43][web:46][web:49].  
- `User`/`Group` запускают его не от root’а [web:46].

### Шаг 3. Запуск и проверка

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-http.service

# статус
sudo systemctl status my-http.service

# логи
sudo journalctl -u my-http.service -f
```

Проверь в браузере `http://server-ip:8080` — должен открыться листинг директории.

Эксперимент: убей процесс руками и посмотри, что systemd его поднимет:

```bash
ps aux | grep server.py
sudo kill -9 <PID>
sudo systemctl status my-http.service
```

---

## 3. Развернуть приложение

Задача: развернуть простое веб‑приложение (frontend + backend + БД или хотя бы backend + статика) на удалённом Linux‑сервере [web:47][web:50].

### Вариант без Docker (минимальный)

Пример на Python (backend) + nginx (отдаёт статику):

1. Установить зависимости:

```bash
sudo apt update
sudo apt install python3 python3-venv nginx
```

2. Залить код на сервер (через `git clone` или `rsync`):

```bash
# на локальной машине
rsync -avz ./myapp/ devops@server:/srv/myapp/
```

3. Настроить виртуальное окружение и запустить backend (любая простая Flask/FastAPI‑апка):

```bash
ssh devops@server
cd /srv/myapp
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

4. Оформить backend как systemd‑сервис по аналогии с `my-http.service` (меняя `ExecStart` и `WorkingDirectory`) [web:46][web:49].

5. Настроить nginx как reverse‑proxy, который принимает HTTP на 80 порту и проксирует на backend (`proxy_pass http://127.0.0.1:8000;`) [web:47][web:50].

### Вариант с Docker (если уже пользуешься)

- Собрать контейнеры для frontend/backend/БД.  
- Описать их в `docker-compose.yml`.  
- На сервере: `docker compose up -d --build` [web:47][web:50].  

В обоих случаях цель одна: всё приложение должно жить на сервере, стартовать одной командой (systemd‑юнит или `docker compose`), а ты должен понимать, где какие логи и конфиги.

---

## 4. Как связывать практику с конспектом и Git

Рекомендуемый рабочий цикл:

1. Выбрал практическую задачу (первичная настройка, systemd‑юнит, деплой).  
2. Выполнил её на реальном сервере.  
3. Зафиксировал шаги, ключевые команды и грабли в `.md` (в соответствующем разделе).  
4. Проверил изменения:  

```bash
git diff
```

5. Зафиксировал в репозитории:

```bash
git add notes/linux/linux_practice.md
git commit -m "Linux: первичная настройка сервера и systemd-юнит"
git push
```

Так у тебя Git становится журналом прокачки: видно, когда ты делал первичную настройку, когда писал юнит, когда впервые задеплоил приложение.

---

## 5. Что делать дальше

- Усложнять юниты: добавлять `Environment`, `Restart=always`, `LimitNOFILE`, зависимость от других сервисов [web:46][web:49].  
- Расширять файрвол: открывать только нужные порты (HTTP/HTTPS, VPN и т.п.) и проверять, что всё остальное закрыто [web:45][web:48][web:51].  
- Наращивать сценарии деплоя: сначала «scp + юнит», дальше — `rsync`, потом Docker и ansible/скрипты поверх этого [web:47][web:50].
```