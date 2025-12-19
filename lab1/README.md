# Лабораторная работа №1

---

## Postgres Cluster

**Дисциплина:** Администрирование компьютерных сетей  
**Факультет:** Программной инженерии и компьютерной техники  
**Университет:** ИТМО

**Студент:** Тутубалин Кирилл, Москалец Данила, Захматов Юрий, Джафари Хоссаин, Мохаджер Али Реза  
**Группа:** К3340-К3341  
**Преподаватель:** Самохин Никита Юрьевич 

**Санкт-Петербург, 2025 г.**

---

**Задача**:
Развернуть и настроить высокодоступный кластер Postgres

## Часть 1, Поднимаем Postgres

Для начала сделаем Dockerfile: 

```dockerfile
FROM postgres:15

RUN apt-get update -y && \
 apt-get install -y netcat-openbsd python3-pip curl python3-psycopg2 python3-venv iputils-ping

RUN python3 -m venv /opt/patroni-venv && \
 /opt/patroni-venv/bin/pip install --upgrade pip && \
 /opt/patroni-venv/bin/pip install patroni[zookeeper] psycopg2-binary

COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml

ENV PATH="/opt/patroni-venv/bin:$PATH"

USER postgres
```

Затем, сделаем compose файл, добавим в него наши ноды и Zookeeper:

```yml
services:
  pg-master:
    build: .
    image: localhost/postgres:patroni
    container_name: pg-master
    restart: always
    hostname: pg-master
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PGDATA: '/var/lib/postgresql/data/pgdata'
    expose:
      - 8008
    ports:
      - 5433:5432
    volumes:
      - pg-master:/var/lib/postgresql/data
    command: patroni /postgres0.yml
  pg-slave:
    build: .
    image: localhost/postgres:patroni
    container_name: pg-slave
    restart: always
    hostname: pg-slave
    expose:
      - 8008
    ports:
      - 5434:5432
    volumes:
      - pg-slave:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PGDATA: '/var/lib/postgresql/data/pgdata'
    command: patroni /postgres1.yml
  zoo:
    image: confluentinc/cp-zookeeper:7.7.1
    container_name: zoo
    restart: always
    hostname: zoo
    ports:
      - 2181:2181
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  haproxy:
    image: haproxy:3.0
    container_name: postgres_entrypoint # Это будет адрес подключения к БД, можно выбрать любой
    ports:
      - 5435:5432 # Это будет порт подключения к БД, можно выбрать любой
      - 7000:7000
    depends_on: # Не забываем убедиться, что сначала все корректно поднялось
      - pg-master
      - pg-slave
      - zoo
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg

volumes:
  pg-master:
  pg-slave:

```

Далее напишем файлы сервисов (postgres0.yml, postgres1.yml):

```yml
scope: my_cluster # Имя нашего кластера
name: postgresql0 # Имя первой ноды
restapi: # Адреса первой ноды
  listen: pg-master:8008
  connect_address: pg-master:8008
zookeeper:
  hosts:
    - zoo:2181 # Адрес Zookeeper
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 10485760
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f
  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5
postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-master:5432 # Адрес первой ноды
  data_dir: /var/lib/postgresql/data/postgresql0 # Место хранения данных первой ноды
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication: # логопасс для репликаци, при желании можно поменять
      username: replicator
      password: rep-pass
    superuser: # админский логопасс, при желании можно поменять (в том числе в файле compose)
      username: postgres
      password: postgres
    parameters:
      unix_socket_directories: '.'
watchdog:
  mode: off
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

```yml
scope: my_cluster # Имя нашего кластера
name: postgresql1 # Имя первой ноды
restapi: # Адреса первой ноды
  listen: pg-slave:8008
  connect_address: pg-slave:8008
zookeeper:
  hosts:
    - zoo:2181 # Адрес Zookeeper
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 10485760
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f
  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5
postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-slave:5432 # Адрес первой ноды
  data_dir: /var/lib/postgresql/data/postgresql1 # Место хранения данных первой ноды
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication: # логопасс для репликаци, при желании можно поменять
      username: replicator
      password: rep-pass
    superuser: # админский логопасс, при желании можно поменять (в том числе в файле compose)
      username: postgres
      password: postgres
    parameters:
      unix_socket_directories: '.'
watchdog:
  mode: off
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

Теперь, время запустить всю эту шайтан-машину 💀:

Воспользуемся командой:

```text
docker compose up -d --build
```

на удивление, все заработало

![img.png](screenshots%2Fimg.png)

Проверим, какая нода стала мастером, а какая слейвом:

Как видно pg-master соответсвует своей роли:

![img_1.png](screenshots%2Fimg_1.png)

Проверим pg-slave:

![img_2.png](screenshots%2Fimg_2.png)

## Проверяем репликацию

### Подключимся к нашим нодам:

![img_3.png](screenshots%2Fimg_3.png)

Теперь создадим таблицу и вставим в нее данные:

![img_4.png](screenshots%2Fimg_4.png)

Теперь сделаем SELECT на pg-slave:

![img_5.png](screenshots%2Fimg_5.png)

Данные скопировались - все работает

Также попробуем создать таблицу из pg-slave

![img_6.png](screenshots%2Fimg_6.png)

Появилась ошибка, так и должно быть

## Установка haproxy

Добавим haproxy в docker-compose:

```yml
  haproxy:
    image: haproxy:3.0
    container_name: postgres_entrypoint # Это будет адрес подключения к БД, можно выбрать любой
    ports:
      - 5435:5432 # Это будет порт подключения к БД, можно выбрать любой
      - 7000:7000
    depends_on: # Не забываем убедиться, что сначала все корректно поднялось
      - pg-master
      - pg-slave
      - zoo
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
```

А также создадим его конфиг (haproxy.cfg)

```cfg
global
    maxconn 100
defaults
    log global
    mode tcp
    retries 3
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
listen postgres
    bind *:5432 # Выбранный порт из docker-compose.yml
    option httpchk
    http-check expect status 200 # Описываем нашу проверку доступности (в данном случае обычный HTTP-пинг)
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgresql_pg_master_5432 pg-master:5432 maxconn 100 check port 8008 # Адрес первой ноды постгреса
    server postgresql_pg_slave_5432 pg-slave:5432 maxconn 100 check port 8008 # Адрес второй ноды постгреса

```

Теперь снова билдим, запускаем и пробуем подключиться через haproxy:

![img_7.png](screenshots%2Fimg_7.png)

и снова успех, осталось только проверить отказоустойчивость:

- отключаем pg-master
![img_8.png](screenshots%2Fimg_8.png)
- заходим в логи pg-slave и видим что он теперь лидер кластера
![img_9.png](screenshots%2Fimg_9.png)

Создаем таблицу через haproxy, теперь, после отключения pg-master она должна создасться только в pg-slave

![img_11.png](screenshots%2Fimg_11.png)

Так оно и происходит

Осталось посмотреть придут ли данные на pg-master после включения:

![img_12.png](screenshots%2Fimg_12.png)

Данные пришли, pg-master стал второстепенной нодой

## Ответы на вопросы 

1.  Порты 8008 и 5432 вынесены в разные директивы, expose и ports. По сути, если записать 8008 в ports, то он тоже станет exposed. В чем разница?

- ports открывает порты для внешнего взаимодействия, expose работает только внутри сети Docker 

2. При обычном перезапуске композ-проекта, будет ли сбилден заново образ? А если предварительно отредактировать файлы postgresX.yml? А если содержимое самого Dockerfile? Почему?

- Нет, при обычном перезапуске (docker-compose restart или docker-compose up) образ не будет пересобран, даже если отредактировать Dockerfile или файлы в контексте сборки, потому что Docker Compose для скорости использует уже существующие образы и не проверяет изменения в исходном коде. Чтобы применить изменения из Dockerfile или контекста, нужно явно указать флаг --build при запуске (docker-compose up --build) или предварительно выполнить docker-compose build.
