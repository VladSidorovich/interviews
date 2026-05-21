# React: сложные вопросы (уровни 1–4)

Один файл: уровни **1–3** — по **3 вопроса** на уровень (всего 9). **Уровень 4** — отдельная **секция на тему**, по **одному короткому вопросу** (forwarding refs, Concurrent Mode, Suspense, PWA, SSR/Next.js, тесты и т.д.). После каждого вопроса — блок **«Суть (устно)»** для интервьюера. Формат — **устные** ответы, без лайв-кодинга. Далее — **эталон по грейдам**: Джун / Мидл / Сеньор — формулировки **как ответ кандидата**, без «понимает / знает / умеет».

---

## Уровень 1

### Вопрос 1. Controlled vs uncontrolled и подъём состояния

Когда в React выбирают **controlled** компонент, а когда **uncontrolled**? Как **lifting state up** связан с этим выбором?

**Суть (устно):**

- **Controlled:** значение поля живёт в **state** React (`value` + `onChange`); DOM только отображает. Один источник правды — проще валидация и связь нескольких полей.
- **Uncontrolled:** значение хранит **DOM**; React читает через **ref** (`defaultValue`, `ref.current.value`). Это **не** «компонент без props» — `placeholder`, `className`, `onBlur` и т.д. могут быть; речь только о том, что **нет** связки `value` из state на каждый символ.
- **Lifting state up:** если двум sibling-компонентам нужны **одни данные** (поиск + список), state кладут в **общего родителя**, детям передают **props вниз** и **колбэки вверх**. Иначе у каждого свой `useState` — данные разъедутся.

**Плюсы и минусы**

| | **Controlled** | **Uncontrolled** |
|---|----------------|------------------|
| **Плюсы** | В любой момент актуальное value в state; валидация на лету; зависимые поля; disabled submit; синхронизация нескольких полей | Меньше ререндеров; проще «простая форма → submit»; file input; интеграция с чужим DOM; фокус через ref без state |
| **Минусы** | Ререндер на каждый символ; больше boilerplate (`useState` на поле) | React «не видит» value до submit/blur; сложнее валидация на лету и синхронизация полей без lift up / библиотеки |

**Когда uncontrolled уместен:** простая форма без live-валидации; `<input type="file" />`; редко нужное поле (комментарий только при submit); legacy-виджет; React Hook Form на больших формах.

**Когда controlled уместен:** маски, «повтор пароля», зависимые select'ы, блокировка кнопки, autosave, undo — нужен state на каждый ввод.

**Lifting state up — мини-пример**

Два ребёнка должны показывать одно и то же слово поиска. Если у каждого свой `useState` — input обновится, список нет. Решение: `const [query, setQuery] = useState('')` в **родителе**; `<SearchInput value={query} onChange={setQuery} />` и `<Results query={query} />`. Поднимают state только **до ближайшего общего предка**, не обязательно в корень `App`.

**Формы: руками vs Formik vs React Hook Form**

| | **Controlled вручную** | **Formik** | **React Hook Form (RHF)** |
|---|------------------------|------------|---------------------------|
| **Как хранит value** | Твой `useState` на каждое поле | Один объект `values` в **state Formik** | В основном **DOM + ref** через `register`; state React не дёргается на каждый символ |
| **Плюсы** | Полный контроль, прозрачно | Удобно для маленьких форм; `values`/`errors`/`touched` из коробки; привычная controlled-модель | Меньше ререндеров на больших формах (20–50 полей); меньше boilerplate |
| **Минусы** | Много кода; ререндеры на больших формах | Ререндер формы при каждом вводе — может лагать | Другая модель мышления; MUI/DatePicker — через `Controller` (controlled-обёртка) |
| **Когда выбирать** | 1–3 поля, учебный пример | Небольшие формы, команда привыкла к «всё в state» | Большие формы, performance; готов объяснить ref-модель |

RHF — **гибрид:** простые `<input>` через ref (uncontrolled-style); сторонние controlled-компоненты — через **`Controller`** (`value` + `onChange`).

**🟡 Джун**

- **Controlled** — у `<input>` есть `value={state}` и `onChange`, который вызывает `setState`; React на каждый символ перерисовывает компонент.
- **Uncontrolled** — начальное значение через `defaultValue`, а при submit или по кнопке читают `ref.current.value`; React не хранит каждый символ в state.
- Если **фильтр** и **список** должны показывать одно и то же слово — state кладут в **родителя** и обоим детям передают `value` + `onChange`; это и есть lifting state up.
- Про uncontrolled для `file` input или «просто отправить форму без валидации на лету» — может не упомянуть без подсказки.

**🟠 Мидл**

- **Controlled** даёт один источник правды: в любой момент `email` в state — это то, что на экране; удобно валидировать, блокировать submit, синхронизировать два поля («повтор пароля»).
- **Uncontrolled** экономит ререндеры: браузер сам держит текст в DOM, React подключается только когда нужно прочитать или сфокусировать через ref.
- **Lifting:** родитель хранит `searchQuery`, `<SearchInput value={…} onChange={…} />` и `<Results query={…} />` — оба controlled, данные не расходятся.
- На одном поле нельзя смешать «иногда `value` из state, иногда нет» — React ругается на переход controlled ↔ uncontrolled.
- Composition здесь — не наследование классов, а **props вниз, события вверх**: дочерний input не «владеет» данными, он их отображает.

**🔴 Сеньор**

- **Controlled** беру, когда нужны маска телефона, зависимые поля («страна → город»), disabled submit при ошибках, единый объект формы перед API, undo/redo — всё это требует актуального state на каждый ввод.
- **Uncontrolled + ref** — когда DOM должен жить своей жизнью: file input, фокус без лишних ререндеров, встраивание jQuery-виджета или rich-editor, который сам пишет в DOM.
- **Anti-pattern:** и родитель, и ребёнок держат копию одного поля без контракта — через секунду они разъедутся; либо один источник (родитель), либо явный uncontrolled с ref только у одного.
- **React Hook Form / Formik** — не «чистый» controlled вручную: Formik держит `values` в state → удобно, но ререндеры; RHF через `register` + ref → меньше ререндеров, на submit собирает `data`. Выбор объясняю trade-off из таблицы выше, не только названием библиотеки.

---

### Вопрос 2. Composition vs Inheritance и паттерн `children`

Почему в React **не наследуют** компоненты друг от друга, а используют **composition**? Что даёт **`children`** и **render props** на уровне идеи?

**Суть (устно):**

