# DAED GraphQL API Документация

## Обзор

DAED (dae daemon) - это веб-интерфейс для управления проксирующим ядром dae. API построен на GraphQL и предоставляет полный набор операций для управления конфигурациями, нодами, подписками, маршрутизацией и DNS настройками.

## Базовая информация

- **Протокол**: GraphQL
- **Endpoint по умолчанию**: `${location.protocol}//${location.hostname}:2023/graphql`
- **Аутентификация**: Bearer Token
- **Транспорт**: HTTP/HTTPS

## Аутентификация

API использует JWT токены для аутентификации. Токен передается в заголовке Authorization:
```
Authorization: Bearer <token>
```

При ошибке доступа ("access denied") токен автоматически сбрасывается.

## Типы данных

### Основные скалярные типы
- `ID`: Строковый идентификатор
- `String`: Строка
- `Boolean`: Логическое значение
- `Int`: Целое число
- `Float`: Число с плавающей точкой
- `Duration`: Продолжительность времени
- `Time`: Временная метка

### Перечисления (Enums)

#### Policy
Политики балансировки нагрузки:
- `fixed` - Фиксированный узел
- `min` - Минимальная задержка
- `min_avg10` - Минимальная средняя задержка за 10 проверок
- `min_moving_avg` - Минимальная скользящая средняя (по умолчанию)
- `random` - Случайный выбор

#### LogLevel
Уровни логирования:
- `error` - Только ошибки
- `warn` - Предупреждения и ошибки
- `info` - Информационные сообщения (по умолчанию)
- `debug` - Отладочная информация
- `trace` - Подробная трассировка

#### DialMode
Режимы подключения:
- `ip` - По IP адресу
- `domain` - По доменному имени (по умолчанию)
- `domain+` - Улучшенный режим домена
- `domain++` - Расширенный режим домена

#### TcpCheckHttpMethod
HTTP методы для TCP проверок:
- `HEAD` (по умолчанию)
- `GET`, `POST`, `PUT`, `DELETE`, `PATCH`
- `CONNECT`, `OPTIONS`, `TRACE`

#### TLSImplementation
Реализации TLS:
- `tls` - Стандартный TLS (по умолчанию)
- `utls` - Маскировка TLS

### Сложные типы

#### Config
Конфигурация системы:
```graphql
type Config {
  id: ID!
  name: String!
  selected: Boolean!
  global: Global!
}
```

#### Global
Глобальные настройки конфигурации:
```graphql
type Global {
  logLevel: String!                    # Уровень логирования
  tproxyPort: Int!                     # Порт tproxy (по умолчанию: 12345)
  allowInsecure: Boolean!              # Разрешить небезопасные соединения
  checkInterval: Duration!             # Интервал проверки (по умолчанию: 30s)
  checkTolerance: Duration!            # Допуск проверки (по умолчанию: 0ms)
  lanInterface: [String!]!             # LAN интерфейсы
  wanInterface: [String!]!             # WAN интерфейсы
  udpCheckDns: [String!]!              # DNS для UDP проверок
  tcpCheckUrl: [String!]!              # URL для TCP проверок
  fallbackResolver: String!            # Резервный DNS (по умолчанию: 8.8.8.8:53)
  dialMode: String!                    # Режим подключения
  tcpCheckHttpMethod: String!          # HTTP метод для проверок
  disableWaitingNetwork: Boolean!      # Отключить ожидание сети
  autoConfigKernelParameter: Boolean!  # Автонастройка параметров ядра
  sniffingTimeout: Duration!           # Таймаут сниффинга
  tlsImplementation: String!           # Реализация TLS
  utlsImitate: String!                 # Имитация uTLS
  tproxyPortProtect: Boolean!          # Защита порта tproxy
  soMarkFromDae: Int!                  # SO_MARK от dae
  pprofPort: Int!                      # Порт профилирования
  enableLocalTcpFastRedirect: Boolean! # Быстрая переадресация TCP
  mptcp: Boolean!                      # Поддержка MPTCP
  bandwidthMaxTx: String!              # Максимальная исходящая полоса
  bandwidthMaxRx: String!              # Максимальная входящая полоса
}
```

#### Node
Узел прокси-сервера:
```graphql
type Node {
  id: ID!
  name: String!
  link: String!                    # Ссылка на конфигурацию узла
  address: String!                 # IP адрес
  protocol: String!                # Протокол (ss, vmess, trojan, etc.)
  tag: String                      # Пользовательский тег
  subscriptionID: ID               # ID подписки (если узел из подписки)
}
```

#### Subscription
Подписка на узлы:
```graphql
type Subscription {
  id: ID!
  tag: String                      # Пользовательский тег
  status: String!                  # Статус подписки
  link: String!                    # URL подписки
  info: String!                    # Информация о подписке
  updatedAt: Time!                 # Время последнего обновления
  nodes: NodesConnection!          # Узлы подписки
}
```

#### Group
Группа узлов с политикой балансировки:
```graphql
type Group {
  id: ID!
  name: String!
  nodes: [Node!]!                  # Узлы в группе
  subscriptions: [Subscription!]!  # Подписки в группе
  policy: Policy!                  # Политика балансировки
  policyParams: [Param!]!          # Параметры политики
}
```

