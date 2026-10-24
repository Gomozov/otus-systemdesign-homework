# ДЗ 3. Проектирование хранения данных — решение

> Условие: [../Задание.md](../Задание.md)
> Шаблоны: [сравнение БД](../Таблица%20сравнения%20БД%20с%20критериями%20выбора.md) ·
> [кэширование](../Чек-лист%20стратегий%20кэширования.md) ·
> [шардирование](../Шаблон%20схемы%20шардирования.md). Схемы — в [diagrams/](diagrams/).

> Выполняется для системы, выбранной в ДЗ 2.

## 1. Модель данных

> **ER-диаграмма** - Entity-Relationship Diagram графическая схема, которая наглядно показывает, какие данные есть в системе и как они связаны между собой.  
> **FK** (foreign key) - внешний ключ: поле, которое ссылается на строку другой таблицы в той же БД.  
> **PK** (primary key) - первичный ключ.  
> **UK** (unique key) - уникальное поле, которое не является PK.

### 1.1. Authentication Service - аккаунты и роли

Курьер может сам заказать еду. Если сделать тип пользователя (role) в ACCOUNT одним значением, то такой случай не описать.  
Альтернатива хранить роли списком, либо вообще убрать role из ACCOUNT и определять роли из наличия строк в CLIENT_PROFILE, COURIER и RESTAURANT_STAFF (но у такого решения будет минус - придется обращаться к другим сервисам).

Authentication Service определяет роли, используя собственную БД, в следующем порядке:

- пользователь вводит телефон и код OTP в приложении
- Authentication Service находит ACCOUNT по телефону и читает все строки ACCOUNT_ROLE этого аккаунта
- Authentication Service кладет список ролей в JWT, например roles: ["client", "courier"], подписывает токен и отдает приложению
- дальше BFF на каждом запросе смотрит роли в токене: эндпоинты курьера вроде accept оффера требуют courier, оформление заказа требует client. В Authentication Service при этом никто не ходит, подпись проверяется по открытому ключу.

Роли выдают разные сервисы в разные моменты, но записывает всегда Authentication Service:

- client выдается автоматически при первом входе по телефону в клиентском приложении
- courier выдается после онбординга в Delivery: когда курьер проверен и активирован, Delivery сообщает Authentication Service, что аккаунту нужно добавить роль. Удобно сделать это событием CourierActivated через Kafka, в том же стиле, что и остальные интеграции.
- restaurant выдает владелец заведения, приглашая сотрудника в панели ресторана. Catalog создает строку RESTAURANT_STAFF и просит Auth добавить роль.
- ops выдает администратор вручную

Что видит человек с двумя ролями. У клиента и курьера разные приложения. Войдя в курьерское приложение, он получает тот же токен с обеими ролями, но приложение использует курьерские эндпоинты. Если у аккаунта нет роли courier, Auth при входе в курьерское приложение отвечает, что нужно пройти онбординг, а не выдает токен. Для этого приложение при входе указывает, какую роль ему нужно.

Предполагается, что БД Authentication Service будет содержать следующие таблицы:

- **ACCOUNT**. Данные об аккаунтах и данных, необходимых для аутентификации;
- **ACCOUNT_ROLE**. Роли вынесены в отдельную таблицу `ACCOUNT_ROLE`, а не в поле `role` таблицы ACCOUNT: курьер может сам заказать еду, сотрудник ресторана - подрабатывать курьером;
- **REFRESH_TOKEN**. Хранит долгоживущую сессию, которую можно отозвать. Сам JWT для этого не подходит: в схеме он нигде не лежит, только проверяется по подписи и JWKS. Короткий JWT клиент предъявляет на каждый запрос. Когда он истекает, приложение присылает refresh-токен, и Auth по строке в этой таблице решает, выпускать ли новую пару. Без таблицы после истечения JWT пришлось бы снова проходить OTP или пароль.

> **OTP** (one-time password) - одноразовый код на телефон. С его помощью входят клиенты и курьеры: Auth кладет код в Redis (otp:{phone}) на 5 минут и ограничивает число попыток. После успешного ввода код больше не нужен.  
> **JWT** (JSON Web Token) - подписанный токен, который приложение прикладывает к следующим запросам. Внутри - кто это (account_id) и какие роли. Auth его нигде не хранит: BFF проверяет подпись и срок. Токен короткий, поэтому предусмотрен еще refresh-токен в таблице REFRESH_TOKEN: по нему выпускается новый JWT без повторного OTP.  
> **JWKS** (JSON Web Key Set) - набор публичных ключей. Ими проверяется подпись JWT. Сам ключ подписи остается у Authentication Service, публичная часть общая для всех регионов, чтобы токен, выданный в одном городе, принимали в другом.

```mermaid
erDiagram
    ACCOUNT ||--o{ ACCOUNT_ROLE : has
    ACCOUNT ||--o{ REFRESH_TOKEN : has

    ACCOUNT {
        uuid id PK
        string phone UK "клиенты, курьеры; nullable"
        string email UK "рестораны, поддержка; nullable"
        string password_hash "только для входа через веб"
        string status "active, blocked"
        timestamptz created_at
    }
    ACCOUNT_ROLE {
        uuid account_id PK, FK
        string role PK "client, courier, restaurant, ops"
        timestamptz granted_at
    }
    REFRESH_TOKEN {
        uuid id PK
        uuid account_id FK
        string token_hash
        string device_id
        timestamptz expires_at
        timestamptz revoked_at "nullable"
    }
```

### 1.2. Profile Service - клиент, адреса, платежные методы

Предполагается, что БД Profile Service будет содержать следующие таблицы:

- **CLIENT_PROFILE**. Профиль клиента: имя, контактный email, язык интерфейса. `home_region` указывает, какой регион владеет записью и имеет право ее менять; `version` нужен CDC-репликации, чтобы остальные регионы получали read-only копию и клиент в поездке видел свои адреса и карты;
- **ADDRESS**. Сохраненные адреса доставки пользователя. Кроме текста и координат хранит информацию для курьера (подъезд, этаж, квартира, домофон, комментарий), которые при заказе целиком копируются в `ORDER.address_details`. `is_default` - адрес, подставляемый по умолчанию;
- **PAYMENT_METHOD**. Способы оплаты. Номер карты не хранится: только `provider_token`, по которому Payment GW создает платеж без повторного ввода карты, и последние 4 цифры номера карты;
- **FAVORITE**. Избранные товары клиента. `menu_item_id` ссылается на Catalog только по значению, без FK между БД: если позицию удалили из меню, она отфильтровывается при отображении, а не каскадом в чужой БД.

```mermaid
erDiagram
    CLIENT_PROFILE ||--o{ ADDRESS : has
    CLIENT_PROFILE ||--o{ PAYMENT_METHOD : has
    CLIENT_PROFILE ||--o{ FAVORITE : has

    CLIENT_PROFILE {
        uuid account_id PK "= ACCOUNT.id из Authentication Service"
        string name
        string email "nullable"
        string home_region "регион регистрации, владелец записи"
        string locale
        timestamptz updated_at
        int version "для CDC-репликации в другие регионы"
    }
    ADDRESS {
        uuid id PK
        uuid client_id FK
        string label "Дом, Работа"
        string text "адрес как ввел пользователь"
        float lat "геокодирован один раз при сохранении"
        float lon
        string entrance
        string floor
        string apartment
        string intercom
        string comment
        bool is_default
        timestamptz deleted_at "мягкое удаление; nullable"
    }
    PAYMENT_METHOD {
        uuid id PK
        uuid client_id FK
        string provider "эквайер или СБП"
        string provider_token "токен провайдера"
        string card_last4
        string card_brand
        bool is_default
    }
    FAVORITE {
        uuid client_id PK, FK
        uuid menu_item_id PK "ссылка на Catalog по id, без FK между БД"
        timestamptz added_at
    }
```

Геокодирование адреса пользователя (адреса доставки) выполняется при сохранении, чтобы при заказе `ValidateAndQuote` и `QuoteFee` получали готовые координаты и гео-сервис не стоял в критическом пути заказа.

### 1.3. Catalog Service - рестораны, меню, промо

Адрес магазина (ресторана) и зона доставки хранятся здесь: адрес меняется редко, координаты вычисляются один раз через гео-сервис при подключении. Зона доставки - полигон PostGIS с GiST-индексом, по нему же `ValidateAndQuote` проверяет «адрес доставки в зоне». Поиск «рестораны в радиусе» идет по кэшу координат в Redis GEO (`restaurants:geo`), истина в PostgreSQL. Меню кэшируется по паре ключей `menu:{restaurant_id}:ver` (текущая версия) и `menu:{restaurant_id}:{version}` (тело этой версии). Стоп-лист в это тело не входит, у него отдельный короткий ключ `stoplist:{restaurant_id}` (п. 4). Фото блюд и логотипы хранятся в object storage, в БД хранится только ключ.

Предполагается, что БД Catalog Service будет содержать следующие таблицы:

- **BRAND**. Информация об организации (бренде), у одной организации может быть несколько (сеть) магазинов (ресторанов). Сеть может работать в нескольких регионах, поэтому это единственная таблица Catalog, которая реплицируется во все регионы как единый реестр (`version` для CDC);
- **RESTAURANT**. Информация о магазинах (ресторанах). Конкретная точка: адрес и координаты (геокодированы один раз при подключении), полигон зоны доставки (PostGIS), график работы и т.п. `pos_type` и `pos_external_id` - привязка к POS-системе заведения: по ним POS GW понимает, куда отправлять `SubmitOrder` и чей webhook пришел. `brand_id` nullable - одиночное заведение без сети;
- **RESTAURANT_STAFF**. Сотрудники ресторана и их права. `role` определяет, что можно делать: `owner` приглашает сотрудников и управляет точкой, `manager` правит меню и цены, `cook` - только стоп-лист и `is_open`;
- **MENU_CATEGORY**. Разделы меню («Пицца», «Напитки») для группировки и порядка показа (`sort_order`). Привязана к ресторану, а не к бренду: даже внутри сети у каждой точки может быть свое меню и свои цены;
- **MENU_ITEM**. Позиция меню: название, описание, цена, ключ фото в object storage, флаг доступности;
- **STOP_LIST**. Временно недоступные позиции - «закончилось сегодня»;
- **PROMO_CODE**. Определение промокода: сам код, скидка, лимит на одного клиента и период действия. Остатка применений здесь нет. `scope = local` заводит региональный маркетинг, строка живет только в этом регионе. `scope = global` приезжает с хаба репликацией и в регионе только читается: регион эту строку не обновляет;
- **PROMO_QUOTA**. Остаток, который этот регион имеет право списать. У локального кода строка создается вместе с кодом, `allocated` равен лимиту кампании. У глобального хаб заранее делит общий бюджет на порции, а регион списывает только свою. Общий счетчик на хабе регион не уменьшает, чужую порцию тоже;
- **PROMO_GRANT**. Одно начисление порции с хаба. Повтор CDC с тем же `id` второй раз квоту не увеличивает;
- **PROMO_RESERVATION**. Резерв на время между checkout и оплатой. В той же транзакции квота уменьшается условием `uses_left > 0`. Без этой таблицы последний экземпляр могли бы одновременно применить два клиента, или код завис бы навсегда при обрыве между checkout и оплатой. Повтор checkout с тем же `idempotency_key` возвращает тот же резерв и единицу повторно не списывает. Отпущенный и истекший резерв возвращает единицу в квоту;
- **PROMO_REDEMPTION**. Сколько раз клиент уже взял этот код в этом регионе, включая еще не оплаченный резерв. Списание - условное обновление `uses = uses + 1 WHERE uses < per_client_limit` в той же транзакции, что и резерв. Обычный лимит - 1. Отпущенный резерв уменьшает `uses` обратно.

Глобальный код с `uses_left` нельзя одновременно привозить репликой и списывать как единственный счетчик. Если регион пишет в свою копию, N регионов потратят один и тот же остаток. Если копия read-only, списать остаток на месте нельзя, а синхронный поход на хаб на каждый checkout ставит оплату в зависимость от межрегиональной сети. Поэтому по CDC едут определение и начисления порций, а списание - локальная CP-транзакция по `PROMO_QUOTA`.

Цена этой модели в том, что хаб не отзывает уже выданную порцию: тихий регион держит свой остаток, пока кампания активна в другом городе. При разрыве с хабом новая порция не приезжает, уже выделенная списывается как обычно, checkout от хаба не зависит. Лимит на клиента тоже региональный: пара `(promo_id, client_id)` уникальна внутри региона. Клиент, который в один день закажет в двух городах, применит глобальный код по разу в каждом. Суммарный бюджет от этого не умножается, каждый регион тратит свою порцию. Синхронный резерв на хабе дал бы и общий остаток, и «один раз на клиента» строго, ценой лишнего сетевого похода на checkout и отказа глобального кода при разрыве с хабом. Его не берем: разрыв с хабом не должен останавливать регион (п. 6.2).

```mermaid
erDiagram
    BRAND ||--o{ RESTAURANT : owns
    RESTAURANT ||--o{ RESTAURANT_STAFF : employs
    RESTAURANT ||--o{ MENU_CATEGORY : has
    MENU_CATEGORY ||--o{ MENU_ITEM : contains
    RESTAURANT ||--o{ STOP_LIST : has
    MENU_ITEM ||--o{ STOP_LIST : "в стоп-листе"
    PROMO_CODE ||--o| PROMO_QUOTA : "квота региона"
    PROMO_CODE ||--o{ PROMO_GRANT : "порции с хаба"
    PROMO_CODE ||--o{ PROMO_RESERVATION : reserved
    PROMO_CODE ||--o{ PROMO_REDEMPTION : "лимит на клиента"

    BRAND {
        uuid id PK
        string name
        string logo_key "object storage"
        timestamptz updated_at
        int version "глобальный реестр, реплицируется"
    }
    RESTAURANT {
        uuid id PK
        uuid brand_id FK "nullable для одиночных заведений"
        string name
        string address "текст"
        float lat "геокодирован при подключении"
        float lon
        geometry delivery_zone "полигон, PostGIS"
        jsonb working_hours
        bool is_open "ручной выключатель панели ресторана"
        string status "onboarding, active, suspended"
        string pos_type "iiko, Syrve, none"
        string pos_external_id "id заведения в POS"
        int rating_x100
        int menu_version "растет при изменении меню, не стоп-листа"
        timestamptz updated_at
    }
    RESTAURANT_STAFF {
        uuid account_id PK, FK "= ACCOUNT.id из Authentication Service"
        uuid restaurant_id PK, FK
        string role "owner, manager, cook"
    }
    MENU_CATEGORY {
        uuid id PK
        uuid restaurant_id FK
        string name
        int sort_order
    }
    MENU_ITEM {
        uuid id PK "= offer_id в API"
        uuid category_id FK
        string name
        string description
        int price_minor
        int price_version "растет при каждом изменении цены"
        string photo_key "object storage"
        bool is_available
        timestamptz updated_at
    }
    STOP_LIST {
        uuid restaurant_id PK, FK
        uuid menu_item_id PK, FK
        timestamptz until "nullable - до ручного снятия"
    }
    PROMO_CODE {
        uuid id PK
        string code UK
        string scope "local, global"
        int discount_minor
        int discount_percent
        int per_client_limit "обычно 1"
        timestamptz valid_from
        timestamptz valid_to
        int version "у global - CDC определения, не остаток"
    }
    PROMO_QUOTA {
        uuid promo_id PK, FK
        int allocated "сколько региону выдано"
        int uses_left "списывается только здесь"
    }
    PROMO_GRANT {
        uuid id PK "id начисления с хаба"
        uuid promo_id FK
        int amount
        timestamptz applied_at
    }
    PROMO_RESERVATION {
        uuid id PK
        uuid promo_id FK
        uuid client_id
        string idempotency_key UK "повтор checkout возвращает тот же резерв"
        timestamptz expires_at "TTL до оплаты"
        string status "reserved, consumed, released"
    }
    PROMO_REDEMPTION {
        uuid promo_id PK, FK
        uuid client_id PK
        int uses "резерв и уже оплаченные"
    }
```

### 1.4. Orders Service - заказ и снимки

Предполагается, что БД Orders Service будет содержать следующие таблицы:

- **ORDER**. Заказ. Хранит, кто и у кого заказал, текущий `status`, снимок адреса доставки (текст, координаты, детали для курьера), суммы (позиции, скидка, тариф доставки, итог) и ссылку на резерв промокода. Имя ресторана и адрес - снимки, а не ссылки: если клиент потом переименует «Дом» или ресторан сменит название, история заказов не изменится;
- **ORDER_ITEM**. Состав заказа: что заказано и сколько. Название, цена и `price_version` скопированы из Catalog в момент `ValidateAndQuote`, `menu_item_id` - тот же `offer_id`, без FK между БД. Снимок нужен для чека, расчета возврата и разбора споров «я заказывал по другой цене», даже если позицию уже удалили из меню;
- **PAYMENT**. Информация о платеже. `client_id` лежит в строке рядом с `payment_id`: в Citus уникальный индекс распределенной таблицы включает колонку распределения, а поиск оплаты по одному `payment_id` ушел бы на все шарды;
- **ORDER_STATUS_HISTORY**. Журнал переходов статуса: какой статус, кто его инициировал (`source` - клиент, платежный webhook, POS, Delivery, оператор поддержки) и когда. В `ORDER.status` лежит только текущее значение; по истории считаются метрики SLA (сколько готовили, сколько везли), восстанавливается хронология при жалобе в Support и строится аналитика. `client_id` дублируется в строке, чтобы запись истории шла в тот же шард, что и заказ;
- **ORDER_BOARD**. Операционный срез «заказы сейчас» для панели ресторана. В нем только живые заказы, колонка распределения - `restaurant_id`. История заказов здесь не копится. Как срез наполняется - в п. 3.2.

```mermaid
erDiagram
    ORDER ||--|{ ORDER_ITEM : contains
    ORDER ||--o{ PAYMENT : "оплачен через"
    ORDER ||--o{ ORDER_STATUS_HISTORY : has

    ORDER {
        uuid client_id PK "колонка распределения, ACCOUNT.id"
        uuid id PK "глобально уникальный uuid"
        uuid restaurant_id "снимок id"
        string restaurant_name "снимок"
        string idempotency_key UK
        string status "pending_payment, paid, cooking, ready, delivering, delivered, cancelled"
        string address_text "снимок из Profile или из тела запроса"
        float address_lat
        float address_lon
        jsonb address_details "подъезд, этаж, квартира, домофон, комментарий"
        int items_minor
        int discount_minor
        int delivery_fee_minor "из QuoteFee"
        int total_minor
        string promo_code "nullable"
        uuid promo_reservation_id "nullable"
        timestamptz created_at
        timestamptz payment_deadline "pending_payment истекает"
        timestamptz updated_at
    }
    ORDER_ITEM {
        uuid client_id PK "колонка распределения"
        uuid id PK
        uuid order_id FK
        uuid menu_item_id "offer_id из Catalog"
        string name "снимок"
        int price_minor "снимок"
        int price_version "снимок"
        int qty
    }
    PAYMENT {
        uuid client_id PK "колонка распределения"
        string payment_id PK "id у провайдера"
        uuid order_id FK
        string provider
        string status "created, succeeded, failed, refunded"
        int amount_minor
        string confirmation_url
        timestamptz created_at
        timestamptz updated_at
    }
    ORDER_STATUS_HISTORY {
        uuid client_id PK "колонка распределения"
        uuid id PK
        uuid order_id FK
        string status
        string source "client, payment, pos, delivery, ops"
        timestamptz at
    }
    ORDER_BOARD {
        uuid restaurant_id PK "колонка распределения панели"
        uuid order_id PK
        uuid client_id "обратный путь в шард заказа"
        string status
        timestamptz updated_at
    }
```

