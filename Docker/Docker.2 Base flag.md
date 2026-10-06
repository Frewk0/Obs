
## 1. `docker` group — що відбувається

Docker daemon працює з високими привілеями. Твій звичайний користувач не має автоматично права керувати `/var/run/docker.sock`.

Тому спочатку:

```bash
sudo docker ps
```

А коли тебе додають у групу:

```bash
sudo usermod -aG docker $USER
```

ти отримуєш доступ до Docker socket через Unix group.

Після цього:

```bash
newgrp docker
```

або новий login, щоб поточна сесія побачила нову групу. Саме ця механіка описана у твоєму файлі.

### Чому це security problem?

Бо Docker daemon має дуже великі привілеї.

Умовно:

```text
user in docker group
        ↓
docker.sock
        ↓
dockerd
        ↓
high privileges on host
```

Тому:

```bash
docker run -v /:/host-root ubuntu
```

може дати контейнеру доступ до файлової системи хоста.

Тобто важлива mental model:

> **`docker` group — це не просто "дозвіл запускати docker без sudo". Це дуже привілейований доступ до Docker Engine.**

---

# 2. `daemon.json` — що це?

Ти вже знаєш:

```text
docker CLI
    ↓
dockerd
```

А тепер треба зрозуміти:

> **Хто визначає, як поводиться `dockerd`?**

Ось тут і з'являється:

```text
/etc/docker/daemon.json
```

Це configuration file **Docker daemon**.

Тобто:

```text
Docker CLI
   ↓
dockerd
   ↑
daemon.json
```

CLI каже:

> "Запусти контейнер".

А `daemon.json` задає **глобальну поведінку daemon**.

Наприклад:

- logging;
    
- storage;
    
- registry-related settings;
    
- інші daemon-level options.
    

---


# 4. Що роблять `max-size` і `max-file`

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

Mental model:

```text
container logs
      ↓
50 MB
      ↓
rotate
      ↓
another file
      ↓
rotate
      ↓
another file
```

Тобто ти не дозволяєш одному контейнеру нескінченно накопичувати локальні logs.

> Важливий нюанс: конкретна поведінка залежить від logging driver. Тут йдеться про `json-file`.

### 13.1. Core Commands (Execution & State)

|**Command**|**Modern Alias**|**Action**|**Risk**|
|---|---|---|---|
|`docker run <image>`|`docker container run`|Створити і запустити новий контейнер.|🟢 Safe|
|`docker ps`|`docker container ls`|Показати **тільки запущені** контейнери.|🟢 Safe|
|`docker ps -a`|`docker container ls -a`|Показати **всі** (вкл. Exited). 🟥 Must Know!|🟢 Safe|
|`docker stop <id>`|`docker container stop`|Відправити `SIGTERM`. Дає процесу 10 сек (default) на graceful shutdown, потім шле `SIGKILL`.|🟢 Safe|
|`docker kill <id>`|`docker container kill`|Одразу відправити `SIGKILL`. Процес вмирає миттєво.|🟡 Caution|
|`docker rm <id>`|`docker container rm`|Видалити зупинений контейнер (знищити RW шар).|🟠 Destructive|
|`docker rm -f <id>`|`docker container rm -f`|Примусово вбити (SIGKILL) і одразу видалити.|🟠 Destructive|

### 13.2. Core Commands (Debugging & Interaction)

| **Command**              | **Action**                                              | **Example**                             |
| ------------------------ | ------------------------------------------------------- | --------------------------------------- |
| `docker logs <id>`       | Читає STDOUT/STDERR процесу контейнера.                 | `docker logs -f my-web` (follow stream) |
| `docker exec <id> <cmd>` | Запускає **додатковий** процес у вже живому контейнері. | `docker exec -it my-web /bin/bash`      |
| `docker inspect <id>`    | Виводить JSON з усіма метаданими (IP, Env, Mounts).     | `docker inspect my-db`                  |
| `docker stats`           | Live-моніторинг використання CPU, RAM, Network I/O.     | `docker stats` (як `htop`)              |
| `docker cp`              | Копіює файли між хостом і контейнером.                  | `docker cp ./conf nginx:/etc/nginx/`    |