- React-модель: UI — **функция от данных** + **дерево компонентов**. Переиспользование — через **вложение** и **props**, а не extends базового класса.
- **Composition:** экран **собирают** из кусочков — Card, Header, Body — поведение **комбинируют**, а не вытягивают из суперкласса.
- **Inheritance** в UI ломается: жёсткая иерархия, один базовый класс раздувается, сложно изменить одну ветку, не сломав другую.
- **`children`** — слот «сюда положи любой контент»; **render props** — «даю логику/данные, ты рисуешь как хочешь».
- **Lifting state up** (вопрос 1) — частный случай composition: общий родитель, не наследование.

**Composition vs Inheritance**

| | **Composition** | **Inheritance (extends)** |
|---|-----------------|---------------------------|
| **Идея** | Маленькие компоненты + props + вложение | Общая логика в базовом классе |
| **Плюсы** | Гибко; явный контракт; легко менять части | Привычно в классическом OOP |
| **Минусы** | Prop drilling (лечат context) | Хрупкая иерархия; zoo из subclass'ов |
| **В React** | Основной путь | Legacy, для нового кода — нет |

**Три паттерна composition (устно)**

1. **`children`:** Dialog рисует рамку, **содержимое** решает родитель.
2. **Props-слоты:** у Layout отдельно sidebar и main — несколько «дырок».
3. **Variant через props:** одна кнопка с `variant` и `size`, а не десять классов-наследников.

**🟡 Джун**

- Компоненты **вкладывают** друг в друга, как матрёшку: Page содержит Header и Content — это composition.
- **`children`** — всё, что написано **между** тегами родителя; Dialog может обернуть любую форму или текст.
- Фраза «composition over inheritance» — React предпочитает **сборку** из частей, а не цепочку наследования классов.

**🟠 Мидл**

- **`class DangerButton extends Button`** плохо масштабируется: комбинаций variant/size/icon слишком много — проще props у одного Button.
- **`children`** — универсальный слот; **header/footer props** — когда слотов несколько и нужны имена.
- Умный контейнер (fetch) + глупый UI (только props) — composition; prop drilling лечат lift up или context, **не** base class.

**🔴 Сеньор**

- Модель React — **дерево**, не IS-A иерархия: Modal с Form внутри, а не ModalWithForm extends Modal.
- **`children` как стабильный слот** — если sidebar тяжёлый, а header часто меняет state, sidebar передают так, чтобы лишний ререндер не тянул его каждый раз (colocation, `useMemo`, layout выше по дереву) — **не** «React сам memoized children».
- Base class с auth + fetch — anti-pattern; **`useAuth()` + guard component** — та же переиспользуемость через composition/hooks.
- HOC и render props (ур. 2) — эволюция composition; design system строят на variants + slots, не на subclass zoo.

---

### Вопрос 3. Синтетические события в React

**Как озвучить (без кода):**

> «Чем **синтетическое событие** в React отличается от **нативного** браузерного? Зачем React вообще использует обёртку?»

**Суть (устно):**

- **SyntheticEvent** — обёртка React над **нативным** событием (`nativeEvent` внутри): тот же клик в браузере, но API **единый** для всех браузеров.
- **Зачем:** нормализация имён и поведения; плюс **делегирование** на root — React сам доводит событие до компонента.
- **На практике:** объект из **пула** — после handler'а поля **обнуляются**; в setTimeout нельзя читать `target`/`value`, только то, что **скопировали сразу**.

**🟡 Джун**

- Нативное — от браузера; синтетическое — **обёртка** React поверх него.
- «Чтобы одинаково работало везде» — без деталей про пул.

**🟠 Мидл**

- Внутри synthetic есть **nativeEvent**.
- Listener на **root**, не на каждую кнопку — React маршрутизирует по дереву.
- event в setTimeout пустой — value/target **копировать сразу**.

**🔴 Сеньор**

- **React 17 и root контейнера:** до React 17 большинство событий вешались на **document** — один общий слушатель на всю страницу. С React 17 делегирование идёт на **DOM-узел, куда смонтировано приложение** (root контейнер). Если на странице **два** React-приложения — у каждого **свой** root и **свои** listener'ы; они не мешают друг другу. Старый код с **native** listener на `document` может вести себя иначе, чем ожидаешь от React.
- **Synthetic vs native и «click outside»:** React-handler — **синтетическая** цепочка по **дереву компонентов**. **Native** listener на `document` — **отдельная** линия, браузерная. `stopPropagation` в React **не останавливает** native listener на document и наоборот. «Клик вне modal» — ref на modal + **native** `mousedown`/`pointerdown` на document (или capture), проверка «клик **вне** ref»; или готовая библиотека. Смешивать React onClick и `document.addEventListener` без понимания порядка — частый баг.
- **Нормализация ≠ магия браузера:** SyntheticEvent выравнивает **API** между браузерами, но **не отменяет** правила движка. **Passive** listener на touch/wheel — `preventDefault` **игнорируется**, scroll всё равно пойдёт. Focus, keyboard, form submit — часть поведения **нативная**; synthetic только доставляет событие в компонент. На форме — **onSubmit**, не только onClick на кнопке.

---

## Уровень 2

### Вопрос 1. `useLayoutEffect` vs `useEffect` и мигание

Чем **`useLayoutEffect`** отличается от **`useEffect`** и **когда без layout будет мигание**?

**Суть (устно):**

- React обновил **DOM** (текст, блоки на странице) — но пользователь **ещё не обязан** это **увидеть**: браузер рисует кадр **чуть позже**.
- **`useLayoutEffect`** — код **между** «DOM уже новый» и «пользователь увидел кадр». Браузер **ждёт**, пока effect отработает, **потом** рисует. Успеваешь **измерить** блок и **подправить** позицию **до показа**.
- **`useEffect`** — код **после** того, как кадр **уже нарисован**. Пользователь **уже видел** экран — потом твой код двигает/меряет.
- **Мигание:** в **`useEffect`** измерил tooltip и сдвинул — на **один кадр** он был **не там**, потом **прыгнул**. В **`useLayoutEffect`** — поправил **до** первого кадра, прыжка **нет**.
- **Правило:** fetch, подписки, аналитика — **`useEffect`**. Измерить DOM, scroll, высоту **без прыжка** — **`useLayoutEffect`**. По умолчанию — **`useEffect`**.

**Пример для кандидата (устно):**

Tooltip под кнопкой: в **`useEffect`** — вставили в DOM → пользователь **мельком** видит кривую позицию → effect меряет и двигает → **прыжок**. В **`useLayoutEffect`** — вставили → **сразу** выставили координаты → **первый** кадр уже правильный.

**🟡 Джун**

- **`useEffect`** — «после отрисовки»; **`useLayoutEffect`** — «после обновления DOM, **до** того как пользователь увидел изменение».
- Мигание — когда элемент **на миг** в неправильном месте/размере, потом **перескакивает**.
- Fetch с API — в **`useEffect`**, не в layout.

