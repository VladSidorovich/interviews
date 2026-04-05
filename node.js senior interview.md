# Блок 1: Node.js Core & Performance

---

## Группа 1: Event Loop & асинхронность (вопросы 1–5)

### 1. ⭐ Какие операции с файловой системой есть в модуле node:fs?

**🟡 Junior**
Знает `readFile`, `writeFile`, понимает что есть sync-версии. Использует callback-стиль.

**🟠 Middle**
Знает все три API (callback / sync / promise), умеет выбрать нужный:
```js
// Callback
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Sync — блокирует Event Loop
const data = fs.readFileSync('file.txt', 'utf8');

// Promise — основной выбор
import { readFile } from 'node:fs/promises';
const data = await readFile('file.txt', 'utf8');
```

Перечисляет основные группы операций:
```js
import { readFile, writeFile, appendFile,
         rename, unlink, mkdir, rm,
         stat, access, readdir, copyFile } from 'node:fs/promises';
```
- **Чтение:** `readFile`, `readdir`, `stat`, `access`
- **Запись:** `writeFile`, `appendFile`
- **Управление:** `rename`, `unlink`, `mkdir`, `rm`, `copyFile`
- **Наблюдение:** `fs.watch`, `fs.watchFile`
- **Потоки:** `createReadStream`, `createWriteStream`

**🔴 Senior**
**Вопрос:** «Является ли `fs.writeFile` атомарной операцией? Что произойдёт с файлом, если процесс упадёт посередине записи?»

**Ответ:** нет — `writeFile` не гарантирует «всё или ничего» для целевого пути; при обрыве (краш, `ENOSPC`, питание и т.д.) файл может остаться **частично записанным**. Безопасное обновление — паттерн **write-to-tmp + rename** на одной FS:
```js
await writeFile('file.tmp', data);
await rename('file.tmp', 'file.json'); // атомарно на уровне ОС
```
Понимает разницу `fs.watch` (нативные события ОС, быстрее) vs `fs.watchFile` (polling, кроссплатформеннее).

---

### 2. ⭐⭐ Когда можно использовать синхронные версии операций с файлами (node:fs)?

**🟡 Junior**
Нельзя использовать в HTTP-запросах — блокируют. Для чтения файлов использует async.

**🟠 Middle**
Чётко разграничивает допустимые контексты:
- ✅ Инициализация приложения до `app.listen()` — конфиги, `.env`
- ✅ CLI-утилиты без параллельных запросов
- ✅ Скрипты миграции/сидинга
- ❌ Обработчики HTTP-запросов
- ❌ Middleware

**🔴 Senior**
Объясняет механизм: sync-операция блокирует весь Event Loop — пока файл читается, Node не может обработать ни один другой входящий запрос. Даже `readFileSync` на 1 мс при 1000 RPS даёт +1 сек суммарной задержки. Знает, что даже при инициализации стоит предпочесть `await readFile()` если порядок не важен — параллельная загрузка конфигов через `Promise.all`.

---

### 3. ⭐⭐ Зачем нужен AbortController?

**🟡 Junior**
Знает что можно отменить `fetch`. Видел в документации.

**🟠 Middle**
Умеет применять сигнал к разным API:
```js
const controller = new AbortController();
const { signal } = controller;

fetch(url, { signal });
fs.readFile(path, { signal }, callback);

controller.abort(); // отменяет все разом
```
Использует `AbortSignal.timeout(5000)` для таймаутов.

**🔴 Senior**
Использует `req.signal` (Node 18+) — при дисконнекте клиента сигнал уходит в **aborted**; это можно протаскивать вниз по цепочке (fetch, fs с `signal` и т.д.):
```js
app.get('/data', async (req, res) => {
  const result = await db.query(sql, { signal: req.signal });
  res.json(result);
});
```
Важно: **отменится ли SQL**, зависит не от Node, а от **драйвера/ORM**: нужна поддержка `AbortSignal` и реальная отмена на стороне БД; иначе запрос в БД может завершиться, хотя клиент уже ушёл. `AbortController` — основа отмены цепочки там, где API сигнал чтут. Связывает с graceful shutdown.

---

### 4. ⭐⭐⭐ Как возможна гонка данных (race condition) в асинхронном программировании?

**🟡 Junior**
Слышал термин, объясняет на уровне «два запроса одновременно могут записать одно и то же».

**🟠 Middle**
Приводит конкретный пример и знает решение через атомарные операции БД:
```js
// Оба запроса прочитают count=5, оба запишут 6 вместо 7
let count = await db.get('likes');
count++;
await db.set('likes', count);

// Решение: атомарная операция
await db.query('UPDATE posts SET likes = likes + 1 WHERE id = ?', [id]);
```

