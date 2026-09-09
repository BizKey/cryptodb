# cryptodb

Репозиторий базы данных **PostgreSQL** проекта *cryptodb*, которая поднимается
в **Docker** и настроена под маломощный хост.

> **Ограничения целевой машины:** 1 ядро CPU · 1 ГБ RAM · 10 ГБ диска.

> **Статус:** Docker-окружение PostgreSQL с тюнингом под слабый хост.
> Миграции и схема БД ведутся **в отдельном репозитории**; прикладного кода здесь нет.

## Структура репозитория

```
cryptodb/
├── docker-compose.yml     # подъём PostgreSQL в Docker (с ограничениями ресурсов)
├── .editorconfig          # единые настройки редактора
├── .env.example           # шаблон переменных окружения
├── .gitignore
├── README.md
└── db/
    ├── init/              # SQL-скрипты, выполняемые при ПЕРВОМ создании тома данных
    └── postgresql.conf    # тюнинг PostgreSQL под 1 ядро / 1 ГБ RAM
```

## Быстрый старт (Docker)

Требуется Docker с compose-плагином (обычно `docker compose`).

```bash
# 1. Создать окружение и задать надёжный пароль
cp .env.example .env
#    отредактируйте .env: PGPASSWORD=<надёжный пароль> (openssl rand -hex 16)

# 2. Поднять базу (первый запуск скачает лёгкий образ postgres:*-alpine)
docker compose up -d

# 3. Проверить состояние
docker compose ps            # STATUS должен стать healthy

# 4. Подключиться (psql уже есть внутри контейнера)
docker compose exec db psql -U cryptodb -d cryptodb

# логи сервера
docker compose logs -f db    # выход — Ctrl+C
```

Сервер слушает `127.0.0.1:5432` (только на этой машине). Если БД должна быть
доступна снаружи, поменяйте в `docker-compose.yml` строку портов на
`"${PGPORT:-5432}:5432"`.

### Остановка и данные

```bash
docker compose down     # остановить; данные сохраняются в томе cryptodb_pgdata
docker compose up -d    # поднять снова — данные на месте

docker compose down -v  # остановить И удалить ВСЕ данные (необратимо!)
```

Данные лежат в именованном томе `cryptodb_pgdata` и не зависят от контейнера,
так что репозиторий можно свободно обновлять (`git pull` + `docker compose up -d`).

## Что настроено под ограничения машины

| Ограничение | Как учтено |
|---|---|
| 1 ГБ RAM | PostgreSQL-параметры под малый объём (`shared_buffers=128MB`, `max_connections=50`, `work_mem=4MB`…); контейнеру выделено не более 512 МБ (`mem_limit: 512m`) — остальное ОС и Docker |
| 1 ядро CPU | `cpus: 1.0`; `autovacuum_max_workers=2`; редкие чекпоинты (`checkpoint_timeout=15min`) — меньше фоновой работы |
| 10 ГБ диск | лёгкий образ `postgres:18-alpine`; ротация логов Docker (10 МБ × 3); ограниченный WAL (`max_wal_size=256MB`); данные — только в одном томе |

Полный список параметров — в [`db/postgresql.conf`](db/postgresql.conf) с пояснениями.

> Если контейнер убивается по памяти (в логах `docker compose logs db` ищете `OOM`),
> уменьшите в `db/postgresql.conf` `shared_buffers` до 96 МБ и выполните
> `docker compose up -d --force-recreate db`.

## PostgreSQL без Docker

Если Docker на машине недоступен, базу можно поднять вручную с тем же тюнингом:

```bash
cp .env.example .env
initdb -D ./pgdata -U "$PGUSER"          # один раз
pg_ctl -D ./pgdata -o "-p $PGPORT -c config_file=$PWD/db/postgresql.conf" start
psql -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" -d "$PGDATABASE"
```

## Скрипты инициализации (`db/init`)

SQL из этой папки выполняется автоматически **при первом старте** (пустой том
данных): расширения, стартовые данные и т. п. Подробности — в
[`db/init/README.md`](db/init/README.md).

## Схема и миграции

Миграции и описание схемы БД ведутся **в отдельном репозитории** — здесь их нет:
этот репозиторий отвечает только за запуск PostgreSQL в Docker.
Первоначальное наполнение при первом старте (расширения, стартовые данные)
делается через [`db/init`](db/init/README.md).

## Переменные окружения

Параметры описаны в [`.env.example`](.env.example). Файл `.env` (с реальными
значениями) в git **не** коммитится.

## План (roadmap)

- [x] Docker-окружение PostgreSQL с тюнингом под 1 ядро / 1 ГБ RAM / 10 ГБ диска
- [ ] По необходимости: скрипты развёртывания, Makefile

Миграции и схема — вне этого репозитория (отдельный репозиторий).