**🟠 Мидл**

- **DOM обновлён ≠ пользователь увидел:** между этим и **`useEffect`** — один **нарисованный** кадр; layout effect **влезает до** этого кадра.
- **Tooltip, popover, auto-scroll к строке** — типичный layout: `getBoundingClientRect`, выставить `top`/`scrollTop` **до** показа кадра.
- **Тяжёлый** код в layout **блокирует** показ страницы — не класть туда fetch и фильтрацию больших списков; layout только для **быстрой** sync-правки DOM.

**🔴 Сеньор**

- **По умолчанию `useEffect`** — layout только когда **видимый** прыжок на одном кадре; иначе лишняя блокировка показа.
- **SSR:** layout на сервере не measure'ит DOM — measure **после hydration** на клиенте.
- **Strict Mode (dev):** effect может **дважды** mount/cleanup — не путать с prod; cleanup в layout тоже обязателен.
- **Альтернатива layout:** CSS (`transform`, `position`) без JS-measure — меньше layout effects, если дизайн позволяет.

---

### Вопрос 2. Архитектура state: local, Context, Redux и server cache

**Как озвучить (без кода):**

«На проекте **Redux**, **Context**, **useState** и **React Query** одновременно. Как **решить**, куда класть: theme, modal open, корзину, список users с API, форму checkout? Что будет, если **продублировать** server data в Redux без invalidation?»

**Суть (устно):**

- **Local state (`useState`)** — UI одного компонента или ближайшей ветки: input, toggle, wizard step; **colocation** — держать state **как можно ниже**, пока не нужен sibling/родитель.
- **Lifting state up** — siblings делят данные через **родителя**; prop drilling на 2–3 уровня — норм, не повод сразу в Redux.
- **Context** — **broadcast** стабильных или редко меняющихся вещей: theme, locale, auth **snapshot**; **плохо** для часто меняющегося cart в одном объекте с theme — ререндер **всех** consumers.
- **Server state (React Query, SWR, RTK Query)** — данные с API: cache, stale/fresh, refetch, dedupe; **не копировать** в Redux «на всякий случай».
- **Redux / Zustand** — **shared client state**: корзина offline, сложные cross-feature flows, единый поток событий, time-travel debug; **не** для каждой формы и не как второй cache users list.

**🟡 Джун**

- **Theme** и **язык** — в Context или глобально; **текст в input** — локальный state компонента.
- **Список users с сервера** — fetch + state или библиотека cache; «положить всё в Redux» — не единственный путь.
- Redux — «один store на приложение»; **selector** — «достать кусок store» без лишних ререндеров (детали reselect — по желанию).

**🟠 Мидл**

- **Разделение:** **server state** (API, cache, invalidation) vs **client UI state** (modal, sidebar, wizard step) vs **URL state** (фильтры в query — shareable link).
- **Context anti-pattern:** `{ user, cart, theme }` один object — смена theme ререндерит cart; **split contexts** или Zustand с selector `useStore(s => s.cart)`.
- **Redux оправдан:** offline cart sync, сложная orchestration (middleware), несколько экранов пишут в **один** domain state, DevTools для support.
- **Redux избыточен:** форма на одной странице, данные только React Query, «положили users в slice» хотя Query уже кэширует.

**🔴 Сеньор**

- **Три кэша одних users** (Redux + Query + local) — **рассинхрон**; один **source of truth per concern**: Query owns server entities, Redux owns client domain if needed.
- **Duplication anti-pattern:** `dispatch(setUsers(data))` после каждого fetch — stale без invalidation strategy; либо Query only, либо явный sync contract + tags.
- **RTK Query** — server cache **внутри** Redux ecosystem; всё равно не смешивать с UI-modal slice без границ.
- **Нормализация** в client store: `{ posts: { byId, allIds } }` — O(1) patch одного post; flat vs nested — trade-off readability vs update cost.
- **Аргументирую выбор:** «checkout form — local + mutation on submit; cart badge — Zustand selector; catalog — React Query stale-while-revalidate» — не «у нас всегда Redux».

---

### Вопрос 3. HOC, render props, compound components и hooks

Сравни **HOC**, **render props**, **compound components** и **custom hooks**. Когда что выбирать?

**Суть (устно):**

- **HOC** — функция `(Component) => EnhancedComponent`; шарит логику через **обёртку**; минусы: **wrapper hell**, неявные props, ref/props collision (`hoist-non-react-statics`).
- **Render props** — компонент принимает **`render`/`children` как функцию** `(state) => ReactNode`; явный контракт; минусы: вложенность JSX.
- **Compound components** — семья компонентов (`Tabs`, `Tabs.List`, `Tabs.Panel`) с **неявным** shared context внутри; гибкий UI, знакомый API (как `<select><option>`).
- **Custom hooks** — вынос **stateful logic** без обёртки UI; предпочтительный современный путь для переиспользования логики.

**🟡 Джун**

- **HOC** — обёртка вроде `withAuth(Dashboard)`: добавляет props или логику снаружи.
- **Custom hook** `useAuth()` — вынести `useState` + `useEffect` в переиспользуемую функцию.
- Render props и compound — слабо или только по названию.

**🟠 Мидл**

- **HOC** пример: `withLogging(Wrapped)` — логирует mount; минус — в DevTools дерево из `Connect(WithRouter(…))`.
- **Render props:** `<DataLoader url="/api">{data => <List items={data} />}</DataLoader>` — явно видно, что получает UI.
- **Compound:** `<Tabs><Tabs.List /><Tabs.Panel /></Tabs>` — внутри Context связывает active tab; API как у нативных элементов.
- **Container/Presentational:** контейнер fetch-ит, презентационный `<UserCard user={…} />` — только JSX из props.

**🔴 Сеньор**

- **Hooks заменили** большинство HOC/render props для **логики**: `const { user } = useAuth()` читается проще, чем `withAuth`. HOC остаются для legacy (`connect`) или cross-cutting обёрток (error boundary как wrapper).
- **Compound + Context:** Tabs могут быть controlled (`value` + `onChange` снаружи) или uncontrolled (internal state). Fat context с часто меняющимся value ререндерит **всех** — split на StateContext + DispatchContext или selectors (Zustand).
- **`useCallback`/`useMemo`** — не «ускорители по умолчанию»: стабильная ссылка нужна, когда **`memo`-ребёнок** или effect deps иначе срабатывают каждый render; зря оборачиваешь всё — лишний overhead.
- **`useReducer`** — локальная state machine (форма с 5 связанными полями, wizard steps); тот же паттерн, что reducer в Redux, без глобального store.
- **Выбор:** hook — переиспользование логики; compound — публичный API UI-библиотеки; render props — когда потребителю нужен **полный контроль** над тем, **куда** воткнуть результат (headless UI).

