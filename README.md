# Stride — Full-Stack Personal Productivity OS

> **Stride** — інтелектуальна персональна операційна система продуктивності, яка об'єднує неструктуровані нотатки, тактичні завдання, щоденні звички та довгострокові цілі в єдиний синхронізований граф даних за допомогою LLM-пайплайна.

---

## Проблема та продуктове бачення

Сучасні цифрові інструменти продуктивності страждають від **фрагментації контексту**:

* **Нотатки** — сховище сирих думок та ідей, які не мають прямого зв'язку з діями й поступово забуваються.
* **Task managers** — ізольовані списки дій, позбавлені контексту походження та стратегічної мети.
* **Habit trackers** — відокремлені чеклісти, які не показують зв'язок між щоденною поведінкою та довгостроковими цілями.

**Stride** розв'язує цю проблему через взаємозв'язок основних сутностей системи.

Неструктурований потік думок — швидкі нотатки або транскрипти — проходить через LLM-пайплайн на базі **Gemini API**. Модель аналізує контекст і перетворює його на структуровані сутності: завдання, звички та цілі.

Таким чином, Stride створює зв'язок:

**Thought → Note → Task / Habit → Goal → Analytics**

---

# Системна архітектура

Stride побудований за принципом **Monolith First** у форматі **Modular Layered Monolith**.

Такий підхід дозволяє:

* уникнути передчасної складності мікросервісної архітектури;
* зберегти низьку latency між внутрішніми модулями;
* використовувати ACID-транзакції PostgreSQL;
* ізолювати бізнес-домен на рівні модулів;
* мати можливість виділити окремі компоненти в сервіси в майбутньому.

## System Flow

Основний потік даних між клієнтами, API, бізнес-модулями, AI-пайплайном та базою даних:

```mermaid
flowchart TD
    WEB["React Web Client"]
    MOBILE["React Native Mobile Client"]

    API["Fastify API"]

    AUTH["Identity Module<br/>Auth / JWT"]
    PROD["Productivity Module<br/>Tasks / Habits / Goals"]
    NOTES["Notes Module"]
    AI["Intelligence Module<br/>Gemini Pipeline"]

    PRISMA["Prisma ORM"]
    DB[("PostgreSQL")]

    WEB -->|HTTP REST / JSON<br/>Bearer JWT| API
    MOBILE -->|HTTP REST / JSON<br/>Bearer JWT| API

    API --> AUTH
    API --> PROD
    API --> NOTES
    API --> AI

    AUTH --> PRISMA
    PROD --> PRISMA
    NOTES --> PRISMA

    NOTES -->|Raw note| AI
    AI -->|Structured Output| PROD

    PRISMA --> DB
```

### Основний сценарій AI Pipeline

```mermaid
flowchart LR
    INPUT["User Note<br/>Free-form text"]
    API["Fastify API"]
    AI["Gemini API"]
    VALIDATE["Zod Validation"]
    TRANSACTION["Prisma Transaction"]
    DB[("PostgreSQL")]

    INPUT --> API
    API --> AI
    AI -->|Structured JSON| VALIDATE
    VALIDATE --> TRANSACTION
    TRANSACTION --> DB
```

LLM не має прямого доступу до бази даних. Модель лише генерує структурований результат, після чого backend:

1. отримує відповідь Gemini;
2. валідовує її через Zod;
3. перевіряє бізнес-правила;
4. створює або оновлює сутності;
5. виконує операції в межах атомарної транзакції PostgreSQL.

---

## Ключові архітектурні принципи

### 1. End-to-End Type Safety

Єдина мова розробки — **TypeScript**.

`Zod` використовується для runtime-валідації API-контрактів та структурованих AI-відповідей.

Спільні контракти винесені в окремий workspace:

```text
packages/contracts
```

Це дозволяє використовувати однакові схеми між frontend та backend.

### 2. Deterministic AI Extraction

AI-пайплайн використовує **Structured Outputs / JSON Schema**, щоб модель повертала передбачувану структуру даних замість довільного тексту.

LLM відповідає лише за **інтерпретацію неструктурованого вводу**.

Бізнес-правила та збереження даних залишаються відповідальністю backend.

### 3. Layered Boundaries

Кожен модуль має чіткий поділ відповідальності:

| Layer                     | Responsibility                              |
| ------------------------- | ------------------------------------------- |
| **Routes / Middleware**   | Routing, authentication, rate limiting      |
| **Controllers**           | HTTP layer, request parsing, DTO validation |
| **Services**              | Business logic and domain rules             |
| **Repositories / Prisma** | Database access                             |
| **Schemas**               | Runtime validation and API contracts        |

