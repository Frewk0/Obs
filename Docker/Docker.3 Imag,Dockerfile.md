
**Part 3 — Images, Dockerfile, Layers, BuildKit & Optimization**


## 18. CONCEPT: DOCKER IMAGE & LAYERS 🟥 MUST KNOW

### 18.1. WHAT (Що таке Image)

Образ (Image) — це read-only (незмінний) шаблон, який містить файлову систему (rootfs), необхідну для запуску застосунку: ОС (наприклад, Alpine або Ubuntu), встановлені пакети, твій код і метадані (які порти відкривати, від якого користувача працювати).

  

### 18.2. INTERNAL MECHANICS (Як працюють шари) 🟪 ADVANCED

Образ не є монолітним файлом (як `.iso`). Це стек незалежних архівів (Layers).

  

- Кожна інструкція в Dockerfile (наприклад, `RUN`, `COPY`) створює **новий шар** (дельту файлової системи).
    
      
    
- _Storage Semantics:_ Якщо ти маєш 10 різних образів на сервері (наприклад, Node.js, Python, Nginx), і всі вони базуються на `ubuntu:22.04`, то базовий шар Ubuntu завантажується і зберігається на диску хоста **лише один раз**. Усі 10 образів ділять цей шар між собою через UnionFS. Це радикально економить дисковий простір.
    
      
    
- _Immutability:_ Шари ніколи не змінюються після створення. Якщо у першому шарі ти створив файл (100 MB), а в другому шарі видалив його (`RUN rm file`), образ усе одно **збільшиться на 100 MB**. Видалення в наступному шарі лише "приховує" файл від UnionFS, але не видаляє його фізично з попереднього шару.
    
      
    

## 19. CLI REFERENCE: IMAGE MANAGEMENT 🟥 MUST KNOW

Команди для управління локальним сховищем образів та взаємодії з Registry.

  

| **Command**                        | **Action**                                                        | **Risk**       | **Note**                                 |
| ---------------------------------- | ----------------------------------------------------------------- | -------------- | ---------------------------------------- |
| `docker image ls`                  | Список завантажених образів.                                      | 🟢 Safe        |                                          |
| `docker pull <image>:<tag>`        | Завантажити образ із Registry.                                    | 🟢 Safe        |                                          |
| `docker push <repo>/<image>:<tag>` | Відправити локальний образ у Registry.                            | 🟢 Safe        | Вимагає `docker login`.                  |
| `docker tag <source> <target>`     | Створити новий тег (вказівник) для існуючого образу.              | 🟢 Safe        | Самі дані не дублюються!                 |
| `docker rmi <image>`               | Видалити образ локально.                                          | 🟡 Caution     | Звільняє диск.                           |
| `docker image prune -a`            | Видалити ВСІ образи, які не використовуються живими контейнерами. | 🟠 Destructive | Доведеться качати з нуля при деплої.     |
| `docker history <image>`           | 🟨 Подивитися, з яких шарів складається образ (і їх розмір).      | 🟢 Safe        | Ідеально для дебагу "товстих" образів.   |
| `docker save -o img.tar <image>`   | 🟦 Експортувати образ у `.tar` архів.                             | 🟢 Safe        | Для серверів без інтернету (Air-gapped). |
| `docker load -i img.tar`           | 🟦 Імпортувати образ із `.tar` архіву.                            | 🟢 Safe        |                                          |

## 20. DOCKERFILE INSTRUCTIONS REFERENCE 🟥 MUST KNOW

`Dockerfile` — це декларативний рецепт збірки.

_Technical Fact:_ Dockerfile читається згори донизу. Кожен рядок може кешуватися.

  

### 20.1. Core Instructions

- **`FROM <image>:<tag>`**: (🟥 MUST KNOW) Завжди перша інструкція (виняток: `ARG` може бути перед `FROM`). Визначає базовий образ.
    
      
    
- **`WORKDIR <path>`**: (🟥 MUST KNOW) Створює директорію і робить її поточною (аналог `mkdir + cd`). Всі наступні `RUN`, `COPY`, `CMD` виконуються тут.
    
      
    
- **`RUN <command>`**: (🟥 MUST KNOW) Виконує команду в shell **ПІД ЧАС ЗБІРКИ** образу. Використовується для встановлення пакетів (`apt install`).
    
      
    
- **`USER <uid>:<gid>`**: (🟨 SHOULD KNOW) Змінює користувача для наступних команд та процесу запуску.
    
      
    
