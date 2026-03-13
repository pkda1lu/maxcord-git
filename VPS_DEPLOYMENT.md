# Развертывание серверной части на Ubuntu VPS

Полное руководство по настройке серверной части Maxcord на Ubuntu VPS.

## Требования

- Ubuntu 20.04 или выше
- Root доступ или пользователь с sudo правами
- Минимум 1GB RAM (рекомендуется 2GB+)
- Открытые порты: 22 (SSH), 80 (HTTP), 443 (HTTPS), 5000 (API), 7880 (LiveKit), 50000-60000 (UDP для WebRTC)

## Шаг 1: Подключение к серверу

```bash
ssh root@your-server-ip
# или
ssh username@your-server-ip
```

## Шаг 2: Обновление системы

```bash
sudo apt update
sudo apt upgrade -y
```

## Шаг 3: Установка Node.js

### Вариант 1: Через NodeSource (рекомендуется, последняя LTS версия)

```bash
# Установка Node.js 20.x LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Проверка версии
node --version
npm --version
```

### Вариант 2: Через nvm (Node Version Manager)

```bash
# Установка nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Перезагрузка терминала или выполните:
source ~/.bashrc

# Установка Node.js
nvm install 20
nvm use 20
nvm alias default 20
```

## Шаг 4: Установка MongoDB

### Вариант 1: MongoDB Community Edition (локально)

```bash
# Импорт ключа MongoDB
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Добавление репозитория
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Обновление и установка
sudo apt update
sudo apt install -y mongodb-org

# Запуск и автозапуск MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod

# Проверка статуса
sudo systemctl status mongod
```

### Вариант 2: MongoDB Atlas (облачный, рекомендуется для продакшена)

1. Перейдите на https://www.mongodb.com/cloud/atlas
2. Создайте бесплатный аккаунт
3. Создайте кластер (бесплатный tier M0)
4. Получите connection string
5. Добавьте IP вашего VPS в whitelist MongoDB Atlas

## Шаг 5: Установка PM2 (менеджер процессов)

```bash
sudo npm install -g pm2
```

## Шаг 6: Клонирование и настройка проекта

```bash
# Создание директории для приложения
sudo mkdir -p /var/www
cd /var/www

# Если используете Git:
# git clone your-repository-url zvon
# cd zvon

# Или загрузите файлы через SCP/SFTP в /var/www/zvon

# Переход в директорию сервера
cd zvon/server

# Установка зависимостей
npm install --production
```

## Шаг 7: Настройка переменных окружения

```bash
# Создание файла .env
nano /var/www/maxcord/server/.env
```

Добавьте следующие переменные:

```env
# Порт сервера
PORT=5000

# MongoDB URI
# Для локального MongoDB:
MONGODB_URI=mongodb://localhost:27017/zvon
# Для MongoDB Atlas:
# MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/zvon?retryWrites=true&w=majority

# JWT Secret (ОБЯЗАТЕЛЬНО измените на случайную строку!)
JWT_SECRET=68f686e0d79b3413453b5b8efa07ae4

# URL клиента (ваш домен или IP)
CLIENT_URL=https://maxcord.fun
# или для разработки:
# CLIENT_URL=http://localhost:3000

# Окружение
NODE_ENV=production

# LiveKit
LIVEKIT_API_KEY=APIkQWmfsaTEiSd
LIVEKIT_API_SECRET=CCbglNtxNW9rDnyxYBHMe4XmmfS3mCx3iwBgVqaQfMfB
LIVEKIT_URL=wss://livekit.maxcord.fun
```

**Важно:** Сгенерируйте безопасный JWT_SECRET:
```bash
openssl rand -base64 32
```

## Шаг 8: Создание директории для загрузок

```bash
mkdir -p /var/www/maxcord-git/server/uploads
chmod 755 /var/www/maxcord-git/server/uploads
```

## Шаг 9: Запуск приложения через PM2

```bash
cd /var/www/maxcord-git/server

# Запуск приложения
pm2 start server.js --name maxcord-server

# Сохранение конфигурации PM2 для автозапуска
pm2 save
pm2 startup

# Проверка статуса
pm2 status
pm2 logs maxcord-server
```

