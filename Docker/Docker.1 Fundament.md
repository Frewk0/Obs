# Docker

> [!abstract] Mental Model  
> **Container ≠ Virtual Machine.**  
> Контейнер — це процес (або дерево процесів), якому Linux надає ізольований простір і контроль ресурсів.
> 
> Основна модель:
> 
> `Docker CLI → dockerd → containerd → OCI runtime → Linux Kernel`
> 
> `Namespaces → isolation`  
> `cgroups → resource control`  
> `Capabilities → privilege control`  
> `OverlayFS/storage driver → layered filesystem`

---

# 🟥 1. What is Docker?

**Docker** — платформа для пакування, доставки та запуску застосунків у контейнерах.

Контейнеризований застосунок може містити:

- application code;
    
- runtime;
    
- dependencies;
    
- libraries;
    
- потрібні файли та конфігурацію.
    

Docker сам по собі **не є віртуальною машиною і не є гіпервізором**. На Linux він використовує механізми самого Linux kernel.

> [!important]  
> Docker не гарантує, що контейнер буде буквально ідентично працювати в будь-якому середовищі. Його перевага в тому, що userspace-залежності та середовище застосунку стандартизовані, а kernel все одно надається хостом.

---

# 🟥 2. Чому DevOps використовує Docker?

## 2.1. Environment consistency

Проблема:

```text
Developer
  ↓
macOS
Python version A

Staging
  ↓
Ubuntu
Python version B

Production
  ↓
RHEL
Python version C
```

Через це можуть виникати проблеми із:

- версіями runtime;
    
- бібліотеками;
    
- системними пакетами;
    
- конфігураціями;
    
- залежностями.
    

Docker дозволяє пакувати userspace-залежності разом із застосунком у стандартний image.

---

## 2.2. Dependency isolation

На одному хості можуть працювати:

```text
Application A → Python 3.x
Application B → Node.js
Application C → Java
```

Контейнери ізолюють їхні середовища, тому вони менше конфліктують між собою.

---

## 2.3. Менший overhead, ніж VM

### VM

```text
Host
 ├── Hypervisor
 ├── VM 1
 │    └── Guest OS
 ├── VM 2
 │    └── Guest OS
 └── VM 3
      └── Guest OS
```

### Container

```text
Host Linux Kernel
 ├── Container 1 → process
 ├── Container 2 → process
 └── Container 3 → process
```

Контейнер не запускає окремий kernel, тому overhead зазвичай значно менший за VM.

> [!warning]  
> Не треба запам'ятовувати "container launch = milliseconds" або "overhead = 0%". Швидкість та overhead залежать від середовища, image, storage, networking і конкретного застосунку.

---

# 🧠 3. Mental Model: Container ≠ VM

## Неправильна модель

> Контейнер — це маленька віртуальна машина.

Ця модель веде до неправильних рішень:

- намагатися ставити SSH server;
    
- запускати повноцінний init/systemd без необхідності;
    
- поводитися з контейнером як із постійним сервером;
    
- зберігати state без persistence mechanism.
    

## Правильна модель

> **Контейнер — це ізольований процес або група процесів.**

Наприклад:

```text
Host
├── nginx process
├── postgres process
├── systemd
└── docker container process
```

Процес усередині контейнера бачить обмежене середовище, створене kernel mechanisms.

---

# 🟥 4. Docker Architecture

Docker використовує client-server model.

```text
┌───────────────┐
│ Docker CLI    │
└───────┬───────┘
        │ API request
        ▼
┌───────────────┐
│ dockerd       │
│ Docker daemon │
└───────┬───────┘
        ▼
┌───────────────┐
│ containerd    │
└───────┬───────┘
        ▼
┌───────────────┐
│ OCI runtime   │
│ e.g. runc     │
└───────┬───────┘
        ▼
┌───────────────┐
│ Linux Kernel  │
└───────────────┘
```

Окремо:

```text
Registry
   ↑
   │ pull / push
   │
Docker
```

---

# 🔑 5. Основні компоненти

## Docker CLI

Команда:

```bash
docker
```

CLI приймає твої команди та взаємодіє з Docker daemon через API.

Приклад:

```bash
docker run nginx
```

CLI **не запускає контейнер напряму**. Воно передає запит daemon.

---

## Docker API

Інтерфейс, через який клієнт взаємодіє з daemon.

Локально Docker часто використовує Unix socket:

```text
/var/run/docker.sock
```

Docker також може працювати через network endpoint залежно від configuration.

> [!warning]  
> Доступ до Docker socket є дуже привілейованим. Контроль доступу до нього — важлива security тема.

---

## dockerd

Docker daemon.

Відповідає за високорівневе керування Docker resources:

- containers;
    
- images;
    
- networks;
    
- volumes;
    
- registries.
    

---

## containerd

Container runtime component, який відповідає за lifecycle контейнерів і взаємодію з нижчим рівнем runtime.

---

## OCI runtime

Runtime, який безпосередньо створює execution environment контейнера.

Типовий приклад:

```text
runc
```

OCI = **Open Container Initiative**.

---

## Registry

Сховище Docker images.

Приклади:

- Docker Hub;
    
- Amazon ECR;
    
- GitHub Container Registry;
    
- GitLab Container Registry.
    

Основні операції:

```bash
docker pull ...
docker push ...
```

---

# 🟪 6. Що відбувається при `docker run`

Приклад:

```bash
docker run nginx
```

Спрощена модель:

```text
docker CLI
   ↓
Docker API
   ↓
dockerd
   ↓
containerd
   ↓
OCI runtime
   ↓
Linux kernel
```

## Крок 1 — CLI

Ти вводиш:

```bash
docker run nginx
```

CLI відправляє запит daemon.

## Крок 2 — daemon

`dockerd` перевіряє, чи є потрібний image локально.

Якщо image немає:

```text
local image?
   ├── yes → use it
   └── no  → pull from registry
```

## Крок 3 — container lifecycle

Docker/containerd готують container environment.

## Крок 4 — OCI runtime

Runtime створює потрібне execution environment та запускає container process.

## Крок 5 — Linux kernel

Kernel застосовує механізми:

- namespaces;
    
- cgroups;
    
- capabilities;
    
- mounts;
    
- networking controls.
    

---

# 🟥 7. Linux Namespaces

Namespaces відповідають за **ізоляцію того, що процес бачить**.

## PID namespace

Ізолює process ID space.

Наприклад:

```text
Host:
PID 14592 → nginx

Container:
PID 1 → nginx
```

Той самий process може мати різні PID у різних namespace.

Контейнер бачить лише доступні йому процеси.

---

## NET namespace

Ізолює network view:

- network interfaces;
    
- IP addresses;
    
- routes;
    
- ports;
    
- network namespace state.
    

Контейнер може мати власний interface:

```text
eth0
```

---

## MNT namespace

Ізолює mount view.

Це допомагає контейнеру мати власне filesystem view:

```text
/
├── bin
├── etc
├── usr
└── app
```

---

## UTS namespace

Дозволяє мати окремий hostname.

---

## IPC namespace

Ізолює певні механізми inter-process communication.

---

## USER namespace

Дозволяє мапити UID/GID між namespace.

Це особливо важливо для rootless/container security scenarios.

---

# 🟥 8. cgroups

**cgroups (control groups)** — механізм Linux для контролю та обліку ресурсів процесів.

Використовується для:

- memory;
    
- CPU;
    
- I/O;
    
- інших ресурсних обмежень.
    

Ментальна модель:

```text
Namespaces
→ що процес може бачити

cgroups
→ скільки ресурсів процес може використовувати
```

---

## Memory

Можна встановлювати memory limits.

Якщо процеси потрапляють у memory pressure і memory limit перевищений, можливий OOM kill залежно від конфігурації та ситуації.