- **`EXPOSE <port>`**: (🟦 REFERENCE) Просто документація. **НЕ** відкриває порт. Порт відкривається тільки через `-p` у `docker run`.
    
      
    

### 20.2. COPY vs ADD (Trade-off & Best Practice)

- **`COPY <src> <dest>`**: Просто копіює локальні файли/папки з хоста в образ.
    
      
    - _Best Practice:_ Використовуй **завжди**, якщо немає специфічних вимог. Це прозоро і безпечно.
        
          
        
- **`ADD <src> <dest>`**: Робить те саме, але має "магію". Вміє автоматично розпаковувати `.tar` архіви та викачувати файли за URL.
    
      
    - _Anti-pattern:_ Використання `ADD` для завантаження файлів з інтернету замість `RUN curl/wget`. `ADD` не дозволяє видалити скачаний архів у тому ж шарі, що збільшує розмір образу.
        
          
        

### 20.3. CMD vs ENTRYPOINT (Interview Classic) 🟥 MUST KNOW

Обидві інструкції визначають, що виконується **ПІД ЧАС ЗАПУСКУ** контейнера (а не під час збірки).

  

- **`ENTRYPOINT ["executable", "param1"]`**: "Незмінна" частина команди. Це бінарник, який має працювати.
    
      
    
- **`CMD ["param2"]`**: Дефолтні аргументи АБО команда, якщо `ENTRYPOINT` не задано.
    
      
    
- _Internal Mechanics:_ Коли визначені обидва, Docker просто конкатенує (склеює) масиви: `[ENTRYPOINT] + [CMD]`.
    
      
    

_Scenario / Example:_

  

Dockerfile

```
ENTRYPOINT ["ping", "-c", "4"]
CMD ["localhost"]
```

- Виклик `docker run myping` виконає: `ping -c 4 localhost`.
    
      
    
- Виклик `docker run myping google.com` перевизначить `CMD`. Виконається: `ping -c 4 google.com`.
    
      
    

### 20.4. ARG vs ENV 🟨 SHOULD KNOW

- **`ARG <name>[=<default>]`**: Доступна **ТІЛЬКИ під час збірки** (наприклад, версія компілятора `ARG COMPILER_VERSION=1.18`). Значення не залишається в готовому образі.
    
      
    
- **`ENV <key>=<value>`**: Доступна і під час збірки, і **ЗАЛИШАЄТЬСЯ в працюючому контейнері** (наприклад, `ENV PORT=8080`).
    
      
    

## 21. CLI REFERENCE: DOCKER BUILD & BUILDKIT 🟥 MUST KNOW

### 21.1. Version Notes (BuildKit)

_Technical Fact:_ З версії Docker 23+ (а також у Docker Desktop), рушієм збірки за замовчуванням є **BuildKit**, а не класичний Builder.

_Performance:_ BuildKit вміє паралельно збирати незалежні етапи, пропускати непотрібні кроки і має набагато кращий механізм кешування.

  

### 21.2. Command Syntax

Bash

```
docker build [OPTIONS] PATH | URL
# Найчастіше PATH — це "." (поточна директорія), що називається Build Context.
```

### 21.3. FLAG MEMORY TABLE

| **Flag**                | **Meaning** | **Use Case**                                                                                                                        |
| ----------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **`-t <name>:<tag>`**   | Tag         | Призначає ім'я і версію (напр., `myapp:1.0`). Обов'язково для використання.                                                         |
| **`-f <file>`**         | File        | Якщо Dockerfile називається нестандартно (напр., `prod.Dockerfile`).                                                                |
| **`--no-cache`**        | No Cache    | Примушує виконати всі інструкції з нуля. Використовується, коли `RUN apt-get update` закешувався тиждень тому і тягне старі пакети. |
| **`--build-arg <k=v>`** | Build Arg   | Передача змінних `ARG` у процес збірки.                                                                                             |
| **`--target <stage>`**  | Target      | (Advanced) Зібрати тільки до певного етапу в Multi-stage build.                                                                     |

## 22. ADVANCED CONCEPT: MULTI-STAGE BUILDS 🟥 MUST KNOW

### 22.1. PROBLEM

Для компіляції програми на Go, Java або C++ потрібен компілятор, SDK та вихідний код. Усе це робить образ масивним (наприклад, 1 ГБ). Але для запуску готової програми в production потрібен лише один скомпільований файл (бінарник) і гола ОС (10 МБ).

  

### 22.2. HOW (The Solution)