**🔴 Senior**
Знает весь спектр решений и когда что применять:
- **Атомарные операции БД** — для простых счётчиков
- **Redis `INCR`/`DECR`** — для распределённых счётчиков
- **`async-mutex`** — для сложной логики внутри одного процесса
- **Очередь задач** — когда операция не идемпотентна и важен порядок
- **Optimistic locking** (version field) — для редких конфликтов в БД

Понимает что в Node.js race condition возможна несмотря на однопоточность JS: await создаёт «окна» между операциями.

---

### 5. ⭐⭐⭐⭐ Почему у Event Loop есть фазы? Чем отличаются микротаски и макротаски?

**🟡 Junior**
Знает что Event Loop существует, что async-код «не блокирует». Путается в деталях порядка выполнения.

**🟠 Middle**
Описывает полную схему выполнения:
```
1. Call Stack (синхронный код)       ← выполняется первым
       ↓ стек пуст
2. Microtask Queue
   ├── process.nextTick              ← наивысший приоритет
   └── Promise.then / queueMicrotask
       ↓ очередь пуста
3. Event Loop — фазы:
   ├── timers      — setTimeout, setInterval
   ├── pending cb  — I/O коллбэки с предыдущей итерации
   ├── poll        — новые I/O события
   ├── check       — setImmediate
   └── close       — socket.on('close', ...)
```
Правильно предсказывает вывод:
```js
setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));
process.nextTick(() => console.log('nextTick'));
// nextTick → Promise → setTimeout
```

**🔴 Senior**
Объясняет **зачем** фазы: в браузере одна Macrotask Queue — Node.js (libuv) разбил её, чтобы I/O не конкурировало с таймерами, а `setImmediate` гарантированно шёл после I/O. Каждая фаза — своя очередь с приоритетом. Знает что Microtask Queue опустошается **после каждой фазы**, а не только между итерациями. Понимает что Event Loop не стартует пока не выполнится весь синхронный код верхнего уровня.

---

## Группа 2: Потоки и backpressure (вопросы 6–9)

### 6. ⭐⭐⭐ Как обрабатывать CPU-интенсивные задачи в Node.js?

> Вопрос 2 закрывает sync I/O — этот вопрос про CPU-bound блокировку

**🟡 Junior**
Не понимает разницы между I/O-блокировкой и CPU-блокировкой. Думает что async/await решает всё.

**🟠 Middle**
Понимает: `await` не спасёт от CPU-блокировки — тяжёлый синхронный код замораживает Event Loop даже внутри async функции. Решение — Worker Thread:
```js
// Плохо: async не помогает если внутри CPU-работа
app.get('/hash', async (req, res) => {
  const result = heavyCrypto(req.body); // блокирует всех!
  res.json(result);
});

// Хорошо: выносим в Worker Thread
app.get('/hash', async (req, res) => {
  const result = await runInWorker('./workers/crypto.js', req.body);
  res.json(result);
});
```

**🔴 Senior**
Выбирает инструмент под задачу:
| Инструмент | Когда |
|---|---|
| **Worker Threads** | CPU-задача, нужна общая память (`SharedArrayBuffer`) |
| **child_process fork** | Изоляция, отдельная память, можно убить при зависании |
| **Чанкование через `setImmediate`** | Итерация по массиву — не хочется создавать поток |
| **BullMQ** | Результат не нужен сразу, важна надёжность и retry |

Чанкование без отдельного потока:
```js
async function processInChunks(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 1000 === 0) {
      await new Promise(r => setImmediate(r)); // отдаём Event Loop
    }
  }
}
```

---

### 7. ⭐⭐⭐ Как могут утечь все соединения из пула подключений к БД?

**🟡 Junior**
Знает что соединения нужно закрывать. Не всегда понимает когда именно.

**🟠 Middle**
Знает паттерн `try/finally` и объясняет почему без него соединение зависнет:
```js
// Плохо: исключение → conn.release() не выполнится
const conn = await pool.connect();
const result = await conn.query('SELECT ...');
conn.release();

// Хорошо
const conn = await pool.connect();
try {
  return await conn.query('SELECT ...');
} finally {
  conn.release(); // выполнится всегда
}
```

**🔴 Senior**
Перечисляет все сценарии утечки:
- `try/finally` без `release` (самое частое)
- Долгая транзакция, ожидающая внешний API без таймаута
- Дисконнект клиента — запрос продолжается, соединение занято, `AbortController` не используется
- Deadlock транзакций — оба процесса ждут друг друга

Знает как мониторить: `pool.totalCount`, `pool.idleCount`, `pool.waitingCount`. Настраивает `connectionTimeoutMillis` и `idleTimeoutMillis`. Понимает что при утечке `pool.waitingCount` растёт до бесконечности.

---

### 8. ⭐⭐⭐ Что такое back pressure для стримов и какая проблема была бы без него?

**🟡 Junior**
Слышал слово в контексте потоков или `pipe`, но путает с «давлением» в сети или не связывает с памятью.

