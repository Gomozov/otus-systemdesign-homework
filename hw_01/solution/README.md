# ДЗ 1. Анализ референсной архитектуры — решение

> Условие: [../Задание.md](../Задание.md) · Чек-лист: [../Чеклист.md](../Чеклист.md)
> Схемы складывайте в [diagrams/](diagrams/).

## Облачное хранилище

### Контекст системы

Облачное хранилище позволяет хранить в публичном облаке файлы: документы, фотографии, видео и другие. 
Доступ к файлам можно осуществлять с любого компьютера, смартфона или планшета.

### Основные пользователи и акторы

Пользователи облачного хранилища:
- люди (через браузер, desktop-приложение и отдельные мобильные приложения для Android/iOS)
- боты (через API)

Зачем:
- хранение данных
- обмен данными

### Ключевые сценарии использования

- загрузка (upload) и скачивание (download) файлов
- поиск по файлам
- обмен данными между своими устройствами (синхронизация, sync)
- обмен данными с другими пользователями (sharing)

### Масштаб

- 10M пользователей, из них 2M DAU (Day Active Users)
- до 10 ГБ для каждого пользователя
- пользователь загружает в среднем 1 файл в день (FBD, File by Day), СРЕДНИЙ размер файла 2 МБ (AFS, Average File Size)
- количество операций чтения и записи примерно равны
- всего места требуется: 10M x 10Гб = 100 Петабайт. Примечание - коэффициент экономии на одинаковых файлах пока неопределен.
- загрузки uQPS (upload Queries by Second): 2M x 1 / (24 * 3600) = 23,15 округляем до ~24
- всего передач файлов tQPS (upload + download): ~48
- метаинформация mQPS (list/sync/search): оценка x3–x10 от transfer -> 150–500 (грубо)
- Пик 2x: дневной паттерн (утро/вечер)

## 1. Компоненты

### Балансировщик нагрузки (Load balancer, LB)

Равномерно распределяет запросы между серверами API.

### Блочная система хранения данных (Block Service)

Основные функции: chunking, хеширование, дедупликация, порядок блоков.
Загружают блоки данных в облачное хранилище. Блочное хранилище - это технология хранения файлов в облачных окружениях. 
Файл может быть разделен на несколько блоков, каждый из которых имеет уникальный хеш и хранится в БД метаданных. 
Каждый блок обрабатывается как независимый объект и записывается в систему хранения (Cloud, например S3). 
Чтобы восстановить файл, блоки соединяются в определенном порядке. 
Что касается размера блоков, мы возьмем за пример сервис Dropbox, в котором блок не может превышать 4 Мб.

### Облачное хранилище (хранение blob-блоков, репликация, durability)

Файл делится на блоки меньшего размера, которые записываются в облачное хранилище.

### Холодное хранилище

Система, предназначенная для хранения неактивных данных (файлов, к которым не обращаются на протяжении длительного времени).

Условие когда файл становится холодным:
* N дней без access;
* только архивные tier;
* отдельная стоимость/латентность при restore.

### Серверы API

Отвечают почти за все, кроме процесса загрузки. Их используют для аутентификации пользователей, управления пользовательскими профилями, 
обновления метаданных файлов и т.д.

### Сервис поиска / индекс

Индексирование и поиск по файлам.

### Очереди сообщений

async: индексация, уведомления, миграция в cold storage

### БД метаданных

Хранит метаданные пользователей, файлов, блоков, версий и т.д. Пожалуйста, обратите внимание на то, что в этой БД находятся только 
метаданные, а сами файлы хранятся в облаке.

### Кэш метаданных

Метаданные кэшируются для быстрого доступа.

### Сервис уведомлений

Система типа PubSub (издатель–подписчик), которая передает данные клиентам при возникновении определенных событий. Сервис оповещает  
пользователей о добавлении/редактировании/удалении файла кем-то другим, чтобы они могли просмотреть последние изменения.

### CDN (опционально)

Ускорение скачивания популярных/крупных файлов.

## 2. Потоки данных

Примечание - Рисунки продублированы в папке /diagrams

