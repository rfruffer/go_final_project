# RUTUBE Plugin: eXpress Notifications for Jira

Jira-плагин для отправки уведомлений о событиях задач в мессенджер **eXpress** (протокол BotX).

## Требования

| Компонент | Версия |
|-----------|--------|
| Jira | 9.4+ |
| Java | 11 |
| eXpress CTS | совместимый с BotX API v3/v4 |

## Установка

1. Собрать JAR:
   ```bash
   mvn package -DskipTests
   ```
2. В Jira: **Администрирование → Управление приложениями (UPM) → Загрузить приложение**.
3. Выбрать JAR из `target/jira-express-notifications-*.jar`.

## Настройка

**Администрирование → eXpress Notifications**

| Поле | Описание |
|------|----------|
| CTS Host | URL CTS-сервера, например `https://cts.example.com` |
| Bot ID | UUID бота в eXpress |
| Secret Key | Секретный ключ для подписи JWT |
| Bot Name | Отображаемое имя бота |
| Bot Link | Ссылка на бота для первого контакта |
| Enabled by Default | Включать уведомления для новых пользователей по умолчанию |

После заполнения нажать **Save**, затем **Test Connection** для проверки связи с CTS.

## Как работает

### Доставка уведомлений

1. Пользователь совершает действие в Jira (создание, обновление, комментарий и т.д.).
2. Плагин получает событие через `IssueEvent` listener.
3. Определяет получателей по схеме уведомлений проекта.
4. Исключает автора события (собственные действия никогда не уведомляются).
5. Проверяет, что у каждого получателя eXpress-уведомления включены.
6. Получает `huid` получателя через `POST /api/v3/botx/users/by_email`.
7. Отправляет сообщение через `POST /api/v4/botx/notifications/direct/sync`.

### Аутентификация с CTS

Каждый запрос подписывается JWT (HS256):

```
Header: {"alg":"HS256","typ":"JWT"}
Payload: {"iss":"<botId>","aud":"<cts_domain>","exp":now+60,"version":2,...}
Signature: HMAC-SHA256(header.payload, secretKey)
```

Токен генерируется заново на каждый запрос (TTL 60 сек).

### Фильтрация получателей

Получатели берутся из схемы уведомлений Jira и проходят через следующие фильтры:

- Стандартные Jira-фильтры (`NotificationFilterManager`), включая настройку пользователя "не уведомлять о своих изменениях"
- Явная проверка: автор события **всегда** исключается из получателей (независимо от настроек)
- Проверка активности аккаунта
- Проверка наличия email-адреса
- Проверка, что у пользователя включены eXpress-уведомления

### Типы обрабатываемых событий

| Событие | Сообщение |
|---------|-----------|
| Создан | ✨ {автор} создал(а) новый запрос {ссылка} |
| Обновлён | 🔄 {автор} обновил(а) запрос {ссылка} |
| Назначен | 👥 {автор} назначил(а) запрос на {исполнитель} |
| Решён | ✔️ {автор} решил(а) запрос как {резолюция} |
| Закрыт | ✔️ {автор} закрыл(а) запрос как {резолюция} |
| Комментарий | 🔔 ссылка + 🗨️ {автор} оставил(а) комментарий |
| Переоткрыт | 🔄 {автор} переоткрыл(а) запрос |
| Удалён | 🗑️ {автор} удалил(а) запрос |
| Журнал работ | ⏱️ {автор} добавил(а) запись в журнал |
| Упоминание | 🗣️ {автор} упомянул(а) вас в запросе |

Для обновлённых полей и комментариев отображается diff (старое значение → новое).

## Пользовательская настройка

Каждый пользователь может включить или отключить eXpress-уведомления в своём профиле Jira:

**Профиль → eXpress Notifications → Edit**

При первом включении необходимо написать боту в eXpress, чтобы открыть диалог (ссылка отображается на панели профиля).

## REST API

Все эндпоинты: `/rest/express/1.0/`

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/admin/config` | Получить текущую конфигурацию |
| `POST` | `/admin/config` | Сохранить конфигурацию |
| `GET` | `/admin/config/test-connection` | Проверить связь с CTS |
| `GET` | `/admin/config/notifications-log` | Последние 50 событий уведомлений (для диагностики) |
| `GET` | `/user/config` | Настройки текущего пользователя |
| `POST` | `/user/config` | Обновить настройки текущего пользователя |

### Диагностика

Эндпоинт `/admin/config/notifications-log` возвращает историю последних 50 событий с момента запуска плагина:

```json
[{
  "time": "2026-06-10 14:01:25",
  "issueKey": "TEST-8",
  "eventType": "CREATED",
  "recipients": [{
    "email":"user@example.com",
    "username": "user",
    "expressEnabled": true,
    "messagePreview": "✨ admin создал(а) новый запрос ...",
    "ctsPayload": "POST https://cts.example.com/api/v3/botx/users/by_email ...",
    "recipientStatus": "SENT"
  }],
  "status": "SENT"
}]
```

Возможные статусы получателя: `SENT`, `FAILED: <ошибка>`, `SKIPPED_DISABLED`, `SKIPPED_OWN_ACTION`, `SKIPPED_INACTIVE`, `SKIPPED_NO_EMAIL`, `SKIPPED_EMPTY_MESSAGE`.

## Сборка и разработка

```bash
# Сборка
mvn package -DskipTests

# Сборка с тестами
mvn package

# Быстрая установка в запущенную Jira через UPM API
TOKEN=$(curl -s -u admin:admin \
  -H "Accept: application/vnd.atl.plugins.installed+json" \
  http://localhost:2990/jira/rest/plugins/1.0/ -D - \
  | grep upm-token | awk '{print $2}' | tr -d '\r')

curl -u admin:admin \
  -H "Accept: application/json" \
  -F "plugin=@target/jira-express-notifications-*.jar" \
  "http://localhost:2990/jira/rest/plugins/1.0/?token=$TOKEN"
```

### Технический стек

- **Jira Plugin Framework** 9.4+, Atlassian Spring Scanner 2
- **JAX-RS** (Jersey 1.x через Jira) — REST API
- **Jackson 2.15** — сериализация запросов/ответов (bundled, не Jira's Jackson 1.x)
- **Unirest 3.14** — HTTP-клиент для запросов к CTS
- **Lombok** — кодогенерация

### Известные особенности

- Jira (Jersey 1.x) использует Jackson 1.x для десериализации тел запросов и сериализации ответов. Строковые поля в DTO сериализуются некорректно. Решение: принимать тело запроса как `String`, парсить через bundled Jackson 2.x; ответы строить через `MAPPER.writeValueAsString()`.
- `@EventListener` в Jira-плагинах требует явной регистрации через `EventPublisher.register(this)`. Реализовано через `InitializingBean.afterPropertiesSet()`.
- ContextProvider для web-panel не поддерживает Spring-инъекции — получение бинов через статический ExpressComponentLocator.
