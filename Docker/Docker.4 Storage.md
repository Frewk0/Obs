## 1. THE PROBLEM: EPHEMERAL CONTAINERS 🟥 MUST KNOW

### 1.1. WHAT & WHY

За своєю природою контейнери є **ephemeral** (тимчасовими). Усі файли, створені або змінені програмою всередині контейнера, записуються в його тонкий **Writable Layer**.

* *Problem 1 (Data Loss):* Коли ти виконуєш `docker rm <container>`, цей Writable Layer знищується назавжди. Якщо там була база даних PostgreSQL — ти втратив усі дані клієнтів.
* *Problem 2 (Performance):* Запис у Writable Layer проходить через драйвер `overlay2` (механізм Copy-on-Write). Це додає overhead (накладні витрати). Для інтенсивних операцій вводу/виводу (I/O), які роблять бази даних, це неприпустимо повільно.

### 1.2. THE SOLUTION

Docker дозволяє обійти UnionFS і монтувати файли або директорії безпосередньо з **Host OS** у контейнер. Цей механізм гарантує:

1. **Persistence:** Дані переживають видалення контейнера.
2. **Native Performance:** Швидкість читання/запису дорівнює швидкості фізичного диска сервера.

---

## 2. TYPES OF MOUNTS (Типи монтування) 🟥 MUST KNOW

Docker підтримує три основні типи роботи зі сховищем.

### 2.1. Volumes (Керовані Томи)

* **WHAT:** Директорія, яка створюється і повністю управляється самим демоном Docker. Вона лежить у спеціально відведеному місці на хості (зазвичай `/var/lib/docker/volumes/`).
* **WHEN TO USE (Best Practice):** Збереження даних баз даних (PostgreSQL, MySQL, Redis), черг повідомлень (RabbitMQ), шаринг файлів між кількома контейнерами.
* **WHEN NOT TO USE:** Коли тобі потрібно редагувати ці файли в IDE (наприклад, VS Code) на своєму комп'ютері. Томи сховані глибоко в системі і часто належать `root`.

### 2.2. Bind Mounts (Прямі прив'язки)

* **WHAT:** Ти береш **будь-яку** існуючу директорію на своїй Host OS (наприклад, `/home/frewk0/project/src`) і монтуєш її в контейнер (наприклад, у `/app/src`).
* **WHEN TO USE (Common Practice):** Локальна розробка (Development). Ти зберігаєш файл коду у своєму редакторі -> файл миттєво оновлюється в контейнері -> веб-сервер робить Live Reload.
* **WHEN NOT TO USE:** У Production. *Anti-pattern:* Зав'язувати production-контейнер на жорсткі шляхи Host-системи (`/opt/custom/path`). Це ламає портативність (на іншому сервері цього шляху може не бути).

### 2.3. tmpfs (Тимчасова пам'ять) 🟨 SHOULD KNOW

* **WHAT:** Дані пишуться безпосередньо в оперативну пам'ять (RAM) сервера. Вони ніколи не зберігаються на фізичний диск.
* **WHEN TO USE:** Зберігання вкрай чутливих секретів (API ключі, токени), які не повинні потрапити на диск. Кешування даних, де потрібна максимальна швидкість і втрата яких після рестарту не є критичною.

---

## 3. CLI SYNTAX: `-v` vs `--mount` 🟥 MUST KNOW

Для підключення сховища під час `docker run` існує два прапорці.

### 3.1. Прапорець `-v` або `--volume` (Old format, Common Practice)

Складається з трьох полів, розділених двокрапкою: `source:target:options`.

```bash
# Named Volume:
docker run -d -v my-db-data:/var/lib/postgresql/data postgres

# Bind Mount (source обов'язково починається з / або ./):
docker run -d -v /home/user/nginx.conf:/etc/nginx/nginx.conf:ro nginx

```

* `ro` (Read-Only) — захищає файл на хості від змін зсередини контейнера.

### 3.2. Прапорець `--mount` (Modern, Best Practice)

Явний і читабельний формат `key=value`. Рекомендується для production скриптів.

```bash
docker run -d \
  --mount type=bind,source=/home/user/nginx.conf,target=/etc/nginx/nginx.conf,readonly \
  nginx

```

### 3.3. КРИТИЧНА РІЗНИЦЯ (Troubleshooting trap)

*Technical Fact:* Якщо ти використовуєш Bind Mount, і директорії `source` на твоєму хості **не існує**:

* Команда з **`-v`** автоматично створить цю директорію. Але вона створить її від імені користувача `root`. Твій застосунок може втратити до неї доступ (`Permission denied`).
* Команда з **`--mount`** просто видасть помилку і не запустить контейнер. Це набагато безпечніше, бо захищає від одруківки (typo).

---

## 4. CLI REFERENCE: VOLUME MANAGEMENT 🟨 SHOULD KNOW