---

## CPU

Можна контролювати:

- CPU quota;
    
- CPU shares/weight;
    
- доступ до CPU.
    

---

## I/O

Можливий контроль I/O resource usage залежно від storage/runtime/kernel configuration.

---

# 🟥 9. Linux Capabilities

У Linux `root` не є єдиним неподільним privilege.

Privileges розбиті на **capabilities**.

Приклади:

```text
CAP_NET_ADMIN
CAP_SYS_ADMIN
CAP_SYS_CHROOT
...
```

Docker зазвичай запускає контейнер із обмеженим набором capabilities порівняно з повним host root.

Це зменшує privilege level контейнерного процесу.

> [!important]  
> `root` усередині контейнера не означає автоматично "повний root над host", але root container process все одно є серйозним security risk, особливо разом із небезпечними capabilities, mounts, `--privileged` та іншими налаштуваннями.

---

# 🟨 10. Layered Filesystem

Docker images складаються з **layers**.

Спрощена модель:

```text
Application layer
        ↓
Dependencies layer
        ↓
Base image layers
```

Image layers переважно read-only.

Під час запуску container використовує writable layer поверх image layers.

```text
┌────────────────────────────┐
│ Container writable layer   │  ← RW
├────────────────────────────┤
│ Image layer                │
├────────────────────────────┤
│ Image layer                │
├────────────────────────────┤
│ Base image layer           │
└────────────────────────────┘
```

---

## Copy-on-Write

Якщо контейнер змінює файл, який походить із read-only layer, storage driver може створити копію у writable layer і змінювати її.

Тому:

```text
Image
→ immutable base

Container
→ image + writable state
```

> [!important]  
> Це simplified mental model. Конкретна storage implementation залежить від Docker storage driver.

---

# 🔑 11. Core Concepts

## Image

Image — immutable artifact, який містить filesystem content + metadata.

```text
Image
├── layers
├── metadata
└── configuration
```

---

## Container

Container — створений із image runtime instance.

Спрощено:

```text
Container
=
Image
+
Writable layer
+
Runtime configuration
+
Isolation
```

---

## Volume

Механізм persistence, який має lifecycle окремо від container writable layer.
Щоб дані жили **незалежно** від контейнера, ми використовуємо **Volumes** (томи).
Використовується для даних, які повинні пережити recreation container.

Наприклад:

```text
PostgreSQL
    ↓
Volume
    ↓
persistent data
```

---

## Network

Docker networking дозволяє контейнерам взаємодіяти:

```text
Container A
     ↕
 Docker Network
     ↕
Container B
```

У user-defined networks Docker також надає DNS-based service discovery.

---

# 🟥 12. Image vs Container

Запам'ятати:

```text
IMAGE
→ template / immutable artifact

CONTAINER
→ runtime instance
```

Аналогія:

```text
Class
 ↓
Object
```

але не сприймай цю аналогію буквально.

---

# 🟥 13. Container Lifecycle

Основна логіка:

```text
Image
 ↓
create
 ↓
start
 ↓
running
 ↓
stop
 ↓
stopped
 ↓
rm
 ↓
deleted
```

Важлива різниця:

```text
stop ≠ delete
```

Після `stop` container існує.

Після `rm` container object видаляється.

Writable layer container при видаленні контейнера також видаляється.

Persistence mechanism, наприклад volume, може залишитися.

---

# 🟨 14. Docker та Linux — зв'язок

```text
Docker
  ↓
container runtime
  ↓
Linux kernel
  ├── namespaces
  ├── cgroups
  ├── capabilities
  ├── mounts
  ├── networking
  └── filesystem/storage
```

Це одна з найважливіших DevOps mental models.

---

# 🟦 15. Docker vs VM

|Docker Container|Virtual Machine|
|---|---|
|Shared host kernel|Separate guest kernel|
|Process isolation|Hardware/VM isolation|
|Usually lower overhead|Usually higher overhead|
|Fast startup|Usually slower startup|
|Containers share kernel|VMs have own OS kernel|
|Images are container artifacts|VM images contain guest OS|