---

# Технологічний стек

| Layer               | Technologies                         | Purpose                                      |
| ------------------- | ------------------------------------ | -------------------------------------------- |
| **Backend Runtime** | `Node.js`, `TypeScript`              | Асинхронний runtime та type safety           |
| **HTTP Framework**  | `Fastify`                            | Lightweight HTTP API                         |
| **Database**        | `PostgreSQL`                         | Relational data storage та ACID transactions |
| **ORM**             | `Prisma`                             | Type-safe database access та migrations      |
| **AI**              | `@google/genai`                      | Gemini API integration                       |
| **Validation**      | `Zod`                                | Runtime validation та shared contracts       |
| **Web**             | `React`, `Vite`, `Tailwind CSS`      | Web dashboard                                |
| **Charts**          | `Chart.js`                           | Productivity analytics                       |
| **Mobile**          | `React Native`, `Expo`, `NativeWind` | Cross-platform mobile client                 |
| **State**           | `Zustand`                            | Client-side application state                |
| **Server State**    | `TanStack Query`                     | API caching та synchronization               |
| **DevOps**          | `Docker`, `GitHub Actions`           | Local infrastructure та CI                   |
| **Code Quality**    | `ESLint`                             | Static analysis                              |

---

# Реляційна модель даних

```mermaid
erDiagram
    USER ||--o{ GOAL : owns
    USER ||--o{ NOTE : creates
    USER ||--o{ TASK : owns
    USER ||--o{ HABIT : owns

    GOAL ||--o{ TASK : contains
    GOAL ||--o{ HABIT : contains

    NOTE ||--o{ TASK : generates

    HABIT ||--o{ HABIT_LOG : has

    USER {
        uuid id
        string email
        string passwordHash
        datetime createdAt
    }

    GOAL {
        uuid id
        uuid userId
        string title
        string description
        datetime createdAt
    }

    NOTE {
        uuid id
        uuid userId
        string content
        datetime createdAt
    }

    TASK {
        uuid id
        uuid userId
        uuid goalId
        uuid sourceNoteId
        string title
        string status
        string priority
        datetime dueDate
    }

    HABIT {
        uuid id
        uuid userId
        uuid goalId
        string name
        string frequency
    }

    HABIT_LOG {
        uuid habitId
        date date
        boolean completed
    }
```

### Основні моделі

#### `Task`

Містить складені індекси:

```text
(userId, status)
(userId, dueDate)
```

Вони оптимізують типові dashboard-запити — фільтрацію завдань за статусом та дедлайном.

`sourceNoteId` зберігає зв'язок із початковою нотаткою, з якої могла бути згенерована задача.

#### `HabitLog`

`HabitLog` зберігає факт виконання звички за конкретну дату.

Унікальність забезпечується композитним ключем:

```prisma
@@unique([habitId, date])
```

Тому немає необхідності щодня скидати стан звички через cron.

Історія виконання зберігається як окремі записи, що дозволяє розраховувати:

* streaks;
* completion rate;
* consistency;
* productivity trends.

#### `Goal`

`Goal` виступає стратегічним рівнем системи.

Завдання та звички можуть бути прив'язані до конкретної цілі, що дозволяє будувати зв'язок між щоденними діями та довгостроковими результатами.

---

# Реалізований та планований функціонал

## 1. Unified Dashboard

Єдиний інтерфейс для перегляду поточного стану продуктивності:

* пріоритетні завдання;
* завдання на сьогодні;
* звички;
* календарний контекст;
* статистика виконання;
* productivity charts.

Візуалізація виконується за допомогою `Chart.js`.

---

## 2. Task & Habit Management

### Tasks

Повноцінний CRUD:

* створення;
* редагування;
* видалення;
* зміна статусу;
* дедлайни;
* пріоритети.

Підтримувані пріоритети:

```text
LOW
MEDIUM
HIGH
URGENT
```

### Habits

Підтримуються:

* щоденні звички;
* звички за конкретними днями тижня;
* історія виконання;
* streak calculation;
* completion analytics.

---

## 3. Context-Aware AI Pipeline

Користувач вводить довільний текст у поле швидкої нотатки.

Наприклад:

```text
I need to prepare for the backend exam next Friday
and start exercising three times a week.
```

