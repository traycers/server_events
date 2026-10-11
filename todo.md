# спецификация проекта `seev`

## 1. Назначение

Демо-проект на Go + React, показывающий:
- **SSE-канал** `/api/events` с событиями о жизненном цикле live-трансляции.
- **Live HTTP-стрим fMP4** `/api/video/stream`, воспроизводимый через MSE.
- **Cross-tab синхронизацию** через `BroadcastChannel('live')` между `/sse` и `/live` в одном браузере.

Трансляция — синтетическая (ffmpeg `lavfi testsrc2`), но с полным жизненным циклом: generate → start → live → finish.

---

## 2. Архитектура

### 2.1 Модель вещания

- **Единая точка правды на сервере.** Состояние live, сгенерированный контент, счётчики — всё на сервере.
- **Broadcast.** Один «насос» (тикер раз в `X-GOP_S`) рассылает сегменты всем активным стрим-соединениям. Все зрители видят один и тот же кадр синхронно.
- **Синхронность между браузерами** достигается за счёт одинакового контента и одинаковой скорости раздачи; для визуальной сверки в кадр вшиты `pts` и номер кадра через `drawtext`.

### 2.2 Состояния live

```
idle ──generate──▶ generating ──auto──▶ live  (если до generate было live)
                       │
                       └──▶ finished  (иначе)
                       │
                       └──▶ idle      (если finish нажат во время генерации)
```

Полный набор: `idle`, `generating`, `live`, `finished`.

### 2.3 Cross-tab

- Вкладка `/sse` держит SSE-соединение, получает `play_again`, постит в `BroadcastChannel('live')`.
- Все вкладки `/live` в том же браузере слушают канал. При получении — перезапускают MSE-стрим.
- Если `/sse` не открыт — вкладка `/live` не узнаёт о `play_again`, стрим обрывается по EOF.

---

## 3. Матрица команд по состояниям

| Команда | `idle` | `generating` | `live` | `finished` |
|---|---|---|---|---|
| `generate` | генерация; остаёмся `idle` | успех, no-op | stop → gen → auto-start + `play_again` | генерация без автостарта |
| `start` | **400**, «нет данных, вызовите generate» | **200** «уже идёт генерация» + `live/start` | **200** + `live/start` (повторное) | старт с тем же контентом + `live/status: live` |
| `finish` | **200**, no-op | **200** + отменяем будущий `play_again`; по окончании ген. → `finished` | stop + `live/status: finished` | **200** + `live/finish` (повторное) |

---

## 4. HTTP API

### 4.1 `GET /api/events` — SSE

`Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`.

Формат события:
```
event: live/status
id: 42
data: {"status":"live","t":1728644400123}

```

Типы событий:
- `server/started` — при старте сервера, несёт время старта.
- `live/status` — при каждом изменении состояния (`idle`/`generating`/`live`/`finished`).
- `generation/started` — начало генерации.
- `generation/finished` — конец генерации.
- `play_again` — сигнал клиентам перезапустить MSE.
- `live/segment/send` — на каждый отправленный сегмент (`gop_index`, `total_sent`, `viewers`).
- `live/start` — нажатие кнопки start (в том числе повторное).
- `live/finish` — нажатие кнопки finish (в том числе повторное).

Ring buffer на `X-EVENT_BUFFER_SIZE` (дефолт 50) событий для `Last-Event-ID`. При переподключении — досылка событий с ID > указанного.

Backpressure: не управляем, для демо пишем напрямую в `http.ResponseWriter` + `Flush`.

### 4.2 `POST /api/generate`

Всегда **200**. Тело:
```json
{"started": true, "already_generating": false}
```
или
```json
{"started": false, "already_generating": true}
```

### 4.3 `POST /api/live/start`, `POST /api/live/finish`

Возвращают **200** с телом `{"status": "..."}` текущего состояния, либо **400** для `start` в `idle`. Коды по матрице §3.