**🟠 Middle**
Объясняет идею: **потребитель** сигнализирует **источнику**, что не успевает; источник **замедляется** или **паузится**, пока не освободится буфер. Знает что `readable.pipe(writable)` в Node это учитывает. Без back pressure быстрое чтение + медленная запись → **растут внутренние буферы** и **память**.

**🔴 Senior**
Связывает с механикой Node: `writable.write(chunk)` возвращает `false`, когда **highWaterMark** достигнут — нужно ждать событие **`drain`** или полагаться на `pipe()`, который вызывает `readable.pause()` / возобновляет по `drain`. При ручной записи:
```js
if (!writable.write(chunk)) {
  await once(writable, 'drain');
}
```
Понимает что без согласования скоростей при большом файле или быстром сокете можно получить **OOM** или нестабильную латентность. Знает что при кастомных потоках и `async` обработчиках `data` легко **сломать** back pressure, если всегда читать вперёд без учёта `write()`.

---

### 9. ⭐⭐⭐⭐ JSON сериализация/десериализация блокирует поток — что делать?

**🟡 Junior**
Знает что `JSON.parse` синхронный. Не знает что с этим делать.

**🟠 Middle**
Знает решение через Worker Thread и понимает порог (~10–50 MB начинает ощутимо тормозить):
```js
// Перенести тяжёлый JSON в worker
const result = await runInWorker('./workers/parse.js', { json: bigString });
```
Знает про пагинацию и проекцию полей как способ не допустить больших payload.

**🔴 Senior**
Знает streaming-парсинг для обработки без загрузки в память:
```js
import { parser } from 'stream-json';
import { streamArray } from 'stream-json/streamers/StreamArray';

fs.createReadStream('big.json')
  .pipe(parser())
  .pipe(streamArray())
  .on('data', ({ value }) => process(value)); // по одному элементу
```
Связывает цепочку с **back pressure** (вопрос 8): `pipe()` согласует скорости источника и преобразований; при кастомных writable / ручной записи — те же правила `write`/`drain`.
Знает про `simdjson` для максимальной скорости парсинга.

---

## Группа 3: Кластеризация и память (вопросы 10–11)

### 10. ⭐⭐⭐ Чем отличается node:cluster от node:child_process?

**🟡 Junior**
`cluster` — для нескольких процессов чтобы использовать все CPU.

**🟠 Middle**
Знает ключевые различия:

| | `cluster` | `child_process` |
|---|---|---|
| Шаринг порта | Да (один порт на всех) | Нет |
| Use case | HTTP-сервер на N CPU | Внешние команды, изоляция |
| IPC | `process.send()` | `stdin/stdout` или channel |

**🔴 Senior**
Знает узкое место: master-процесс `cluster` принимает **все** соединения и раздаёт воркерам — при высоком RPS сам становится bottleneck. Решения:
- Linux `SO_REUSEPORT` — ядро само балансирует без master-посредника
- PM2 cluster mode использует `SO_REUSEPORT` там где доступно
- В Kubernetes — несколько подов за Service лучше кластера внутри пода

Понимает что `cluster` и `child_process` — оба создают отдельные процессы (отдельная память), в отличие от Worker Threads (один процесс, разные потоки).

---

### 11. ⭐⭐⭐ Типичные причины утечек памяти в Node.js и как с ними бороться?

**🟡 Junior**
Перечисляет 1–2 штампа: забыли `removeListener`, не вызвали `clearInterval`. Не связывает с heap и инструментами.

**🟠 Middle**
Систематизирует основные классы причин:
- **События** — подписка без отписки (`EventEmitter`, DOM-подобные API в тестах)
- **Таймеры** — `setInterval` / `setTimeout` без очистки; замыкания, держащие большие объекты
- **Кэши без eviction** — `Map` / объект без лимита и TTL
- **Замыкания** — колбэк держит в лексическом окружении больше данных, чем нужно
- **Глобалы и модульные синглтоны** — массивы/очереди, растущие без политики очистки
- **Ресурсы** — неотпущенные соединения БД, незакрытые стримы (см. вопрос 7)

Знает направление поиска: рост **heap / RSS** в метриках, снимки heap в DevTools, сравнение двух snapshot.

**🔴 Senior**

**Утечка vs «рост по дизайну».** V8 собирает только **недостижимые** с глобальных корней и активных стеков объекты. Если объект **по-прежнему достижим** — это не сбой GC, а следствие ссылок в коде. **«Настоящая» утечка** в инженерном смысле — когда вы **ожидали**, что данные перестанут быть нужны, но из-за замыкания, глобального массива, кэша без eviction, забытых слушателей или циклических структур, достижимых из корня, память **никогда не освобождается**. **Рост по дизайну** — когда приложение **намеренно** хранит всё (логи в памяти, полный кэш ответов, `Map` по `userId` без TTL): GC работает штатно, но политика хранения ведёт к OOM; лечится не «починкой GC», а **лимитами, TTL, выносом в Redis/диск**.