#### Routing
Конфигурация маршрутизации:
```graphql
type Routing {
  id: ID!
  name: String!
  selected: Boolean!
  routing: DaeRouting!
  referenceGroups: [String!]!      # Группы, на которые ссылается маршрутизация
}
```

#### DNS
Конфигурация DNS:
```graphql
type Dns {
  id: ID!
  name: String!
  selected: Boolean!
  dns: DaeDns!
}
```

## Query операции

### Пользователи и аутентификация

#### numberUsers
Получить количество пользователей в системе:
```graphql
query NumberUsers {
  numberUsers
}
```

#### token
Получить токен аутентификации:
```graphql
query Token($username: String!, $password: String!) {
  token(username: $username, password: $password)
}
```

#### user
Получить информацию о текущем пользователе:
```graphql
query User {
  user {
    username
    name
    avatar
  }
}
```

### Системная информация

#### general
Получить общую информацию о системе:
```graphql
query General($up: Boolean) {
  general {
    dae {
      running      # Запущен ли dae
      modified     # Изменена ли конфигурация
      version      # Версия dae
    }
    interfaces(up: $up) {
      name
      ifindex
      ip
      flag {
        default {
          gateway
        }
      }
    }
  }
}
```

#### interfaces
Получить информацию о сетевых интерфейсах:
```graphql
query Interfaces($up: Boolean) {
  general {
    interfaces(up: $up) {
      name
      ifindex
      ip
      flag {
        default {
          gateway
        }
      }
    }
  }
}
```

#### jsonStorage
Получить данные из JSON хранилища:
```graphql
query JsonStorage($paths: [String!]) {
  jsonStorage(paths: $paths)
}
```

### Конфигурации

#### configs
Получить список конфигураций:
```graphql
query Configs {
  configs {
    id
    name
    selected
    global {
      logLevel
      tproxyPort
      allowInsecure
      checkInterval
      checkTolerance
      lanInterface
      wanInterface
      udpCheckDns
      tcpCheckUrl
      fallbackResolver
      dialMode
      tcpCheckHttpMethod
      disableWaitingNetwork
      autoConfigKernelParameter
      sniffingTimeout
      tlsImplementation
      utlsImitate
      tproxyPortProtect
      soMarkFromDae
      pprofPort
      enableLocalTcpFastRedirect
      mptcp
      bandwidthMaxTx
      bandwidthMaxRx
    }
  }
}
```

### Узлы

#### nodes
Получить список узлов:
```graphql
query Nodes($first: Int, $after: ID, $id: ID, $subscriptionId: ID) {
  nodes(first: $first, after: $after, id: $id, subscriptionId: $subscriptionId) {
    edges {
      id
      name
      link
      address
      protocol
      tag
    }
    pageInfo {
      hasNextPage
      endCursor
      startCursor
    }
    totalCount
  }
}
```

### Подписки

#### subscriptions
Получить список подписок:
```graphql
query Subscriptions($id: ID) {
  subscriptions(id: $id) {
    id
    tag
    status
    link
    info
    updatedAt
    nodes {
      edges {
        id
        name
        protocol
        link
      }
    }
  }
}
```

### Группы

#### groups
Получить список групп:
```graphql
query Groups($id: ID) {
  groups(id: $id) {
    id
    name
    policy
    nodes {
      id
      name
      address
      protocol
      tag
      subscriptionID
    }
    subscriptions {
      id
      tag
      link
      status
      info
      updatedAt
    }
    policyParams {
      key
      val
    }
  }
}
```

### Маршрутизация

#### routings
Получить список конфигураций маршрутизации:
```graphql
query Routings($id: ID, $selected: Boolean) {
  routings(id: $id, selected: $selected) {
    id
    name
    selected
    routing {
      string
    }
  }
}
```

### DNS

#### dnss
Получить список DNS конфигураций:
```graphql
query DNSs($id: ID, $selected: Boolean) {
  dnss(id: $id, selected: $selected) {
    id
    name
    selected
    dns {
      string
      routing {
        request {
          string
        }
        response {
          string
        }
      }
    }
  }
}
```

## Mutation операции

### Пользователи

#### createUser
Создать нового пользователя:
```graphql
mutation CreateUser($username: String!, $password: String!) {
  createUser(username: $username, password: $password)
}
```

**Валидация:**
- `username`: 4-20 символов, обязательно
- `password`: 6-20 символов, обязательно

#### updateAvatar
Обновить аватар пользователя:
```graphql
mutation UpdateAvatar($avatar: String) {
  updateAvatar(avatar: $avatar)
}
```

#### updateName
Обновить имя пользователя:
```graphql
mutation UpdateName($name: String) {
  updateName(name: $name)
}
```

### Конфигурации

#### createConfig
Создать новую конфигурацию:
```graphql
mutation CreateConfig($name: String, $global: GlobalInput) {
  createConfig(name: $name, global: $global) {
    id
  }
}
```

#### updateConfig
Обновить конфигурацию:
```graphql
mutation UpdateConfig($id: ID!, $global: GlobalInput!) {
  updateConfig(id: $id, global: $global) {
    id
  }
}
```

#### removeConfig
Удалить конфигурацию:
```graphql
mutation RemoveConfig($id: ID!) {
  removeConfig(id: $id)
}
```

#### selectConfig
Выбрать активную конфигурацию:
```graphql
mutation SelectConfig($id: ID!) {
  selectConfig(id: $id)
}
```