### Общая схема
```
flowchart TB
    Client[Client]
    LB[Load Balancer]
    API[API Server]
    BS[Block Service]
    Cloud[(Object Storage)]
    Cold[(Cold Storage)]
    Meta[(Metadata DB<br/>files · blocks · manifests)]
    Cache[(Metadata Cache)]
    CDN[CDN optional]
    Q[[Message Queue]]
    Search[Search Index]
    Notify[Notification]

    Client -->|upload / API| LB
    LB --> API
    LB --> BS

    API --> Cache
    Cache --> Meta
    API --> Meta
    API --> Q

    BS -->|dedup lookup<br/>register block<br/>file_blocks| Meta
    BS -->|PUT blob| Cloud
    Cloud --> Cold

    CDN --> Cloud
    API --> Cloud
    Client -->|download private| API
    Client -->|download public| CDN

    Q --> Search
    Q --> Notify
    Notify --> Client
```


### 2.1. Загрузка файла

* Путь: client -> LB -> API -> Block Service -> Cloud + Metadata DB (+ dedup по hash)
* Async: индексация и уведомления через очередь
* Сбой:
    - Cloud timeout -> retry блока
    - обрыв upload -> resume по hash
    - Metadata недоступна -> 503 Service Unavailable, клиент повторяет commit POST /upload/commit с тем же session_id / upload_id

```
sequenceDiagram
    autonumber
    participant C as Client
    participant LB as Load Balancer
    participant API as API Server
    participant BS as Block Service
    participant Cloud as Object Storage
    participant M as Metadata DB
    participant Q as Message Queue
    participant SI as Search Index

    C->>LB: POST /upload/init (auth, path, size)
    LB->>API: forward
    API->>M: create file record (status=pending)
    API-->>C: upload_session_id, block_size=4MB

    loop for each chunk
        C->>LB: PUT /upload/chunk (hash, data)
        LB->>BS: forward
        BS->>M: block exists? (dedup by hash)
        alt block already in Cloud
            BS-->>C: 200 OK (skipped)
        else new block
            BS->>Cloud: PUT object (content-addressed key)
            alt Cloud error
                BS-->>C: 503 + retry-after
            else success
                BS->>M: register block ref
                BS-->>C: 200 OK
            end
        end
    end

    C->>LB: POST /upload/commit
    LB->>API: forward
    API->>M: finalize manifest, status=ready
    API->>Q: FileIndexed event (async)
    Q-->>SI: update index (async)
    API-->>C: 201 Created

```

### 2.2. Скачивание

* Путь:  Client -> LB -> API -> Metadata -> blocks -> API (assemble) -> client
* Async: нет (критический путь синхронный)
* Сбой:
    - block missing в Cloud -> 404 + alert
    - CDN недоступен -> fallback на origin Cloud

```
sequenceDiagram
    autonumber
    participant C as Client
    participant LB as Load Balancer
    participant API as API Server
    participant Cache as Metadata Cache
    participant M as Metadata DB
    participant CDN as CDN optional
    participant Cloud as Object Storage

    C->>LB: GET /files/{id}/download
    LB->>API: forward
    API->>Cache: get file metadata + ACL
    alt cache hit
        Cache-->>API: metadata + block list
    else cache miss
        API->>M: read metadata + ACL check
        M-->>API: block manifest
        API->>Cache: populate cache
    end

    alt access denied
        API-->>C: 403 Forbidden
    else public or shared hot file
        C->>CDN: GET /object/{hash}
        alt CDN hit
            CDN-->>C: file bytes
        else CDN miss
            CDN->>Cloud: origin fetch
            Cloud-->>CDN: blocks
            CDN-->>C: file bytes
        end
    else private file
        loop for each block in order
            API->>Cloud: GET block by hash
            alt block not found
                API-->>C: 404 + internal alert
            else
                Cloud-->>API: block data
            end
        end
        API-->>C: assembled file stream
    end

```


### 2.3. Sync между устройствами 

* Путь: delta sync, long polling / WebSocket через Notification
* Async: push через PubSub; catch-up по cursor при reconnect
* Сбой:
    - offline device -> при reconnect GET /sync?cursor=... (только дельта с последней точки, не полагаемся на push-уведомление)
    - duplicate event -> dedup по event_id (защита от повторной обработки одного и того же события)

