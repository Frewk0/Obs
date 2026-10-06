## 1. THE PROBLEM: IMPERATIVE HELL 🟥 MUST KNOW

### 1.1. WHAT & WHY

До цього моменту ти запускав контейнери імперативно (Imperative approach) — писав довгі команди в терміналі.

Уяви, що твій project складається з трьох частин: Frontend (React), Backend (Python), і Database (PostgreSQL).

Щоб запустити цей стек вручну, тобі треба:

  

1. Створити Network (`docker network create...`).
    
      
    
2. Запустити БД з Volume та Env-змінними (`docker run -d --name db -e POSTGRES_PASSWORD=sec -v pgdata:/var/lib/postgresql/data --network my-net postgres`).
    
      
    
3. Запустити Backend, прокинути порти, підключити до Network...
    
      
    
4. Запустити Frontend...
    
      
    

- _Problem:_ Це неможливо запам'ятати. Це неможливо передати іншому розробнику. Якщо сервер перезавантажиться, ти будеш вводити це знову.
    
      
    
- _Solution:_ **Docker Compose**. Це інструмент для декларативного (Declarative) опису інфраструктури. Ти один раз описуєш бажаний стан (Desired State) у файлі `docker-compose.yml`, і система сама розбирається, як його досягти. Це твій перший крок до **Infrastructure as Code (IaC)**.
    
      
    

## 2. ANATOMY OF `docker-compose.yml` 🟥 MUST KNOW

Файл написаний мовою YAML (відступи мають значення, використовуй тільки пробіли, ніколи не TAB).

Маніфест завжди складається з трьох головних Root-блоків: `services`, `volumes`, `networks`.

  

### 2.1. The Ultimate Production Example (PostgreSQL + API)

YAML

```
# Версія синтаксису (сучасний Compose ігнорує це поле, але його часто залишають для сумісності)
name: my-awesome-project # Задає префікс для всіх ресурсів (опціонально)

services:
  # 1. Database Service
  db:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: mysecretpassword
      POSTGRES_DB: app_db
    volumes:
      - pg-data:/var/lib/postgresql/data # Підключення Named Volume
    networks:
      - backend-net # Підключення до кастомної мережі

  # 2. Backend API Service
  api:
    build: 
      context: ./backend # Папка, де лежить Dockerfile
      dockerfile: Dockerfile
    restart: on-failure
    ports:
      - "8000:8000" # Publish порту на хост
    depends_on:
      - db # API запуститься ТІЛЬКИ після старту db
    networks:
      - backend-net

# Декларація ресурсів (без цього блоки services не зможуть їх використати)
volumes:
  pg-data: # Docker автоматично зробить 'docker volume create pg-data'

networks:
  backend-net: # Docker автоматично зробить 'docker network create backend-net'
```

## 3. NETWORKING UNDER THE HOOD (Як працює мережа в Compose) 🟪 ADVANCED

Це найсильніша сторона Docker Compose.

  

- _Internal Mechanics:_ Коли ти робиш `docker compose up`, Docker автоматично створює нову bridge network (якщо ти не вказав кастомну). Назва мережі формується з імені папки + `_default` (наприклад, `myproject_default`).
    
      
    
- **Service Discovery (Магія DNS):** Усі контейнери в одному `docker-compose.yml` автоматично додаються в цю мережу. Тобі БІЛЬШЕ НЕ ТРЕБА знати IP-адреси!
    
      
    