#### renameConfig
Переименовать конфигурацию:
```graphql
mutation RenameConfig($id: ID!, $name: String!) {
  renameConfig(id: $id, name: $name)
}
```

### Узлы

#### importNodes
Импортировать узлы:
```graphql
mutation ImportNodes($rollbackError: Boolean!, $args: [ImportArgument!]!) {
  importNodes(rollbackError: $rollbackError, args: $args) {
    link
    error
    node {
      id
    }
  }
}
```

**ImportArgument:**
```graphql
input ImportArgument {
  link: String!     # Ссылка на конфигурацию узла
  tag: String       # Пользовательский тег
}
```

#### removeNodes
Удалить узлы:
```graphql
mutation RemoveNodes($ids: [ID!]!) {
  removeNodes(ids: $ids)
}
```

### Подписки

#### importSubscription
Импортировать подписку:
```graphql
mutation ImportSubscription($rollbackError: Boolean!, $arg: ImportArgument!) {
  importSubscription(rollbackError: $rollbackError, arg: $arg) {
    link
    sub {
      id
    }
    nodeImportResult {
      node {
        id
      }
    }
  }
}
```

#### updateSubscription
Обновить подписку:
```graphql
mutation UpdateSubscription($id: ID!) {
  updateSubscription(id: $id) {
    id
  }
}
```

#### removeSubscriptions
Удалить подписки:
```graphql
mutation RemoveSubscriptions($ids: [ID!]!) {
  removeSubscriptions(ids: $ids)
}
```

### Группы

#### createGroup
Создать новую группу:
```graphql
mutation CreateGroup($name: String!, $policy: Policy!, $policyParams: [PolicyParam!]) {
  createGroup(name: $name, policy: $policy, policyParams: $policyParams) {
    id
  }
}
```

#### removeGroup
Удалить группу:
```graphql
mutation RemoveGroup($id: ID!) {
  removeGroup(id: $id)
}
```

#### renameGroup
Переименовать группу:
```graphql
mutation RenameGroup($id: ID!, $name: String!) {
  renameGroup(id: $id, name: $name)
}
```

#### groupSetPolicy
Установить политику группы:
```graphql
mutation GroupSetPolicy($id: ID!, $policy: Policy!, $policyParams: [PolicyParam!]) {
  groupSetPolicy(id: $id, policy: $policy, policyParams: $policyParams)
}
```

#### groupAddNodes
Добавить узлы в группу:
```graphql
mutation GroupAddNodes($id: ID!, $nodeIDs: [ID!]!) {
  groupAddNodes(id: $id, nodeIDs: $nodeIDs)
}
```

#### groupDelNodes
Удалить узлы из группы:
```graphql
mutation GroupDelNodes($id: ID!, $nodeIDs: [ID!]!) {
  groupDelNodes(id: $id, nodeIDs: $nodeIDs)
}
```

#### groupAddSubscriptions
Добавить подписки в группу:
```graphql
mutation GroupAddSubscriptions($id: ID!, $subscriptionIDs: [ID!]!) {
  groupAddSubscriptions(id: $id, subscriptionIDs: $subscriptionIDs)
}
```

#### groupDelSubscriptions
Удалить подписки из группы:
```graphql
mutation GroupDelSubscriptions($id: ID!, $subscriptionIDs: [ID!]!) {
  groupDelSubscriptions(id: $id, subscriptionIDs: $subscriptionIDs)
}
```

### Маршрутизация

#### createRouting
Создать конфигурацию маршрутизации:
```graphql
mutation CreateRouting($name: String, $routing: String) {
  createRouting(name: $name, routing: $routing) {
    id
  }
}
```

#### updateRouting
Обновить конфигурацию маршрутизации:
```graphql
mutation UpdateRouting($id: ID!, $routing: String!) {
  updateRouting(id: $id, routing: $routing) {
    id
  }
}
```

#### removeRouting
Удалить конфигурацию маршрутизации:
```graphql
mutation RemoveRouting($id: ID!) {
  removeRouting(id: $id)
}
```

#### selectRouting
Выбрать активную конфигурацию маршрутизации:
```graphql
mutation SelectRouting($id: ID!) {
  selectRouting(id: $id)
}
```

#### renameRouting
Переименовать конфигурацию маршрутизации:
```graphql
mutation RenameRouting($id: ID!, $name: String!) {
  renameRouting(id: $id, name: $name)
}
```

### DNS

#### createDns
Создать DNS конфигурацию:
```graphql
mutation CreateDNS($name: String, $dns: String) {
  createDns(name: $name, dns: $dns) {
    id
  }
}
```

#### updateDns
Обновить DNS конфигурацию:
```graphql
mutation UpdateDNS($id: ID!, $dns: String!) {
  updateDns(id: $id, dns: $dns) {
    id
  }
}
```

#### removeDns
Удалить DNS конфигурацию:
```graphql
mutation RemoveDNS($id: ID!) {
  removeDns(id: $id)
}
```

#### selectDns
Выбрать активную DNS конфигурацию:
```graphql
mutation SelectDNS($id: ID!) {
  selectDns(id: $id)
}
```

#### renameDns
Переименовать DNS конфигурацию:
```graphql
mutation RenameDNS($id: ID!, $name: String!) {
  renameDns(id: $id, name: $name)
}
```