### 4.4 `GET /api/live/status`

**200** `{"status":"idle"}` — только статус.

### 4.5 `GET /api/video/stream`

- `Content-Type: video/mp4`, `Cache-Control: no-store`, без `Content-Length`, без `Accept-Ranges`.
- При `live`/`finished`: init + сегменты в реальном времени, EOF по `finish`.
- При `idle`/`generating`: **404**.
- **При `finished` соединение висит** и ждёт нового `live` (не закрываем сразу).

---

## 5. Генерация контента (ffmpeg)

### 5.1 Параметры (env)

- `X-FPS` (25)
- `X-GOP_S` (1) — длительность GOP в секундах
- `X-VIDEO_DURATION_S` (10) — всего секунд = число GOP
- `X-RESOLUTION` (1920x1080)
- `X-FFMPEG_PATH` (ffmpeg)
- `X-FONT_PATH` (`/usr/share/fonts/truetype/iosevka/Iosevka-Regular.ttf`)
- `X-MSE_CODECS` (`avc1.42E028`)

Производные: `-g = X-GOP_S * X-FPS`, `-t = X-GOP_S`, `N = X-VIDEO_DURATION_S / X-GOP_S`.

### 5.2 Первый вызов ffmpeg (init + GOP 0)

```bash
ffmpeg \
  -f lavfi -i "testsrc2=size=${RES}:rate=${FPS}" \
  -vf "drawtext=fontfile=${FONT}:\
       text='%{pts\:hms}  #%{n}':x=20:y=20:fontsize=48:\
       fontcolor=white:box=1:boxcolor=black@0.5" \
  -t ${GOP_S} \
  -output_ts_offset 0 \
  -c:v libx264 -profile:v baseline -level 4.0 \
  -g ${G} -keyint_min ${G} -sc_threshold 0 \
  -movflags empty_moov+default_base_moof+frag_keyframe \
  -f mp4 pipe:1
```

Из вывода берём:
- `ftyp` + `moov` → **Init** (сохраняем один раз).
- `moof` + `mdat` → **Segments[0]**.

### 5.3 Вызовы 1..N-1

Те же параметры, но `-output_ts_offset K*GOP_S` (K = 1..N-1). Из вывода берём только `moof`+`mdat`. Если в выводе присутствует `moov` — выбрасываем.

### 5.4 Разбор вывода ffmpeg в Go

Минимальный примитив — обход top-level боксов:

```go
type Box struct {
    Type string
    Data []byte // весь бокс, включая 8-байтный заголовок
}

func SplitTopLevelBoxes(data []byte) ([]Box, error)
```

Алгоритм: читаем `size` (uint32 BE), `type` (4 байта ASCII), продвигаемся. Если `size == 1` — 64-битный size идёт следом. Если `size == 0` — до конца буфера.

Никакого разбора внутренностей `moov`/`tfdt` не требуется — `-output_ts_offset` сдвигает PTS автоматически.

### 5.5 Fallback

Если на первом запуске окажется, что `-output_ts_offset` не сдвигает `tfdt` в `moof`, переходим на `-itsoffset K*GOP_S -i ...` (перед `-i`) или фильтр `setpts`. Проверяется эмпирически.

### 5.6 Хранение

```go
type Content struct {
    Init     []byte    // ftyp + moov
    Segments [][]byte  // N штук, каждая — moof + mdat
}
```

Хранится в памяти, перезаписывается при каждом `generate`.

---

## 6. Раздача live (`/api/video/stream`)

### 6.1 Жизненный цикл соединения

1. Клиент подключается, `live = live` — сервер:
   - пишет `Init`,
   - пишет `Segments[0]`,
   - регистрируется как подписчик в broadcast-канале.