**Диагностика.** Умеет читать **heap snapshot** в Chrome DevTools: два снимка до/после нагрузки, **Comparison**, сортировка по **Retained Size**, цепочка **Retainers** — кто держит живым кластер объектов. Знает **clinic heapprofiler** как способ получить профиль кучи на прод-подобной нагрузке без ручной возни. Понимает что **рост RSS при стабильном heap** может указывать на **внешнюю** память (буферы, нативные модули), а не только на объекты JS.

**Нативная память и addon’ы.** Крупные **`Buffer`**, долгоживущие ссылки на нативные структуры, баги или особенности **C++ addon** могут раздувать процесс в обход привычной картины «только heap». Важно не смешивать в отчётах **heap Used** и **RSS** и при подозрении смотреть нативный слой и документацию модуля.

**Профилактика в коде.** Явные **лимиты и eviction** для любых in-memory кэшей; **TTL / LRU**; для метаданных о объектах, которые должны умирать вместе с владельцем, — **`WeakMap` / `WeakSet`** (ключи только объекты; при отсутствии других ссылок на ключ запись пропадает). На **shutdown**: снять подписки, `clearInterval`/`clearTimeout`, закрыть серверы и пулы — чтобы не копить «хвосты» при рестартах и тестах.

**In-memory rate limiting.** Счётчики вида `Map<ip | userId, { count, resetAt }>` при **атаке** или при **слишком грубом ключе** (например, только IP за NAT) раздувают память: миллионы уникальных ключей → миллионы записей в `Map`. Это снова **политика хранения**: нужны **жёсткий лимит размера структуры**, **просрочка записей**, либо **вынесение счётчиков в Redis** с TTL, плюс лимит на периметре (LB/WAF).

---

## Группа 4: Безопасность (вопросы 12–13)

### 12. ⭐⭐⭐ Откуда берутся уязвимости в Node-проектах?

**🟡 Junior**
Упоминает «уязвимые пакеты» и `npm audit`. Не разделяет код приложения и зависимости.

**🟠 Middle**
Делит поверхность атаки на уровни:
- **Собственный код** — небезопасная работа с вводом (SQL, пути, HTML), слабая авторизация, утечки в логах
- **Зависимости** — известные CVE, транзитивные пакеты, typosquatting
- **Конфигурация** — секреты в репозитории, открытый debug, избыточный CORS, старый Node
- **Инфраструктура** — права процесса, открытые порты, незащищённый TLS

Использует **`npm audit` / `pnpm audit`**, lockfile, обновления патчей; понимает что audit — не полная гарантия (false sense, delayed advisories).

**🔴 Senior**
Говорит про **supply chain**: pinned версии, приватный registry, SBOM, политика approve для новых зависимостей, **least privilege** в CI, подпись артефактов где уместно. Различает **severity** и **reachability** (есть ли реальный путь вызова уязвимого кода). Связывает с **Dependabot/Renovate** и процессом emergency patch. Не обещает «ноль уязвимостей», а описывает **непрерывный** процесс.

---

### 13. ⭐⭐⭐ Какие уязвимости чаще встречаются в Node API / веб-приложении и как от них защищаться?

**🟡 Junior**
Слышал про **инъекции**, **XSS**, слабые пароли. Может назвать 1–2 пункта из OWASP Top 10 по памяти без связи с конкретными мерами в Node.

**🟠 Middle**
Даёт **классификацию по типам** (не обязательно по номерам OWASP, но в том же духе) и для каждого — **типовую причину** и **базовую защиту**:

- **Инъекции** (SQL, NoSQL, команды) — неверное смешивание **данных пользователя** с **кодом** запроса/шелла. Защита: **параметризованные запросы**, API ORM с биндингом, не строить динамический SQL/путь к коллекции конкатенацией; для shell — не вызывать оболочку с сырой строкой.
- **Небезопасная работа с путями** (path traversal, **zip slip** при распаковке) — доверие к фрагменту пути от клиента. Защита: **базовая директория**, `path.resolve`, проверка что итог **внутри** allowed root; осторожность с symlink.
- **XSS** при SSR или хранении HTML — неэкранированный вывод. Защита: экранирование, безопасные шаблоны, **CSP**; для rich text — санитайзеры, а не «доверие» строке.
- **CSRF** — браузер автоматически шлёт **куки** на ваш домен с чужого сайта. Защита: **`SameSite`**, CSRF-токен для cookie-сессий; для pure **Bearer в заголовке** CSRF обычно не применим.
- **Сломанная авторизация / IDOR** — можно читать чужие ресурсы, сменив id. Защита: проверка прав **на каждом** эндпоинте, не полагаться только на «секретный» URL.
- **Утечки и секреты** — ключи в репо, лишнее в логах. Защита: `.env`/секрет-хранилища, маскирование, минимум данных в ответах об ошибках в проде.