### Системные операции

#### run
Запустить/перезапустить dae:
```graphql
mutation Run($dry: Boolean!) {
  run(dry: $dry)
}
```

#### setJsonStorage
Сохранить данные в JSON хранилище:
```graphql
mutation SetJsonStorage($paths: [String!]!, $values: [String!]!) {
  setJsonStorage(paths: $paths, values: $values)
}
```

## Значения по умолчанию

### Конфигурация
- **Имя конфигурации**: "global"
- **Порт tproxy**: 12345
- **Уровень логирования**: "info"
- **Интервал проверки**: 30 секунд
- **DNS для UDP проверок**: `['dns.google:53', '8.8.8.8', '2001:4860:4860::8888']`
- **URL для TCP проверок**: `['http://cp.cloudflare.com', '1.1.1.1', '2606:4700:4700::1111']`
- **Резервный DNS**: "8.8.8.8:53"

### Маршрутизация по умолчанию
```
pname(NetworkManager, systemd-resolved, dnsmasq) -> must_direct
dip(geoip:private) -> direct
dip(geoip:cn) -> direct
domain(geosite:cn) -> direct
fallback: proxy
```

### DNS по умолчанию
```
upstream {
  alidns: 'udp://223.5.5.5:53'
  googledns: 'tcp+udp://8.8.8.8:53'
}
routing {
  request {
    qname(geosite:cn) -> alidns
    fallback: googledns
  }
}
```

## Поддерживаемые протоколы узлов

### VMess/V2Ray
Параметры схемы v2raySchema:
- `ps`: Описание
- `add`: Адрес сервера (обязательно)
- `port`: Порт (0-65535)
- `id`: UUID (обязательно)
- `aid`: AlterID (0-65535)
- `net`: Сеть ('tcp', 'kcp', 'ws', 'h2', 'grpc')
- `type`: Тип ('none', 'http', 'srtp', 'utp', 'wechat-video', 'dtls', 'wireguard')
- `host`: Хост
- `path`: Путь
- `tls`: TLS ('none', 'tls')
- `scy`: Шифрование ('auto', 'aes-128-gcm', 'chacha20-poly1305', 'none', 'zero')

### Shadowsocks
Параметры схемы ssSchema:
- `method`: Метод шифрования
- `plugin`: Плагин ('', 'simple-obfs', 'v2ray-plugin')
- `password`: Пароль (обязательно)
- `server`: Сервер (обязательно)
- `port`: Порт (0-65535)

### ShadowsocksR
Параметры схемы ssrSchema:
- `method`: Метод шифрования
- `proto`: Протокол
- `obfs`: Обфускация
- `password`: Пароль (обязательно)
- `server`: Сервер (обязательно)
- `port`: Порт (0-65535)

### Trojan
Параметры схемы trojanSchema:
- `server`: Сервер (обязательно)
- `port`: Порт (0-65535)
- `password`: Пароль (обязательно)
- `method`: Метод ('origin', 'shadowsocks')
- `obfs`: Обфускация ('none', 'websocket')

### TUIC
Параметры схемы tuicSchema:
- `server`: Сервер (обязательно)
- `port`: Порт (0-65535)
- `uuid`: UUID (обязательно)
- `password`: Пароль (обязательно)

### Juicity
Параметры схемы juicitySchema:
- `server`: Сервер (обязательно)
- `port`: Порт (0-65535)
- `uuid`: UUID (обязательно)
- `password`: Пароль (обязательно)

### Hysteria2
Параметры схемы hysteria2Schema:
- `server`: Сервер (обязательно)
- `port`: Порт (0-65535, по умолчанию: 443)
- `auth`: Аутентификация

### HTTP/SOCKS5
Параметры для HTTP и SOCKS5 прокси:
- `host`: Хост (обязательно)
- `port`: Порт (0-65535)
- `username`: Имя пользователя
- `password`: Пароль

## Обработка ошибок

API автоматически обрабатывает ошибки через middleware. При получении ошибки:
1. Показывается уведомление с сообщением об ошибке
2. При ошибке "access denied" токен автоматически сбрасывается
3. Ошибки возвращаются в стандартном формате GraphQL

## Примеры использования

### Настройка системы (Setup Flow)

1. **Проверка количества пользователей:**
```graphql
query {
  numberUsers
}
```

2. **Создание первого пользователя (если numberUsers = 0):**
```graphql
mutation {
  createUser(username: "admin", password: "password123")
}
```

3. **Вход в систему:**
```graphql
query {
  token(username: "admin", password: "password123")
}
```

### Создание базовой конфигурации

1. **Создание конфигурации:**
```graphql
mutation {
  createConfig(
    name: "my-config"
    global: {
      logLevel: "info"
      tproxyPort: 12345
      lanInterface: ["eth0"]
      wanInterface: ["auto"]
    }
  ) {
    id
  }
}
```

2. **Создание группы:**
```graphql
mutation {
  createGroup(
    name: "proxy"
    policy: min_moving_avg
  ) {
    id
  }
}
```

3. **Импорт узлов:**
```graphql
mutation {
  importNodes(
    rollbackError: true
    args: [
      {
        link: "vmess://..."
        tag: "server1"
      }
    ]
  ) {
    link
    error
    node {
      id
    }
  }
}
```