```
sequenceDiagram
    autonumber
    participant A as Device A
    participant LB as Load Balancer
    participant API as API Server
    participant M as Metadata DB
    participant N as Notification PubSub
    participant B as Device B

    Note over B,N: Device B держит long-poll / WebSocket
    B->>LB: GET /sync/subscribe (cursor=rev_100)
    LB->>API: forward (sticky session)
    API->>N: subscribe user channel

    A->>LB: PUT /files/{id} (edit/upload)
    LB->>API: forward
    API->>M: update file, revision=rev_101
    API->>N: publish ChangeEvent(rev_101)
    N-->>B: push: new revision available

    B->>LB: GET /sync/delta?since=rev_100
    LB->>API: forward
    API->>M: fetch changes rev_100..rev_101
    M-->>API: delta (added/modified/deleted)
    API-->>B: delta + cursor=rev_101

    alt Device B was offline
        Note over B: пропущены push-события
        B->>API: GET /sync/delta?since=last_known_cursor
        API->>M: catch-up all missed revisions
        M-->>API: full delta batch
        API-->>B: delta + new cursor
    end

```


### 2.4. Sharing с другим пользователем

* Путь: owner -> API -> Metadata (Access Control List, ACL) -> Queue -> Notification -> collaborator
* Async: уведомление других пользователей (collaborator) через очередь/PubSub
* Сбой:
    - concurrent ACL update -> optimistic lock - вместо того чтобы заранее блокировать данные, система разрешает свободный доступ и проверяет наличие конфликтов только в момент сохранения изменений, т.е. кто первым успешно записал (commit прошёл при актуальной version) — его изменения сохраняются. (конфликт версий -> 409 Conflict, retry)

```
sequenceDiagram
    autonumber
    participant O as Owner Client
    participant LB as Load Balancer
    participant API as API Server
    participant M as Metadata DB
    participant Q as Message Queue
    participant N as Notification PubSub
    participant R as Recipient Client

    O->>LB: POST /share (file_id, user_id, role=editor)
    LB->>API: forward
    API->>M: check owner ACL
    API->>M: insert share record + ACL entry
    alt version conflict
        M-->>API: 409 Conflict
        API-->>O: retry with latest version
    else success
        M-->>API: share_id, acl_version
        API->>Q: ShareGranted event (async)
        Q->>N: deliver notification
        N-->>R: push: "Folder shared with you"
        API-->>O: 200 OK
    end

    R->>LB: GET /files/shared
    LB->>API: forward
    API->>M: list shares for recipient
    M-->>API: shared files + permissions
    API-->>R: file list (read/write per ACL)

```


### 2.5. Поиск

* Путь: client -> API -> Search Index (+ Metadata для enrichment)
* Async: индекс обновляется из очереди после upload/edit
* Сбой:
    - Search недоступен -> fallback prefix search по имени в Metadata; index lag -> результаты могут отставать на секунды

```
sequenceDiagram
    autonumber
    participant C as Client
    participant LB as Load Balancer
    participant API as API Server
    participant SI as Search Index
    participant M as Metadata DB
    participant Q as Message Queue

    Note over Q,SI: Background: после upload commit
    Q->>SI: index document (file_id, name, content)

    C->>LB: GET /search?q=report
    LB->>API: forward
    API->>SI: query "report" (user scope)

    alt Search Index available
        SI-->>API: ranked file_ids + snippets
        API->>M: enrich metadata (path, size, modified)
        M-->>API: file details
        API-->>C: 200 search results
    else Search Index down
        API->>M: fallback prefix match on filename
        M-->>API: limited results
        API-->>C: 200 degraded results
    end

    Note over C,SI: Index lag: новый файл может<br/>появиться в поиске через N сек

```

## 3. Проблемы и решения

### 3.1. Сводная таблица

