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

## Группа 2: Потоки и backpressure (вопросы 6–8)

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

### 8. ⭐⭐⭐⭐ JSON сериализация/десериализация блокирует поток — что делать?

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
Объясняет **backpressure**: если потребитель медленнее производителя — `pipe()` автоматически приостанавливает readable. При ручном `writable.write()` — проверяет возвращаемое значение и ждёт `drain`:
```js
if (!writable.write(chunk)) {
  await once(writable, 'drain');
}
```
Знает про `simdjson` для максимальной скорости парсинга.

---

## Группа 3: Кластеризация и память (вопрос 9)

### 9. ⭐⭐⭐ Чем отличается node:cluster от node:child_process?

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

# Блок 3: AI in Practice (12 мин)

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