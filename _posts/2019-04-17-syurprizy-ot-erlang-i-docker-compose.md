---
title: "Сюрпризы от Erlang и Docker-compose"
date: "2019-04-17 10:12:34 +0300"
---

<!-- excerpt -->

## Преамбула

Есть такая интересная штука, как RabbitMQ. Написана на Erlang (это важно). В Docker запускается с пол-пинка, в кластер собирается тремя с половиной командами. Это если всё делать вручную по [инструкции](https://hub.docker.com/r/zimniy/rabbitmq). Сразу скажу, корректные FQDN серверов и контейнеров — обязательное условие работоспособности.

Запускаем на первом сервере первую ноду будущего кластера:

```shell
docker run -dit -h docker-rabbitmq-01.domain.tld               \
           --name rabbit                                       \
           -p "4369:4369"                                      \
           -p "5671:5671"                                      \
           -p "15671:15671"                                    \
           -p "25672:25672"                                    \
           -p "35197:35197"                                    \
           -e "RABBITMQ_ERLANG_COOKIE=S0meL3ttersAndNumber5"   \
           -v rabbitmq_data:/var/lib/rabbitmq                  \
           -v /opt/ssl:/opt/ssl:ro                             \
           --restart unless-stopped                            \
           zimniy/rabbitmq:latest
```

Сразу определяем политику HA (запускаем также на первом сервере):

```shell
docker exec rabbit rabbitmqctl set_policy ha-all ".*" '{"ha-mode":"all", "ha-sync-mode": "automatic"}'
```

Запускаем на втором сервере вторую ноду будущего кластера:

```shell
docker run -dit -h docker-rabbitmq-02.domain.tld               \
           --name rabbit                                       \
           -p "4369:4369"                                      \
           -p "5671:5671"                                      \
           -p "15671:15671"                                    \
           -p "25672:25672"                                    \
           -p "35197:35197"                                    \
           -e "RABBITMQ_ERLANG_COOKIE=S0meL3ttersAndNumber5"   \
           -v rabbitmq_data:/var/lib/rabbitmq                  \
           -v /opt/ssl:/opt/ssl:ro                             \
           --restart unless-stopped                            \
           zimniy/rabbitmq:latest
```

Со второго сервера подключаем вторую ноду к первой:

```shell
docker exec rabbit rabbitmqctl stop_app
docker exec rabbit rabbitmqctl join_cluster rabbit@docker-rabbitmq-01.domain.tld
docker exec rabbit rabbitmqctl start_app
```

Всё, кластер собран и уже работает. Можно подключаться с учёткой guest:guest.

## Фабула

Короче, теперь про то, какая гадость этот docker-compose…

Оказывается, у него есть два параметра: hostname и domainname.

И если задать только hostname как FQDN «docker-rabbitmq-01.domain.tld», то оно в контейнере обрезается до «docker-rabbitmq-01». Так что нужно обязательно задать ещё и domainname «domain.tld» иначе Erlang, на котором написан RabbitMQ, свой FQDN не определит и будет игнорировать попытки подключения.

В общем, используя docker-compose, нужно делать так:

```docker
docker-rabbitmq:
  image: zimniy/rabbitmq:latest
  container_name: rabbit
  hostname: docker-rabbitmq-01.domain.tld 
  domainname: domain.tld
  restart: unless-stopped
  ports:
    - "4369:4369"
    - "5671:5671"
    - "15671:15671"
    - "25672:25672"
    - "35197:35197"
  volumes:
    - "/opt/ssl:/opt/ssl:ro"
    - "rabbitmq_data:/var/lib/rabbitmq"
```

Внимание, hostname задать именно как FQDN.

И, для того, чтобы это выяснить, пришлось аж в исходники лезть…