| Компонент | Какую проблему решает | Что сломается без него | Trade-off |
| --------- | --------------------- | ---------------------- | --------- |
| Load Balancer | Единая точка входа; распределение ~48 tQPS + сотни mQPS | Один API-сервер — bottleneck и SPOF ( single point of failure) | +1 hop latency; нужен health check |
| Block Service | Chunking, dedup (дедупликация), resume upload | Полный re-upload файла при обрыве; дубли блоков | Сложность сборки файла и учёта hash |
| Object Storage (Cloud, например S3) | Хранение blob-блоков (ПБ масштаб) | Metadata DB не выдержит | Vendor lock-in,  (стоимость исходящего трафика) |
| Cold Storage | Снижение cost при 100 PB квоты | Все данные в hot tier — ×5–×10 cost | Restore latency минуты–часы |
| Серверы API | Auth, ACL, sharing, sync cursor | Block Service смешивает бизнес-логику и I/O | Риск «супер-сервиса» без чётких границ |
| Search Index | Full-text search за O(log n) | Scan Metadata — не масштабируется | Eventual consistency, lag индекса |
| Message Queue | Decouple spikes (index, notify, migrate) | Sync coupling, потери при пиках | At-least-once -> нужна идемпотентность |
| Metadata DB | Файловая структура, версии, ACL, block map | Нет «файловой системы в облаке» | Шардирование усложняет cross-user queries |
| Metadata Cache | Latency list/sync при 150–500 mQPS | Metadata DB перегружена «на чтение» | Stale reads, сложная инвалидация |
| Notification (PubSub) | Real-time sync без периодических опросов (polling) | Клиенты poll каждые N сек — нагрузка ×10 | Delivery guarantees, reconnect logic |
| CDN (optional) | Latency download для hot/large files | Все через origin Cloud, высокий egress | Cache invalidation при delete/rename |

### 3.2. Чуть детальнее по компонентам