`ORDER_BOARD` не связан внешним ключом с `ORDER`: строки лежат в разных шардах Citus, а ссылка между шардами держалась бы распределенной транзакцией. `client_id` в строке среза нужен панели, когда она открывает сам заказ.

Таблица `PAYMENT` - ссылка на платеж у провайдера и его статус по webhook. Webhook не ищет строку по одному `payment_id`: вместе с `order_id` в событие приходит `client_id`, и обновление идет в один шард по паре `(client_id, payment_id)`. Откуда шлюз берет `client_id` - в п. 3.2. `payment_deadline` закрывает заказы, по которым клиент так и не оплатил: по истечении заказ переводится в `cancelled`, резерв промо освобождается.

### 1.5. Delivery Service - курьеры, смены, доставки

Предполагается, что БД Delivery Service будет содержать следующие таблицы:

- **COURIER**. Профиль курьера: имя, тип транспорта (влияет на радиус геопоиска и тариф в `QuoteFee`), рейтинг и число выполненных доставок. `status` - жизненный цикл онбординга: переход в `active` порождает событие `CourierActivated`, по которому Auth добавляет аккаунту роль `courier`; `blocked` снимает курьера с линии без удаления истории. PK = `ACCOUNT.id`. `home_region` и `version` - запись реплицируется во все регионы, чтобы курьер мог выйти на смену в другом городе;
- **SHIFT**. Смена курьера: когда начал, когда закончил, в каком районе (`zone_id`). Открытая смена (`ended_at` IS NULL) - условие, при котором курьер вообще получает офферы; район смены ограничивает, какие доставки ему предлагать;
- **DELIVERY**. Доставка одного заказа. Создается событием `OrderPaid` в `waiting_kitchen`, по `KitchenReady` переходит в `offering`, после accept - в `assigned`, дальше отметки курьера `picked_up` и `delivered`. `courier_id` пуст до назначения. Точки забора и доставки - снимки из заказа, чтобы матчинг не ходил в Orders и Catalog. Временные отметки этапов нужны для ETA, выплат и SLA;
- **DELIVERY_OFFER**. Предложение конкретной доставки конкретному курьеру с дедлайном `expires_at`. У одной доставки несколько офферов: волны геопоиска, пока кто-то не примет. История нужна, чтобы не предлагать ту же доставку отказавшемуся повторно.

```mermaid
erDiagram
    COURIER ||--o{ SHIFT : works
    COURIER ||--o{ DELIVERY : delivers
    DELIVERY ||--o{ DELIVERY_OFFER : "предложена"
    COURIER ||--o{ DELIVERY_OFFER : receives

    COURIER {
        uuid account_id PK "= ACCOUNT.id из Authentication Service"
        string name
        string vehicle_type "foot, bike, car"
        string status "onboarding, active, blocked"
        string home_region "регион регистрации, владелец записи"
        int rating_x100
        int deliveries_done
        timestamptz updated_at
        int version "для CDC-репликации"
    }
    SHIFT {
        uuid id PK
        uuid courier_id FK
        string zone_id "район, выбранный на смену"
        timestamptz started_at
        timestamptz ended_at "nullable - смена идет"
    }
    DELIVERY {
        uuid id PK
        uuid order_id UK "уникальный частичный индекс, кроме cancelled"
        uuid courier_id FK "nullable до назначения"
        string status "waiting_kitchen, offering, assigned, picked_up, delivered, cancelled"
        uuid restaurant_id
        float pickup_lat "снимок из заказа"
        float pickup_lon
        float dropoff_lat "снимок из заказа"
        float dropoff_lon
        string dropoff_text
        int fee_minor "из QuoteFee"
        timestamptz assigned_at
        timestamptz picked_up_at
        timestamptz delivered_at
    }
    DELIVERY_OFFER {
        uuid id PK
        uuid delivery_id FK
        uuid courier_id FK
        string status "offered, accepted, declined, expired, lost"
        timestamptz expires_at
    }
```

### 1.6. Tracking Service - только Redis

Реляционных таблиц нет: точка курьера живет минуты, писать ее в PostgreSQL бессмысленно. Структуры кэша (Redis):

| Ключ | Тип | Что хранит | Срок жизни |
| --- | --- | --- | --- |
| `couriers:{zone_id}:online` | GEO (sorted set) | курьеры, чья текущая точка лежит в этом районе. `{zone_id}` - hash tag Redis Cluster, слот считается по нему | у членов GEO нет TTL |
| `couriers:{zone_id}:last_seen` | sorted set | `courier_id` → unix time последней точки в том же районе. Тот же hash tag, поэтому GEO и last_seen района стоят на одном слоте | фоновая задача этого ключа удаляет молчащих > 60 с из обоих наборов района |
| `courier:{id}:pos` | hash | lat, lon, heading, ts, `order_id`, `zone_id` | TTL 60 с. Отсюда читатель узнает текущий район, если курьер на миг попал в два GEO |
| `order:{id}:eta` | string | ETA в минутах от гео-сервиса | TTL 60–120 с, пересчет не на каждую точку |
| `order:{id}:track` | pub/sub канал | кадры `{lat, lon, eta_min, ts}` подписчикам | - |

История точек не хранится. Одного ключа `couriers:last_seen` на регион нет: один sorted set занимает один слот одного мастера, и `ZADD` на каждую точку (5–10k операций/с) собрался бы на нем. Район в ключе - район текущей точки, не район смены. Как курьер переходит между районами - в п. 3.3.

> ETA (estimated time of arrival) — оценка, через сколько минут курьер будет у клиента.

### 1.7. Support Service - обращения, оценки, звонки

Один объект «обращение по заказу» объединяет чат, жалобу и оценку, как решено в ДЗ 2. Непрочитанные сообщения и веерная рассылка (fan-out) по открытым сокетам - в Redis (`thread:{id}:unread:{account_id}`, pub/sub `thread:{id}`), текст в PostgreSQL. Вложения чата и записи звонков - в object storage.

Предполагается, что БД Support Service будет содержать следующие таблицы:

- **THREAD**. Обращение - один объект, объединяющий чат, жалобу, оценку и звонки по заказу. Идентификаторы заказа, клиента, курьера и ресторана - просто uuid без FK между БД;
- **MESSAGE**. Сообщения в треде: автор, текст и т.п.
- **RATING**. Оценка заказа клиентом: ресторану и курьеру отдельно, с комментарием. UK по `order_id` - одна оценка на заказ. `thread_id` nullable: оценка может быть и без переписки, а при жалобе связывается с тредом;
- **CALL**. Маскированный звонок через Telephony GW: `provider_call_id` - идентификатор у оператора телефонии, по нему сопоставляются webhook-и статусов; кто кому звонил (`from_account_id`, `to_account_id`) без реальных номеров; статус, длительность, ключ записи в object storage. Привязан к треду, чтобы звонок был виден в истории обращения вместе с сообщениями.

```mermaid
erDiagram
    THREAD ||--o{ MESSAGE : contains
    THREAD ||--o{ CALL : has
    THREAD ||--o| RATING : "завершается"

    THREAD {
        uuid id PK
        uuid order_id "без FK между БД; nullable для общих обращений"
        string kind "client_courier, client_restaurant, client_support, complaint"
        uuid client_id
        uuid courier_id "nullable"
        uuid restaurant_id "nullable"
        uuid assignee_id "сотрудник поддержки; nullable"
        string status "open, waiting, resolved, closed"
        timestamptz created_at
        timestamptz updated_at
    }
    MESSAGE {
        uuid id PK
        uuid thread_id FK
        uuid author_id "ACCOUNT.id"
        string text
        jsonb attachments "ключи object storage"
        timestamptz sent_at
        timestamptz read_at "nullable"
    }
    RATING {
        uuid id PK
        uuid order_id UK
        uuid thread_id FK "nullable"
        uuid client_id
        int restaurant_score "1–5"
        int courier_score "1–5"
        string comment
        timestamptz created_at
    }
    CALL {
        uuid id PK
        uuid thread_id FK
        string provider_call_id "id у оператора телефонии"
        uuid from_account_id
        uuid to_account_id
        string status "initiated, connected, finished, failed"
        string recording_key "object storage; nullable"
        timestamptz started_at
        int duration_s
    }
```

### 1.8. Сводка: кто чем владеет

| Сущности | Сервис | Хранилище | Область |
| --- | --- | --- | --- |
| ACCOUNT, ACCOUNT_ROLE, REFRESH_TOKEN | Auth | PostgreSQL (своя БД) | регион регистрации, JWKS общий |
| OTP, лимиты попыток | Auth | Redis | регион |
| CLIENT_PROFILE, ADDRESS, PAYMENT_METHOD, FAVORITE | Profile | PostgreSQL | домашний регион → все регионы read-only |
| BRAND | Catalog | PostgreSQL | единый реестр → все регионы |
| RESTAURANT, RESTAURANT_STAFF, MENU_*, STOP_LIST | Catalog | PostgreSQL + PostGIS | регион |
| PROMO_CODE local, квота, резерв, лимит на клиента | Catalog | PostgreSQL | регион, списание CP |
| PROMO_CODE global (определение) | Catalog | PostgreSQL, реплика с хаба | read-only, AP, минуты |
| кэш меню, указатель версии, стоп-лист, `restaurants:geo` | Catalog | Redis | регион |
| фото блюд, логотипы | Catalog | Object storage | регион |
| ORDER, ORDER_ITEM, PAYMENT, ORDER_STATUS_HISTORY | Orders | PostgreSQL, шард по `client_id` | регион |
| ORDER_BOARD | Orders | PostgreSQL, шард по `restaurant_id` | регион, живые заказы панели |
| COURIER | Delivery | PostgreSQL | регион регистрации → все регионы |
| SHIFT, DELIVERY, DELIVERY_OFFER | Delivery | PostgreSQL | регион |
| `couriers:{zone}:online`, `couriers:{zone}:last_seen`, `courier:{id}:pos`, ETA, pub/sub | Tracking | Redis | регион |
| THREAD, MESSAGE, RATING, CALL | Support | PostgreSQL | регион |
| вложения, записи звонков | Support | Object storage | регион |
| `outbox`, `processed_events` | Orders, Delivery, Support | PostgreSQL, та же БД сервиса | регион |
| события, точки для аналитики | все → Analytics | ClickHouse (центр) | one-way поток |