## Константы и настройки по умолчанию

### Ключи запросов (Query Keys)
- `QUERY_KEY_HEALTH_CHECK`: ['healthCheck']
- `QUERY_KEY_GENERAL`: ['general']
- `QUERY_KEY_USER`: ['user']
- `QUERY_KEY_NODE`: ['node']
- `QUERY_KEY_SUBSCRIPTION`: ['subscription']
- `QUERY_KEY_CONFIG`: ['config']
- `QUERY_KEY_ROUTING`: ['routing']
- `QUERY_KEY_DNS`: ['dns']
- `QUERY_KEY_GROUP`: ['group']

### Режимы работы
- `MODE.simple`: Простой режим
- `MODE.advanced`: Расширенный режим

### Типы ресурсов для Drag & Drop
- `DraggableResourceType.node`: Узел
- `DraggableResourceType.subscription`: Подписка
- `DraggableResourceType.subscription_node`: Узел подписки
- `DraggableResourceType.groupNode`: Узел группы
- `DraggableResourceType.groupSubscription`: Подписка группы

## Состояние приложения

Приложение использует nanostores для управления глобальным состоянием:

- `endpointURLAtom`: URL GraphQL endpoint
- `tokenAtom`: JWT токен аутентификации

## Контексты React

### GQLClientContext
Предоставляет GraphQL клиент для выполнения запросов:
```typescript
const client = useGQLQueryClient()
```

### QueryProvider
Обертка для React Query и GraphQL клиента с автоматической обработкой ошибок и управлением токенами.

## Валидация данных

Все формы используют Zod схемы для валидации:

### Endpoint URL
```typescript
const endpointURLSchema = z.object({
  endpointURL: z.string().url().nonempty(),
})
```

### Регистрация/Вход
```typescript
const signupSchema = z.object({
  username: z.string().min(4).max(20).nonempty(),
  password: z.string().min(6).max(20).nonempty(),
})
```

## Интернационализация

API поддерживает многоязычность через react-i18next. Основные переводимые ключи:

- `welcome to` - Приветствие
- `what for` - Описание назначения
- `step` - Шаг в процессе настройки
- `setup endpoint` - Настройка endpoint
- `login account` - Вход в аккаунт
- `endpointURL` - URL endpoint
- `username` - Имя пользователя
- `password` - Пароль
- `actions.continue` - Продолжить
- `actions.create account` - Создать аккаунт
- `actions.login` - Войти
- `actions.start your journey` - Начать работу
- `notifications.login succeeded` - Успешный вход

## Схемы протоколов

### V2Ray Schema
```typescript
const v2raySchema = z.object({
  ps: z.string(),                    // Псевдоним
  add: z.string().nonempty(),        // Адрес сервера
  port: z.number().min(0).max(65535), // Порт
  id: z.string().nonempty(),         // UUID
  aid: z.number().min(0).max(65535), // AlterID
  net: z.enum(['tcp', 'kcp', 'ws', 'h2', 'grpc']), // Тип сети
  type: z.enum(['none', 'http', 'srtp', 'utp', 'wechat-video', 'dtls', 'wireguard']), // Тип обфускации
  host: z.string(),                  // Хост
  path: z.string(),                  // Путь
  tls: z.enum(['none', 'tls']),      // TLS
  flow: z.enum(['none', 'xtls-rprx-origin', 'xtls-rprx-origin-udp443', 'xtls-rprx-vision', 'xtls-rprx-vision-udp443']), // Поток XTLS
  alpn: z.string(),                  // ALPN
  scy: z.enum(['auto', 'aes-128-gcm', 'chacha20-poly1305', 'none', 'zero']), // Шифрование
  v: z.literal(''),                  // Версия
  allowInsecure: z.boolean(),        // Разрешить небезопасные соединения
  sni: z.string(),                   // SNI
})
```

### Shadowsocks Schema
```typescript
const ssSchema = z.object({
  method: z.enum(['aes-128-gcm', 'aes-256-gcm', 'chacha20-poly1305', 'chacha20-ietf-poly1305', 'plain', 'none']), // Метод шифрования
  plugin: z.enum(['', 'simple-obfs', 'v2ray-plugin']), // Плагин
  obfs: z.enum(['http', 'tls']),     // Обфускация
  tls: z.enum(['', 'tls']),          // TLS
  path: z.string(),                  // Путь
  mode: z.string(),                  // Режим
  host: z.string(),                  // Хост
  password: z.string().nonempty(),   // Пароль
  server: z.string().nonempty(),     // Сервер
  port: z.number().min(0).max(65535), // Порт
  name: z.string(),                  // Имя
  impl: z.enum(['', 'chained', 'transport']), // Реализация
})
```