| Command | Action | Risk | Note |
| --- | --- | --- | --- |
| `docker volume create <name>` | Створити пустий том. | 🟢 Safe | Можна не робити вручну, Docker створить його автоматично під час `run`. |
| `docker volume ls` | Список усіх томів на сервері. | 🟢 Safe |  |
| `docker volume inspect <name>` | Показати метадані тому. | 🟢 Safe | 🟥 Must Know: В полі `Mountpoint` ти побачиш реальний шлях на фізичному диску (де лежать файли). |
| `docker volume rm <name>` | Видалити конкретний том. | 🟠 Destructive | Знищує дані назавжди. Контейнер, який його використовує, має бути видалений перед цим. |
| `docker volume prune` | Видалити **УСІ** томи, які не підключені до живих або зупинених контейнерів. | 🔴 Data-destructive | Дуже обережно в Prod. Якщо ти зупинив і видалив БД, щоб її оновити, `prune` зітре її дані. |

---

## 5. DEEP TROUBLESHOOTING: THE PERMISSION HELL 🟪 ADVANCED

Це найчастіший біль DevOps інженерів при роботі з Bind Mounts та Volumes.

**Symptom:** Застосунок (наприклад, Node.js або PostgreSQL) падає з помилкою `Error: EACCES: permission denied, mkdir '/data'`.
**Context:** Ти підключив папку з хоста в контейнер через `-v /home/user/data:/data`.

### 5.1. INTERNAL MECHANICS (Чому це відбувається)

Ядро Linux нічого не знає про імена користувачів (username). Воно оперує виключно цифрами — **UID (User ID)** та **GID (Group ID)**.

1. На твоїй Host OS ти створив папку `/home/user/data`. Ти звичайний користувач, твій UID = `1000`. Папка належить UID `1000`.
2. Всередині контейнера образ налаштований так (через інструкцію `USER node`), що процес `node` працює від імені користувача з UID `1001`.
3. Процес (UID `1001`) намагається записати файл у папку (належить UID `1000`).
4. Ядро Linux каже: "1001 != 1000. Доступ заборонено."

### 5.2. Investigation Tree & Fixes

**1. Дізнайся UID на хості:**

```bash
ls -ln /home/user/data  # Покаже числа замість імен, наприклад 1000 1000

```

**2. Дізнайся UID процесу в контейнері:**

```bash
docker exec my-app id   # Покаже uid=1001(node) gid=1001(node)

```

**3. The Fix (Виправлення):**

* *Option A (Fix on Host - Recommended):* Зміни власника папки на хості, щоб він збігався з процесом контейнера.
```bash
sudo chown -R 1001:1001 /home/user/data

```


* *Option B (Fix via Docker):* Змусь контейнер працювати від твого UID за допомогою прапорця `--user`.
```bash
docker run -v /home/user/data:/data --user 1000:1000 my-app

```



---

## 6. REAL TASK: BACKUP & RESTORE A VOLUME 🟨 SHOULD KNOW

**Problem:** В тебе є Volume `pg-data`, де лежить production база даних. Її треба збекапити у звичайний `.tar.gz` архів на твій комп'ютер, **не зупиняючи** саму базу даних, або мінімізувавши downtime.

**HOW (The Docker Way):**
Використовуємо тимчасовий контейнер (часто Alpine або Ubuntu), який примонтує той самий Volume і виконає архівацію.

```bash
# 1. Створюємо бекап в архів /tmp/backup.tar.gz на хості
docker run --rm \
  -v pg-data:/data:ro \
  -v /tmp:/backup \
  alpine \
  tar -czvf /backup/pg_backup.tar.gz -C /data .

```

*Investigation / Mechanics of this command:*

* `--rm` — контейнер-архіватор самознищиться після завершення.
* `-v pg-data:/data:ro` — ми монтуємо том з БД у папку `/data` ТІЛЬКИ для читання (`ro`), щоб нічого не пошкодити.
* `-v /tmp:/backup` — ми робимо Bind Mount папки `/tmp` з хоста в `/backup` контейнера, щоб мати куди покласти готовий архів.
* `alpine tar ...` — викликаємо утиліту архівації всередині контейнера.

---

## 7. MEMORY CORE

**Що я повинен пам'ятати без підглядання:**

* **Volumes:** Для баз даних та постійного зберігання (управляє Docker).
* **Bind Mounts:** Для коду та конфігів (управляєш ти, вказуєш точний шлях).
* Дані у Volumes повністю обходять UnionFS і записуються прямо на диск з native performance.
* Якщо примонтована папка видає `Permission denied`, проблема майже завжди у конфлікті **UID/GID** між хостом і контейнером.
* Команда `docker volume prune` — надзвичайно небезпечна в production, бо видаляє всі непідключені томи.
