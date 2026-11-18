# Technical Specification (SPEC) — Финансовый трекер "Salary"

## 1. Стек технологий

| Уровень | Технология | Обоснование |
|---|---|---|
| **Framework** | Next.js 15 (App Router, React 19) | Server Components, Server Actions, высокая скорость, SEO/мета-теги |
| **Язык** | TypeScript 5+ | Строгая типизация, снижение риска ошибок с денежными вычислениями |
| **База данных** | Turso (LibSQL Cloud SQLite) | Бесплатный щедрый тариф, молниеносная скорость SQLite, облачная репликация |
| **ORM** | Drizzle ORM | Идеальная синергия с `@libsql/client`, нулевой оверхед, типобезопасный SQL-подобный синтаксис |
| **Стилизация** | Tailwind CSS + Shadcn UI (Radix Primitives) | Доступные, современные, легко кастомизируемые компоненты, поддержка темной темы |
| **Иконки** | Lucide React | Легковесные аккуратные векторные иконки для категорий и интерфейса |
| **Графика / Чарты** | Recharts (или Tremor / Chart.js) | Гибкие интерактивные SVG-графики с поддержкой анимаций и подсказок (tooltips) |
| **Аутентификация** | NextAuth.js v5 (Auth.js) или Better-Auth / JWT Session | Сессии с сохранением в Turso, безопасные HTTP-only cookies |
| **Валидация** | Zod + React Hook Form | Сквозная валидация схем на клиенте и сервере |
| **Форматирование валют** | Intl.NumberFormat | Корректный вывод чисел, валют (RUB, USD, EUR и др.) и дат (date-fns / Day.js) |

---

## 2. Архитектура приложения

```
salary/
├── src/
│   ├── app/                      # Next.js App Router
│   │   ├── (auth)/               # Группа роутов аутентификации
│   │   │   ├── login/page.tsx
│   │   │   └── register/page.tsx
│   │   ├── (dashboard)/          # Защищенная рабочая зона
│   │   │   ├── layout.tsx        # Боковое меню / мобильный таббар
│   │   │   ├── page.tsx          # Главный дашборд с балансом и быстрыми действиями
│   │   │   ├── transactions/     # Список и фильтры транзакций
│   │   │   ├── analytics/        # Развернутые графики и отчеты
│   │   │   ├── categories/       # Управление категориями
│   │   │   └── settings/         # Профиль, валюта, тема
│   │   ├── api/                  # API endpoints (при необходимости)
│   │   ├── layout.tsx            # Корневой layout с провайдерами
│   │   └── globals.css           # Tailwind стили и переменные темы
│   ├── components/               # UI компоненты
│   │   ├── ui/                   # Shadcn UI (Button, Dialog, Card, Input, etc.)
│   │   ├── charts/               # Компоненты графиков (ExpenseDonut, CashflowBar)
│   │   ├── forms/                # Формы транзакций, категорий, профиля
│   │   └── layout/               # Header, Sidebar, MobileNav
│   ├── db/                       # Конфигурация базы данных
│   │   ├── schema/               # Drizzle схемы таблиц
│   │   │   ├── users.ts
│   │   │   ├── categories.ts
│   │   │   ├── transactions.ts
│   │   │   └── budgets.ts
│   │   ├── index.ts              # Подключение LibSQL клиента
│   │   └── migrate.ts            # Скрипт миграций
│   ├── actions/                  # Server Actions (мутации данных)
│   │   ├── auth.ts
│   │   ├── transactions.ts
│   │   ├── categories.ts
│   │   └── analytics.ts
│   ├── lib/                      # Вспомогательные утилиты
│   │   ├── utils.ts              # cn (clsx + tailwind-merge), форматирование валюты
│   │   ├── auth.ts               # Конфигурация аутентификации
│   │   └── constants.ts          # Дефолтные категории, цвета, валюты
│   └── types/                    # Общие интерфейсы TypeScript
├── drizzle.config.ts             # Конфиг Drizzle Kit
├── .env.example                  # Образец переменных окружения
├── package.json
└── tsconfig.json
```

---