### ShadowsocksR Schema
```typescript
const ssrSchema = z.object({
  method: z.enum([
    'aes-128-cfb', 'aes-192-cfb', 'aes-256-cfb',
    'aes-128-ctr', 'aes-192-ctr', 'aes-256-ctr',
    'aes-128-ofb', 'aes-192-ofb', 'aes-256-ofb',
    'des-cfb', 'bf-cfb', 'cast5-cfb', 'rc4-md5',
    'chacha20-ietf', 'salsa20',
    'camellia-128-cfb', 'camellia-192-cfb', 'camellia-256-cfb',
    'idea-cfb', 'rc2-cfb', 'seed-cfb', 'none'
  ]), // Метод шифрования
  password: z.string().nonempty(),   // Пароль
  server: z.string().nonempty(),     // Сервер
  port: z.number().min(0).max(65535).positive(), // Порт
  name: z.string(),                  // Имя
  proto: z.enum([
    'origin', 'verify_sha1', 'auth_sha1_v4',
    'auth_aes128_md5', 'auth_aes128_sha1',
    'auth_chain_a', 'auth_chain_b'
  ]), // Протокол
  protoParam: z.string(),            // Параметры протокола
  obfs: z.enum(['plain', 'http_simple', 'http_post', 'random_head', 'tls1.2_ticket_auth']), // Обфускация
  obfsParam: z.string(),             // Параметры обфускации
})
```

### Trojan Schema
```typescript
const trojanSchema = z.object({
  name: z.string(),                  // Имя
  server: z.string().nonempty(),     // Сервер
  peer: z.string(),                  // Peer
  host: z.string(),                  // Хост
  path: z.string(),                  // Путь
  allowInsecure: z.boolean(),        // Разрешить небезопасные соединения
  port: z.number().min(0).max(65535), // Порт
  password: z.string().nonempty(),   // Пароль
  method: z.enum(['origin', 'shadowsocks']), // Метод
  ssCipher: z.enum(['aes-128-gcm', 'aes-256-gcm', 'chacha20-poly1305', 'chacha20-ietf-poly1305']), // Шифр SS
  ssPassword: z.string(),            // Пароль SS
  obfs: z.enum(['none', 'websocket']), // Обфускация
})
```

### TUIC Schema
```typescript
const tuicSchema = z.object({
  name: z.string(),                  // Имя
  server: z.string().nonempty(),     // Сервер
  port: z.number().min(0).max(65535), // Порт
  uuid: z.string().nonempty(),       // UUID
  password: z.string().nonempty(),   // Пароль
  allowInsecure: z.boolean(),        // Разрешить небезопасные соединения
  disable_sni: z.boolean(),          // Отключить SNI
  sni: z.string(),                   // SNI
  congestion_control: z.string(),    // Контроль перегрузки
  alpn: z.string(),                  // ALPN
  udp_relay_mode: z.string(),        // Режим UDP relay
})
```

### Juicity Schema
```typescript
const juicitySchema = z.object({
  name: z.string(),                  // Имя
  server: z.string().nonempty(),     // Сервер
  port: z.number().min(0).max(65535), // Порт
  uuid: z.string().nonempty(),       // UUID
  password: z.string().nonempty(),   // Пароль
  allowInsecure: z.boolean(),        // Разрешить небезопасные соединения
  pinned_certchain_sha256: z.string(), // SHA256 цепочки сертификатов
  sni: z.string(),                   // SNI
  congestion_control: z.string(),    // Контроль перегрузки
})
```

### Hysteria2 Schema
```typescript
const hysteria2Schema = z.object({
  name: z.string(),                  // Имя
  server: z.string().nonempty(),     // Сервер
  port: z.number().min(0).max(65535), // Порт
  auth: z.string(),                  // Аутентификация
  obfs: z.string(),                  // Обфускация
  obfsPassword: z.string(),          // Пароль обфускации
  sni: z.string(),                   // SNI
  allowInsecure: z.boolean(),        // Разрешить небезопасные соединения
  pinSHA256: z.string(),             // Закрепленный SHA256
})
```

### HTTP/SOCKS5 Schema
```typescript
const httpSchema = z.object({
  username: z.string(),              // Имя пользователя
  password: z.string(),              // Пароль
  host: z.string().nonempty(),       // Хост
  port: z.number().min(0).max(65535), // Порт
  name: z.string(),                  // Имя
})

const socks5Schema = z.object({
  username: z.string(),              // Имя пользователя
  password: z.string(),              // Пароль
  host: z.string().nonempty(),       // Хост
  port: z.number().min(0).max(65535), // Порт
  name: z.string(),                  // Имя
})
```

## Расширенные типы GraphQL

### DaeRouting
Структура маршрутизации dae:
```graphql
type DaeRouting {
  string: String!                    # Строковое представление
  fallback: FunctionOrPlaintext!     # Fallback правило
  rules: [RoutingRule!]!             # Правила маршрутизации
}
```

### DaeDns
Структура DNS конфигурации dae:
```graphql
type DaeDns {
  string: String!                    # Строковое представление
  upstream: [Param!]!                # Upstream серверы
  routing: DnsRouting!               # DNS маршрутизация
}
```

### RoutingRule
Правило маршрутизации:
```graphql
type RoutingRule {
  conditions: AndFunctions!          # Условия (логическое И)
  outbound: Function!                # Исходящее действие
}
```

### Function
Функция в правилах:
```graphql
type Function {
  name: String!                      # Имя функции
  not: Boolean!                      # Инверсия
  params: [Param!]!                  # Параметры
}
```

### Param
Параметр ключ-значение:
```graphql
type Param {
  key: String!                       # Ключ
  val: String!                       # Значение
}
```

### Interface
Сетевой интерфейс:
```graphql
type Interface {
  name: String!                      # Имя интерфейса
  ifindex: Int!                      # Индекс интерфейса
  ip: [String!]!                     # IP адреса
  flag: InterfaceFlag!               # Флаги интерфейса
}
```