Между БД разных сервисов внешних ключей нет: `client_id`, `restaurant_id`, `order_id` в чужих таблицах - просто uuid, целостность обеспечивается событиями и снимками, а не ограничениями СУБД.

## 2. Выбор БД

Исходные цифры из ДЗ 2 на одну агломерацию:

- 1M DAU
- до 30k заведений
- 20–30k курьеров одновременно на линии
- 1.5M заказов в сутки
- пик около 100 RPS заказов

Это не «хайлоад» на запись для реляционной БД: один заказ дает порядка десяти INSERT/UPDATE за весь жизненный цикл, то есть сотни операций записи в секунду в пике. При этом на checkout и на accept оффера нужны транзакции и условные UPDATE. Поэтому для транзакционных сервисов выбран PostgreSQL, а другие БД взяты там, где форма данных или нагрузка другие: геоданные курьеров, аналитика, медиа-файлы.

| Сервис | Что хранит | Форма данных | Ключевые запросы | Выбранная БД | Почему именно она |
| --- | --- | --- | --- | --- | --- |
| Auth | аккаунты, роли, refresh-токены | реляционная, 3 таблицы, уникальность телефона и email | точечно по телефону / email при входе, по `token_hash` при refresh | PostgreSQL (своя БД, отдельная от остальных) | уникальные индексы и транзакция «аккаунт + роль» при первом входе; отдельная БД, чтобы утечка или падение другого сервиса не задели учетные данные |
| Auth | OTP-коды, счетчики попыток, rate limit | ключ → значение с TTL | `GET`/`INCR` по `otp:{phone}` | Redis | код существует 5 минут; счетчики атомарны; PostgreSQL для этого пришлось бы чистить вручную |
| Profile | профиль, адреса, платежные методы, избранное | реляционная, один клиент → много адресов и карт | по `account_id` из JWT, редкие обновления | PostgreSQL + логическая репликация (CDC) в другие регионы | данные персональные и нужны для чека и возврата, важна точность; объем мал (2M клиентов × единицы адресов); CDC по WAL дает read-only копии в других регионах |
| Catalog | рестораны, зоны доставки, меню, стоп-листы, промокоды | реляционная + геометрия (полигоны зон) | «ресторан в радиусе», «точка в полигоне», меню ресторана целиком, атомарное `uses_left - 1` у региональной квоты | PostgreSQL + PostGIS | GiST-индекс по полигону закрывает «адрес в зоне» в `ValidateAndQuote`; резерв и лимит на клиента - одна транзакция с условием на остаток квоты. Определение глобального кода реплицируется, остаток - нет (п. 1.3); 30k заведений × десятки позиций - единицы миллионов строк, один узел |
| Catalog | кэш меню, стоп-лист, координаты ресторанов для поиска в радиусе | key-value, GEO-набор | `GET menu:{id}:ver`, затем `GET menu:{id}:{version}`, `GET stoplist:{id}`, `GEOSEARCH restaurants:geo` | Redis | витрину смотрят чаще, чем заказывают; версию читатель берет из Redis, а не из PostgreSQL и не из карточки ресторана. GEO-набор отвечает на радиус за миллисекунды |
| Catalog | фото блюд, логотипы | бинарные объекты | отдача по ключу | Object storage (S3-совместимый, региональный) | миллионы картинок не место в СУБД; в таблице только ключ. CDN не нужен, трафик региональный для каждой агломерации (ДЗ 2) |
| Orders | заказы, позиции, платежи, история статусов, срез панели | реляционная, агрегат «заказ» | по `(client_id, id)`, по `(client_id, idempotency_key)`, «мои заказы» по `client_id`, панель ресторана по `restaurant_id` из `ORDER_BOARD` | PostgreSQL | checkout фиксирует заказ, позиции и строку outbox одной транзакцией на шарде клиента; условные UPDATE статуса - то, что дает реляционная СУБД. Деньги считаются здесь, eventual consistency неприемлема. Запрос по одному `order_id` шарды не обходит (п. 3.2) |
| Delivery | курьеры, смены, доставки, офферы | реляционная, гонка за одну строку | условный `UPDATE ... WHERE status = 'offered'`, частичный уникальный индекс по `order_id` | PostgreSQL | решение «кто повез» принимает одна строка в одной БД: ноль обновленных строк - HTTP 409 «оффер уже принят». Никакого отдельного кода для гонки писать не нужно: атомарность условия и обновления дает сама СУБД. В документной или wide-column БД такой инвариант пришлось бы реализовывать "вручную" |
| Delivery | кэш офферов, что кому предложено | ключ → значение с TTL | по `offer_id` | Redis | оффер живет десятки секунд; результат в итоге - в PostgreSQL |
| Tracking | текущая точка курьера, ETA, поток точек клиенту | гео-точки с TTL, pub/sub | `GEOADD`/`GEOSEARCH`, `GET` ETA, подписка на канал заказа | Redis (только), без реляционной БД | 20–30k курьеров × точка каждые 3–5 с - 5–10k записей в секунду, каждая живет минуту. В PostgreSQL это был бы бесполезный WAL и vacuum; потеря одной точки терпима, поэтому persistence не нужен |
| Support | треды, сообщения, оценки, звонки | реляционная, тред → сообщения | по `thread_id` с сортировкой по времени, по `order_id` | PostgreSQL | история чата должна сохраниться для разбора жалоб; одна оценка на заказ - UK; объем умеренный (не каждый заказ приводит к появлению треда) |
| Support | непрочитанные, рассылка (fan-out) по сокетам | счетчики, pub/sub | `INCR thread:{id}:unread:{account_id}`, `PUBLISH thread:{id}` | Redis | счетчик и рассылка живым подключениям - не то, что хочется писать в журнал транзакций на каждое сообщение |
| Support | вложения, записи звонков | бинарные объекты | по ключу | Object storage | как у Catalog |
| Payment GW, POS GW | дедупликация webhook и маршрут в шард заказа | ключ с TTL | `SET NX webhook:{provider}:{event_id}`; `payroute:{provider}:{payment_id}` и `posroute:{pos_type}:{external_id}` → `(client_id, order_id)` | Redis (тот же инстанс состояния, `noeviction`) | доменное состояние заказа - у Orders. Шлюзу хватает двух ключей: «этот webhook уже видели» и «какой `client_id` у этого платежа или заказа POS», чтобы не искать заказ по всем шардам (п. 3.2) |
| Telephony GW | дедупликация входящих webhook по `event_id` | ключ с TTL | `SET NX webhook:{provider}:{event_id}` | Redis | адаптер без своей БД; ему нужно помнить, что `event_id` уже видели, пару недель. Доменное состояние звонка - у Support |
| Notify | очередь на отправку, дедупликация | события | consumer group Kafka | без своей БД (Kafka + Redis для дедупликации) | шлюзы SMS/PUSH сами возвращают статус; журнал отправок уходит в ClickHouse, он нужен для аналитики, а не для транзакций |
| Sync | смещения CDC, версии реплицируемых сущностей | лог изменений | последовательное чтение WAL | без своей БД: Debezium (Kafka Connect) хранит offset в топиках Kafka | второй СУБД не нужно, состояние коннектора и так живет в Kafka |
| Analytics | события всех регионов, точки курьеров, журнал отправок | события append-only, агрегаты | «средняя длительность готовки по ресторанам за неделю», «acceptance rate по районам» и т.п. | ClickHouse (центр) | агрегаты по миллиардам строк - колоночная задача; в PostgreSQL такие запросы отнимали бы ресурсы у заказов. Загрузка one-way из Kafka, update не нужен |

## 3. Шардирование

### 3.0. Нужно ли шардировать?

Оценка для Orders одной агломерации:

| Показатель | Оценка | Откуда |
| --- | --- | --- |
| Заказов в сутки | 1.5M | ДЗ 2, «утро-обед-вечер» 0.5M × 3 |
| Запись в пике | ~100 заказов/с × ~10 строк (заказ, 2–4 позиции, платеж, 6 статусов, outbox) ≈ 1 000 строк/с | 100 RPS пик, модель данных п. 1.4 |
| Размер одного заказа со всеми строками | ~2.5 КБ | uuid, снимок адреса в `jsonb`, 3 позиции, история |
| Прирост | ~3.7 ГБ/сутки, ~1.4 ТБ/год | 1.5M × 2.5 КБ |
| Горячие данные | последние 90 дней ≈ 340 ГБ | ops и Support смотрят недавние заказы, старое - в ClickHouse |

Тысяча строк в секунду и сотни гигабайт горячих данных - рабочий режим одной PostgreSQL на NVMe с синхронной репликой и 1–2 репликами для чтения. Ответ по рекомендуемому шаблону: **внутри региона шардирование Orders не требуется**. Но шардирование по регионам уже есть, оно принято в ДЗ 2, и именно его надо описать как уровень 1; а ключ для уровня 2 стоит выбрать сейчас, потому что он влияет на то, как писать запросы, и менять его потом дорого.

### 3.1. Уровень 1: geo-шардирование по агломерации (уже есть)