> [!important]  
> Контейнерна ізоляція та VM-ізоляція — не одне й те саме. VM зазвичай забезпечує сильнішу межу ізоляції завдяки окремому guest kernel.

---

# 🟦 16. Docker Desktop

На Linux Docker працює нативно поверх Linux kernel.

На macOS та Windows desktop Docker Desktop потребує Linux environment для Linux containers, тому використовує VM/virtualization layer під капотом.

Спрощено:

```text
macOS / Windows
      ↓
Docker Desktop
      ↓
Linux VM
      ↓
Linux kernel
      ↓
Containers
```

На Windows також існують native Windows containers, тому не треба змішувати Linux-container model і Windows-container model.

---

# 🟦 17. Version Notes

## Docker vs Moby

**Moby** — open-source project/ecosystem, на якому базуються компоненти Docker Engine.

Не плутати:

```text
Docker
→ product / ecosystem

Moby
→ open-source project
```

## cgroups v1 vs v2

Сучасні Linux systems все частіше використовують **cgroup v2**.

Docker підтримує cgroup v2 у сучасних версіях Linux environments.

> [!warning]  
> Не прив'язуй cgroup v2 лише до конкретної версії дистрибутива. Потрібно дивитися фактичну конфігурацію системи.

Перевіряти можна, наприклад:

```bash
stat -fc %T /sys/fs/cgroup
```

та через Docker/system information.

---

# 🏭 18. Production Mental Model

Типовий flow:

```text
Developer
   ↓
Git
   ↓
Dockerfile
   ↓
docker build
   ↓
Image
   ↓
Registry
   ↓
Deployment
   ↓
Container runtime
   ↓
Application
   ↓
Monitoring / Logging
```

Контейнер зазвичай розглядається як **ephemeral workload**.

Stateful data повинні бути винесені в відповідний persistence mechanism.

---

# ⚠️ 19. Common Mistakes

## Mistake 1 — "Container = VM"

Неправильно.

Контейнер — процес із isolation/resource controls.

---

## Mistake 2 — зберігати важливі дані тільки у writable layer

Після recreation/removal container ці дані можуть бути втрачені.

Для persistence використовують volumes, bind mounts або інші відповідні storage mechanisms.

---

## Mistake 3 — вважати root у контейнері harmless

Root container process може створювати серйозні security risks.

---

## Mistake 4 — вважати Docker socket звичайним файлом

```text
/var/run/docker.sock
```

Доступ до нього фактично дає дуже високі привілеї над Docker Engine.

---

## Mistake 5 — вчити oversimplified internals

Модель:

```text
dockerd → containerd → runc → kernel
```

корисна, але реальна реалізація має більше компонентів та залежить від версії й конфігурації.

---

# 🔗 20. Connections

## Docker ← Linux

Docker використовує:

```text
namespaces
cgroups
capabilities
mounts
networking
filesystem/storage
```

## Docker → CI/CD

```text
Git
 ↓
CI
 ↓
docker build
 ↓
image
 ↓
registry
```

## Docker → Kubernetes

```text
Docker/container image
        ↓
Container runtime
        ↓
Kubernetes
        ↓
Pods / Workloads
```

> [!important]  
> Kubernetes не слід мислити як "Docker для великої кількості контейнерів". Kubernetes — orchestration system, яка працює з container images через container runtime.

---

# 🧠 21. MEMORY CORE

## Must remember

1. **Container ≠ VM.**
    
2. Container — це процес/група процесів ізоляції.
    
3. `Namespaces` → isolation.
    
4. `cgroups` → resource control/accounting.
    
5. `Capabilities` → privilege control.
    
6. Image → immutable artifact.
    
7. Container → runtime instance of an image.
    
8. Container writable layer — ephemeral.
    