### InterfaceFlag
Флаги сетевого интерфейса:
```graphql
type InterfaceFlag {
  up: Boolean!                       # Интерфейс активен
  default: [DefaultRoute]            # Маршруты по умолчанию
}
```

### DefaultRoute
Маршрут по умолчанию:
```graphql
type DefaultRoute {
  gateway: String                    # Шлюз
  ipVersion: String!                 # Версия IP
  source: String                     # Источник
}
```

## Полезные утилиты

### Fragment Masking
API предоставляет утилиты для работы с GraphQL фрагментами:

```typescript
// Использование фрагмента
function useFragment<TType>(
  documentNode: DocumentTypeDecoration<TType, any>,
  fragmentType: FragmentType<DocumentTypeDecoration<TType, any>>
): TType

// Создание данных фрагмента
function makeFragmentData<F, FT>(
  data: FT,
  fragment: F
): FragmentType<F>

// Проверка готовности фрагмента
function isFragmentReady<TQuery, TFrag>(
  queryNode: DocumentTypeDecoration<TQuery, any>,
  fragmentNode: TypedDocumentNode<TFrag>,
  data: FragmentType<TypedDocumentNode<Incremental<TFrag>, any>>
): boolean
```

## Конфигурационные паттерны

### Типичная конфигурация с LAN интерфейсами
```typescript
const DEFAULT_CONFIG_WITH_LAN_INTERFACEs = (interfaces: string[] = []): GlobalInput => ({
  logLevel: 'info',
  tproxyPort: 12345,
  tproxyPortProtect: true,
  pprofPort: 0,
  soMarkFromDae: 0,
  allowInsecure: false,
  checkInterval: '30s',
  checkTolerance: '0ms',
  sniffingTimeout: '100ms',
  lanInterface: interfaces,
  wanInterface: ['auto'],
  udpCheckDns: ['dns.google:53', '8.8.8.8', '2001:4860:4860::8888'],
  tcpCheckUrl: ['http://cp.cloudflare.com', '1.1.1.1', '2606:4700:4700::1111'],
  tcpCheckHttpMethod: 'HEAD',
  dialMode: 'domain',
  autoConfigKernelParameter: true,
  tlsImplementation: 'tls',
  utlsImitate: 'chrome_auto',
  disableWaitingNetwork: false,
  enableLocalTcpFastRedirect: false,
  mptcp: false,
  bandwidthMaxTx: '200 mbps',
  bandwidthMaxRx: '1 gbps',
  fallbackResolver: '8.8.8.8:53',
})
```

## Обработка соединений

### NodesConnection
Пагинированный список узлов:
```graphql
type NodesConnection {
  edges: [Node!]!                    # Список узлов
  pageInfo: PageInfo!                # Информация о пагинации
  totalCount: Int!                   # Общее количество
}
```

### PageInfo
Информация о пагинации:
```graphql
type PageInfo {
  hasNextPage: Boolean!              # Есть ли следующая страница
  endCursor: ID                      # Курсор конца
  startCursor: ID                    # Курсор начала
}
```

## Результаты импорта

### NodeImportResult
Результат импорта узла:
```graphql
type NodeImportResult {
  link: String!                      # Исходная ссылка
  error: String                      # Ошибка импорта
  node: Node                         # Импортированный узел
}
```

### SubscriptionImportResult
Результат импорта подписки:
```graphql
type SubscriptionImportResult {
  link: String!                      # Исходная ссылка
  sub: Subscription!                 # Импортированная подписка
  nodeImportResult: [NodeImportResult!]! # Результаты импорта узлов
}
```

## Примеры сложных операций

### Полная настройка с нуля

1. **Проверка состояния системы:**
```graphql
query SystemStatus {
  numberUsers
  general {
    dae {
      running
      version
    }
    interfaces(up: true) {
      name
      ip
      flag {
        default {
          gateway
        }
      }
    }
  }
}
```

2. **Создание пользователя и получение токена:**
```graphql
# Если numberUsers = 0
mutation CreateFirstUser {
  createUser(username: "admin", password: "securepassword")
}

# Получение токена
query GetToken {
  token(username: "admin", password: "securepassword")
}
```

3. **Создание базовой конфигурации:**
```graphql
mutation SetupBasicConfig {
  config: createConfig(name: "main") {
    id
  }
  
  routing: createRouting(
    name: "default"
    routing: """
      pname(NetworkManager, systemd-resolved, dnsmasq) -> must_direct
      dip(geoip:private) -> direct
      dip(geoip:cn) -> direct
      domain(geosite:cn) -> direct
      fallback: proxy
    """
  ) {
    id
  }
  
  dns: createDns(
    name: "default"
    dns: """
      upstream {
        alidns: 'udp://223.5.5.5:53'
        googledns: 'tcp+udp://8.8.8.8:53'
      }
      routing {
        request {
          qname(geosite:cn) -> alidns
          fallback: googledns
        }
      }
    """
  ) {
    id
  }
  
  group: createGroup(
    name: "proxy"
    policy: min_moving_avg
  ) {
    id
  }
}
```