---

## Уровень 3

### Вопрос 1. Error boundaries, Suspense и отказоустойчивый UI

Что ловят **Error Boundaries** и чего они **не** ловят? Чем **Suspense** отличается от boundary?

**Суть (устно):**

- **Error Boundary** — **обёртка** вокруг части дерева: если **внутри** упал **render** дочернего дерева, React **не роняет** всё приложение — показывается **fallback UI** («Что-то пошло не так», кнопка «Попробовать снова»).
- На практике — **`react-error-boundary`** или готовый boundary-компонент: `<ErrorBoundary fallback={…} onError={log}>`. **Не спрашиваем** class API (`getDerivedStateFromError`, `componentDidCatch`) — только **поведение**, **fallback** и **куда** ставить обёртку.
- **Не ловят:** ошибки в **обработчиках событий** (`onClick`) — нужен `try/catch`; **async** в effect без проброса в render; ошибки **в самой** обёртке boundary; **SSR** — отдельная стратегия на сервере.
- **Suspense** — **ожидание** неготового UI: `React.lazy` (chunk), data layer с throw promise; показывает **fallback** (spinner/skeleton), **не** «ошибка упала».
- **Boundary vs Suspense:** Suspense — «ещё **грузится**»; Boundary — «**сломалось** при render». Часто **рядом**: `<ErrorBoundary><Suspense fallback={…}>…</Suspense></ErrorBoundary>`.

**🟡 Джун**

- **ErrorBoundary** оборачивает виджет: упал render внутри — пользователь видит **запасной экран**, а не белую страницу всего сайта.
- **Suspense** + **`React.lazy`** — пока JS-chunk грузится, показывается spinner.
- Ошибка в **onClick** — boundary **не** поймает; нужен `try/catch` или сообщение пользователю из handler — часто не упомянет без подсказки.

**🟠 Мидл**

- Boundary на **границе фичи** (route, виджет «Рекомендации»), не один на root «на всякий случай» — иначе мелкий баг **убивает** весь layout.
- **Async fetch:** `setError` + UI ошибки в компоненте **или** `throw error` в render — **осознанно**, чтобы boundary показал fallback; boundary **сам** fetch не ловит.
- **`React.lazy` + Suspense** — code splitting; общий Suspense на группу lazy-страниц с одним skeleton.
- **Reset:** смена **`key`** на boundary после «Попробовать снова» — **remount** детей, чистый state.

**🔴 Сеньор**

- **Granular boundaries:** упал один виджет — header, sidebar, checkout **живы**; мониторинг через **`onError`** (Sentry + componentStack), в fallback **не** показываю stack и PII пользователю.
- **Suspense for data** (Next/RSC, React Query suspense mode): throw promise → fallback **без** ручного `isLoading`; trade-off — нужен **совместимый** data layer; иначе `useQuery` + skeleton вручную.
- **Streaming SSR:** ошибка в одном Suspense-сегменте может отдать **fallback HTML** для куска страницы, не весь 500 — зависит от framework (App Router).
- **Portal** для modal «Ошибка» — поверх layout, scroll основной страницы не ломается.
- **Не путаю:** Suspense fallback ≠ error UI; loading skeleton ≠ «Что-то пошло не так».

---

### Вопрос 2. Производительность: ререндеры, memo, virtualization

Как **диагностировать** и **снижать** лишние ререндеры в React?

**Суть (устно):**

- **Причины ререндера:** state/props/context изменились, **родитель** ререндернулся (ребёнок по умолчанию тоже), **forceUpdate**, context consumer.
- **Инструменты:** React DevTools **Profiler**, **«Highlight updates»**, why-did-you-render (осторожно в dev).
- **Приёмы:** **`React.memo`**, **`useMemo`/`useCallback`**, colocation state, **split context**, **`children` as prop** (stable slot), **virtualization** (`react-window`) для длинных списков, **lazy** + code splitting.

**🟡 Джун**

- Компонент ререндерится, когда изменились его **props** или **собственный state**; если родитель ререндернулся — ребёнок тоже, даже с теми же props (без memo).
- **`React.memo(Row)`** — пропустить ререндер, если props shallow-equal.
- Длинный список — «рендерить только видимое» (virtualization) — слышал, деталей мало.

**🟠 Мидл**

- **`React.memo`** сравнивает props **поверхностно**; `{ user: { id: 1 } }` каждый render новый объект — memo бесполезен; custom `arePropsEqual` или стабильные ссылки.
- **`useMemo(() => heavyCalc(a), [a])`** — кэш результата; **`useCallback(fn, deps)`** — стабильная функция для memo-child или effect deps.
- **Context:** один объект `{ user, theme, cart }` — любое изменение ререндерит всех consumers; split: ThemeContext редко меняется, CartContext отдельно.
- **Virtualization (`react-window`):** в DOM только ~20 видимых row + overscan; dynamic height — сложнее, нужны измерения.

**🔴 Сеньор**

- Сначала **Profiler** — record interaction, смотрю flamegraph: часто виновник не «мелкий memo-child», а **тяжёлый render родителя** или context broadcast.
- **Anti-pattern:** `memo` на всё + inline `onClick={() => …}` и `style={{ … }}` — каждый render новые ссылки, memo только добавляет compare overhead.
- **State colocation:** state поиска в layout ререндерит всю страницу — опустить input state в sidebar или использовать **`children` slot**, чтобы тяжёлый main не трогать.
- **`useTransition` / `useDeferredValue`** — ввод в search остаётся urgent, фильтрация 10k items — transition; это **приоритет scheduler**, не debounce по таймеру.
- **Bundle:** route split, preload on hover, vite/webpack analyzer — иногда лаг не от ререндеров, а от **parse/eval** JS; **LCP** страдает от fat main chunk.

---

### Вопрос 3. SSR, Next.js и isomorphic React

Чем **SSR** отличается от **CSR** и какие **подводные камни** у **hydration**?

**Суть (устно):**

- **CSR:** пустой HTML + bundle → React монтируется на клиенте; SEO/TTFP слабее без доп. мер.
- **SSR:** сервер рендерит HTML → клиент **hydrate** (привязка event listeners, восстановление React tree). HTML и client render **должны совпасть** — иначе **hydration mismatch**.
- **Next.js:** file routing, **RSC** (App Router), SSR/SSG/ISR, streaming, **`'use client'`** граница.

**🟡 Джун**