Ключ - `region_id`, он вычисляется из адреса доставки на BFF и дальше запрос целиком живет в стеке региона. Это стратегия «Geo / по атрибуту» из шаблона.

- **Равномерность.** Перекос между регионами есть и большой: самая крупная агломерация в разы больше остальных. Это терпимо, потому что регион - не кусок одной БД, а отдельный стек, который масштабируется независимо. Маленькие регионы можно разворачивать на "общем" железе.
- **Локальность.** Заказ, ресторан, курьер и кухня в одной агломерации, границы не пересекаются (ДЗ 2). Cross-shard запросов в пути заказа нет вообще.
- **Рост.** Новая агломерация - новый стек, данные никуда не переезжают. Ребалансировки нет по построению.
- **Транзакции.** Все транзакции checkout и accept внутри одного региона, SAGA из ДЗ 2 не пересекает границу шардов.
- **Что пересекает границу.** Профиль, курьер, бренд и определение глобального промо реплицируются CDC через хаб, регион их не запрашивает по сети на checkout. Остаток глобального промо границу живым счетчиком не пересекает: хаб присылает порцию (`PROMO_GRANT`), регион один раз прибавляет ее к своей `PROMO_QUOTA` и дальше списывает локально (п. 1.3). Клиент в чужом городе читает локальную read-only копию своего профиля.

### 3.2. Уровень 2: Orders внутри региона, когда одна нода перестанет вмещать

Триггер перехода: 90 дней не влезают в память и кэш страниц, либо запись упирается в один WAL.

| Кандидат в ключи | Равномерность | Локальность запросов | Риск hotspot | Вывод |
| --- | --- | --- | --- | --- |
| `order_id` (uuid) | идеальная | «мои заказы» клиента размазываются по всем шардам; `idempotency_key` тоже глобальный | нет | плохо: самый частый запрос клиента становится scatter-gather |
| `client_id` | высокая: 2M клиентов, у каждого единицы заказов в день | «мои заказы», повтор по `idempotency_key` (ключ выдает клиент), история для Support - все в одном шарде | низкий: нет клиента с тысячами заказов | **подходит** |
| `restaurant_id` | плохая: сеть с сотней точек и дарк-китчены дают перекос | панель ресторана локальна, но клиентские запросы - scatter | высокий: популярная точка в обед | нет |
| `created_at` (range) | нет: вся запись идет в свежий диапазон | хорошо для архива и отчетов | максимальный: один «текущий» шард принимает все | нет как ключ шардирования, да как партиционирование внутри шарда |

> **Scatter-gather** - это запрос клиента, который нельзя направить в один шард, поэтому его приходится "разбросать" (scatter) по всем шардам и потом "собрать" (gather) результаты. Допустим, Orders разрезан на 4 шарда по order_id. Каждый заказ попадает в шард по хэшу своего идентификатора, и заказы одного клиента оказываются размазаны по всем четырем. Клиент открывает экран «Мои заказы». Сервис знает только client_id, а по нему нельзя вычислить, в каком шарде лежат заказы. Остается одно: отправить SELECT ... WHERE client_id = X во все четыре шарда, дождаться четырех ответов, слить их, отсортировать по дате и отдать клиенту.

Итог для Orders:

| Что шардируем | Ключ | Стратегия | Кол-во шардов (старт) | Ребалансировка | Главный риск + как лечим |
| --- | --- | --- | --- | --- | --- |
| ORDER, ORDER_ITEM, PAYMENT, ORDER_STATUS_HISTORY, outbox - колокация по `client_id` | `client_id` | hash, через Citus (`distribution column = client_id`) | 4 логических шарда на 2 нодах, Citus держит 32 физических шарда и переносит их целиком при добавлении нод | добавление ноды - перенос части из 32 шардов без изменения хэш-функции (pre-splitting); онлайн, с копированием и короткой блокировкой на переключение | `UPDATE`/`SELECT` по одному `order_id` или `payment_id` в Citus уходит на все 32 шарда. Это тот же scatter-gather, из-за которого ключом не взят `order_id`, и на пути оплаты его быть не должно. Каждый вход в Orders несет `client_id` (таблица ниже), условие всегда `WHERE client_id = ? AND id = ?` |
| ORDER_BOARD, живые заказы панели ресторана | `restaurant_id` | hash, отдельная распределенная таблица | те же 32 шарда | вместе с остальными шардами Citus | панель «заказы сейчас» - операционный путь, в ClickHouse ее не отправляем: там данные отстают. Отчет за день и старше остается в ClickHouse |

> **Citus** - это расширение PostgreSQL, которое превращает несколько обычных инстансов PostgreSQL в один шардированный кластер.

`ORDER.id` остается глобально уникальным uuid, поэтому ссылка `DELIVERY.order_id` и `RATING.order_id` в других сервисах не меняется. В Citus уникальный индекс распределенной таблицы включает колонку распределения, поэтому первичный ключ заказа - `(client_id, id)`, платежа - `(client_id, payment_id)`, идемпотентности - `(client_id, idempotency_key)`. Глобальную уникальность `order_id` доказывает генерация uuid, а не индекс на все шарды. Внутри каждого шарда таблицы партиционируются по `created_at` помесячно: это не шардирование, а способ дешево отрезать старые месяцы (`DETACH PARTITION`) после выгрузки в ClickHouse.

Строки одного заказа лежат в одном шарде: у ORDER_ITEM, PAYMENT и ORDER_STATUS_HISTORY `client_id` - колонка распределения, и транзакция checkout шарды не пересекает. Единственная транзакция, пересекающая сервисы, уже решена SAGA в ДЗ 2. Индекса по `idempotency_key` на шард достаточно: ключ выдает клиент, а все заказы клиента в одном шарде.

`ORDER_BOARD` в эту транзакцию не входит. У среза другая колонка распределения, `restaurant_id`, и общая транзакция с заказом была бы 2PC между шардами на каждом checkout. Срез пишет тот же потребитель Orders, который применяет смену статуса: сразу после обновления заказа он делает идемпотентный upsert по `(restaurant_id, order_id)`. Если процесс упал между двумя записями, событие придет еще раз (at-least-once, п. 5.3) и допишет срез. Кухня видит заказ с лагом этого потребителя, это секунды. В строке среза лежит `client_id`, поэтому открытие карточки из панели - опять один шард, уже заказа. Доставленный и отмененный заказ из среза удаляется. При 100 заказах/с и жизни заказа около 40 минут в срезе порядка 250 тысяч строк, не история на год.

Ops-экран «все застрявшие в `cooking`» читает этот же срез. Таблица маленькая, обход ее шардов - не обход истории заказов и не критический путь оплаты. Отчет за день и старше строится в ClickHouse.

```mermaid
flowchart LR
    REQ["POST /v1/orders<br/>JWT: client_id<br/>address → region"]
    REQ --> BFF["BFF"]
    BFF -->|"уровень 1: region_id из адреса"| RA

    subgraph RA["Регион A - свой стек"]
        ORD["Orders Service"]
        COORD["Citus coordinator"]
        ORD -->|"INSERT ... client_id = X"| COORD
        COORD -->|"hash(client_id) → шард"| S0[("нода 1<br/>шарды 0..15")]
        COORD -.-> S1[("нода 2<br/>шарды 16..31")]
        ORD -->|"после смены статуса"| BOARD[("ORDER_BOARD<br/>hash restaurant_id")]
    end

    subgraph RB["Регион B"]
        STB["такой же стек"]
    end
    BFF -.->|"другой адрес"| STB

    S0 -->|"CDC: outbox"| KA{{"Kafka A"}}
    KA -->|"OrderPaid(client_id, order_id)"| DLV["Delivery"]
    DLV -->|"CourierAssigned(client_id, order_id)"| ORD
    PAY["Payment GW<br/>payroute в Redis"] -->|"PaymentSucceeded(client_id, order_id)"| ORD
    POS["POS GW<br/>posroute в Redis"] -->|"KitchenReady(client_id, order_id)"| ORD
    KA -->|"события"| CH[("ClickHouse<br/>отчеты, не панель кухни")]
```

Откуда в каждом входе берется `client_id`. Ключ партиции Kafka остается `order_id`: он держит порядок событий одного заказа. Шард считается по полю тела, это другой ключ.

| Кто меняет заказ | Откуда `client_id` | Запрос в Orders |
| --- | --- | --- |
| Клиент: checkout, «мои заказы», отмена | JWT | `WHERE client_id = ?` |
| Delivery | забирает `client_id` из `OrderPaid`, хранит на `DELIVERY` и кладет в каждое свое событие (`CourierAssigned`, `PickedUp`, `Delivered`) | `WHERE client_id = ? AND id = ?` |
| Payment GW | `CreatePayment` получает пару от Orders. Шлюз пишет в свой Redis `payroute:{provider}:{payment_id}` → `(client_id, order_id)` с TTL 14 дней (тот же горизонт, что у дедупликации webhook) и передает `client_id` в metadata платежа, чтобы провайдер вернул его в webhook. Webhook читает `payroute`. Metadata - второй экземпляр, если ключ в Redis потерян | `PAYMENT` и `ORDER` по `(client_id, payment_id)` и `(client_id, id)`. Поиска по одному `payment_id` нет |
| POS GW | `SubmitOrder` содержит `client_id` и `order_id`, то же кладется в заказ на стороне POS. Шлюз пишет `posroute:{pos_type}:{external_id}` с TTL 7 дней. Webhook кухни резолвится в шлюзе | `WHERE client_id = ? AND id = ?` |
| Панель ресторана | срез `ORDER_BOARD` по `restaurant_id`; в строке уже есть `client_id` | список - один шард панели, карточка заказа - один шард клиента |
| Support | `client_id` приходит в событии заказа и лежит на `THREAD` | пара `(client_id, order_id)` |