4. **Добавление узлов и подписок:**
```graphql
# Импорт отдельных узлов
mutation ImportProxyNodes {
  importNodes(
    rollbackError: true
    args: [
      { link: "vmess://...", tag: "server1" }
      { link: "ss://...", tag: "server2" }
    ]
  ) {
    link
    error
    node {
      id
      name
      protocol
    }
  }
}

# Импорт подписки
mutation ImportProxySubscription {
  importSubscription(
    rollbackError: true
    arg: { link: "https://example.com/subscription", tag: "my-sub" }
  ) {
    link
    sub {
      id
      status
    }
    nodeImportResult {
      node {
        id
        name
      }
      error
    }
  }
}
```

5. **Добавление ресурсов в группу:**
```graphql
mutation AddResourcesToGroup {
  addNodes: groupAddNodes(
    id: "group-id"
    nodeIDs: ["node1", "node2"]
  )
  
  addSubscriptions: groupAddSubscriptions(
    id: "group-id"
    subscriptionIDs: ["sub1"]
  )
}
```

6. **Запуск системы:**
```graphql
# Проверка конфигурации (dry run)
mutation TestConfig {
  run(dry: true)
}

# Применение конфигурации
mutation ApplyConfig {
  run(dry: false)
}
```

## Мониторинг и диагностика

### Health Check
```graphql
query HealthCheck {
  healthCheck
}
```

### Состояние dae
```graphql
query DaeStatus {
  general {
    dae {
      running      # Запущен ли процесс
      modified     # Изменена ли конфигурация
      version      # Версия dae
    }
  }
}
```

### Схема конфигурации
```graphql
query ConfigSchema {
  general {
    schema         # JSON Schema конфигурации
  }
  
  configFlatDesc   # Плоское описание конфигурации
}
```

## Управление хранилищем

### Чтение настроек
```graphql
query GetSettings {
  mode: jsonStorage(paths: ["mode"])
  defaults: jsonStorage(paths: ["defaults"])
}
```

### Сохранение настроек
```graphql
mutation SaveSettings {
  setJsonStorage(
    paths: ["mode", "theme"]
    values: ["advanced", "dark"]
  )
}
```

## Продвинутые функции

### Парсинг конфигураций
```graphql
query ParseConfigs {
  parsedRouting(raw: "fallback: proxy") {
    string
    rules {
      conditions {
        and {
          name
          params {
            key
            val
          }
        }
      }
      outbound {
        name
        params {
          key
          val
        }
      }
    }
  }
  
  parsedDns(raw: "upstream { google: '8.8.8.8' }") {
    string
    upstream {
      key
      val
    }
    routing {
      request {
        string
      }
      response {
        string
      }
    }
  }
}
```

## Константы приложения

### Порты и адреса
- `DEFAULT_TPROXY_PORT`: 12345
- `DEFAULT_PPROF_PORT`: 0
- `DEFAULT_SO_MARK_FROM_DAE`: 0

### Временные интервалы
- `DEFAULT_CHECK_INTERVAL_SECONDS`: 30
- `DEFAULT_CHECK_TOLERANCE_MS`: 0
- `DEFAULT_SNIFFING_TIMEOUT_MS`: 100

### Сетевые настройки
- `DEFAULT_UDP_CHECK_DNS`: ['dns.google:53', '8.8.8.8', '2001:4860:4860::8888']
- `DEFAULT_TCP_CHECK_URL`: ['http://cp.cloudflare.com', '1.1.1.1', '2606:4700:4700::1111']
- `DEFAULT_FALLBACK_RESOLVER`: '8.8.8.8:53'

### Пропускная способность
- `DEFAULT_BANDWIDTH_MAX_TX`: '200 mbps'
- `DEFAULT_BANDWIDTH_MAX_RX`: '1 gbps'

## Безопасность

### Валидация
- Все входные данные проходят валидацию через Zod схемы
- URL endpoint должен быть валидным URL
- Пароли должны быть от 6 до 20 символов
- Имена пользователей от 4 до 20 символов

### Обработка ошибок
- Автоматическое отображение ошибок через уведомления
- Автоматический сброс токена при ошибках доступа
- Rollback при ошибках импорта (опционально)

## Интеграция с клиентом

### React Query
API интегрирован с React Query для кэширования и синхронизации данных:

```typescript
// Использование в компоненте
const { data: configs } = useQuery(['config'], () => 
  client.request(graphql(`
    query Configs {
      configs {
        id
        name
        selected
      }
    }
  `))
)
```

### Nanostores
Глобальное состояние управляется через nanostores:

```typescript
import { endpointURLAtom, tokenAtom } from '~/store'

// Чтение состояния
const endpointURL = useStore(endpointURLAtom)
const token = useStore(tokenAtom)

// Изменение состояния
endpointURLAtom.set('https://new-endpoint.com/graphql')
tokenAtom.set('new-jwt-token')
```

### TypeScript Support
API полностью типизирован с использованием кодогенерации GraphQL:

```typescript
import { graphql } from '~/schemas/gql'
import type { NodesQuery, NodesQueryVariables } from '~/schemas/gql/graphql'

const query = graphql(`
  query Nodes {
    nodes {
      edges {
        id
        name
        protocol
      }
    }
  }
`)
```

Эта документация покрывает все основные аспекты DAED GraphQL API, включая все операции, типы данных, схемы валидации и примеры использования, основанные на анализе предоставленных исходных файлов.