## Шаг 10: Установка LiveKit Server (SFU)

Для работы голосовой и видеосвязи используется LiveKit. Рекомендуется установка через Docker.

### 10.1 Установка Docker
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```

### 10.2 Настройка LiveKit
Создайте директорию для конфигурации и файл `livekit.yaml`:
```bash
sudo mkdir -p /opt/livekit
sudo nano /opt/livekit/livekit.yaml
```

Вставьте следующую конфигурацию (используйте ключи из вашего `.env` файла):
```yaml
port: 7880
rtc:
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true
keys:
  APIkQWmfsaTEiSd: CCbglNtxNW9rDnyxYBHMe4XmmfS3mCx3iwBgVqaQfMfB
```

### 10.3 Запуск через Docker Compose
Создайте `docker-compose.yaml`:
```bash
sudo nano /opt/livekit/docker-compose.yaml
```

```yaml
version: '3'
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /livekit.yaml
    restart: always
    network_mode: "host"
    volumes:
      - ./livekit.yaml:/livekit.yaml
```

Запустите сервер:
```bash
cd /opt/livekit
sudo docker-compose up -d
```

## Шаг 11: Настройка Nginx (Reverse Proxy)

### Установка Nginx

```bash
sudo apt install -y nginx
```

### Создание конфигурации

```bash
sudo nano /etc/nginx/sites-available/zvon
```

Добавьте следующую конфигурацию:

```nginx
# ВНИМАНИЕ: На этом этапе мы создаем ВРЕМЕННЫЙ конфиг только для HTTP.
# Это нужно, чтобы Nginx запустился и Certbot смог выпустить сертификат.