9. Persistent data потребує окремого storage mechanism.
    
10. `dockerd` керує Docker Engine.
    
11. `containerd` бере участь у lifecycle management.
    
12. OCI runtime, наприклад `runc`, запускає контейнерний process environment.
    
13. Linux kernel забезпечує базові механізми ізоляції.
    

---

# 🧠 22. Що треба вміти пояснити на співбесіді

### "Що таке Docker?"

> Docker дозволяє пакувати application та його userspace dependencies у container image і запускати його як ізольований workload, використовуючи механізми ОС.

### "Контейнер — це VM?"

> Ні. Linux container — це процес або група процесів, ізольованих через kernel mechanisms, зокрема namespaces та resource controls через cgroups.

### "Що роблять namespaces?"

> Ізолюють view процесу на системні ресурси, наприклад processes, networking, mounts та hostname.

### "Що роблять cgroups?"

> Контролюють та обліковують використання ресурсів процесами, наприклад CPU та memory.

### "Чому контейнер не вважається VM?"

> Він не запускає окремий guest kernel. Контейнери використовують kernel хоста.

---

# 🧠 23. Що пам'ятати vs що дивитися

## Пам'ятати

```text
Image ≠ Container
Container ≠ VM
Namespaces → isolation
cgroups → resource control
Capabilities → privileges
Volume → persistence
```

## Відновлювати з документації

- рідкісні CLI flags;
    
- повний список Linux capabilities;
    
- точні OCI details;
    
- storage-driver internals;
    
- cgroup implementation details;
    
- version-specific behavior.
    

---

# 📋 24. CHEAT SHEET

```text
DOCKER
│
├── Image
│   └── immutable artifact
│
├── Container
│   └── runtime instance
│
├── Docker CLI
│   └── sends API requests
│
├── dockerd
│   └── Docker Engine daemon
│
├── containerd
│   └── container lifecycle/runtime management
│
├── OCI runtime
│   └── e.g. runc
│
└── Linux Kernel
    ├── namespaces
    ├── cgroups
    ├── capabilities
    ├── mounts
    └── networking
```

### Core mental model

```text
Docker
 ↓
runtime
 ↓
Linux kernel
 ↓
process
```

### Isolation

```text
Namespaces
```

### Resources

```text
cgroups
```

### Privileges

```text
Capabilities
```

### Filesystem

```text
Image layers
+
Writable container layer
```

### Persistence

```text
Volume / Bind Mount / other storage
```

### Distribution

```text
Image
 ↓
Registry
 ↓
Pull
 ↓
Run
```

---

# 🔍 25. Troubleshooting Mental Model

Коли контейнер поводиться неправильно, не починай із випадкового набору команд.

Мисли по шарах:

```text
Application
   ↓
Container process
   ↓
Container configuration
   ↓
Runtime
   ↓
Networking
   ↓
Storage
   ↓
Host resources
   ↓
Linux kernel
```

При проблемі спитай:

1. Чи існує container?
    
2. Чи він running?
    
3. Який process запущений?
    
4. Який exit status?
    
5. Що кажуть logs?
    
6. Яка configuration?
    
7. Чи доступна потрібна network?
    
8. Чи є доступ до storage?
    
9. Чи вистачає CPU/RAM/disk?
    
10. Чи проблема в application або infrastructure?
    

---

# 🏭 26. Production Checklist

Перед production deployment перевір:

-  Image має контрольовану версію/tag.
    
-  Не використовуються secrets без потреби в image.
    
-  Container не працює як root без необхідності.
    
-  Exposed ports мінімальні.
    
-  Volumes правильно налаштовані.
    
-  Logs контролюються.
    
-  Resource limits продумані.
    
-  Healthcheck потрібний та налаштований.
    
-  Restart behavior зрозумілий.
    
-  Image source/registry trusted.
    
-  Monitoring та alerting присутні.
    
-  Backup/recovery продумані для stateful data.
    

---