- **CSR:** браузер получает пустую страницу, качает JS, React рисует UI.
- **SSR:** сервер отдаёт готовый HTML, потом React **подключается** к нему (hydration).
- **Next.js** — фреймворк поверх React с SSR «из коробки»; mismatch — не объяснит без подсказки.

**🟠 Мидл**

- **Mismatch** если server HTML ≠ первый client render: `Date.now()`, `Math.random()`, `window.localStorage`, разная locale — на сервере одно, на клиенте другое.
- **Fix:** перенести browser-only в **`useEffect`**, **`dynamic(..., { ssr: false })`**, **`suppressHydrationWarning`** только точечно для известных diff (datetime).
- **SSG** — HTML на build; **SSR** — на каждый request; **ISR** — static + revalidate через N секунд.
- **Code splitting:** `React.lazy`, `next/dynamic`; bundler превращает JSX в JS, tree-shaking — на уровне «зачем нужен bundler».

**🔴 Сеньор**

- **Streaming SSR:** сначала shell + skeleton, потом догружаются Suspense segments — лучше TTFB, чем ждать весь page data.
- **RSC (Server Components):** рендер на сервере, **не** попадает в client bundle; **`'use client'`** — граница, где нужны hooks/events; props сериализуются — функции/classes не передать.
- **Data:** waterfall `await a; await b` на page vs **parallel** `Promise.all` в layout; Next **`fetch` cache**, **`revalidate`**, tags для invalidation.
- **Security:** `process.env.SECRET` не утекает в client; только **`NEXT_PUBLIC_*`** в bundle.
- **Isomorphic код:** `typeof window !== 'undefined'` или отдельные entry; **localStorage**, **matchMedia** — только client.
- **A11y:** SSR отдаёт семантический HTML до JS — screen reader видит контент сразу; после hydrate — **focus management** (modals, skip links).

---

## Уровень 4

По каждой теме — **один короткий вопрос**, затем **«Суть (устно)»** и эталоны по грейдам.

---

### Секция 1. Forwarding refs

Зачем **`forwardRef`** и **`useImperativeHandle`**?

**Суть (устно):**

- **`ref`** на function component **не работает** без **`forwardRef`** — ref идёт на **DOM** или **imperative API** дочернего компонента.
- **`useImperativeHandle(ref, () => ({ focus, scroll }), deps)`** — ограничить наружный API (не отдавать весь DOM node).
- Use case: **input** библиотеки, **modal focus**, integration с non-React.

**🟡 Джун**

- **`ref`** на `<input ref={r} />` даёт доступ к DOM-узлу — `r.current.focus()`.
- На **своём** function component ref **не прилипает** сам — нужен **`forwardRef`**, чтобы пробросить ref на внутренний input.

**🟠 Мидл**

- Пишу `const TextField = forwardRef((props, ref) => <input ref={ref} {...props} />)` — родитель может фокусировать «обёрнутый» input.
- Ref — для **imperative** действий (focus, scrollIntoView, measure); данные по-прежнему через props, не через ref.

**🔴 Сеньор**

- Ref **не заменяет** props для data flow — `value`/`onChange` декларативно; ref только edge cases (focus trap, интеграция с non-React).
- **React 19:** `ref` как обычный prop на function components — evolution API, но **`useImperativeHandle`** всё ещё для узкого public API (`{ focus, reset }` вместо whole `<input>`).
- В тестах **RTL + userEvent** ближе к пользователю, чем `ref.current.click()`; в modal — focus trap + return focus on close.

---

### Секция 2. Concurrent Mode (Concurrent Features)

Что даёт **Concurrent React** и чем **`startTransition`** / **`useDeferredValue`** отличаются от debounce?

**Суть (устно):**

- **Concurrent rendering** — React может **прерывать**, **возобновлять** и **приоритизировать** обновления (urgent vs transition).
- **`startTransition(fn)`** / **`useTransition`** — пометить setState как **низкий приоритет**; UI остаётся отзывчивым (input).
- **`useDeferredValue`** — отложить **отображение** тяжёлого значения, пока urgent обновления не завершены.
- Не magic: **тяжёлый render** всё равно нужно оптимизировать или разбивать.

**🟡 Джун**

- **Concurrent React** — React может «не блокировать» интерфейс при тяжёлом обновлении; путает с `async/await` в компоненте.

**🟠 Мидл**

- **`startTransition(() => setFiltered(hugeFilter(query)))`** — пока фильтруется 10k items, input продолжает печататься без лага.
- **Debounce** — ждёт N ms тишины; **transition** — без таймера, через **приоритет** в scheduler React: urgent (ввод) vs transition (список).

**🔴 Сеньор**

- **`useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)`** — подписка на store вне React без **tearing** (разный snapshot в одном commit при concurrent).
- **Suspense + transition:** pending state (`isPending`) показываю stale UI + indicator, input не блокируется.
- Нужен **`createRoot`**, не legacy `ReactDOM.render`; profiling — смотрю concurrent commits в Profiler; transition **не отменяет** работу — только deprioritize, тяжёлый list всё равно оптимизировать.

---

### Секция 3. Suspense (углублённо)

Где **Suspense** работает «из коробки» и где нужна **интеграция** (data/router)?

**Суть (устно):**

- **Lazy components** — throw promise при загрузке chunk → Suspense fallback.
- **Data Suspense** — нужен **framework/cache**, который throw promise при pending (RSC, Relay, experimental routers).
- **Nested Suspense** — гранулярные fallbacks; **avoid waterfall** через parallel fetching на уровне route/layout.

**🟡 Джун**

- Оборачиваю lazy-комponent: `<Suspense fallback={<Spinner />}><LazyPage /></Suspense>` — пока chunk грузится, spinner.

**🟠 Мидл**

- **Nested Suspense:** outer — layout skeleton, inner — content; пользователь видит прогрессивную загрузку.
- Обычный `fetch` в `useEffect` **не** включает Suspense — нужна обёртка, которая throw promise при pending (React Query suspense mode, framework loader).

**🔴 Сеньор**

- **Streaming HTML:** сервер шлёт shell, Suspense boundaries догружают segments — клиент hydrate по частям.
- **Error boundary** рядом с Suspense: loading vs error — разные fallback; throw в render ловит boundary, pending — Suspense.
- Миграция: вместо `isLoading && <Spinner>` — Suspense-aware data layer + nested boundaries; убираю request waterfall через parallel fetch в layout.

---

### Секция 4. PWA и React

Как **PWA** (service worker, manifest) сочетается с **React SPA**?

**Суть (устно):**

- **Manifest** — installability, icons, theme.
- **Service Worker** — cache static assets, offline shell, **не** кэшировать API без стратегии.
- **Workbox** / vite-plugin-pwa — precache bundle; **обновление** SW → prompt reload (`skipWaiting`).
- React остаётся CSR внутри shell; **SSR/PWA** — offline fallback page.