2. Серверный тикер раз в `X-GOP_S` публикует следующий сегмент в канал.
3. Все подписчики получают сегмент, пишут в `http.ResponseWriter`, вызывают `Flusher.Flush()`.
4. При `finish` — сервер закрывает соединения (EOF), клиент вызывает `mediaSource.endOfStream()`.
5. При `live = finished` в момент подключения — соединение открыто, но ничего не пишется, пока не начнётся новый `live`.

### 6.2 Обработка `idle`/`generating`

**404** сразу.

### 6.3 Отмена

`ctx.Done()` в горутине раздачи, отписка от канала, лог `stream client disconnected`.

---

## 7. Клиент (React + TypeScript + Vite + Bootstrap dark)

### 7.1 Страницы

- `/` → редирект на `/sse`.
- `/sse` — страница событий.
- `/live` — страница трансляции.

### 7.2 `/sse`

- Кнопки: `connect` / `disconnect`, `generate`.
- Лента последних событий (последние ~200, кольцевой буфер на клиенте).
- Кнопка `generate` → `POST /api/generate`.
- `connect` создаёт `EventSource('/api/events')`, `disconnect` — `es.close()`.
- По приёму `play_again` — `broadcastChannel.postMessage({type: 'play_again'})`.
- Отображение статуса соединения: `connecting / open / error / closed`.

### 7.3 `/live`

- Кнопки: `refresh`, `play`, `stop`, `start`, `finish`.
- `refresh` → `GET /api/live/status`, обновляет отображение.
- `play` → создаёт MSE, `fetch('/api/video/stream')`, читает `ReadableStream`, кормит `SourceBuffer`. При `404` — сообщение «нет вещания».
- `stop` — `AbortController.abort()`, `mediaSource.endOfStream()`, отписка от BroadcastChannel. Клиентское.
- `start` → `POST /api/live/start`.
- `finish` → `POST /api/live/finish`.
- Слушает `BroadcastChannel('live')`; при `play_again` — перезапускает MSE (новый SourceBuffer, новый fetch).

### 7.4 MSE-плеер

- MIME: `video/mp4; codecs="${X-MSE_CODECS}"`.
- Очередь `appendBuffer`: не более одного активного в момент (`updateend`).
- Очистка буфера: `sourceBuffer.remove(0, video.currentTime - 1)` когда `buffered.length > 3` или по достижении 3 GOP. Кольцевой буфер 3 GOP на клиенте.
- Autoplay запрещён, старт — по кнопке `play`.
- Отображаемые счётчики: номер текущего GOP, всего получено сегментов клиентом, размер буфера.

### 7.5 Cross-tab

`const ch = new BroadcastChannel('live')` — на обеих страницах. `/sse` только постит, `/live` только слушает.

### 7.6 Сборка

Vite + TypeScript, без внешних рантайм-зависимостей. Bootstrap устанавливается npm-пакетом, шрифты/иконки — из пакета, никаких CDN.

---

## 8. Env-переменные (полный список)

| Имя | Дефолт | Назначение |
|---|---|---|
| `X-PORT` | `28080` | Порт сервера |
| `X-LOG_LEVEL` | `info` | `debug` / `info` / `warn` / `error` |
| `X-GOP_S` | `1` | Длительность GOP в секундах |
| `X-VIDEO_DURATION_S` | `10` | Длительность всего контента |
| `X-FFMPEG_PATH` | `ffmpeg` | Путь к бинарю |
| `X-FPS` | `25` | Частота кадров |
| `X-RESOLUTION` | `1920x1080` | Разрешение `WxH` |
| `X-FONT_PATH` | `/usr/share/fonts/truetype/iosevka/Iosevka-Regular.ttf` | Шрифт для drawtext |
| `X-EVENT_BUFFER_SIZE` | `50` | Ring buffer для `Last-Event-ID` |
| `X-MSE_CODECS` | `avc1.42E028` | Codec string для MSE |

`X-SSE_INTERVAL_MS` — **убран**.

---

## 9. Структура репозитория

