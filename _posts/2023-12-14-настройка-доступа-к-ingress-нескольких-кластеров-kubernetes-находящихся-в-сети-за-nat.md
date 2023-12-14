---
title: "Настройка доступа к Ingress нескольких кластеров Kubernetes, находящихся в сети за NAT"
date: "2023-12-14 15:39:00 +0400"
---

Есть у меня две мелких инсталляции кубера на домашнем сервере. Использую их для тестирования различных пайплайнов деплоя разрабатываемых приложений. Один кубер условно "тестовый", другой условно "продакшен". Нужно сделать приложения, запущенные в этих кубах, доступными из интернетов по https с минимальными усилиями по настройке tls с моей стороны.

Disclaimer: настройки сети, пробросы портов и т.д. оставлю за кадром.

## Nginx to the rescue

Для проброса TCP-соединений буду использовать модуль stream. В этом случае мне не придётся настраивать терминацию SSL на Nginx.

### Установка модуля stream

```bash
sudo apt-get install libnginx-mod-stream
```

Подразумевается, что сам Nginx уже установлен и кеш пакетов apt свежий.

### Настройка модуля stream

Добавляю в `/etc/nginx/nginx.conf` секцию stream для **трафика с TLS**:

```bash
stream {
    map_hash_bucket_size 64;

    map $ssl_preread_server_name $targetBackend {
        hostnames;

        *.domain1.tld   k8s1;
        domain1.tld     k8s1;

        *.domain2.tld   k8s2;
        domain2.tld     k8s2;

        default         local;
    }

    upstream k8s1 {
        server 10.0.0.2:443;
    }

    upstream k8s2 {
        server 10.0.0.3:443;
    }

    upstream local {
        server 127.0.0.1:8443;
    }

    server {
        listen 0.0.0.0:443;

        ssl_preread on;

        proxy_pass $targetBackend;
    }
}
```

Где `upstream k8s1` - адрес ингресса первого кубера, `upstream k8s2` - адрес ингресса второго кубера, `upstream local` - локальный сервер nginx.

Для того, чтобы корректно перенаправить SSL-трафик на разные upstream, использую параметр `ssl_preread on`, позволяющий получить FQDN из SNI и записать его в переменную `$ssl_preread_server_name`. Далее, на основе заданного маппинга сопоставляем `$ssl_preread_server_name` с соответствующим `$targetBackend`.

Локальный nginx придётся перенастроить чтобы он слушал обращения к сайтам с TLS на <https://127.0.0.1:8443>.

Перезапускаю nginx после изменения конфигурации для применения новых настроек.

```bash
systemctl restart nginx
```

### Дополнительная настройка nginx для корректной работы Cert Manager

Cert Manager используется для автоматизации получения SSL-сертификатов от Let's Encrypt в Kubernetes. Создаём для Nginx по конфигу сайта под каждый кубер, чтобы туда мог подключиться бот Letsencrypt, проверяющий адреса. Приведена минимальная конфигурация, дополняйте по обстоятельствам.

`/etc/nginx/sites-enabled/k8s1`:

```bash
# HTTP
server {
    listen      80;

    server_name *.domain1.tld domain1.tld;

    location    / {
        allow all;

        proxy_pass http://10.0.0.2:80;
        proxy_pass_header Set-Cookie;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

`/etc/nginx/sites-enabled/k8s2`:

```bash
# HTTP
server {
    listen      80;

    server_name *.domain2.tld domain2.tld;

    location    / {
        allow all;

        proxy_pass http://10.0.0.3:80;
        proxy_pass_header Set-Cookie;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

## Бонусы

### Перенаправление https на https

`/etc/nginx/sites-enabled/http2https`:

```bash
server {
    listen      80 default_server;

    return      301 https://$host$request_uri;
}
```

### Сайт-заглушка

`/etc/nginx/sites-enabled/landing`:

```bash
server {
    listen      127.0.0.1:8443 ssl http2 default_server;

    root        /srv/landing;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