Понимает что одной библиотеки мало: нужны **валидация ввода**, **least privilege** к БД, **security headers** (Helmet) как слой поверх правильной логики.

**🔴 Senior**
Связывает классы уязвимостей с **реальными ошибками в Node**: например **second-order SQL injection** (данные сохранили и потом вставили в запрос как «доверенные»); ORM не спасает при **шаблонных строках** в `query()`. Различает **CORS** (читает ли *другой origin ответ*) и **CSRF** (выполняется ли *запрос с чужого сайта с вашими куками*) — это разные механизмы. Знает когда **SSRF** актуален (сервер сам ходит по URL от пользователя). Упоминает **нормализацию** путей и **различие cookie / Authorization** для модели угроз. Не подменяет глубину одной фразой «используй Helmet» — видит **defense in depth**: валидация, заголовки, права, мониторинг аномалий.

---

# Блок 2: System Design (25 мин)

---

## Группа 1: Очереди и масштабирование (вопросы 1–2)

### 1. ⭐⭐ Для чего нужны внутренние очереди и внешние MQ-системы?

**🟡 Junior**
Знает что очереди нужны для фоновых задач (отправка email).

**🟠 Middle**
Различает внутренние и внешние:
- **BullMQ / p-queue** — в памяти, retry с backoff, ограничение concurrency
- **RabbitMQ / Kafka** — переживают рестарт, несколько воркеров, гарантированная доставка

**🔴 Senior**
Выбирает под требования: задача должна выжить при крэше → только внешняя очередь. Знает паттерны: at-least-once vs exactly-once, dead letter queue, идемпотентные обработчики. Понимает когда Kafka (высокий throughput, retention) vs RabbitMQ (routing, приоритеты).

---

### 2. ⭐⭐⭐⭐ Стратегии масштабирования Node.js-приложений

**🟡 Junior**
Запустить несколько процессов через PM2.

**🟠 Middle**
Сравнивает основные стратегии:

| Стратегия | Плюсы | Минусы |
|---|---|---|
| Cluster / PM2 | Просто, все CPU | Один сервер |
| Horizontal + LB | Надёжно | Нужен stateless |
| Worker Threads | CPU-задачи | Только compute |
| Serverless | Эластично | Cold start |

**🔴 Senior**
Знает что горизонтальное масштабирование требует **stateless**: сессии в Redis, файлы в S3, кэш внешний. Понимает что PM2 cluster + Nginx решает 80% задач, но Kubernetes даёт auto-scaling, health checks и rolling deploys. Выбирает Serverless осознанно: подходит для редких/пиковых нагрузок, не подходит для long-running соединений (WebSocket, SSE).

---

## Группа 2: Границы слоёв и транспорта (вопрос 3)

### 3. ⭐⭐⭐⭐ Как изолировать бизнес-логику от фреймворка (Express/Fastify) и от транспорта (HTTP, gRPC, очередь)?

**🟡 Junior**
Предлагает «вынести функции в отдельные файлы» / `services/` без чёткого правила зависимостей. Контроллеры всё ещё содержат правила домена.

**🟠 Middle**
Описывает **слои**: handlers тонкие (валидация DTO, маппинг статусов), **use case** / **application service** содержит сценарий, доменные типы без `req`/`res`. Понимает что один и тот же сценарий можно вызвать из HTTP-роута, CLI и consumer очереди, передавая **plain objects**.

Знает направление **dependency injection** (конструктор, фабрика) чтобы подставлять репозитории и не импортировать `pg` в домене.

**🔴 Senior**
Формулирует через **порты и адаптеры (hexagonal)**:
- **Domain** — сущности, инварианты, доменные ошибки; без ORM и HTTP.
- **Application** — use cases, **интерфейсы** портов (`OrderRepository`, `EmailGateway`); зависимости направлены **внутрь**.
- **Infrastructure** — реализации портов (Postgres, SendGrid), HTTP/gRPC **адаптеры** только переводят запрос → вызов use case → ответ.

Объясняет **маппинг ошибок**: домен `InsufficientStock` → HTTP 409, в gRPC — другой status. Упоминает **тестируемость**: use case тестируется с in-memory fake без поднятия сервера. Оговаривает прагматизм: не весь CRUD обязан быть «идеально чистым», но **критичные правила и деньги** — за границей фреймворка.

---

# Блок 3: AI in Practice (12 мин) — **всегда в конце интервью**

> **Внутри блока 3** (если в резюме нет AI): на **английском** интервью **первым шагом блока** задать **вступление про CV** (скрипт ниже) — причина отсутствия упоминаний и **есть ли опыт** использования AI в работе. **Затем** — **Группа A** (day-to-day development). Весь блок можно **сократить до 5–8 минут**, если не задавать вопросы **1–2** про устройство LLM и интеграцию в Node. Вопросы **1–2** — для кандидатов с **AI в продукте** / сильным интересом к моделям.