#### Балансировщик нагрузки (LB)
* Проблема: при 2M DAU запросы не должны упираться в один API-инстанс; нужна маршрутизация (routing) на основе доступности-состояния узлов.
* Без LB: один сервер обрабатывает весь трафик -> нехватка CPU/memory, немасштабируемо, downtime при деплое.
* Trade-off: дополнительная сетевая задержка (~1–5 мс); Липкие сессии (sticky sessions, https://habr.com/ru/companies/domclick/articles/548610/) иногда нужны для long-poll/WebSocket.
* Масштабирование: горизонтально добавляем API-инстансы за LB; L7 маршрутизация по path (/upload -> Block Service pool).
* Надёжность: active-passive или multi-AZ (изоляция по зонам доступности) LB; недоступные/нерабочие инстансы исключаются из пула.

#### Block Service (chunking, hash, dedup)
* Проблема: файлы до 10 ГБ нельзя загружать атомарно; нужны восстановления после сбоя (resume), дедупликация (dedup) и CAS (content-addressed storage).
* Без него: обрыв на 99% загрузки 2 ГБ файла -> перезагрузка с нуля; одинаковые блоки хранятся N раз -> лишние ПБ.
* Trade-off: клиент и сервер усложняются (manifest блоков, порядок, обработка коллизий hash); dedup cross-user — privacy-вопросы.
* Масштабирование: stateless сервис — масштабируется копированием; dedup снижает нагрузку на запись в облачном хранилище.
* Надёжность: идемпотентный PUT блока по hash — повторная загрузка того же блока безопасна; частичная загрузка хранится как "pending" в Metadata до commit.

#### Облачное хранилище (например S3)
* Проблема: надёжное хранение blob-данных на порядке десятков–сотен ПБ вне реляционных СУБД.
* Без него: пришлось бы хранить файлы в Metadata DB или на дисках API — ни масштабируемо, ни надёжно.
* Trade-off: задержка (latency) выше, чем local disk; стоимость исходящего трафика (egress); семантика eventual consistency на уровне объекта (компромисс между доступностью и согласованностью: система гарантирует, что после записи объект рано или поздно станет виден в списках, но не сразу).
* Масштабирование: как правило масштабируется автоматически; prefix sharding hash-блоков (/ab/cd/...) для равномерной нагрузки.
* Надёжность: репликация в ≥3 AZ.

#### Холодное хранилище
* Проблема: 100 ПБ — суммарная квота; даже при 30–40% заполнения и дедупликации это десятки ПБ; hot tier (S3) дорог.
* Без него: экономика не сходится — стоимость хранения доминирует над доходом.
* Trade-off: файлы без доступа N дней (например, 90) -> Glacier/Archive tier; восстановление 1–12+ часов, отдельная плата за retrieval.
* Масштабирование: lifecycle policy автоматически tiering по last_access_at; не требует ручного шардирования.
* Надёжность: те же durability guarantees; риск — пользователь ждёт restore; нужен async job + уведомление «файл готов».

#### Серверы API
* Проблема: единая бизнес-логика — auth (OAuth/JWT), квоты (10 ГБ/user), ACL sharing, sync cursor, orchestration upload/download.
* Без них: Block Service и клиенты дублируют правила доступа; нет централизованной авторизации.
* Trade-off: риск «god service» — mitigated тем, что тяжёлый I/O (chunk upload) делегируется Block Service.
* Масштабирование: stateless; горизонтально за LB; rate limiting per user/API key.
* Надёжность: circuit breaker к Metadata DB и Search; graceful degradation (read-only mode при partial outage).

#### Сервис поиска / индекс
* Проблема: сценарий «поиск по файлам» при 10M users — full scan Metadata невозможен.
* Без него: SELECT ... LIKE '%query%' по шардам — секунды/минуты, перегрузка БД.
* Trade-off: индекс отстаёт от Metadata на секунды–минуты (eventual consistency); extra storage (~10–30% от metadata size).
* Масштабирование: inverted index шардируется по user_id; частые запросы кэшируются.
* Надёжность: reindex из очереди при пропуске event; search degraded -> fallback на prefix match по имени файла.

#### Очереди сообщений
* Проблема: upload не должен блокироваться на индексации, push-уведомлениях и cold migration; пики (утро/вечер, 2×) нужно сгладить.
* Без неё: API синхронно вызывает Search + Notification -> latency upload ×3, cascade failure при падении Search.
* Trade-off: at-least-once delivery -> duplicate events; consumers должны быть idempotent.
* Масштабирование: partition queue по user_id или file_id; consumer groups масштабируются независимо.
* Надёжность: queue как buffer при spike; messages persist on disk; DLQ для poison messages.

#### БД метаданных
* Проблема: source of truth — users, files, block manifests, versions, shares, sync revision.
* Без неё: нет иерархии папок, sharing, conflict detection, dedup reference counting.
* Trade-off: relational model удобна для ACL, но cross-shard joins дороги; нужен careful schema design.
* Масштабирование: шардирование по user_id (10M users -> N shards); read replicas для list/sync reads; block hash table может быть global shard по hash prefix.
* Надёжность: синхронная репликация в quorum (RPO ≈ 0 для metadata); periodic backup + point-in-time recovery.

#### Кэш метаданных
* Проблема: list folder, sync delta, get file metadata — hot path при 150–500 mQPS; каждый read в DB — дорого.
* Без него: Metadata DB становится первым bottleneck при росте DAU.
* Trade-off: cache invalidation при write (delete/rename/share) — сложность; короткий TTL + explicit invalidate.
* Масштабирование: Redis cluster; cache-aside pattern; hot users/devices получают 80% hit rate.
* Надёжность: cache miss -> fallback to DB (не fail); Redis replica для HA.

#### Сервис уведомлений (PubSub)
* Проблема: sync между устройствами — клиент должен узнать об изменениях без polling каждые 5 сек (2M devices × poll = огромная нагрузка).
* Без него: polling -> mQPS ×10; задержка sync до интервала poll.
* Trade-off: offline devices пропускают push -> catch-up по sync cursor при reconnect; WebSocket/long-poll держат connections.
* Масштабирование: pub/sub broker (Kafka, Redis Streams, dedicated); channels per user/device.
* Надёжность: at-least-once notify; client deduplicates by event_id; heartbeat + reconnect.

#### CDN (опционально)
* Проблема: повторяющаяся загрузка одного файла (sharing link, популярный файл) — origin Cloud + egress cost + latency.
* Без CDN: каждый пользователь идёт в Cloud region; пользователи далеко от DC — RTT 100–300 мс.
* Trade-off: cache invalidation при delete/update; не все файлы CDN-friendly (private, E2EE).
* Масштабирование: edge PoP глобально; cache hit ratio target 60–80% для public/shared content.
* Надёжность: потеря CDN -> получение данных напрямую из облачного хранилища; stale-while-revalidate для metadata headers.

## 4. Вопросы к авторам архитектуры

* защита данных (шифрование client-side E2EE vs server-side), групповой доступ и обновление ключей при истечении лимита времени, количества обращений или утечках
* стратегии резервного копирования, в т.ч. с учетом версионирования
* обработка сбоев компонент, представленных в разделе 1
* стратегии работы с конфликтами синхронизации (особенно если пользователей > 2). Sync/conflict resolution — отдельный сервис или явная зона ответственности API.
* экономика vs. задел для развития