**🟡 Джун**

- **PWA** — сайт можно «установить» на телефон; есть **manifest.json** с иконками и названием.

**🟠 Мидл**

- **Service Worker** кэширует JS/CSS (cache-first) — offline открывается shell React-приложения; API — **network-first**, иначе stale данные.
- После deploy новый SW ждёт закрытия вкладок — показываю «Доступна новая версия → обновить» (`skipWaiting` + reload).

**🔴 Сеньор**

- **HTTPS** обязателен; scope SW не шире нужного; **не кэширую** персонализированный HTML с PII.
- **Push / background sync** — trade-offs iOS (ограничения); не обещаю parity с native.
- **Next + PWA:** App Router + custom SW — осторожно с precache server chunks; offline fallback page отдельно от dynamic routes.

---

### Секция 5. SSR и Next.js (углублённо)

Как устроены **App Router**, **RSC** и **кэширование** в Next.js?

**Суть (устно):**

- **Server Components** — рендер на сервере, **zero** client JS для них by default; **`'use client'`** — boundary.
- **Caching:** `fetch` options, **`revalidate`**, **`unstable_cache`**, segment config **`dynamic`/`force-static`**.
- **Layouts** сохраняют state при navigation; **parallel routes**, **intercepting routes** — advanced routing.

**🟡 Джун**

- **Pages Router** — файлы в `pages/`; **App Router** — `app/` с layouts и server components — «новый способ» без деталей.

**🟠 Мидл**

- **`'use client'`** нужен, где hooks, onClick, browser APIs — server component **не** может `useState`.
- **`export const revalidate = 60`** — ISR: страница static, обновляется раз в минуту.

**🔴 Сеньор**

- **PPR (Partial Prerendering):** static shell с dynamic holes — быстрый first byte + персонализация.
- **Server Actions:** mutation с revalidateTag/revalidatePath — не тащить отдельный API route для каждой формы.
- **Deploy:** edge runtime — нет Node API (fs, некоторые npm); cold start vs regional latency; streaming улучшает TTFB.
- Debug: hydration mismatch, RSC payload errors — смотрю server vs client component boundary и serializable props.

---

### Секция 6. React Testing Library

Как тестировать React-компоненты через **React Testing Library**?

**Суть (устно):**

- Философия: тест как **пользователь** — **roles**, **labels**, **text**, не implementation details (state, class names).
- **`render`**, **`screen`**, **`userEvent`** (preferred over fireEvent), **`waitFor`**, **`findBy*`** для async.
- **Mock** fetch/router на границе; **MSW** для API integration.

**🟡 Джун**

- `render(<Login />)`, `screen.getByText('Submit')`, `await userEvent.click(button)` — проверяю, что UI реагирует.

**🟠 Мидл**

- Приоритет queries: **`getByRole('button', { name: 'Save' })`**, **`getByLabelText('Email')`** — ближе к a11y и устойчивее к refactor className.
- Async: **`await findByText('Loaded')`** или **`waitFor(() => expect(…))`** после fetch; fake timers — аккуратно с userEvent.

**🔴 Сеньор**

- Не тестирую **`useState` call count** — тестирую поведение: «после submit показана ошибка валидации».
- **Custom render** с providers (Redux, QueryClient, Router) — integration на уровне feature.
- **E2E** (Playwright) — checkout, auth; RTL — компоненты и flows; flaky: забытый `await`, shared mutable state между tests, act warnings.

---

### Секция 7. Context API (продвинуто)

Когда **Context** — правильный выбор и как **не убить** производительность?

**Суть (устно):**

- Context — **broadcast** значения без prop drilling: theme, locale, auth **snapshot**, dispatch-only context.
- **Проблема:** любое изменение value → **все** consumers ререндер (если не split/memo).
- **Паттерны:** разделить **StateContext** / **DispatchContext**; memoize value; **`useContextSelector`** (libs) или colocation.

**🟡 Джун**

- `<ThemeProvider value="dark">` + `useContext(ThemeContext)` — не тащить theme через 5 уровней props.

**🟠 Мидл**

- Не кладу **часто меняющийся cart** в один context с **theme** — смена theme ререндерит cart consumers без нужды.
- Альтернатива при частых updates: **Zustand/Jotai** с selector или Redux.

**🔴 Сеньор**

- Локальная проблема — **composition** (`children` slot), не global context; context для **действительно global** stable или редко меняющихся вещей.
- **RSC:** React context работает в client components; server components не используют context consumers напрямую — граница `'use client'`.
- Тесты: оборачиваю минимальным Provider с mock value; не тяну весь App provider в unit test.

---

### Секция 8. Portal

Зачем **`createPortal`**?

**Суть (устно):**

- Рендер children в **другой DOM node** (часто `document.body`) при сохранении **React tree** (events bubble по **React** иерархии, не DOM).
- Modal, tooltip, dropdown — **escape overflow:hidden**, **z-index** stacking.
- **Focus trap**, **aria-modal**, return focus on close — a11y обязательна.

**🟡 Джун**

- Modal рисуется в **`document.body`**, а не внутри `<div id="root">` — чтобы был поверх всего и не обрезался `overflow: hidden` родителя.

**🟠 Мидл**

- **`createPortal(modal, document.body)`** — DOM-узел другой, но **onClick** на modal всё ещё **всплывает по React-дереву** к общему ancestor в логике React.
- z-index и stacking context — portal решает типичную боль nested layout.

**🔴 Сеньор**

- **SSR:** `document.body` нет на сервере — portal только после mount или dynamic ssr:false.
- **Nested portals**, **scroll lock** on body, **`inert`** на backdrop — фокус не уходит под modal; при close — return focus на кнопку, открывшую dialog.

---

### Секция 9. Custom hooks (углублённо)

Как проектировать **custom hooks** и какие **правила** Hooks соблюдать?

**Суть (устно):**

- Extract **stateful logic**: имя **`use*`**, вызывать только on top level / только из React functions.
- **Contract:** входы, возвращаемое value/stable fns, **document deps** for caller.
- **Composition:** маленькие hooks (`useToggle`, `useFetch`) → **`useUserProfile`**.
- **Shared state:** hook с module-level store — осторожно; предпочитать context/external store.

**🟡 Джун**

- Выношу повторяющиеся `useState` + `useEffect` в **`useFetch(url)`** — один раз написал, везде вызываю.

**🟠 Мидл**

- Hook возвращает **`{ data, error, isLoading, refetch }`** — стабильный контракт; **`useCallback`** на refetch если отдаю в deps детям.
- Тестирую через **`renderHook`** + **`act`**.

