# Диаграмма последовательности: UC-001 Регистрация пользователя

**Дата создания:** 2024-11-08
**Описание:** Детальный флоу регистрации пользователя

Эта диаграмма показывает последовательность взаимодействий при регистрации.

---

## Основной сценарий (Happy Path)

```mermaid
sequenceDiagram
    participant U as 👤 Пользователь
    participant UI as 🖥️ Интерфейс
    participant API as ⚙️ API Server
    participant DB as 🗄️ База данных
    participant Email as 📧 Email Service

    autonumber

    U->>UI: Нажимает "Регистрация"
    UI->>U: Показывает форму

    U->>UI: Заполняет данные и отправляет
    UI->>UI: Валидация на клиенте

    UI->>API: POST /api/register<br/>{email, name, password}

    API->>API: Валидация данных
    API->>DB: SELECT email FROM users
    DB-->>API: Email не найден

    API->>API: Хеширование пароля (bcrypt)
    API->>DB: INSERT новый пользователь
    DB-->>API: ID пользователя

    API->>API: Генерация токена подтверждения
    API->>DB: INSERT токен подтверждения

    API->>Email: Отправить письмо подтверждения
    Email-->>API: Письмо отправлено

    API-->>UI: 201 Created {userId, message}
    UI-->>U: "Проверьте вашу почту!"

    Note over U,Email: Пользователь переходит по ссылке из письма

    U->>UI: Клик по ссылке с токеном
    UI->>API: GET /api/confirm/{token}
    API->>DB: SELECT token, проверка срока
    DB-->>API: Токен валиден

    API->>DB: UPDATE user SET active=true
    API->>DB: DELETE token

    API-->>UI: 200 OK
    UI-->>U: "Email подтвержден! Можете войти"
```

---

## Альтернативный сценарий: Email уже существует

```mermaid
sequenceDiagram
    participant U as 👤 Пользователь
    participant UI as 🖥️ Интерфейс
    participant API as ⚙️ API Server
    participant DB as 🗄️ База данных

    U->>UI: Заполняет форму с существующим email
    UI->>API: POST /api/register

    API->>DB: SELECT email FROM users
    DB-->>API: ❌ Email найден!

    API-->>UI: 409 Conflict<br/>{error: "Email already exists"}
    UI-->>U: ❌ "Пользователь уже существует"<br/>Предложить: Войти | Восстановить пароль
```

---

## Исключительная ситуация: Ошибка email сервиса

```mermaid
sequenceDiagram
    participant U as 👤 Пользователь
    participant UI as 🖥️ Интерфейс
    participant API as ⚙️ API Server
    participant DB as 🗄️ База данных
    participant Email as 📧 Email Service

    U->>UI: Отправляет форму регистрации
    UI->>API: POST /api/register

    API->>DB: Создание пользователя
    DB-->>API: ✅ Создан

    API->>Email: Отправить письмо
    Email-->>API: ❌ Ошибка (503 Service Unavailable)

    API->>DB: UPDATE user SET email_pending=true
    API->>API: Логирование ошибки

    API-->>UI: 200 OK (partial)<br/>{warning: "Email pending"}
    UI-->>U: ⚠️ "Регистрация завершена,<br/>но письмо не отправлено.<br/>Попробуйте повторить позже"

    Note over API: Background job попытается<br/>отправить письмо позже
```

---

## Ключевые моменты

### Безопасность:
1. Пароль хешируется на сервере (bcrypt)
2. Токен подтверждения криптографически безопасен
3. Токен имеет срок действия (24 часа)

### Надежность:
1. Транзакционность операций с БД
2. Обработка ошибок email-сервиса
3. Валидация на клиенте и сервере

### Производительность:
1. Хеширование пароля асинхронно
2. Отправка email в фоновом режиме (опционально)

---

## Связанные документы

- **[UC-001: Регистрация пользователя](../../use-cases/UC-001-example.md)** - полное описание юз кейса
- **[REQ-F-001: Регистрация через email](../../requirements/functional/REQ-F-001-example.md)** - функциональное требование
- **[REQ-NF-001: Безопасность паролей](../../requirements/non-functional/REQ-NF-001-example.md)** - требование безопасности