## 3. Схема базы данных (Turso / Drizzle)

### 3.1. Таблица `users`
- `id`: `text` (UUID / cuid2), Primary Key
- `name`: `text`
- `email`: `text`, Unique, Not Null
- `password_hash`: `text`, Not Null
- `currency`: `text`, Default `'RUB'`
- `created_at`: `integer` (timestamp), Default current
- `updated_at`: `integer` (timestamp)

### 3.2. Таблица `categories`
- `id`: `text`, Primary Key
- `user_id`: `text` (FK -> users.id, nullable: `null` для глобальных дефолтных категорий)
- `name`: `text`, Not Null
- `type`: `text` (enum: `'income'`, `'expense'`), Not Null
- `icon`: `text` (имя иконки Lucide, например `'shopping-cart'`, `'wallet'`, `'coffee'`)
- `color`: `text` (HEX или класс Tailwind, например `'#10B981'`, `'#EF4444'`)
- `is_default`: `integer` (boolean 0/1), Default 0
- `created_at`: `integer` (timestamp)

### 3.3. Таблица `transactions`
- `id`: `text`, Primary Key
- `user_id`: `text` (FK -> users.id, cascade delete), Not Null, Index
- `category_id`: `text` (FK -> categories.id), Not Null, Index
- `amount`: `integer` (сумма хранится в минимальных единицах / копейках/центах, чтобы исключить ошибки float: 100 руб. = 10000)
- `type`: `text` (enum: `'income'`, `'expense'`), Not Null
- `date`: `integer` (Unix timestamp даты операции), Not Null, Index
- `description`: `text` (опциональный комментарий)
- `created_at`: `integer` (timestamp)
- `updated_at`: `integer` (timestamp)

### 3.4. Таблица `budgets` (лимиты расходов)
- `id`: `text`, Primary Key
- `user_id`: `text` (FK -> users.id), Not Null
- `category_id`: `text` (FK -> categories.id), Not Null
- `limit_amount`: `integer` (в копейках)
- `month`: `integer` (1-12)
- `year`: `integer` (например 2026)

---

## 4. Спецификация серверных действий (Server Actions / API)

### 4.1. Аутентификация (`actions/auth.ts`)
- `registerUser(formData)`: валидация Zod, проверка уникальности email, хеширование пароля, создание дефолтных категорий для пользователя, открытие сессии.
- `loginUser(formData)`: валидация, сверка пароля, создание session token (HTTP-only cookie).
- `logoutUser()`: очистка cookie сессии.

### 4.2. Транзакции (`actions/transactions.ts`)
- `createTransaction(data)`:
  - Input: `{ type, amount, categoryId, date, description }`
  - Revalidate tag: `transactions`, `analytics`
- `updateTransaction(id, data)`: обновление выбранной записи.
- `deleteTransaction(id)`: удаление с проверкой прав владения (`user_id`).
- `getTransactions(filters)`:
  - Фильтры: `startDate`, `endDate`, `type`, `categoryId`, `searchQuery`, `page`, `limit`.

### 4.3. Аналитика (`actions/analytics.ts`)
- `getDashboardSummary(period)`:
  - Вычисляет: Суммарный доход, Суммарный расход, Чистый баланс, Дельту в % по сравнению с прошлым периодом.
- `getCategoryExpensesBreakdown(period)`:
  - Группировка трат по `category_id`, подсчет процентного соотношения для Donut Chart.
- `getCashflowTimeline(period, grouping: 'day' | 'month')`:
  - Временной ряд (доход vs расход по датам) для построения Bar/Area Chart.

---

## 5. Безопасность и хранение секретов
- Переменные окружения (`.env`):
  - `TURSO_DATABASE_URL`: `libsql://your-db-org.turso.io`
  - `TURSO_AUTH_TOKEN`: токен доступа к БД
  - `AUTH_SECRET`: секретный ключ для шифрования сессий и JWT
- Все SQL-запросы параметризованы через Drizzle ORM (защита от SQL Injection).
- Все формы экранируются и типизируются Zod.
- Все операции выборки/модификации жестко фильтруются по `user_id` текущей сессии.