**🔴 Сеньор**

- В **`useFetch`** — **AbortController** на unmount и смену url; не setState после unmount.
- **eslint-plugin-react-hooks** в CI — rules of hooks не negotiable.
- Публичный hook в библиотеке — **semver** на shape return value; breaking rename `loading` → `isPending` — major bump.

---

### Секция 10. Code splitting и module preloading

Какие **стратегии** code splitting и **preloading** модулей в React-приложении?

**Суть (устно):**

- **Route-based** split — основной выигрыш; **component-level** `React.lazy`.
- **Preload:** `import()` on **link hover/focus**, **`rel="modulepreload"`**, webpack **magic comments** `webpackPrefetch`, Next **`<Link prefetch>`**.
- Balance: **too many chunks** → HTTP overhead; **analyze bundle**.

**🟡 Джун**

- **`React.lazy(() => import('./Page'))`** — страница в отдельном JS-файле, грузится при первом заходе.

**🟠 Мидл**

- Split по **routes** в React Router / **`next/dynamic`** — main bundle маленький, TTI быстрее.
- **Prefetch:** on hover на `<Link>` вызываю `import('./Dashboard')` — при клике chunk уже в cache.

**🔴 Сеньор**

- **Critical path:** above-the-fold в main, charts/editors — lazy; на SSR split влияет на TTFB vs client TTI — балансирую.
- **Vendor chunk**, **shared chunks**, **`sideEffects: false`** в package.json для tree-shaking.
- **Module federation** — micro-frontends на уровне архитектуры; не дроблю до 200 chunks без метрик.

---

### Секция 11. Accessibility (a11y)

Какие **a11y** практики обязательны в React UI?

**Суть (устно):**

- Семантика: **button** vs div onClick; **labels** для inputs; **heading hierarchy**.
- Keyboard: **focus order**, **Escape** закрывает modal, **roving tabindex** в menus.
- **ARIA** когда нет нативного элемента; не дублировать роль нативного (`button role=button`).
- Live regions: **`aria-live`** для toasts/async errors.

**🟡 Джун**

- **`<label htmlFor="email">`** связан с input; у картинок **`alt`**; кнопка — **`<button>`**, не `<div onClick>`.

**🟠 Мидл**

- Modal: **focus trap**, Escape закрывает, **return focus**; видимый **focus ring** не вырезаю в CSS.
- **eslint-plugin-jsx-a11y** ловит `img` без alt, click на div без role.

**🔴 Сеньор**

- Проверяю critical flows в **VoiceOver/NVDA** — не только Lighthouse score.
- **Virtualized list:** `aria-rowcount`, keyboard nav при scroll — row в DOM ≠ row в data.
- Design system: **contrast tokens**, **skip link**, form errors через **`aria-describedby`** + focus first invalid field.

---

### Секция 12. Styled-components и CSS-in-JS

Плюсы и минусы **styled-components** / CSS-in-JS в React?

**Суть (устно):**

- **Colocation**, dynamic styles from props, theming via **ThemeProvider**.
- **Runtime cost** — injection, specificity; **SSR** — **`ServerStyleSheet`**, hydrate class names.
- **Trend:** zero-runtime (**Linaria**, **Vanilla Extract**, **Tailwind**) для performance.

**🟡 Джун**

- **`styled.button`** — стили рядом с компонентом; props можно в template: `` background: ${p => p.primary ? 'blue' : 'gray'} ``.

**🟠 Мидл**

- **ThemeProvider** — цвета/spacing из темы; transient props **`$variant`** — не попадают в DOM.
- На SSR собираю стили в **`ServerStyleSheet`**, иначе flash unstyled.

**🔴 Сеньор**

- Runtime CSS-in-JS — cost на каждый render (insertRule); на больших SPA метрики хуже, чем CSS modules / Tailwind.
- **RSC + runtime CSS-in-JS** — проблемная связка; в App Router чаще **compile-time** (Vanilla Extract) или Tailwind.
- **FOUC/hydration mismatch** class names — critical CSS или static extraction.

---

### Секция 13. UI-библиотеки (MUI и др.)

Как **Material UI** (или аналог) влияет на **bundle** и **кастомизацию**?

**Суть (устно):**

- **Tree shaking** named imports; avoid barrel import whole lib.
- **Theming:** `createTheme`, **`sx`**, **`styled` API** MUI.
- **Trade-off:** скорость разработки vs bundle size vs design uniqueness.

**🟡 Джун**

- Импортирую **`import Button from '@mui/material/Button'`**, а не весь `@mui/material` разом — «чтобы bundle не раздулся» (интуиция).

**🟠 Мидл**

- Icons: **`import DeleteIcon from '@mui/icons-material/Delete'`** — не весь icons pack.
- Кастомизация через **`theme.components.MuiButton.styleOverrides`**.

**🔴 Сеньор**

- **Emotion cache** на SSR — иначе duplicate styles / wrong insertion order.
- MUI v6+ / **Pigment CSS** — движение к zero-runtime; для уникального дизайна — **headless Radix + Tailwind**, MUI когда скорость важнее uniqueness.

---

### Секция 14. Form libraries (Formik и др.)

Когда **Formik** / **React Hook Form**, а когда **нативный** controlled state?

**Суть (устно):**

- **RHF:** uncontrolled + ref, меньше ререндеров; **`register`**, **`controller`** для controlled third-party.
- **Formik:** values in state, проще ментальная модель для маленьких форм.
- Validation: **Yup/Zod** schema; **server errors** mapping to fields.

**🟡 Джун**

- Библиотека «собирает поля и submit» — не писать десять `useState` руками.

**🟠 Мидл**

- **Formik** — каждое поле трогает values в state → ререндер формы; ок для 5 полей.
- **RHF** — **`register('email')`** + ref, валидация **`resolver: zodResolver(schema)`**; **`Controller`** для MUI DatePicker (controlled third-party).

**🔴 Сеньор**

- Форма на **50+ полей** — RHF default: меньше re-render storm; Formik без memoization страдает.
- **A11y:** error summary, **`aria-invalid`**, **`aria-describedby`**, focus на первую ошибку после submit.
- **Server Actions (Next):** progressive enhancement — form работает без JS, React enhance сверху.

---

### Секция 15. State management alternatives (Recoil, Zustand, Jotai)

Чем **Recoil** / **atoms** отличаются от **Redux** и от **Context**?

**Суть (устно):**

- **Recoil/Jotai:** atomic **fine-grained** subscriptions — компонент подписан на **atom**, не на весь store.
- **Zustand:** простой store + selectors, minimal boilerplate.
- **React Query/SWR:** **server state** отдельно от **client UI state** — не дублировать entities в Redux без нужды.