---

## Вступление (шаг 1 блока 3): резюме без упоминаний AI (скрипт на английском, 1–2 мин)

**Зачем:** снять неловкость «в резюме пусто», отделить **намеренное молчание** (политика, NDA, не считает релевантным) от **отсутствия практики**. *Не заменяет* техническую часть — идёт **после** core и design.

**Спросить по-английски (нейтральный тон):**
> *I noticed your résumé doesn’t mention AI tools or LLMs, so I’d like to ask: **Do you use AI in your day-to-day work** as a developer — for coding, review, debugging, or similar? **Have you also looked into how LLMs work** beyond everyday tooling — courses, documentation, side projects, or integrating with an API?*

**Уточняющие (по ответу):**
- Если **политика компании / запрет** — зафиксировать, не давить; всё равно можно коротко спросить про **опыт вне работы** или **как смотрит на инструменты** (без деталей NDA).
- Если **«не использую»** — *Is that a choice, or you haven’t felt the need?* — без оценки «хорошо/плохо».
- Если **использует, но не вписал в CV** — нормальная ситуация; переход к **Группе A** для детализации workflow.
- Если **копал глубже** (теория, свой API-опыт) — уточнить что именно; при сильном ответе можно перейти к **вопросам 1–2** блока.

**На русском интервью** — тот же смысл своими словами: *«В CV нет упоминаний про AI — хочу уточнить: пользуетесь ли AI в повседневной разработке — код, ревью, отладка и т.п.? Разбирались ли глубже, как устроены LLM: курсы, документация, pet-проекты, интеграция через API?»*

---

## Группа A: AI в повседневной разработке (шаг 2 блока 3, после вступления про CV; 5–8 мин)

### Какие AI-инструменты используете в работе и как — хотя бы для написания кода?

**Слабый ответ / нет практики**
Не пользуется или «пробовал ради интереса», не может описать **когда** и **зачем** включает инструмент в типичный день.

**Базовый уровень**
Называет конкретные продукты (**GitHub Copilot, Cursor, ChatGPT, Claude, JetBrains AI** и т.д.). Использует для **автодополнения**, **объяснения стека ошибок**, **черновика функций или тестов**, **поиска по незнакомому API**. Понимает, что результат нужно **прочитать и прогнать** (тесты, линтер), а не вставлять вслепую.

**Сильный уровень**
Описывает **осознанный workflow**: для чего берёт **inline-completion**, для чего — **чат** или **agent** (рефакторинг по всему репо, миграции, бойлерплейт, документация). Умеет **давать контекст** (релевантные файлы, ограничения стека, стиль проекта). Ревьюит сгенерированное как **чужой PR**: краевые случаи, безопасность, производительность. Называет **ограничения**: галлюцинации, устаревшие ответы, **политика компании** и **куда нельзя слать код** (публичные чаты vs enterprise-версии), **секреты и PII** в промптах. Может связать с **code review** и парным программированием — AI как ускоритель, не замена ответственности.

**Про настройку среды (Cursor и аналоги):** умеет объяснить роль **Skills** — переиспользуемые инструкции для агента (стандарты проекта, чеклисты перед коммитом, запреты вроде «не трогай без запроса»), различие **user-** vs **project-**уровня и зачем класть skill в репозиторий для команды. Про **Rules** (например **`.cursor/rules`**, `RULE.md`, иногда **`AGENTS.md`**) — постоянные **проектные** указания, которые подмешиваются в контекст агента/чата: стиль кода, запреты на лишние рефакторы, соглашения по коммитам, «как мы деплоим»; умеет отличить **глобальные user rules** от **правил репозитория**, настраивать **привязку к путям** (glob), чтобы в одном монорепо для `frontend/` и `services/api/` были разные ограничения; понимает что rules — **короткие и проверяемые** предпочтительнее простыни, иначе их игнорируют или конфликтуют друг с другом. Про **MCP-серверы** — зачем подключать (актуальная документация библиотек, браузер, внутренние API), как **включает/отключает** серверы в настройках IDE, что они дают модели **инструменты** с сетью и файлами; понимает **риски** (утечка контекста в сторонний сервис, лишние разрешения, секреты в env для MCP) и что список серверов — часть **гигиены** onboarding’а в команде.

---

## Группа 1: LLM и интеграция (вопросы 1–2)

### 1. Как работает LLM под капотом?

**Level 1–2**
Знает что LLM — это нейросеть, обученная на текстах. Понимает что модель генерирует текст токен за токеном.