Если у webhook нет ни `payroute`/`posroute`, ни `client_id` в теле провайдера, шлюз кладет событие в свой DLQ. Orders все 32 шарда в поисках заказа не обходит.

### 3.3. Tracking: Redis Cluster, позиция по `courier_id`, GEO и last_seen по району

Здесь шардирование может понадобиться раньше, чем в Orders, потому что нагрузка другая: 20–30k курьеров × точка каждые 3–5 с = 5–10k точек в секунду, каждая точка - три команды (`GEOADD`, `HSET`, `ZADD`), итого 15–30k операций в секунду. Один инстанс Redis на это способен, но запас по пику двойной, а не десятикратный, и один процесс - одна точка отказа для всего трекинга региона. Берем Redis Cluster (3 master + 3 replica).

Один ключ кластер не разрезает. И `couriers:online`, и `couriers:last_seen` как единственный sorted set на регион занимают один слот одного мастера. `ZADD` на каждую точку - те же 5–10k операций/с на этом мастере, ради которых GEO-набор и дробится. Фраза «чистка по last_seen каждого слота» к одному ключу не относится: слотов у него один.

| Кандидат в ключи | Равномерность | Локальность запросов | Риск hotspot | Вывод |
| --- | --- | --- | --- | --- |
| `courier_id` в `courier:{id}:pos` | идеальная, Redis Cluster сам раскладывает по 16384 хэш-слотам | запись точки и чтение позиции одного курьера - один слот | нет | **подходит** для hash позиции |
| один ключ `couriers:online` или один `couriers:last_seen` | нет: один ключ = один слот | `GEOSEARCH` и чистка - один вызов | весь трафик региона на одном мастере | плохо |
| `couriers:{zone_id}:online` и `couriers:{zone_id}:last_seen` | районов десятки, hash tag `{zone_id}` кладет оба ключа района на один слот и разносит районы по мастерам | `GEOSEARCH` при матчинге и так вокруг ресторана, Delivery знает `zone_id` ресторана. Чистка района - `ZRANGEBYSCORE` его же ключа | центр крупнее окраины в разы. Если один район упрется в мастер, он делится на бакеты | **подходит** |
| `order:{id}:track` pub/sub | по `order_id` | подписчик - один клиент | нет | подходит, но нужен sharded pub/sub (Redis, `SSUBSCRIBE`), иначе классический pub/sub рассылает на все ноды кластера |

Итог для Tracking:

| Что шардируем | Ключ | Стратегия | Кол-во шардов (старт) | Ребалансировка | Главный риск + как лечим |
| --- | --- | --- | --- | --- | --- |
| `courier:{id}:pos` | `courier_id` | hash-слоты Redis Cluster | 3 мастера | штатная миграция слотов, без простоя | - |
| `couriers:{zone_id}:online`, `couriers:{zone_id}:last_seen` | `zone_id` текущей точки | по атрибуту, общий hash tag | десятки пар ключей на тех же 3 мастерах | при добавлении мастера ключи переезжают вместе со слотами | граница района и смена района - ниже |
| `order:{id}:track` | `order_id` | sharded pub/sub | те же 3 мастера | - | при падении мастера подписчики переподключаются через BFF и теряют 1–2 кадра, это допустимо |

В какой набор пишется курьер. В ключ попадает район, в котором лежит текущая точка, а не район смены из `SHIFT.zone_id`. Район смены остается фильтром Delivery в момент оффера: курьера, найденного геопоиском, можно не звать, если его смена этот район не покрывает. Если бы ключом был район смены, курьер, доехавший до соседнего района, не попал бы в `GEOSEARCH` ресторана, рядом с которым стоит.

Смена района. В хэше `courier:{id}:pos` записан `zone_id` предыдущей точки. Пока район тот же, Tracking делает `GEOADD` и `ZADD` в ключи этого района. Когда район изменился, сначала `GEOADD` и `ZADD` в новый район, потом `ZREM` из старого. Добавление раньше удаления, поэтому на время перехода курьер из поиска не пропадает. Ключи разных районов - разные слоты, одной команды на оба нет, и курьер ненадолго состоит в двух наборах. Второго назначения из этого не получается: Delivery после `GEOSEARCH` читает `courier:{id}:pos` и оставляет кандидата, только если `zone_id` в хэше - это район набора или соседний район того же поиска, и точка свежая. Попадания одного `courier_id` из двух наборов схлопываются до оффера. Чистка района заодно удаляет члена, чей хэш уже указывает на другой район.

Ресторан у границы района ищется как раньше: `GEOSEARCH` своего района и соседних, объединение на стороне Delivery.

Бакеты. Пока район умещается в один слот, бакет один. Если запись точек одного района подходит к бюджету одного мастера, ключи становятся `couriers:{zone_id:bucket}:online` и `couriers:{zone_id:bucket}:last_seen`, `bucket = hash(courier_id) % B`. Скобки hash tag охватывают и район, и бакет. Если оставить tag только на `{zone_id}`, все бакеты района останутся на одном слоте и дробление ничего не даст. `GEOSEARCH` опрашивает все бакеты района. Старт - B = 1.

Чистка «молчит > 60 с» идет по `couriers:{zone_id}:last_seen` каждого района и каждого бакета. Это обычный `ZRANGEBYSCORE` по ключу, который и так лежит на мастере этого района. Атомарности с чем-либо точка курьера не требует: расхождение двух GEO на время смены района закрывает чтение хэша `courier:{id}:pos`.

## 4. Кэширование

Кэшируем то, что читают на порядки чаще, чем пишут, и где отставание на секунды-минуты никому не вредит. Витрина - главный кандидат: по оценке ДЗ 2 просмотр каталога и трекинг дают примерно ×10 к RPS заказов. Что меняется на каждом шаге заказа - статус, остаток промокода, назначение курьера - не кэшируем: там отставание стоит денег или дает двойное назначение.

| Что кэшируем | Уровень | Стратегия | TTL | Инвалидация | Зачем |
| --- | --- | --- | --- | --- | --- |
| Меню ресторана (категории, позиции, цены; без стоп-листа) | Redis (app): указатель `menu:{restaurant_id}:ver` и тело `menu:{restaurant_id}:{version}` | cache-aside | указатель без TTL. Тело текущей версии продлевается при чтении, у предыдущей версии TTL 10 мин и его никто не продлевает | изменение меню увеличивает `RESTAURANT.menu_version` в PostgreSQL, пишет новое тело и после этого переставляет указатель. Читатель берет версию из указателя, не из PostgreSQL и не из карточки `restaurant:{id}` (у той TTL 5 мин, и по ней инвалидация превратилась бы в пятиминутное отставание). Промах указателя бывает после рестарта Redis: один раз читается `menu_version`, указатель восстанавливается. Клиенту отдается `ETag = version` | клиент открывает меню в разы чаще, чем заказывает; один JOIN по меню на 30k заведений × 1M DAU снимаем с PostgreSQL. 10 минут здесь - уборка старой версии, а не свежесть витрины |
| Стоп-лист ресторана | Redis `stoplist:{restaurant_id}`, множество `menu_item_id` | cache-aside: обработчик `StopListChanged` перезаписывает ключ сразу после записи в PostgreSQL | 30 с | событие перезаписывает ключ; TTL страхует потерянное событие | «закончилось» в обед меняется часто. Стоп-лист внутри меню либо отставал бы вместе с тяжелым ключом, либо заставлял бы собирать меню заново на каждое снятие. Короткий ключ клиент читает вместе с меню. `ValidateAndQuote` все равно сверяет стоп-лист с PostgreSQL, поэтому 422 на checkout остается проверкой, а не обычным ответом в обед |
| Координаты ресторанов для «в радиусе 10 км» | Redis GEO `restaurants:geo` | write-through: Catalog при изменении адреса или `is_open` пишет в PostgreSQL и в GEO-набор в одном коде; открытые и закрытые - отдельные наборы | без TTL | по событию (`RestaurantUpdated`), плюс полная пересборка из PostgreSQL раз в сутки и после рестарта Redis | `GEOSEARCH` за миллисекунды вместо `ST_DWithin` по 30k строк на каждый открытый экран |
| Карточка ресторана (название, рейтинг, график, `is_open`, лого) | Redis (app), `restaurant:{id}` | cache-aside | 5 мин | по событию `RestaurantUpdated`; `is_open` меняется через панель ресторана и сбрасывает ключ сразу | список «рестораны рядом» собирается из GEO-набора + N карточек, без обращения в PostgreSQL |
| Фото блюд, логотипы | клиент (HTTP-кэш) + object storage | ключ объекта содержит хэш содержимого, `Cache-Control: max-age=30d, immutable` | 30 дней | не нужна: новое фото - новый ключ | картинки - основной трафик витрины; CDN не нужен, object storage региональный (ДЗ 2) |
| JWKS (публичные ключи для проверки JWT) | в памяти процесса BFF и сервисов | read-through с фоновым обновлением | 1 ч | ротация ключа: новый ключ публикуется заранее, старый остается в наборе еще на TTL токенов | проверка подписи на каждый запрос без обращения в Auth |
| Расчет тарифа `QuoteFee(restaurant_id, zone_id)` | Redis (app) | cache-aside, ключ по ресторану и району адреса с округлением координат до ~100 м | 60 с | по TTL; пиковый коэффициент меняется раз в минуты | клиент видит стоимость доставки при каждом изменении корзины и адреса, это 5–7 одинаковых расчетов на один заказ; расчет требует дистанции и пиковый коэффициент |
| ETA заказа | Redis `order:{id}:eta` | пересчет по таймеру: Tracking пишет новое значение не чаще раза в минуту. Это не write-behind. Write-behind - запись в кэш с отложенным сбросом в БД, а у этого ключа базы нет: Redis и есть хранилище посчитанного значения | 60–120 с | по TTL и при следующей точке, если минута уже прошла | внешний гео-сервис платный и медленный; клиенту достаточно ETA с точностью до минуты |
| Текущая позиция курьера | Redis `courier:{id}:pos`, `couriers:{zone_id}:online` | это не кэш, а основное хранилище (п. 1.6), истины в PostgreSQL нет | 60 с | по TTL и чистке «молчит > 60 с» | упомянуто для полноты: TTL здесь задает, когда курьер считается пропавшим |
| Непрочитанные в чате | Redis `thread:{id}:unread:{account_id}` | write-through: `INCR` при сообщении, `DEL` при чтении | без TTL | по действию пользователя | счетчик на бейдже читается при каждом открытии приложения |
| Профиль и адреса клиента | не кэшируем | - | - | - | читается один раз на checkout по PK, PostgreSQL отдает за миллисекунды; а устаревший адрес - это заказ не туда |
| Статус заказа | не кэшируем | - | - | - | меняется каждые минуты, клиент получает изменения push-ем по WSS; stale-статус в кэше хуже одного `SELECT` по PK |
| Промокод, `PROMO_QUOTA.uses_left`, лимит клиента | не кэшируем | - | - | - | остаток квоты и `PROMO_REDEMPTION.uses` уменьшаются в транзакции резерва; кэш дал бы два применения последнего кода в регионе |
| «Мои заказы» | не кэшируем | - | - | - | индекс по `(client_id, created_at DESC)`, единицы строк на клиента; инвалидировать пришлось бы на каждую смену статуса |