**🟡 Джун**

- **Redux** — один большой store; **Recoil/Zustand** — «полегче», names слышал.

**🟠 Мидл**

- **Context** ререндерит всех при любом изменении value; **Zustand** `useStore(s => s.cart)` — только cart slice.
- **React Query** для API; Redux/Zustand для UI (modal open, sidebar) — не кладу users list в Redux, если Query уже кэширует.

**🔴 Сеньор**

- Relational data — **normalization** нужна и в atoms, не только в Redux.
- **Recoil** — maintenance/migration риски; **Jotai** atoms + derived atoms — fine-grained, дружат с concurrent.
- **Три кэша одних users** (Redux + Query + local state) — рассинхрон; один source of truth per concern.

---

### Секция 16. Webpack и Babel (React toolchain)

Какую роль **Babel** и **Webpack/Vite** играют в React-проекте?

**Суть (устно):**

- **Babel:** JSX → JS, **polyfills** (preset-env), **class properties**; **не** typecheck (TS — отдельно).
- **Bundler:** module graph, **HMR**, **code split**, **asset pipeline**.
- **Vite:** dev esbuild, prod rollup; **fast refresh** vs classic HMR.

**🟡 Джун**

- **Babel** превращает JSX в `React.createElement` / `_jsx`; браузер JSX не понимает.
- **Webpack/Vite** собирают модули в bundle для браузера.

**🟠 Мидл**

- **`@babel/preset-react` `runtime: 'automatic'`** — не нужен `import React` в каждом файле.
- Env variables: **`import.meta.env`** (Vite) / **DefinePlugin** — только public vars в client bundle.

**🔴 Сеньор**

- **Module Federation**, **SSR dual bundles** (server vs client entry), **source maps** in prod — policy (скрыть или Sentry upload).
- **SWC/esbuild** вместо Babel на 80% кейсов; Babel остаётся для exotic plugins.
- **Fast Refresh** сохраняет state при edit — не путать с full page HMR reload.

---

### Секция 17. StrictMode

Зачем **`StrictMode`** в development?

**Суть (устно):**

- **Double invoke** render/setup/cleanup для effects — выявить **non-idempotent** side effects.
- **Deprecated API** warnings; **prepare** for concurrent features.
- **Не** дублирует работу в production build.

**🟡 Джун**

- В dev компоненты и effects «как будто два раза» — это нормально, в prod один раз.

**🟠 Мидл**

- Effect с `[]` в StrictMode: mount → cleanup → mount — если после ухода со страницы остался interval, это баг; cleanup обязан всё снять.

**🔴 Сеньор**

- «Двойной fetch в dev» — не повод отключать StrictMode; фикс: **AbortController**, idempotent subscribe, или accept dev-only double call.
- Объясняю команде: StrictMode — **линтер на жизненный цикл**, не баг React.

---

### Секция 18. Debugging React

Как **отлаживать** React-приложения?

**Суть (устно):**

- **React DevTools:** components tree, props/state, **Profiler**, **⚛️** badge for highlights.
- **Sources** breakpoints; **Log** render counts (temporary).
- **Common bugs:** wrong deps, mutation, key, context over-render, stale closure.

**🟡 Джун**

- **React DevTools** — смотрю props/state компонента; **`console.log`** в render — «сколько раз рендерится».

**🟠 Мидл**

- **Profiler** — record click flow, ищу компонент с longest render time; **highlight updates** — кто лишний раз мигает.

**🔴 Сеньор**

- **Redux DevTools** time-travel — replay action sequence; production **session replay** (LogRocket) — только без PII в recording policy.
- Error overlay **component stack** → source maps в monorepo (правильный path mapping).
- Чеклист: deps effect, mutation state, index key, fat context, stale closure — системно, не random logs.

---

### Секция 19. Props validation

Как валидировать **props** в React (PropTypes vs TypeScript)?

**Суть (устно):**

- **`prop-types`** — runtime dev warnings; legacy JS codebases.
- **TypeScript** — compile-time; **`React.FC`** vs explicit props type; **children** typing in React 18+.
- **Default props** — default params vs `defaultProps` (deprecated for function components).

**🟡 Джун**

- **TypeScript interface** для props или **PropTypes** — «чтобы не передать string вместо number».

**🟠 Мидл**

- **Discriminated union** для variant: `{ variant: 'link', href: string } | { variant: 'button', onClick: … }`.
- **`ComponentProps<'button'>`** — расширяю нативный button без дублирования атрибутов.

**🔴 Сеньор**

- **Generic** `<List<T> items={T[]} renderItem={(item: T) => …} />`.
- Runtime validation **на границе API** (zod parse response) — не дублировать всю схему в PropTypes и TS; TS для compile-time контракта компонентов.

---

### Секция 20. Redux Saga (async orchestration)

Когда **redux-saga** предпочтительнее **thunk**?

**Суть (устно):**

- **Saga:** **`takeEvery`/`takeLatest`**, **debounce**, **race**, **fork/join**, **cancellation** через **`cancel`/` cancelled`**.
- **Thunk:** проще, imperative chains; cancellation сложнее.
- **RTK Query** — многие data workflows без ручных sagas.

**🟡 Джун**

- **Saga** — «generators для side effects в Redux» — без деталей takeLatest.

**🟠 Мидл**

- **`takeLatest('SEARCH', fetchResults)`** — пользователь быстро печатает, предыдущий fetch **отменяется**, UI не перезаписывается старым ответом.
- Thunk для простого `fetch → dispatch`; saga когда **много шагов**, debounce, race между actions.

**🔴 Сеньор**

- Saga **не ловит** React render errors — channel для errors, logging, retry policy отдельно от error boundary.
- Тесты: **`redux-saga-test-plan`** — declarative step assertions.
- Новый проект: **RTK Query** / React Query вместо ручных sagas для CRUD; saga остаётся для сложных orchestration (websocket + queue + offline sync).

---

## Как пользоваться документом на собеседовании

- **«Суть (устно)»** — краткий ориентир для интервьюера перед эталонами; не обязательно зачитывать кандидату.
- **Эталоны по грейдам** — примеры **формулировок ответа**, не чеклист «кандидат понимает». Джун короче, сеньор глубже и с продакшен-деталями.
- **Джун:** базовые определения + один бытовой пример; пробелы по подсказке — норма.
- **Мидл:** связная модель + термины + что писать в коде.
- **Сеньор:** объясняет **простым языком**, называет компромиссы, anti-patterns, диагностику — без «понимает / знает / умеет».
- **Уровень 1** опирается на базовый JavaScript (`javascript-interview-levels-1-4.md`); **уровни 3–4** — production React.
