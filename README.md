# Remna Routing Updater

Микросервис для автоматического обновления кастомного заголовка ответа `routing` в Remna панели при появлении новых данных в GitHub-репозитории [roscomvpn-happ-routing](https://github.com/hydraponique/roscomvpn-happ-routing).

## Как работает

1. При запуске получает заголовок `routing` из настроек подписки и из настроек каждого настроенного внешнего сквада
2. Проверяет файлы с роутингом на GitHub — по интервалу (`CHECK_INTERVAL`) или по расписанию (`CRON_SCHEDULE`)
3. Если содержимое изменилось — отправляет новое содержимое заголовка `routing` в Remna
4. Если изменений нет — ничего не делает

Настройки подписки и каждый внешний сквад отслеживаются **независимо**: у каждого свой GitHub URL и свой кеш текущего роутинга. Изменение в одном не затрагивает остальные.

> **Требуется Remnawave 3.0.0+.** Начиная с 3.0.0 роутинг хранится не в поле `happRouting`, а в
> кастомном заголовке ответа `routing`: у настроек подписки — в `customResponseHeaders`, у внешнего
> сквада — в `responseHeadersAdd`. На панели 2.x этот апдейтер работать не будет — для неё нужен
> коммит форка до мерджа поддержки 3.0.0.

Панель принимает объект заголовков целиком, поэтому апдейтер перед каждой записью перечитывает
текущие заголовки и подменяет в них только `routing`. Остальные заголовки (`profile-title`,
`support-url` и прочие), в том числе добавленные в панели уже после запуска контейнера,
сохраняются. Если сквад явно удалял `routing` через `responseHeadersRemove`, эта запись убирается,
иначе она перебивала бы добавленный заголовок.

## Быстрый старт

```bash
mkdir remna-routing-updater && cd remna-routing-updater
```

### Внешняя панель (HTTPS)

Создайте файл `.env`:

```env
REMNA_BASE_URL=https://your-host/api
REMNA_TOKEN=your_bearer_token
# GITHUB_RAW_URL=https://raw.githubusercontent.com/hydraponique/roscomvpn-happ-routing/refs/heads/main/HAPP/DEFAULT.DEEPLINK
# CHECK_INTERVAL=300
```

Создайте файл `docker-compose.yml`:

```yaml
services:
  routing-updater:
    image: ghcr.io/lifeindarkside/remnawave-routing-update:latest
    container_name: remna-routing-updater
    restart: unless-stopped
    env_file:
      - .env
```

### Локальная панель (Docker)

Если RemnaWave панель запущена локально в Docker (образ `remnawave/backend:latest`), контейнер updater нужно подключить к той же сети `remnawave-network` и обращаться к панели по имени контейнера.

Создайте файл `.env`:

```env
REMNA_BASE_URL=http://remnawave:3000/api
REMNA_TOKEN=your_bearer_token
# GITHUB_RAW_URL=https://raw.githubusercontent.com/hydraponique/roscomvpn-happ-routing/refs/heads/main/HAPP/DEFAULT.DEEPLINK
# CHECK_INTERVAL=300
```

> `remnawave` — имя контейнера панели, `3000` — порт по умолчанию. Измените при необходимости.

Создайте файл `docker-compose.yml`:

```yaml
services:
  routing-updater:
    image: ghcr.io/lifeindarkside/remnawave-routing-update:latest
    container_name: remna-routing-updater
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - remnawave-network

networks:
  remnawave-network:
    name: remnawave-network
    external: true
```

> Сеть `remnawave-network` должна уже существовать (создаётся docker-compose панели RemnaWave).

Запуск:

```bash
docker compose up -d
```

### Сборка из исходников

Если хотите собрать образ самостоятельно:

```bash
git clone https://github.com/lifeindarkside/Remnawave-Routing-update.git
cd Remnawave-Routing-update
cp .env.example .env
# отредактируйте .env
docker build -t remna-routing-updater .
docker compose up -d
```

## Переменные окружения

| Переменная | Обязательная | По умолчанию | Описание |
|---|---|---|---|
| `REMNA_BASE_URL` | да | — | Базовый URL API Remna (например `https://host/api` или `http://remnawave:3000/api`) |
| `REMNA_TOKEN` | да | — | Bearer-токен для авторизации в Remna API |
| `GITHUB_RAW_URL` | нет | [DEFAULT.DEEPLINK](https://raw.githubusercontent.com/hydraponique/roscomvpn-happ-routing/refs/heads/main/HAPP/DEFAULT.DEEPLINK) | URL файла с роутингом для настроек подписки |
| `CHECK_INTERVAL` | нет | `300` | Интервал проверки обновлений (в секундах), общий для всех |
| `CRON_SCHEDULE` | нет | — | Запуск проверки по расписанию (cron-выражение, напр. `0 9 * * *`). Если задано — заменяет `CHECK_INTERVAL`. Время в часовом поясе контейнера (по умолчанию UTC) |
| `SQUAD_N_UUID` | нет | — | UUID внешнего сквада (N = 1, 2, 3, ...) |
| `SQUAD_N_URL` | нет | — | GitHub URL файла с роутингом для этого сквада |
| `ROUTING_EXTRA_DIRECT_SITES` | нет | — | Дополнительные правила direct (сайты), через запятую. Пример: `geosite:my-domain,geosite:another` |
| `ROUTING_EXTRA_DIRECT_IP` | нет | — | Дополнительные правила direct (IP), через запятую. Пример: `geoip:my-range` |
| `ROUTING_EXTRA_PROXY_SITES` | нет | — | Дополнительные правила proxy (сайты) |
| `ROUTING_EXTRA_PROXY_IP` | нет | — | Дополнительные правила proxy (IP) |
| `ROUTING_EXTRA_BLOCK_SITES` | нет | — | Дополнительные правила block (сайты) |
| `ROUTING_EXTRA_BLOCK_IP` | нет | — | Дополнительные правила block (IP) |
| `ROUTING_REMOVE_DIRECT_SITES` | нет | — | Правила, вырезаемые из direct (сайты), через запятую. Пример: `geosite:whitelist` |
| `ROUTING_REMOVE_DIRECT_IP` | нет | — | Правила, вырезаемые из direct (IP) |
| `ROUTING_REMOVE_PROXY_SITES` | нет | — | Правила, вырезаемые из proxy (сайты) |
| `ROUTING_REMOVE_PROXY_IP` | нет | — | Правила, вырезаемые из proxy (IP) |
| `ROUTING_REMOVE_BLOCK_SITES` | нет | — | Правила, вырезаемые из block (сайты) |
| `ROUTING_REMOVE_BLOCK_IP` | нет | — | Правила, вырезаемые из block (IP) |
| `SQUAD_N_EXTRA_*` | нет | — | То же, что `ROUTING_EXTRA_*`, но для конкретного сквада (например `SQUAD_1_EXTRA_DIRECT_SITES`) |
| `SQUAD_N_REMOVE_*` | нет | — | То же, что `ROUTING_REMOVE_*`, но для конкретного сквада удаление применяется **до** добавления extras. Оно нужно, когда апстримный роутинг ссылается на коды из кастомных geosite/geoip (например `geosite:twitch-ads`, `geosite:whitelist`, `geosite:torrent`, `geoip:direct`): клиенты со стандартными базами такие коды не резолвят и падают с ошибкой `failed to check code TWITCH-ADS from geosite.dat > failed to build routing configuration`. |
| `GEO_URL_MIRROR` | нет | — | Зеркало для geo-баз: подменяет хост в `Geoipurl` и `Geositeurl`, остаток пути сохраняется. Может включать путь (`https://mirror.example.com/jsd/`). Общая для настроек подписки и всех сквадов |
| `ROUTING_NAME` | нет | — | Название роутинга (поле `Name`, показывается в Happ). Заменяет значение из GitHub-конфига |
| `SQUAD_N_NAME` | нет | — | То же, что `ROUTING_NAME`, но для конкретного сквада |

### Режим запуска: интервал или расписание

Сервис поддерживает два взаимоисключающих режима проверки:

- **Интервал** (по умолчанию) — проверка каждые `CHECK_INTERVAL` секунд.
- **Расписание** — если задан `CRON_SCHEDULE` (cron-выражение), проверка идёт по расписанию, а `CHECK_INTERVAL` игнорируется. Дополнительно одна проверка выполняется сразу при старте, чтобы не ждать первого срабатывания после рестарта или деплоя.

**Зачем расписание.** Списки роутинга в [roscomvpn-happ-routing](https://github.com/hydraponique/roscomvpn-happ-routing) пересобираются примерно раз в сутки (автосборкой по утрам UTC). При интервале в 5 минут это ≈ 288 запросов к GitHub в сутки ради одного реального изменения. `CRON_SCHEDULE` позволяет синхронизироваться один раз — например, утром после пересборки — и сократить число запросов в сотни раз:

```env
# каждый день в 09:00 (часовой пояс контейнера, по умолчанию UTC)
CRON_SCHEDULE=0 9 * * *
```

> Часовой пояс берётся из контейнера (по умолчанию UTC). Для другого пояса задайте переменную `TZ`, например `TZ=Europe/Moscow`.

### Внешние сквады

Для каждого сквада задаётся пара переменных с порядковым номером:

```env
SQUAD_1_UUID=your-first-squad-uuid-here
SQUAD_1_URL=https://raw.githubusercontent.com/.../SQUAD1.DEEPLINK

SQUAD_2_UUID=your-second-squad-uuid-here
SQUAD_2_URL=https://raw.githubusercontent.com/.../SQUAD2.DEEPLINK
```

Количество сквадов не ограничено. Если переменные не заданы — синхронизируются только настройки подписки.

### Дополнительные правила роутинга

Помимо синхронизации с GitHub, можно добавить свои правила в поля `DirectSites`, `DirectIp`, `ProxySites`, `ProxyIp`, `BlockSites` и `BlockIp`. Значения из переменных окружения **дописываются** к спискам из GitHub (дубликаты игнорируются).

```env
# direct: сайты и IP
ROUTING_EXTRA_DIRECT_SITES=geosite:my-bank,geosite:local-service
ROUTING_EXTRA_DIRECT_IP=geoip:direct

# для конкретного сквада
SQUAD_1_EXTRA_DIRECT_SITES=geosite:corp-internal
```

### Название роутинга

Поле `Name` из GitHub-конфига (например `RoscomVPN`) можно заменить на своё — оно показывается пользователю в приложении Happ:

```env
ROUTING_NAME=Березка VPN
SQUAD_1_NAME=Березка VPN
```

### Зеркало для geo-баз

Апстримный конфиг раздаёт `geoip.dat` / `geosite.dat` через jsDelivr, который доступен не везде.
`GEO_URL_MIRROR` подменяет хост в полях `Geoipurl` и `Geositeurl`, сохраняя остаток пути — версия баз
(`@202609120752`) и структура путей продолжают приходить из апстрима, меняется только источник раздачи:

```env
GEO_URL_MIRROR=https://mirror.example.com/jsd/
```

```
было:  https://cdn.jsdelivr.net/gh/hydraponique/roscomvpn-geoip@202609120752/release/geoip.dat
стало: https://mirror.example.com/jsd/gh/hydraponique/roscomvpn-geoip@202609120752/release/geoip.dat
```

Зеркало может включать путь (`https://mirror.example.com/jsd/`), хвостовой слэш не обязателен.
Подменяется любой хост из апстрим-конфига, не только `cdn.jsdelivr.net`, — так подмена не сломается,
если апстрим переедет на другой CDN. Переменная одна на весь сервис: варианта `SQUAD_N_GEO_URL_MIRROR` нет,
значение применяется и к настройкам подписки, и ко всем сквадам.

## Логи

```bash
docker compose logs -f
```

## Лицензия

MIT

## Форк: отличия от оригинала

Оригинал — [lifeindarkside/Remnawave-Routing-update](https://github.com/lifeindarkside/Remnawave-Routing-update).
`upstream/main` влит, поддержка Remnawave 3.0.0+ (заголовок `routing`) на месте — прежнее
ограничение «не мерджить upstream» снято.

Проверить расхождение:

```bash
git remote add upstream https://github.com/lifeindarkside/Remnawave-Routing-update.git
git fetch upstream && git log --oneline HEAD..upstream/main
```

### Что добавлено в форке

- `ROUTING_NAME` / `SQUAD_N_NAME` — подмена поля `Name` в деплинке.
- `ROUTING_EXTRA_*` — добавление своих правил в списки (`DirectSites`, `DirectIp` и др.).
- `ROUTING_REMOVE_*` — вырезание правил апстрима. Нужно потому, что апстрим использует коды
  из кастомных баз roscomvpn (`geosite:twitch-ads`, `geosite:whitelist`, `geosite:torrent`,
  `geoip:direct`), которых нет в стандартных базах — клиенты на них не стартуют.
- `GEO_URL_MIRROR` — подмена хоста в `Geoipurl`/`Geositeurl` на своё зеркало. Нужно потому, что
  апстрим раздаёт `geoip.dat` / `geosite.dat` через jsDelivr, доступный не везде. Остаток пути
  сохраняется, поэтому версия баз продолжает приходить из апстрима.
- Перечитывание заголовков перед записью. Оригинал держит снимок заголовков со старта контейнера
  и отправляет его обратно при каждом обновлении, затирая всё, что админ поменял в панели позже
  (в режиме `CRON_SCHEDULE` снимок живёт сутками). Форк обновляет снимок непосредственно перед PATCH.
- Сравнение роутинга по содержимому, а не по строке: деплинк декодируется из base64 и сравнивается
  как JSON, поэтому разница в паддинге или порядке ключей не вызывает лишних записей в панель.

Деплой на проде — локальной сборкой (`build: .` в docker-compose.yml), образы из реестра
не используются.