**Level 3**
Объясняет ключевые концепции:
- **Токенизация** — текст разбивается на токены (≈ слова/части слов). 1 токен ≈ 4 символа. От количества токенов зависит цена и лимит контекста.
- **Context window** — максимальное количество токенов, которые модель «видит» за один раз (вход + выход). Всё что не влезло — модель не знает.
- **Temperature** — степень «случайности» ответа. 0 — детерминировано, 1+ — творчески/непредсказуемо.
- **Autoregressive generation** — каждый следующий токен предсказывается на основе всех предыдущих. Потоковый ответ (`stream: true`) — токены приходят по мере генерации.

**Level 4**
Понимает архитектурные ограничения и их влияние на продукт:
- **Hallucinations** — модель не «знает» факты, она предсказывает вероятный текст → нельзя доверять без верификации
- **Stateless** — каждый запрос независим, история передаётся вручную в `messages[]`
- **Latency** — первый токен может приходить через 1–5 сек → streaming обязателен для UX
- **Cost** — платят за input + output токены → важно контролировать размер промптов

---

### 2. Как безопасно интегрировать AI в Node.js? Кэширование промптов, fallback, observability.

**Level 2**
Знает базовые меры:
- API-ключ в `.env`, не в коде
- Таймаут на запрос к AI API
- `try/catch` вокруг вызова

**Level 3**
Знает паттерны надёжной интеграции:

**Prompt caching** — дорогой системный промпт передаётся один раз и кэшируется на стороне провайдера:
```js
// Anthropic: cache_control на длинном неизменяемом блоке
{
  role: 'user',
  content: [{
    type: 'text',
    text: longSystemContext,
    cache_control: { type: 'ephemeral' } // кэш на 5 мин
  }]
}
// Повторные запросы с тем же блоком — дешевле и быстрее
```

**Fallback** — если основная модель недоступна, переключаться на резервную:
```js
async function callAI(prompt) {
  try {
    return await anthropic.complete(prompt);
  } catch (err) {
    if (err.status === 529 || err.status === 503) {
      return await openai.complete(prompt); // fallback
    }
    throw err;
  }
}
```

**Rate limiting** — не отправлять в AI больше чем позволяет квота:
```js
const limiter = new Bottleneck({ maxConcurrent: 5, minTime: 200 });
const safeCall = limiter.wrap(callAI);
```

**Level 4**
Строит полноценную observability:
- **Логирует** каждый запрос: модель, токены input/output, latency, стоимость
- **Трекает** hallucinations и ошибки через Sentry или кастомный pipeline
- **Кэширует ответы** на уровне приложения (Redis) для идентичных промптов — экономия и скорость
- **Circuit breaker** — при N ошибках подряд перестаёт звать AI и возвращает graceful degradation
- **Prompt injection защита** — валидация и санитизация пользовательского ввода перед вставкой в промпт:
```js
// Плохо: пользователь может вставить "Ignore all previous instructions"
const prompt = `Ответь на вопрос: ${userInput}`;

// Хорошо: изолировать пользовательский ввод
const prompt = `
Ты помощник. Отвечай только на вопросы по теме X.
<user_input>${sanitize(userInput)}</user_input>
`;
```

---

# Appendix: Block 3 — English (mirror of «Блок 3» above)

> Same structure as **Блок 3: AI in Practice** — use for interviews conducted in English.
>
> **Inside Block 3:** if the CV has **no AI mentions**, begin the block with **Before Group A** (below), then **Group A**.

---

# Block 3: AI in Practice (12 min) — **always at the end**

> **Inside Block 3:** if the résumé has **no AI mentions**, **first** use the **CV opener** (below): why it’s absent and whether they **actually use** AI tooling. **Then** **Group A** (day-to-day development). Keep the block to **5–8 minutes** if you skip questions **1–2**. Use **1–2** when the candidate has **product LLM experience** or strong model-level interest.

---

## Before Group A — step 1 of Block 3: résumé gap (1–2 min)

**Purpose:** acknowledge the empty CV line on AI; separate **intentional omission** (policy, relevance) from **no practice**. Does **not** replace the technical sections — comes **after** core and design.

**Ask (neutral):**
> *I noticed your résumé doesn’t mention AI tools or LLMs, so I’d like to ask: **Do you use AI in your day-to-day work** as a developer — for coding, review, debugging, or similar? **Have you also looked into how LLMs work** beyond everyday tooling — courses, documentation, side projects, or integrating with an API?*

**Follow-ups:**
- **Company policy / not allowed** — acknowledge; optionally ask how they **think about** these tools in the industry (no confidential detail).
- **“I don’t use them”** — *Is that a deliberate choice, or you haven’t felt the need?* — stay non-judgmental.
- **They use tools but didn’t list them** — common; move to **Group A** for depth.
- **They went deeper** (theory, own API integration) — ask what exactly; a strong answer may justify moving to **questions 1–2** in this block.

---

## Group A: AI in day-to-day development — step 2 of Block 3 (after CV opener; 5–8 min)

### Which AI tools do you use at work and how — at least for writing code?