Проблемы, которые надо адресовать (из чек-листа):

- **Cache stampede.** Тяжелый промах - это холодный указатель версии или тело версии, которого еще нет. Single-flight вешается на тело: `SET menu:{id}:{version}:lock NX EX 5`, остальные ждут 50–100 мс и перечитывают кэш. Пока до истечения тела текущей версии меньше 10 % TTL, первый читатель продлевает ключ в фоне, остальные получают старое значение: так текущая версия не исчезает каждые 10 минут. Для `restaurants:geo` проблемы нет: набор не истекает. Стоп-лист при промахе - короткий запрос, не сборка меню.
- **Cold start.** После рестарта Redis `restaurants:geo` пересобирается из PostgreSQL за секунды (30k строк) до того, как инстанс объявляется готовым. Меню не прогреваем: первый запрос восстанавливает указатель из `menu_version` и собирает тело, stampede ограничен single-flight. Трекинг после рестарта пуст и наполняется за 5 с - следующей точкой от каждого курьера.
- **Консистентность.** Цена и состав меню отстают, пока обработчик не переставил указатель версии: это секунды события, не TTL карточки ресторана. Стоп-лист отстает на время того же события или, если событие потеряно, не дольше 30 с. И то и другое еще раз проверяется в `ValidateAndQuote` по PostgreSQL: клиент может получить HTTP 422 «позиция недоступна», но не закажет по старой цене. То же с `is_open`: витрина может показать закрытый ресторан, заказ в него не пройдет.
- **Вытеснение и память.** Два Redis-инстанса (кластера) на регион с разной политикой: *кэш* (`allkeys-lru`, вытеснение допустимо: меню, указатели версий, стоп-листы, карточки, тарифы, ETA) и *состояние* (`noeviction`, persistence AOF: OTP, трекинг, офферы, счетчики непрочитанных, дедупликация webhook, `payroute` и `posroute`). Смешивать нельзя: LRU мог бы вытеснить OTP-код или маршрут платежа под нагрузкой на меню. Указатель версии меню живет в кэше: если его вытеснит, следующий просмотр один раз читает `menu_version` по первичному ключу и кладет указатель обратно. Оценка памяти кэша: 30k меню × ~50 КБ = 1.5 ГБ, карточки, стоп-листы и GEO - сотни МБ; состояние: 30k × 3 ключа × ~200 Б - десятки МБ, маршрутные ключи шлюзов того же порядка.

> **Stampede** (cache stampede, thundering herd) - ситуация, когда ключа в кэше нет, а запросов на него приходит много одновременно. Все они промахиваются и все идут в PostgreSQL за одним и тем же меню. База делает сотни одинаковых тяжелых запросов вместо одного, а каждый из них еще и кладет в кэш одно и то же значение. Так бывает в двух случаях: ключ истек по TTL у популярного ресторана в обед, или Redis только что перезапустился и пуст вообще.  
> **Single-flight** - прием, когда на один ключ в полете разрешен ровно один запрос к источнику. Первый промахнувшийся ставит в Redis короткую блокировку:  
> `SET menu:{id}:{version}:lock NX EX 5`  
> NX означает «только если ключа еще нет», поэтому блокировку получит один процесс. Он идет в PostgreSQL, собирает меню, пишет его в кэш и снимает блокировку. Остальные, кому SET NX не удался, не идут в базу, а ждут 50–100 мс и перечитывают кэш, где значение уже есть. EX 5 - страховка: если держатель блокировки упал, через 5 секунд она исчезнет сама и следующий возьмет работу на себя.

## 5. Очереди

Брокер нужен там, где ответ ждет человека, кухню или внешнего провайдера, а запрос клиента к этому моменту уже закрыт (ДЗ 2, раздел 5). Внутри региона один кластер Kafka, между регионом и хабом - та же Kafka через Sync Service.

### 5.1. Почему Kafka, а не RabbitMQ или очередь в PostgreSQL

| Требование | Kafka | RabbitMQ | Очередь в таблице PostgreSQL |
| --- | --- | --- | --- |
| Несколько независимых потребителей одного события (`OrderPaid` читают Delivery, POS GW, Notify, Analytics) | consumer groups, каждая читает со своего смещения | нужен fan-out exchange и очередь на потребителя, сообщение копируется | каждый потребитель должен вести свой курсор, при трех-четырех потребителях таблица превращается в брокер |
| Порядок событий одного заказа (`paid` раньше `cooking`) | гарантирован внутри партиции при ключе `order_id` | только в одной очереди с одним потребителем | `ORDER BY`, но с блокировками при конкурентных воркерах |
| Повторное чтение: новый потребитель, пересчет аналитики, восстановление после бага | retention в днях, replay с любого смещения | сообщение удалено после ack | возможно, но таблица растет |
| CDC: transactional outbox и репликация регион ↔ хаб | Debezium пишет в Kafka напрямую, offsets в Kafka | через Kafka Connect все равно нужен Kafka | - |
| Нагрузка | тысячи событий в секунду в пике на регион - малая доля возможностей | тоже справится | справится, но нагружает основную БД |

Kafka выбран не из-за объема, а из-за трех вещей: несколько потребителей на событие, порядок по ключу и replay. Эти же свойства нужны потоку в ClickHouse и Sync Service, то есть брокер один на все задачи.

### 5.2. Топики региона

| Топик | Ключ партиции | Продюсер | Потребители | Зачем асинхронно |
| --- | --- | --- | --- | --- |
| `orders` (`OrderCreated`, `OrderPaid`, `OrderCancelled`, `OrderDelivered`) | `order_id` | Orders через outbox + CDC | Delivery, POS GW, Notify, Analytics | после оплаты заказ затрагивает кухню, курьера и уведомления; держать HTTP клиента нельзя, а порядок статусов одного заказа важен |
| `payments` (`PaymentSucceeded`, `PaymentFailed`, `RefundDone`) | `order_id` | Payment GW из webhook | Orders | 3-D Secure и СБП занимают секунды-минуты; webhook приходит, когда `POST /v1/orders` давно закрыт |
| `kitchen` (`KitchenSubmitted`, `KitchenRejected`, `KitchenReady`) | `order_id` | POS GW из webhook POS | Orders, Delivery, Notify | лежачая POS одного ресторана не должна останавливать остальных: у POS GW свой retry-топик и DLQ |
| `delivery` (`CourierAssigned`, `PickedUp`, `Delivered`, `CourierCancelled`) | `order_id` | Delivery через outbox | Orders, Notify, Analytics | оффер висит десятки секунд; Orders узнает о назначении из события, а не из ответа |
| `couriers` (`CourierActivated`, `CourierBlocked`) | `courier_id` | Delivery | Auth (роль `courier`), Sync | онбординг и роли - разные сервисы с разными БД |
| `catalog` (`MenuUpdated`, `StopListChanged`, `RestaurantUpdated`) | `restaurant_id` | Catalog | Catalog-кэш, POS GW (обратная синхронизация), Analytics | инвалидация кэша и синхронизация с POS не должны блокировать панель ресторана |
| `notifications` (`SendSms`, `SendPush`, `SendEmail`) | `account_id` | Auth (OTP), Orders, Delivery, Support | Notify | шлюзы SMS и PUSH медленные и с квотами; отказ SMS не откатывает заказ |
| `support` (`MessageSent`, `ThreadAssigned`) | `thread_id` | Support через outbox | Notify (получатель офлайн), Analytics | PUSH необязателен: сообщение уже в PostgreSQL |
| `cdc.*` (строки outbox и реплицируемых таблиц) | PK таблицы | Debezium из WAL PostgreSQL | роутер outbox → доменные топики; Sync Service → хаб | единый механизм «записал в БД - событие ушло» без двойной записи |

Ключ партиции - `order_id` почти везде. Все события одного заказа попадают в одну партицию и читаются одним потребителем по порядку, а разные заказы распределяются равномерно (uuid). Партиций на старте 12 на топик: это больше, чем инстансов любого потребителя, и позволяет масштабировать потребителей без перепартиционирования. Retention 7 дней для доменных топиков, чтобы пережить выходные с лежащим потребителем и дать replay; `cdc.*` - 3 дня.

Ключ партиции не подсказывает шард Orders. В теле `payments`, `kitchen` и `delivery` лежит `client_id`: Payment GW и POS GW достают его из `payroute` и `posroute` до публикации, Delivery - из `OrderPaid` (п. 3.2). `catalog` по `StopListChanged` перезаписывает `stoplist:{restaurant_id}`, по `MenuUpdated` переставляет `menu:{restaurant_id}:ver`.