- _Application Code Configuration:_ Твій Python Backend з прикладу вище має підключатися до бази даних за хостом **`db`** (ім'я сервісу), а не `127.0.0.1`. Вбудований DNS Докера автоматично відрезолвить (перетворить) `db` на актуальний IP-адрес контейнера бази даних.
    
      
    

## 4. STARTUP ORDER & HEALTHCHECKS 🟨 SHOULD KNOW

- _The Trap:_ Інструкція `depends_on: - db` каже Compose: "Запусти контейнер `api` після того, як створиш контейнер `db`". Але Compose **не знає**, чи база даних всередині контейнера вже готова приймати з'єднання (можливо, PostgreSQL ще 10 секунд ініціалізує файли на диску). Backend спробує підключитися, отримає _Connection Refused_ і впаде.
    
      
    
- _Best Practice (Service Healthy):_ Щоб зробити надійний pipeline, використовують `healthcheck`.
    
      
    

YAML

```
services:
  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "admin"] # Команда всередині контейнера
      interval: 5s
      timeout: 5s
      retries: 5
      
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy # Чекає, поки healthcheck БД не поверне SUCCESS
```

## 5. ENVIRONMENT VARIABLES (Управління секретами) 🟨 SHOULD KNOW

Хардкодити паролі прямо в `docker-compose.yml` (як у прикладі вище) — це грубий Security Anti-pattern (якщо цей файл пушиться в Git).

  

- _The DevOps Way:_ Використовувати файл `.env`.
    
      
    

**1. Створюєш файл `.env` поруч із `docker-compose.yml`:**

  

Plaintext

```
DB_PASS=SuperSecret123
API_PORT=8080
```

_(Цей файл ОБОВ'ЯЗКОВО додається в `.gitignore`)_

  

**2. Використовуєш змінні в `docker-compose.yml` (Interpolation):**

  

YAML

```
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASS} # Підставить значення з .env
  api:
    ports:
      - "${API_PORT}:8000"
```

## 6. CLI REFERENCE (Modern `docker compose`) 🟥 MUST KNOW

_Technical Note:_ Раніше це була окрема програма `docker-compose` (з дефісом), написана на Python. Зараз це вбудований плагін, написаний на Go — `docker compose` (через пробіл). Використовуй новий варіант.

  

|**Command**|**Action**|**Behavior / Output**|
|---|---|---|
|`docker compose up -d`|🟥 Start Stack|Збирає образи (якщо треба), створює мережі/томи, запускає контейнери в detached mode (фоні).|
|`docker compose down`|🟥 Stop Stack|Зупиняє і **видаляє** контейнери та мережі. _Томи залишаються цілими!_|
|`docker compose down -v`|🟠 Destroy Stack|Видаляє контейнери, мережі І ТОМИ. **(Destroys Database Data!)**.|
|`docker compose logs -f`|🟨 View Logs|Показує агреговані (зібрані разом) логи з УСІХ сервісів у реальному часі.|
|`docker compose ps`|🟢 Status|Показує статус контейнерів поточного проєкту.|
|`docker compose build`|🟨 Build Images|Примусово перезбирає образи (які описані блоком `build:`), не запускаючи їх.|

## 7. DEEP TROUBLESHOOTING 🟪 ADVANCED

### 7.1. "Я змінив код, але Compose запускає стару версію"

- **Root Cause:** Якщо ти один раз зробив `docker compose up`, Docker зібрав твій образ і закешував його. При наступному `up` він бачить, що образ з таким ім'ям вже є, і не перезбирає його, навіть якщо ти змінив файли коду.
    
      
    
- **The Fix:** `docker compose up -d --build`. Прапорець `--build` примушує Compose перевірити Dockerfile і перезібрати образ перед запуском.
    
      
    

### 7.2. "Orphan containers warning"

- **Symptom:** `WARNING: Found orphan containers (...) for this project.`
    
      
    
- **Root Cause:** Ти запустив проєкт, де був сервіс `api_v1`. Потім ти перейменував його в `docker-compose.yml` на `api_v2` і зробив `up`. Compose запустив `api_v2`, але старий контейнер `api_v1` все ще висить у пам'яті (став "сиротою"), бо його більше немає в YAML-файлі, і Compose не знає, що з ним робити.
    
      
    
- **The Fix:** `docker compose up -d --remove-orphans`.
    
      
    

### 7.3. "Port is already allocated"

- **Symptom:** `Error starting userland proxy: listen tcp4 0.0.0.0:80: bind: address already in use`.
    
      
    
- **Root Cause:** Ти намагаєшся відкрити порт 80, але на твоєму хості ВЖЕ працює щось на цьому порту (можливо, локальний Nginx, або інший запущений контейнер).
    
      
    
- **The Fix:** Знайди процес, який тримає порт: `sudo ss -tulpn | grep :80`. Зупини його або зміни порт у Compose (`"8080:80"`).
    
      
    

## 8. MEMORY CORE

**Що я повинен пам'ятати без підглядання:**

  

- `docker compose` (через пробіл) — сучасний CLI плагін.
    
      
    
- Контейнери в Compose автоматично бачать один одного за **іменами сервісів** (вбудований DNS). Ніяких IP-адрес.
    
      
    
- `docker compose down` не видаляє дані (Volumes). Щоб знищити дані, треба додати `-v`.
    
      
    
- Якщо змінив код програми — роби `up -d --build`. Без `--build` зміни не підтягнуться.
    
      
    
- Секрети (паролі) мають лежати в `.env` файлі, а в YAML передаватися як `${VAR_NAME}`.


1. Сучасний та стандартний спосіб (Compose V2 / Специфікація Docker Compose)

Цей формат є універсальним і працює як для звичайного запуску через `docker compose up`, так і для Docker Swarm. Блок обмежень прописується всередині секції `deploy`.

yaml

```
version: "3.8" # Або без вказання версії для сучасного Compose

services:
  web_app:
    image: nginx:latest
    deploy:
      resources:
        limits:
          cpus: '0.50' # Максимум 50% від одного ядра CPU
          memory: 512M # Максимум 512 Мегабайт оперативної пам'яті
        reservations:
          cpus: '0.25' # Гарантовано виділити 25% CPU
          memory: 256M # Гарантовано виділити 256 Мегабайт пам'яті
```

Используйте код с осторожностью.

**Що означають ці параметри:**

- **`limits` (Ліміти):** Жорстке обмеження. Контейнер не зможе взяти більше ресурсів, ніж указано. Якщо контейнер перевищить ліміт пам'яті (`memory`), Docker його перезапустить (помилка OOM — Out Of Memory).

- **`reservations` (Резервування):** Мінімально гарантовані ресурси, які Docker намагатиметься виділити для контейнера під час старту.

 ==**PROFILES**==
### 1. Що це таке і яку проблему вирішує?

**Проблема:** Уяви, що ти прийшов на реальний проєкт. У вас є:

1. Frontend (React)
    
2. Backend (Python/Django)
    
3. База даних (PostgreSQL)
    
4. Кеш (Redis)
    
5. Сервіс для E2E тестування (Cypress)
    
6. Графічна панель для бази даних (pgAdmin)
    
7. Сервіс для моніторингу (Prometheus/Grafana)
    

- **Frontend-розробнику** потрібні тільки Frontend, Backend і База. Йому наплювати на моніторинг.
    
- **QA-інженеру** потрібні Backend, База і сервіс тестів (Cypress).
    
- **DevOps-інженеру** потрібно підняти моніторинг і бази даних для дебагу.
    

Без профайлів тобі довелося б писати окремі файли: `docker-compose.front.yml`, `docker-compose.qa.yml`, `docker-compose.monitoring.yml`. Це пекло для підтримки: змінив версію бази в одному файлі — забув змінити в інших.

**Рішення (Profiles):** Profiles дозволяють призначити кожному сервісу "тег" (категорію). І при запуску ти просто кажеш Docker-у: _"Підніми мені тільки ті сервіси, які належать до профайлу frontend"_.

### 2. Як це працює під капотом?

Ти просто додаєш блок `profiles: ["назва_профайлу"]` до конфігурації сервісу.

**Головне правило логіки Docker Compose Profiles:**

1. Якщо в сервісу **НЕМАЄ** блоку `profiles` — він вважається базовим (core). Він буде запускатися **ЗАВЖДИ**, коли ти пишеш просто `docker compose up`.
    
2. Якщо в сервісу **Є** блок `profiles` — він стає "сплячим". Він не запуститься, поки ти явно не вкажеш його профайл при запуску.
    

Запустити конкретний профайл можна двома способами (на співбесідах часто питають обидва):

- Через прапорець: `docker compose --profile debug up -d`
    
- Через змінну оточення (корисно для CI/CD): `COMPOSE_PROFILES=debug docker compose up -d`
    

Можна комбінувати кілька профайлів одночасно: `docker compose --profile frontend --profile debug up -d`

### 3. Приклади від простого до Production

#### Junior приклад (Проста логіка)

Тут база даних стартує завжди, а панель керування нею (pgadmin) — тільки якщо ми спеціально її покличемо.

YAML

```
services:
  db:
    image: postgres:15
    # Немає profiles -> стартує завжди

  pgadmin:
    image: dpage/pgadmin4
    profiles: ["debug"] # Спить за замовчуванням
```

#### Production приклад (Розподіл по командах)

Ось як це виглядає в реальних компаніях:

YAML

```
services:
  # БАЗОВІ СЕРВІСИ (Стартують завжди)
  postgres:
    image: postgres:15
  
  redis:
    image: redis:alpine

  # БЕКЕНД КОМАНДА
  api:
    image: my-backend-api
    profiles: ["backend", "fullstack"]
    depends_on:
      - postgres

  # ФРОНТЕНД КОМАНДА
  web:
    image: my-react-app
    profiles: ["frontend", "fullstack"]

  # QA КОМАНДА (Тестування)
  cypress-tests:
    image: cypress/included:latest
    profiles: ["testing"]
    depends_on:
      - api
      - web
```

- Backend-розробник пише: `docker compose --profile backend up -d` (Отримає: Postgres, Redis, API).
    
- Frontend-розробник пише: `docker compose --profile frontend up -d` (Отримає: Postgres, Redis, Web).
    
- На CI/CD сервері при деплої запускається: `COMPOSE_PROFILES=testing docker compose up` (Отримає все необхідне для тестів).
    

### 4. Типові помилки новачків (Як дебажити)

**Помилка 1: Невидимі залежності (depends_on)** Уяви, що сервіс `api` має `profiles: ["backend"]`, і він має `depends_on: db`. Якщо ти запустиш `docker compose up` (без профайлів), база `db` запуститься, а `api` — ні. Але якщо `db` має `profiles: ["database"]`, а `api` (без профайлу) залежить від нього, то при звичайному `docker compose up` Docker **автоматично** підтягне і запустить базу даних, навіть якщо ти не вказав її профайл, бо `api` без неї не виживе.

**Помилка 2: Видалення контейнерів** Коли ти хочеш зупинити і видалити всі контейнери, ти пишеш `docker compose down`. Але ця команда видалить **тільки активні профайли** або ті сервіси, що без профайлів! Якщо в тебе висів запущений сервіс з профайлу `debug`, він залишиться працювати у фоні. _Правильно видаляти все:_ `docker compose --profile "*" down` (зірочка означає "всі профайли").