```
.
├── Dockerfile              # multistage: web-build → server-build → runtime
├── Makefile
├── README.md
├── web/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── index.html
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── pages/
│       │   ├── SsePage.tsx
│       │   └── LivePage.tsx
│       ├── components/
│       │   └── MsePlayer.tsx
│       ├── lib/
│       │   ├── api.ts
│       │   └── broadcast.ts
│       └── styles.css
└── server/
    ├── go.mod              # module seev
    ├── cmd/seev/
    │   └── main.go
    └── internal/
        ├── config/         # env-парсинг
        ├── log/            # slog-обёртка
        ├── sse/            # /api/events, ring buffer, broadcast
        ├── live/           # состояния, команды, тикер
        ├── ffmpeg/         # запуск, парсинг, генерация Content
        ├── mp4/            # SplitTopLevelBoxes
        ├── video/          # /api/video/stream
        └── web/            # go:embed dist, SPA-fallback, API-роуты
```

---

## 10. Docker (один файл, multistage)

**Stage 1 — web-build:** `node:20-alpine`, `npm ci && npm run build`, артефакт `/web/dist`.

**Stage 2 — server-build:** `golang:1.24`, копируем `server/` + `web/dist`, `go build -o /seev ./cmd/seev`.

**Stage 3 — runtime:** `debian:bookworm-slim` +
- `apt-get install -y ffmpeg fonts-iosevka ca-certificates`
- копируем бинарь `/seev`
- копируем скачанный Iosevka (если `fonts-iosevka` не подходит по имени файла — берём `.ttf` с GitHub releases и кладём в `/usr/share/fonts/truetype/iosevka/`)
- `EXPOSE 28080`, `ENTRYPOINT ["/seev"]`

Шрифт скачивается в build-stage (или в runtime-stage) с GitHub releases; путь и имя — выясняются на этапе сборки и фиксируются в `X-FONT_PATH`.

---

## 11. Логирование

`slog` → stdout, текстовый формат (не JSON), уровень из `X-LOG_LEVEL`.

Обязательные INFO-логи: старт сервера, старт/конец генерации, смена состояния, старт/конец раздачи, подключение/отключение зрителя, нажатия кнопок (с пометкой `repeated` при повторных).

DEBUG: каждый отправленный сегмент (номер, размер), каждый запуск ffmpeg с полной командой, задержки имитации.

---

## 12. Порядок реализации

1. **Скелет репо + Makefile + Dockerfile (пока без ffmpeg).** Проверка: `make build` собирает, `make run` отдаёт заглушку.
2. **`internal/mp4` + unit-тест на `SplitTopLevelBoxes`** (без внешних зависимостей).
3. **`internal/ffmpeg`** — генерация контента, проверка вручную: init + N сегментов в памяти, `xxd` показывает корректные боксы.
4. **`internal/live` + `internal/sse`** — состояния, broadcast, события. Проверка через `curl -N /api/events`.
5. **`internal/video`** — стрим, broadcast, отдача. Проверка через `curl -N /api/video/stream | xxd`.
6. **React-скелет** — две страницы, роутинг, статика в embed.
7. **`/sse` страница** — EventSource, лента, кнопки.
8. **`/live` страница + MsePlayer** — MSE, refresh, play/stop/start/finish.
9. **`BroadcastChannel`** — связка двух страниц.
10. **Финальный Dockerfile** — multistage с ffmpeg и шрифтом.

---

## 13. Что НЕ делаем (напоминание)

- Range-запросы и скачивание файла.
- Отдельный init-сегмент «на каждый GOP».
- `X-SSE_INTERVAL_MS` и любые heartbeat-события.
- Тесты (кроме unit на `SplitTopLevelBoxes`).
- HLS/DASH-манифесты, аудио, транскодирование.
- Аутентификация, CORS, multi-tenancy.
- JSON-логи, structured logging за пределами `slog`.