## 14. FLAG MEMORY TABLE: `docker run` 🟥 MUST KNOW

`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

### 14.1. Flags I should remember (Must Know)

|**Flag**|**Name**|**Mechanics / What it does**|**Example**|
|---|---|---|---|
|**`-d`**|Detached|Запускає контейнер у фоні і повертає керування терміналом. Виводить ID контейнера.|`docker run -d nginx`|
|**`-p`**|Publish|`HOST_PORT:CONTAINER_PORT`. Налаштовує DNAT в iptables хоста.|`docker run -p 8080:80 nginx`|
|**`-v`**|Volume|Монтує директорію хоста (Bind Mount) або керований том (Named Volume) у контейнер.|`docker run -v pgdata:/var/lib/postgresql/data postgres`|
|**`-e`**|Environment|Передає змінні середовища всередину процесу. Можна передавати багато разів.|`docker run -e MYSQL_ROOT_PASSWORD=sec mysql`|
|**`--name`**|Name|Призначає кастомне ім'я замість згенерованого (напр. `jolly_turing`). Полегшує управління.|`docker run --name my-api api-image`|
|**`-it`**|Interactive + TTY|`-i` тримає STDIN відкритим (навіть без attach). `-t` виділяє псевдо-термінал. Необхідно для запуску bash/sh.|`docker run -it ubuntu /bin/bash`|

### 14.2. Flags I can safely look up (Useful / Advanced)

| **Flag**           | **Mechanics**                                                                     | **When to use**                                                                           |
| ------------------ | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **`--rm`**         | Контейнер автоматично видаляється (робить `docker rm`), щойно процес завершиться. | Тимчасові задачі (`docker run --rm alpine ls -l`). _Exception:_ Не сумісно з `--restart`. |
| **`--network`**    | Підключає контейнер до конкретної віртуальної мережі.                             | `docker run --network my-net ...`                                                         |
| **`--restart`**    | Restart Policy. Можливі значення: `no`, `on-failure`, `always`, `unless-stopped`. | Production-деплої без оркестратора (`always`).                                            |
| **`--user`**       | Запускає процес (PID 1) від вказаного UID, ігноруючи `USER` з Dockerfile.         | `docker run --user 1000:1000 ...` (вирішення проблем з правами на томи).                  |
| **`--entrypoint`** | Перевизначає `ENTRYPOINT` з Dockerfile.                                           | Дебаг контейнера, який падає при старті.                                                  |

# 18. `-v` — теж треба бачити логікою

```bash
-v pgdata:/var/lib/postgresql/data
```

означає:

```text
Docker volume: pgdata
       ↓
container:
/var/lib/postgresql/data
```

А:

```bash
-v /home/me/app:/app
```

означає:

```text
Host directory
/home/me/app
       ↓
Container
/app
```

Тобто `-v` може використовуватися для різних типів mounts.

---

# 19. Чому `--entrypoint` такий корисний?

Уяви:

```text
docker run my-app
       ↓
application starts
       ↓
CRASH
       ↓
container exits
```

Ти не встигаєш:

```bash
docker exec ...
```

бо container уже мертвий.

Тоді:

```bash
docker run -it --entrypoint /bin/sh my-app
```

Ти кажеш:

> "Не запускай стандартний application. Замість нього запусти shell."

Тепер можна зайти всередину і перевірити:

```text
filesystem
environment
files
permissions
dependencies
```

Це дуже хороший debugging pattern.

# 21. Troubleshooting: контейнер одразу помирає

```text
1. docker ps -a
        ↓
2. inspect state / exit code
        ↓
3. docker logs
        ↓
4. inspect config
        ↓
5. перевір command / entrypoint
        ↓
6. dependencies
        ↓
7. filesystem / permissions
        ↓
8. resources
```


# 22. Exit codes — тут обережно

У твоєму файлі:

```text
137 → OOM
127 → command not found
1/255 → application/config error
```

Це **корисні сигнали**, але не сприймай їх як абсолютну таблицю:

> "137 завжди OOM."

Наприклад, 137 часто відповідає завершенню через signal 9 (`128 + 9`), а OOM kill — лише один із типових шляхів, як це може статися.

Тобто правильна модель:

```text
Exit code
   ↓
clue
   ↓
investigate
```