Stride передає текст до Gemini разом із системним prompt та структурованою схемою.

AI може перетворити його на структурований результат:

```text
Note
 ├── Task
 │    └── Prepare for backend exam
 │
 └── Habit
      └── Exercise 3 times per week
```

Після цього backend:

```text
Gemini
   ↓
Structured Output
   ↓
Zod Validation
   ↓
Business Rules
   ↓
Prisma Transaction
   ↓
PostgreSQL
```

Усі зміни виконуються атомарно.

---

# Структура проєкту

Stride організований як **pnpm monorepo**.

```text
stride/
├── apps/
│   ├── web/
│   │   └── # React + Vite web dashboard
│   │
│   ├── mobile/
│   │   └── # React Native + Expo mobile client
│   │
│   └── api/
│       ├── prisma/
│       │   ├── schema.prisma
│       │   └── migrations/
│       │
│       └── src/
│           ├── modules/
│           │   ├── auth/
│           │   │   ├── routes/
│           │   │   ├── controllers/
│           │   │   ├── services/
│           │   │   └── schemas/
│           │   │
│           │   ├── productivity/
│           │   │   ├── tasks/
│           │   │   ├── habits/
│           │   │   └── goals/
│           │   │
│           │   ├── notes/
│           │   │   ├── routes/
│           │   │   ├── controllers/
│           │   │   └── services/
│           │   │
│           │   └── ai/
│           │       ├── gemini/
│           │       ├── prompts/
│           │       ├── schemas/
│           │       └── services/
│           │
│           └── shared/
│               ├── db/
│               ├── errors/
│               ├── middleware/
│               └── utils/
│
├── packages/
│   └── contracts/
│       ├── schemas/
│       └── types/
│
├── docker-compose.yml
├── package.json
├── pnpm-workspace.yaml
└── turbo.json
```

### Архітектурний принцип

Структура розділена за двома рівнями:

```text
apps/
    → deployable applications

packages/
    → shared libraries and contracts
```

Backend додатково організований за **domain modules**, а не за глобальними папками типу:

```text
controllers/
services/
repositories/
```

Це дозволяє зберігати пов'язану бізнес-логіку поруч:

```text
productivity/
├── tasks/
├── habits/
└── goals/
```

замість розподілення її по всьому проєкту.

---

# Швидкий старт

## Передумови

* Node.js `>= 20.x`
* pnpm `>= 9.x`
* Docker
* Docker Compose

## 1. Клонування та встановлення

```bash
git clone https://github.com/your-username/stride.git

cd stride

pnpm install
```

## 2. Запуск PostgreSQL

```bash
docker compose up -d postgres
```

## 3. Environment variables

Створіть `.env` у `apps/api/`:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/stride_db?schema=public"

JWT_SECRET="your-super-secret-key"

GEMINI_API_KEY="your-google-gemini-api-key"

PORT=3001
```

## 4. Prisma migrations

```bash
pnpm --filter api prisma migrate dev
```

## 5. Запуск development environment

```bash
pnpm dev
```

Після запуску:

```text
Web Client
http://localhost:5173

Fastify API
http://localhost:3001

Swagger / API Docs
http://localhost:3001/documentation
```

---

# Подальший розвиток

## 1. Asynchronous AI Processing

Важкі AI-запити можуть бути винесені у background workers:

```text
API
 ↓
Redis / BullMQ
 ↓
AI Worker
 ↓
Gemini
 ↓
PostgreSQL
```

Це дозволить:

* уникнути довгих HTTP-запитів;
* краще працювати з rate limits;
* повторювати failed jobs;
* масштабувати AI processing незалежно від API.

---

## 2. Offline-First Synchronization

Mobile client може використовувати локальне сховище для миттєвого UI:

```text
Mobile UI
   ↓
Local Storage
   ↓
Background Sync
   ↓
Fastify API
   ↓
PostgreSQL
```

Для синхронізації можуть використовуватися:

* `updatedAt`;
* versioning;
* optimistic updates;
* delta synchronization.

---

## 3. Advanced AI Features

Подальший розвиток AI-рівня може включати:

* виявлення когнітивного перевантаження;
* аналіз накопичених невиконаних завдань;
* автоматичне визначення пріоритетів;
* рекомендації щодо перенесення дедлайнів;
* аналіз productivity patterns;
* персональні рекомендації щодо розподілу навантаження.

---

# Ліцензія

Проєкт розробляється в межах курсової роботи кафедри комп'ютерних наук ЧНУ.
