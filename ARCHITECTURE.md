# Архитектура системы ботов — схемы по этапам

Связка с [ROADMAP.md](./ROADMAP.md). Каждая схема — mermaid (рендерится на GitHub автоматически).

---

## 1. Общая архитектура целевой системы (все этапы вместе)

```mermaid
flowchart TB
    subgraph ADMIN["Управление (Этап 4)"]
        UI["Веб-панель<br/>Go templates + htmx"]
        API["REST API<br/>chi/gin"]
        PROM["Prometheus + Grafana<br/>метрики, алерты"]
        TG["Telegram-алерты<br/>волны банов"]
    end

    subgraph CORE["Ядро (Этап 2)"]
        DISP["Диспетчер задач<br/>rate-limit: аккаунт/IP/платформа"]
        Q[("Очередь<br/>NATS JetStream / Redis+asynq")]
        DB[("PostgreSQL<br/>task, account, proxy,<br/>worker, result")]
        DLQ["Dead-letter queue<br/>«отравленные» задачи"]
    end

    subgraph WORKERS["Боты-воркеры Go, stateless (Этап 1→2→6)"]
        W1["Worker #1"]
        W2["Worker #2"]
        WN["Worker #N ..."]
    end

    subgraph ANTIBAN["Антибан-слой (Этап 3)"]
        ACC["Пул аккаунтов<br/>ротация, карантин"]
        PRX["Пул резидентных прокси<br/>учёт качества IP"]
        BEH["Хаотизатор поведения<br/>jitter, паузы"]
        CAP["Решатель капчи<br/>2captcha/CapMonster"]
    end

    SOC[("Целевая соцсеть<br/>VK / YouTube / TikTok...")]

    UI --> API --> DISP
    DISP --> Q --> W1 & W2 & WN
    DISP <--> DB
    W1 & W2 & WN <--> ANTIBAN
    W1 & W2 & WN --> SOC
    W1 & W2 & WN -- "результат / ошибка" --> DB
    W1 -.-> DLQ
    PROM --- CORE & WORKERS
    PROM --> TG
```

**Как читать:** панель → диспетчер кладёт задачи в очередь → N одинаковых воркеров тянут их, перед действием берут «чистый» аккаунт + прокси → бьют в соцсеть → результат возвращается в БД; метрики и алерты идут отдельным контуром.

---

## 2. Жизненный цикл одной задачи (что происходит внутри системы)

```mermaid
sequenceDiagram
    participant P as Панель/API
    participant D as Диспетчер
    participant Q as Очередь (JetStream)
    participant W as Worker (Go)
    participant A as Антибан-слой
    participant S as Соцсеть
    participant B as PostgreSQL

    P->>D: Новая задача (просмотр X)
    D->>B: Проверить идемпотентность (не дубль?)
    D->>Q: publish(task_id) c учётом лимитов
    W->>Q: pull(task)
    W->>A: Запросить аккаунт + прокси (свободные, не в карантине)
    A-->>W: creds + proxy + поведенческий профиль
    W->>S: Выполнить действие (API/браузер)
    alt Успех
        S-->>W: 200 OK
        W->>B: result=ok, +1 к счётчику аккаунта
    else Ошибка/подозрение
        S-->>W: 429 / captcha / ban
        W->>A: Карантин аккаунта/IP
        W->>Q: retry с backoff (или → dead-letter после N попыток)
    end
    W->>Q: ack(task)
```

---

## 3. Эволюция системы по этапам (roadmap как рост архитектуры)

```mermaid
flowchart LR
    E0["Этап 0<br/>Ручной MVP<br/>5–10 аккаунтов,<br/>скрипт 200 строк"]
    E1["Этап 1<br/>1 воркер Go<br/>SQLite, cron,<br/>Docker, VPS"]
    E2["Этап 2<br/>Очередь + диспетчер<br/>10–50 воркеров,<br/>Postgres"]
    E3["Этап 3<br/>Антибан-слой<br/>пулы, карантин,<br/>капча, алерты"]
    E4["Этап 4<br/>Панель + метрики<br/>Grafana, аудит"]
    E5["Этап 5<br/>Мультипользователь<br/>JWT, тенанты, квоты"]
    E6["Этап 6<br/>Продакшн-IaC<br/>Terraform, CI/CD,<br/>Nomad/K8s → физик"]

    E0 --> E1 --> E2 --> E4
    E2 --> E3
    E3 -.параллельно, постоянно.-> E2
    E4 --> E5
    E4 --> E6
    E6 ==>|"при >5 серверов"| PHYS["Физический сервер<br/>в data-center"]
```

---

## 4. Инфраструктура: облако → рост → физический сервер

```mermaid
flowchart TB
    subgraph S1["Старт: VPS-облако (~10–20 €/мес)<br/>Этапы 0–2"]
        V1["VPS #1: 2–4 CPU / 8 GB<br/>docker-compose:<br/>worker×N + NATS + Postgres"]
    end

    subgraph S2["Рост: dedicated ~40–70 €/мес<br/>Этапы 3–4"]
        D1["Dedicated #1<br/>NATS + Postgres + панель"]
        D2["Dedicated #2..K<br/>только воркеры<br/>(scale = добавить машину)"]
    end

    subgraph S3["Физический сервер<br/>когда ≥50–100 браузерных сессий"]
        PHY["Свой сервер в DC (DE/FI/NL)<br/>много RAM + NVMe, канал 1 Gbps<br/>Nomad/K8s поверх"]
    end

    PX["💰 Прокси — всегда внешние<br/>резидентные (ISP),<br/>2–10× дороже сервера"]

    S1 ==>|"Terraform: сменить provider,<br/>код не трогаем"| S2
    S2 ==>|"окупилось по трафику/RAM"| S3
    PX -.-> S1 & S2 & S3
```

Ключевая идея: **stateless-воркеры** — масштабирование это просто «+1 контейнер / +1 машина», а переписывание под физик сведено к смене Terraform-provider, потому что IaC заведён с этапа 1.

---

## 5. Что покупать vs что строить своими руками

```mermaid
flowchart TB
    ROOT{"Что нужно<br/>системе?"}
    ROOT --> BUY["✅ КУПИТЬ / арендовать"]
    ROOT --> BUILD["🔨 СТРОИТЬ самим"]
    ROOT --> SKIP["❌ НЕ покупать"]

    BUY --> B1["Резидентные прокси<br/>(главный дефицит)"]
    BUY --> B2["Решатель капчи<br/>(2captcha API)"]
    BUY --> B3["Серверы/VPS"]
    BUY --> B4["Open-source кирпичики:<br/>asynq, chromedp,<br/>playwright-go, vkapi"]

    BUILD --> S1["Диспетчер + очереди<br/>(Go, NATS)"]
    BUILD --> S2["Воркеры-боты"]
    BUILD --> S3["Антибан-логика,<br/>пулы аккаунтов/IP"]
    BUILD --> S4["Панель + метрики"]

    SKIP --> X1["Готовые «базы ботов»<br/>с форумов: закрытый код,<br/>баны пачками, бэкдоры"]
    SKIP --> X2["SMM-панели как основа:<br/>нет контроля, маржа<br/>у посредника"]
```
