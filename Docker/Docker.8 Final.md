## 1. LOGGING MANAGEMENT (Захист диска) 🟥 MUST KNOW

### 1.1. The Root Cause of Disk Full

Коли застосунок пише щось у консоль (`print()`, `console.log()`), Docker перехоплює це і зберігає на диск. За замовчуванням використовується `json-file` драйвер.

  

- _The Danger:_ Якщо твій Nginx генерує 1 ГБ логів на день, за місяць файл логів одного контейнера займе 30 ГБ. Диск заповниться до 100%, і система зупиниться (No space left on device).
    
      
    

### 1.2. The Solution: Log Rotation

Ти мусиш налаштувати ротацію логів (обмеження розміру) **до** того, як випускати сервіс у production.

  

**CLI Option:**

  

Bash

```
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx
# Зберігає максимум 3 файли по 10 мегабайт. Старі видаляються автоматично.
```

**Docker Compose Option (Best Practice):**

  

YAML

```
services:
  app:
    image: my-app:v1
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 1.3. External Log Forwarding (🟪 ADVANCED)

У серйозних інфраструктурах логи не зберігають на серверах взагалі. Їх відправляють у централізовані системи (ELK, Loki) або хмару (AWS CloudWatch).

Docker підтримує різні **Logging Drivers**:

  

- `syslog` — стандартний Linux логер.
    
      
    
- `fluentd` — відправка логів агенту Fluentd.
    
      
    
- `awslogs` — пряма відправка в Amazon CloudWatch.
    
      
    

## 2. RESTART POLICIES (Автоматичне відновлення) 🟥 MUST KNOW

Ти не повинен прокидатися о 3-й ночі, якщо бекенд впав через дрібну помилку в коді. Docker може перезапускати контейнери автоматично.

  

|**Policy**|**Behavior (Як працює)**|**Best Use Case**|
|---|---|---|
|`no`|**(Default)** Не перезапускати ніколи.|Локальна розробка, разові скрипти.|
|`on-failure`|Перезапускає, тільки якщо процес впав з помилкою (Exit Code != 0). Не реагує на штатні завершення.|Обробники черг, Cron-job контейнери.|
|`always`|Завжди перезапускає, незалежно від причини падіння. Перезапустить контейнери **після ребуту самого сервера**.|**Production (Більшість сервісів)**: Web-сервери, Бази даних.|
|`unless-stopped`|Працює як `always`, але якщо ти сам сказав `docker stop`, він не підніме контейнер після ребуту сервера.|Production (більш гнучкий варіант).|

## 3. CLEANUP & MAINTENANCE (Догляд за сервером) 🟨 SHOULD KNOW

Контейнери, старі образи та невикористані томи накопичуються і з'їдають місце.

DevOps інженери часто додають у `crontab` (планувальник задач Linux) регулярне очищення системи.

  

- `docker system df` — показує, скільки дискового простору займає Docker і скільки з нього можна звільнити (Reclaimable).
    
      
    
- `docker container prune` — видаляє всі зупинені контейнери.
    
      
    
- `docker image prune -a` — видаляє всі образи, які не використовуються жодним контейнером.
    
      
    
- `docker system prune` — масова чистка (контейнери, мережі, образи).
    
      
    
- `docker system prune --volumes` — 🔴 **DANGER ZONE:** Очищає все, включно з томами баз даних, якщо контейнер був тимчасово зупинений.
    
      
    

## 4. MULTI-ARCHITECTURE BUILDS (ARM vs AMD64) 🟪 ADVANCED

- _The Context:_ Сучасні сервери (наприклад, AWS Graviton) та комп'ютери (Apple Silicon M1/M2/M3) використовують архітектуру процесора **ARM64**. Старі сервери та більшість ПК — **AMD64** (x86_64).
    
      
    
- _The Problem:_ Якщо ти зібрав Docker Image на своєму процесорі AMD64 і відправив його на сервер з ARM64 — контейнер не запуститься з помилкою `exec format error`.
    
      
    
- _The Solution:_ Використовуй `buildx`. Це сучасний плагін, який дозволяє зібрати образ для кількох архітектур одночасно.
    
      
    

Bash

```
# Збирає образ для обох архітектур і відразу пушить його в Registry
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 --push .
```

## 5. DOCKER MASTER CHEAT SHEET (Фінальний довідник)

Ось вичавка всіх найважливіших команд з 8 блоків, які формують щоденну рутину DevOps-інженера.

  

### Lifecycle & Inspect

- `docker run -d -p 80:80 --name web nginx` (Запуск у фоні з прокиданням порту)
    
      
    
- `docker exec -it web /bin/sh` (Вхід у термінал працюючого контейнера)
    
      
    
- `docker logs -f --tail 100 web` (Дивитися останні 100 рядків логів у реальному часі)
    
      
    
- `docker inspect web` (Отримати всі метадані: IP, шляхи до Volumes, налаштування)
    
      
    
- `docker stats` (Моніторинг споживання CPU та RAM у реальному часі)
    
      
    

### Images & Build

- `docker build -t myapp:latest .` (Збірка образу)
    
      
    
- `docker build --no-cache -t myapp:latest .` (Збірка без використання кешу)
    
      
    
- `docker push myapp:latest` (Відправка в Registry)
    
      
    
- `docker history myapp:latest` (Аналіз розміру шарів образу)
    
      
    

### System & Cleanup

- `docker system df` (Аналіз дискового простору)
    
      
    
- `docker system prune -a` (Глибоке очищення сервера від сміття)
    
      
    

### Docker Compose

- `docker compose up -d` (Запуск інфраструктури)
    
      
    
- `docker compose up -d --build` (Перезбірка зі зміненим кодом і запуск)
    
      
    
- `docker compose down` (Зупинка і видалення)
    
      
    
- `docker compose logs -f` (Перегляд логів усіх сервісів одночасно)