Multi-stage builds дозволяють використовувати кілька інструкцій `FROM` в одному Dockerfile. Кожен `FROM` починає новий етап (stage). Ти можеш скопіювати артефакт (бінарник) з попереднього етапу в фінальний, залишивши все "сміття" (компілятори) поза готовим образом.

  

### 22.3. EXAMPLE (Go Application)

Dockerfile

```
# Етап 1: Builder (Великий образ із компілятором)
FROM golang:1.20-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Створюємо бінарник /app/main
RUN go build -o main . 

# Етап 2: Фінальний образ (Production, дуже легкий)
FROM alpine:3.18
WORKDIR /root/
# Магія: копіюємо ТІЛЬКИ готовий файл з етапу 'builder'
COPY --from=builder /app/main .
# Фінальний образ не містить вихідного коду і Go-компілятора!
CMD ["./main"]
```

## 23. BEST PRACTICES & ANTI-PATTERNS 🟨 SHOULD KNOW

### 23.1. The `.dockerignore` File (Security & Performance)

- _Problem:_ Коли ти запускаєш `docker build .`, демон копіює **весь** поточний каталог (Build Context) у свою пам'ять. Якщо в папці є `.git` (100MB історії) або `node_modules` (500MB), збірка буде довгою, а в образ можуть потрапити файли `.env` із секретами (Security risk).
    
      
    
- _Best Practice:_ Завжди створюй `.dockerignore` поруч із `Dockerfile`.
    
      
    
    Plaintext
    
    ```
    .git
    node_modules/
    .env
    ```
    

### 23.2. Cache Optimization (Порядок інструкцій) 🟥 MUST KNOW

- _Technical Fact:_ Якщо кеш інвалідується (скидається) на якомусь кроці, **усі наступні кроки виконуються без кешу**.
    
      
    
- _Anti-pattern:_
    
      
    
    Dockerfile
    
    ```
    COPY . . 
    # Код змінюється часто. Цей рядок завжди скидатиме кеш.
    RUN npm install 
    # Залежності будуть скачуватися щоразу при зміні 1 рядка коду!
    ```
    
- _Best Practice:_ Спочатку копіювати файли залежностей (які змінюються рідко), встановлювати їх, і тільки потім копіювати код.
    
      
    
    Dockerfile
    
    ```
    COPY package.json .
    RUN npm install
    COPY . . 
    ```
    

### 23.3. One RUN per logical step

- _Anti-pattern:_ Створення купи `RUN`-інструкцій для оновлення системи. Це створює зайві шари.
    
      
    
    Dockerfile
    
    ```
    RUN apt-get update
    RUN apt-get install -y curl
    RUN apt-get install -y git
    ```
    
- _Best Practice:_ Об'єднуй логічно пов'язані команди через `&&`.
    
      
    
    Dockerfile
    
    ```
    RUN apt-get update && apt-get install -y \
        curl \
        git \
     && rm -rf /var/lib/apt/lists/* # Очищення сміття в ТОМУ Ж шарі!
    ```
    

## 24. TROUBLESHOOTING: IMAGE BUILD 🟨 SHOULD KNOW

**Symptom:** Збірка `docker build` падає на команді `RUN npm install` або `RUN apt-get update` з помилкою `Temporary failure in name resolution` або `Connection timeout`.

  

- **Possible Causes:**
    
      
    1. На хост-системі є Firewall (ufw/iptables), який блокує вихідний трафік від інтерфейсу `docker0`.
        
          
        
    2. У твоїй мережі корпоративний проксі або специфічний DNS.
        
          
        
- **Investigation:** Перевір, чи є інтернет під час збірки.
    
    `docker build --network=host -t test-net .` (якщо з `--network=host` працює — проблема у віртуальному мості Docker або Firewall).
    
      
    

## 25. MEMORY CORE & AUDIT

**Що я повинен пам'ятати без підглядання:**

  

- Образ — це набір Read-Only шарів. Кожна інструкція = шар.
    
      
    
- `CMD` — можна легко перевизначити під час `docker run`. `ENTRYPOINT` — це жорсткий фундамент команди.
    
      
    
- Спочатку `COPY package.json` -> `RUN install` -> потім `COPY . .` (Основа оптимізації кешу).
    
      
    
- Завжди використовувати `.dockerignore`, щоб не злити секрети і пришвидшити збірку.
    
      
    
- Multi-stage build використовується для зменшення фінального розміру образу та безпеки (викидаємо компілятори).
    
      