### 5.3. Гарантии доставки

- **At-least-once + идемпотентность**, не exactly-once. Продюсер - Debezium с `acks=all` и `min.insync.replicas=2`, событие не теряется после коммита транзакции в PostgreSQL. Потребитель может получить дубль при ребалансе группы, поэтому каждый пишет `processed_events(event_id)` в той же транзакции, что и свое изменение (ДЗ 2). Повтор `OrderPaid` упирается в частичный уникальный индекс `deliveries(order_id)`.
- **Transactional outbox** в Orders, Delivery, Support: событие - строка в таблице `outbox` той же транзакции, что и доменное изменение; Debezium читает WAL, роутер раскладывает по топикам. Процесс может упасть между коммитом и публикацией - строку заберет CDC.
- **DLQ** только у адаптеров внешних систем (POS GW, Notify, Payment GW): после N попыток с backoff сообщение уходит в `*.dlq`, ops видит его в панели и может повторить вручную. У доменных потребителей (Orders, Delivery) DLQ нет: ошибка обработки `PaymentSucceeded` - это баг, который надо чинить, а не откладывать; потребитель останавливает партицию и поднимает алерт.
- **Ordering.** Один заказ - одна партиция - один потребитель. Между заказами порядок не нужен.

> **DLQ** - Это отдельный топик, куда потребитель откладывает сообщение, которое не смог обработать после нескольких попыток, чтобы не блокировать остальные и не потерять его. Потребитель Kafka читает партицию по порядку. Если сообщение не обрабатывается, у него два плохих варианта: пропустить и потерять, или повторять бесконечно, и тогда все сообщения за ним стоят. DLQ дает третий: после N попыток с нарастающей паузой переложить сообщение в *.dlq, зафиксировать смещение и идти дальше. Сообщение не потеряно, его видно, его можно разобрать и повторить руками.

### 5.4. Где брокер не нужен

- `ValidateAndQuote`, `QuoteFee` на checkout и accept оффера - синхронные gRPC: клиенту нужен ответ сейчас, а решение принимает одна строка в PostgreSQL.
- Трекинг - Redis pub/sub, не Kafka: тысячи кадров в секунду, каждый нужен одному подписчику на 3–5 с, replay и ретеншн вредны. В Kafka уходит только разреженная выборка точек для аналитики (раз в 30 с на курьера).
- Чат онлайн - Redis pub/sub по той же причине; Kafka только для PUSH офлайновому получателю.
- Синхронизация регион ↔ хаб - не отдельная очередь, а те же `cdc.*` топики, которые Sync Service фильтрует и пересылает.

> **Ретеншн** (retention, удержание) - это срок, в течение которого Kafka хранит сообщения на диске после того, как их записали. В разделе 5.2 он задан как 7 дней для доменных топиков и 3 дня для cdc.*.

## 6. CAP trade-offs

CAP выбирается не для системы целиком, а для каждого типа данных и для каждой границы, где возможен разрыв сети. В этой архитектуре таких границ две: внутри региона (между нодами PostgreSQL, Redis, Kafka) и между регионом и хабом.

### 6.1. Внутри региона: CP там, где деньги и инварианты

Деньги (транзакции) - данные, где расхождение стоит реальных денег: заказ и его сумма, факт оплаты, остаток применений промокода. Выбираем Consistency + Partition tolerance

| Данные | Хранилище | Выбор | Что происходит при partition | Почему так |
| --- | --- | --- | --- | --- |
| Заказ, суммы, резерв промокода, региональная квота и лимит на клиента | PostgreSQL Orders, Catalog | **CP**: один primary, синхронная реплика (`synchronous_commit = on`), автофейловер через Patroni | если primary отрезан от реплики - запись останавливается на время фейловера (десятки секунд), клиент получает 503 на checkout. Уже созданные заказы не теряются | лучше 30 секунд без новых заказов, чем два заказа с одной квотой региона или потерянная оплата при переключении. Списывается `PROMO_QUOTA` этого региона, не реплика общего счетчика |
| Назначение курьера | PostgreSQL Delivery | **CP**: условный `UPDATE` на одном primary | при фейловере accept возвращает ошибку, курьер повторяет через несколько секунд | два курьера на один заказ - худший сценарий, чем задержка accept |
| Аккаунты, refresh-токены | PostgreSQL Auth | **CP** | вход и refresh недоступны на время фейловера; уже выданные JWT продолжают работать по JWKS | JWT делает Auth не критичным для текущих сессий |
| События (outbox → Kafka) | Kafka, `acks=all`, `min.insync.replicas=2`, RF=3 | **CP** для записи | при потере кворума продюсер (Debezium) ждет, события копятся в WAL PostgreSQL, ничего не теряется, доставка откладывается | потеря `OrderPaid` означает неприготовленный оплаченный заказ |
| Кэш меню, указатель версии, стоп-лист, карточки, тарифы | Redis-кэш | **AP**, фактически «без C»: меню отстает до обработки события, стоп-лист - до 30 с, если событие потеряно | Redis недоступен - читаем PostgreSQL напрямую (деградация по латентности), витрина живет | stale-меню и стоп-лист исправляются на checkout проверкой по PostgreSQL |
| Точки курьеров, ETA | Redis-состояние | **AP**: асинхронная репликация в кластере | при фейловере мастера теряются последние секунды точек; курьер пропадает с карты на 5 с до следующей точки | точка живет минуту, терять ее дешевле, чем ждать подтверждения реплики на каждую |
| OTP, дедупликация webhook, `payroute` / `posroute` | Redis-состояние, AOF | **AP** с оговоркой | при потере мастера клиент запросит код заново; повторный webhook ловит `processed_events` в PostgreSQL. Маршрут платежа и кухни при потере ключа берется из metadata, которую провайдер вернул в webhook; если нет и ее, событие идет в DLQ шлюза | вторая линия защиты в CP-хранилище, поэтому первая может быть AP. Поиск заказа по всем шардам из-за потерянного ключа не включается |

Заказ - целиком CP. При 100 RPS заказов цена простоя в 30 секунд - несколько тысяч клиентов, которые нажмут «повторить», а цена несогласованности - реальные деньги и двойные доставки.  
Все, что вокруг пути заказа (витрина, трекинг, чат), сделано AP и переживает падение любого компонента деградацией, а не отказом.

### 6.2. Между регионами: AP, консистентность eventual через CDC

| Данные | Направление | Выбор | Что видит пользователь при разрыве регион ↔ хаб |
| --- | --- | --- | --- |
| Профиль, адреса, карты | домашний регион → все | **AP**, свежесть минуты | клиент в чужом городе видит адреса на момент последней синхронизации; новый адрес, добавленный дома час назад, может не появиться. Заказ оформляется по локальной копии, снимок адреса в заказе все равно есть |
| Аккаунты курьеров | регион регистрации → все | **AP**, часы | курьер не сможет выйти на смену в чужом городе, если его только что активировали; дома - без ограничений |
| Бренды, определение глобального промо (код, скидка, срок, `per_client_limit`) | центр → все | **AP**, минуты | новый код появляется с задержкой. Строку определения регион не списывает; локальные промо от хаба не зависят |
| Порция квоты глобального промо | хаб → регион начислением `PROMO_GRANT` | выдача порции - **AP** (без связи новой порции нет); списание уже выданной `PROMO_QUOTA` - **CP** внутри региона | при разрыве регион тратит `uses_left`, который уже лежит у него, и этот остаток не превышает. Общий бюджет живым счетчиком не реплицируется, поэтому копии регионов его не умножают |
| Антифрод, черные списки | центр → все | **AP**, но ≤ 1 мин, с алертом при отставании | при долгом разрыве регион работает по последнему списку; риск принять заказ от только что заблокированного - принятый |
| JWKS | Auth → все | **AP**, ключи публикуются заранее | токены, выданные в другом регионе, проверяются по кэшированному набору ключей; ротация откладывается до восстановления связи |
| Аналитика | все → центр | **AP**, one-way | ClickHouse отстает; Kafka региона держит события 7 дней, после восстановления поток догоняет |

Разрыв с хабом не останавливает ни один регион: заказы, кухня, курьеры и витрина локальны по построению (ДЗ 2). Это главный выигрыш от geo-шардирования с точки зрения CAP: самая дорогая partition - между городами - не затрагивает CP-данные вообще, потому что CP-данные не пересекают границу.

### 6.3. PACELC: что мы отдаем в отсутствие partition

CAP говорит только о поведении при разрыве. В нормальном режиме выбор другой: латентность против консистентности (PACELC, «else latency or consistency»).

- **Orders, Delivery: EC.** Синхронная реплика добавляет единицы миллисекунд к каждому коммиту. Для 100 RPS это незаметно, зато фейловер без потери подтвержденных заказов.
- **Витрина: EL.** Меню читается из Redis. Свежесть задает перестановка указателя версии по событию, стоп-лист - свой короткий ключ. Клиент получает ответ за миллисекунды, а расхождение по цене и наличию ловится на checkout.
- **Трекинг: EL.** Redis без ожидания реплики; точка у клиента через 3–5 с после курьера, что и требуется.
- **Репликация между регионами: EL.** Профиль в чужом городе отстает на минуты, но checkout не ходит в домашний регион по сети. Альтернатива - синхронный запрос домой на каждый заказ в поездке - дала бы консистентность ценой зависимости пути заказа от межрегиональной сети, что противоречит п. 6.2.

> Фейловер (failover) - автоматическое переключение на резервный сервер, когда основной вышел из строя.

Единственное место, где латентность сознательно принесена в жертву консистентности, - путь заказа. Везде остальное наоборот, и граница между ними проходит ровно там, где заканчиваются финансы и начинается витрина.