**Weak / no real practice**
Does not use them, or only “tried once,” and cannot explain **when** and **why** they reach for a tool on a typical day.

**Solid baseline**
Names concrete products (**GitHub Copilot, Cursor, ChatGPT, Claude, JetBrains AI**, etc.). Uses them for **inline completion**, **explaining stack traces**, **drafting functions or tests**, **exploring unfamiliar APIs**. Knows output must be **read and exercised** (tests, linter), not pasted blindly.

**Strong**
Describes a deliberate **workflow**: when **inline completion** is enough vs **chat** or **agent** mode (repo-wide refactors, migrations, boilerplate, docs). Knows how to **provide context** (relevant files, stack constraints, project style). Reviews generated code like a **peer’s PR**: edge cases, security, performance. Names **limits**: hallucinations, stale training, **company policy** and **where code must not go** (public chat vs enterprise tier), **secrets and PII** in prompts. Can relate this to **code review** and pairing — AI as an accelerator, not a substitute for ownership.

**Environment setup (Cursor and similar):** can explain **Skills** — reusable agent instructions (project standards, pre-commit checklists, guardrails like “don’t change without asking”), **user** vs **project** scope, and why committing skills helps the team. **Rules** (e.g. **`.cursor/rules`**, `RULE.md`, sometimes **`AGENTS.md`**) — standing **project** instructions merged into agent/chat context: code style, no drive-by refactors, commit conventions, how you deploy; distinguishes **global user rules** from **repo rules**, uses **path globs** so `frontend/` and `services/api/` can differ in a monorepo; prefers **short, enforceable** rules over huge walls of text that get ignored or conflict. **MCP servers** — why wire them up (fresh library docs, browser, internal APIs), how to **enable/disable** in IDE settings, that they give the model **tools** with network and filesystem access; understands **risks** (context leakage to third-party services, over-broad permissions, secrets in env for MCP) and that the server list is part of team **onboarding hygiene**.

---

## Group 1: LLM fundamentals and integration (questions 1–2)

### 1. How does an LLM work under the hood?

**Level 1–2**
Knows an LLM is a neural net trained on text. Understands it generates text **one token at a time**.

**Level 3**
Explains core ideas:
- **Tokenization** — text is split into tokens (≈ words/subwords). ~1 token ≈ 4 characters. Token count drives **price** and **context limits**.
- **Context window** — max tokens the model “sees” in one shot (input + output). Anything outside it is unknown to the model.
- **Temperature** — how “random” the completion is. 0 ≈ deterministic; higher ≈ more creative / less predictable.
- **Autoregressive generation** — each next token depends on all previous ones. **`stream: true`** streams tokens as they are produced.

**Level 4**
Understands product-facing constraints:
- **Hallucinations** — the model does not “know” facts; it predicts plausible text → never trust without verification.
- **Stateless** — each request is independent; conversation history is passed manually in `messages[]`.
- **Latency** — first token may take seconds → streaming matters for UX.
- **Cost** — billed on input + output tokens → control prompt size.

---

### 2. How do you integrate AI into Node.js safely? Prompt caching, fallback, observability.

**Level 2**
Basic hygiene:
- API key in `.env`, not in source
- Timeout on calls to the AI API
- `try/catch` around the call

**Level 3**
Reliable integration patterns:

**Prompt caching** — expensive static system context sent once and cached by the provider:
```js
// Anthropic: cache_control on a large, stable block
{
  role: 'user',
  content: [{
    type: 'text',
    text: longSystemContext,
    cache_control: { type: 'ephemeral' } // e.g. short-lived server-side cache
  }]
}
// Repeat calls with the same block: cheaper and faster
```

**Fallback** — if the primary model is down, switch to a backup:
```js
async function callAI(prompt) {
  try {
    return await anthropic.complete(prompt);
  } catch (err) {
    if (err.status === 529 || err.status === 503) {
      return await openai.complete(prompt); // fallback
    }
    throw err;
  }
}
```

**Rate limiting** — do not exceed provider / budget quotas:
```js
const limiter = new Bottleneck({ maxConcurrent: 5, minTime: 200 });
const safeCall = limiter.wrap(callAI);
```

**Level 4**
Full **observability** mindset:
- **Log** each call: model, input/output tokens, latency, estimated cost
- **Track** bad outputs and errors (Sentry or a custom pipeline)
- **Cache responses** at the app layer (e.g. Redis) for identical prompts — cost and latency
- **Circuit breaker** — after N failures, stop calling the model and degrade gracefully
- **Prompt-injection hardening** — validate and sanitize user text before it lands in the prompt:
```js
// Bad: user can inject "Ignore all previous instructions"
const prompt = `Answer the question: ${userInput}`;

// Better: isolate untrusted input
const prompt = `
You are an assistant. Answer only about topic X.
<user_input>${sanitize(userInput)}</user_input>
`;
```

---