server {
    listen 80;
    server_name maxcord.fun;

    client_max_body_size 50M;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

### Шаг 12: Настройка SSL (Certbot)

Теперь, когда Nginx работает на 80 порту, выпускаем сертификат:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d maxcord.fun
```

### Шаг 13: Финальная настройка Nginx (API + LiveKit + SSL)

После того как Certbot закончит работу, он сам изменит файл `/etc/nginx/sites-available/zvon`. Вам нужно будет **отредактировать его еще раз**, чтобы добавить специфические настройки для LiveKit и статики.

Откройте файл: `sudo nano /etc/nginx/sites-available/zvon`
Приведите его к такому виду (замените всё содержимое):

```nginx
server {
    listen 443 ssl http2;
    server_name maxcord.fun;

    # Эти пути Certbot пропишет сам, убедитесь что они верные
    ssl_certificate     /etc/letsencrypt/live/maxcord.fun/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/maxcord.fun/privkey.pem;

    client_max_body_size 50M;

    # Статика вложений с CORS
    location /api/uploads/ {
        alias /var/www/zvon/server/uploads/;
        add_header Access-Control-Allow-Origin "*" always;
        add_header Cache-Control "public, immutable";
        expires 30d;
        try_files $uri $uri/ =404;
    }

    # LiveKit WebSockets (SFU)
    location ~* /rtc$|/rtc/ {
        proxy_pass http://localhost:7880;
        proxy_http_version 1.1;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Основной API и React Route
    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name maxcord.fun;
    return 301 https://$host$request_uri;
}
```

После редактирования:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Активация конфигурации

```bash
# Создание символической ссылки
sudo ln -s /etc/nginx/sites-available/zvon /etc/nginx/sites-enabled/

# Удаление дефолтной конфигурации (опционально)
sudo rm /etc/nginx/sites-enabled/default

# Проверка конфигурации
sudo nginx -t

# Перезагрузка Nginx
sudo systemctl reload nginx
sudo systemctl enable nginx
```

## Шаг 11: Настройка Firewall (UFW)

```bash
# Разрешение SSH (важно сделать первым!)
sudo ufw allow 22/tcp

# Разрешение HTTP и HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Разрешение портов LiveKit
sudo ufw allow 7880/tcp
sudo ufw allow 50000:60000/udp

# Включение firewall
sudo ufw enable

# Проверка статуса
sudo ufw status
```

## Шаг 12: Настройка SSL (Let's Encrypt) - опционально, но рекомендуется

```bash
# Установка Certbot
sudo apt install -y certbot python3-certbot-nginx

# Получение SSL сертификата
sudo certbot --nginx -d maxcord.fun

# Автоматическое обновление (настроено автоматически)
sudo certbot renew --dry-run
```

После этого Certbot автоматически обновит конфигурацию Nginx для использования HTTPS.

## Шаг 13: Обновление переменных окружения для HTTPS

Если вы настроили SSL, обновите `.env`:

```bash
nano /var/www/zvon/server/.env
```

Измените `CLIENT_URL`:
```env
CLIENT_URL=https://maxcord.fun
```

Перезапустите приложение:
```bash
pm2 restart maxcord-server
```

## Полезные команды PM2

```bash
# Просмотр логов
pm2 logs zvon-server

# Просмотр статуса
pm2 status

# Перезапуск
pm2 restart zvon-server

# Остановка
pm2 stop zvon-server

# Удаление из PM2
pm2 delete zvon-server

# Мониторинг
pm2 monit
```

## Обновление приложения

```bash
cd /var/www/zvon/server

# Если используете Git:
git pull origin main

# Установка новых зависимостей
npm install --production

# Перезапуск
pm2 restart zvon-server
```

## Проверка работы

1. Проверьте статус PM2: `pm2 status`
2. Проверьте логи: `pm2 logs zvon-server`
3. Проверьте Nginx: `sudo systemctl status nginx`
4. Проверьте MongoDB: `sudo systemctl status mongod`
5. Откройте в браузере: `http://maxcord.fun` или `http://your-server-ip`

## Решение проблем

### Приложение не запускается

```bash
# Проверьте логи
pm2 logs zvon-server --lines 50

# Проверьте переменные окружения
cat /var/www/zvon/server/.env

# Проверьте подключение к MongoDB
mongosh
# или
mongo
```

### MongoDB не подключается

```bash
# Проверьте статус
sudo systemctl status mongod

# Проверьте логи
sudo journalctl -u mongod -n 50

# Перезапустите MongoDB
sudo systemctl restart mongod
```

### Nginx не работает

```bash
# Проверьте конфигурацию
sudo nginx -t

# Проверьте логи
sudo tail -f /var/log/nginx/error.log

# Перезапустите Nginx
sudo systemctl restart nginx
```

### Порт занят

```bash
# Проверьте, что использует порт 5000
sudo lsof -i :5000
# или
sudo netstat -tulpn | grep 5000

# Убейте процесс или измените порт в .env
```

### Проблемы с правами доступа

```bash
# Установите правильные права
sudo chown -R $USER:$USER /var/www/zvon
sudo chmod -R 755 /var/www/zvon
```

## Резервное копирование

### Резервное копирование MongoDB

```bash
# Создание бэкапа
mongodump --out /var/backups/mongodb/$(date +%Y%m%d)

# Восстановление
mongorestore /var/backups/mongodb/YYYYMMDD
```

### Резервное копирование загрузок

```bash
# Создание архива
tar -czf /var/backups/zvon/uploads-$(date +%Y%m%d).tar.gz /var/www/zvon/server/uploads
```

## Мониторинг

### Установка мониторинга PM2

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
```

### Мониторинг ресурсов

```bash
# Использование памяти и CPU
pm2 monit

# Системные ресурсы
htop
# или
top
```

## Дополнительные настройки безопасности

1. **Отключение root SSH** (если еще не сделано):
```bash
sudo nano /etc/ssh/sshd_config
# Установите: PermitRootLogin no
sudo systemctl restart sshd
```

2. **Настройка fail2ban** (защита от брутфорса):
```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

3. **Регулярные обновления**:
```bash
# Настройка автоматических обновлений безопасности
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Готово!

Ваш сервер должен быть доступен по адресу:
- HTTP: `http://livekit.maxcord.fun` или `http://your-server-ip`
- HTTPS: `https://livekit.maxcord.fun` (если настроен SSL)

API будет доступно на том же адресе, например:
- `http://livekit.maxcord.fun/api/auth/register`
- `http://livekit.maxcord.fun/api/servers`



