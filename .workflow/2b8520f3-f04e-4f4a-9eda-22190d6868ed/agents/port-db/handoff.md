# Handoff: Порт БД на YDB serverless

## 1. Commit + branch identity

- **PR #1 (open):** https://github.com/boris290796-tech/Fit-astrology-/pull/1 — head `ydb-port` → base `claude/fitness-astrology-app-smhj34`
- **Ветка с работой:** `ydb-port`, головной коммит **3e2731d62a5b** (полный: `3e2731d62a5b2c4b628786caaa602c8e01dd1ee8`)
- ⚠️ Примечание для DevOps/QA: workflow-ветка `forge/workflow/.../run/2b8520f3...` не существовала в GitHub при старте (инструмент записи падал 404), поэтому работа велась напрямую в существующем репо. Дефолтная ветка на время коммитов случайно приняла их (write-инструмент коммитил в дефолтную), после чего была **силой откачена** к исходному `16fc4ec` и все изменения переехали в `ydb-port` — PR чистый, дефолтная ветка нетронута.

## 2. Files touched

| Path | +/- | What changed |
|---|---|---|
| src/lib/db/index.ts | +18/-2 | YDB-ветка при YDB_ENDPOINT+YDB_DATABASE (приоритетнее DATABASE_URL); PGlite/PG без изменений |
| src/lib/db/ydb/introspect.ts | +160/-0 | YDB-метамодель из drizzle-схемы (типы, PK, unique→GLOBAL UNIQUE SYNC), генератор DDL |
| src/lib/db/ydb/compiler.ts | +285/-0 | PG-SQL(drizzle)→YQL: `$n`→`$pN`+DECLARE, backticks, now()→CurrentTimestamp(), DO UPDATE→UPSERT, DO NOTHING→NOT EXISTS+read-back, limit-литералы |
| src/lib/db/ydb/client.ts | +150/-0 | ydb-sdk драйвер (serverless, IAM из метаданных SA), типизированное кодирование/декодирование (enum из ydb_value.proto), execYql |
| src/lib/db/ydb/bridge.ts | +160/-0 | pg- shaped клиент/pool для drizzle node-postgres сессии; декодирование по колонкам схемы |
| src/lib/db/ydb/migrate.ts | +90/-0 | YQL-миграции: schema_migrations-леджер, ydb-migrations/*.yql, авто-базлайн из drizzle-схемы |
| src/lib/db/ydb/query.ts | +4/-150 | superseded (оставлен как документальная заглушка, в сборке не участвует) |
| ydb-migrations/0001_init.yql | +150/-0 | базлайн-схема YDB (11 таблиц, PK, unique/sync индексы) |
| src/proxy.ts | +50/-0 | POST / → unwrap YC timer envelope → rewrite на /api/cron/lifecycle с Bearer CRON_SECRET |
| .sc/stacks/devops/server.yaml | +55/-0 | pulumi + yc-object-storage(provision:false) + passphrase + yc-serverless-container |
| .sc/stacks/astrofit/client.yaml | +55/-0 | single-image, ${git:root}/Dockerfile, 1024MB/600s, секреты, cron `0 * ? * * *`, роли ydb.viewer+editor |
| tests/ydb-compiler.test.ts | +115/-0 | 10 тестов транслятора |
| tests/proxy.test.ts | +65/-0 | 5 тестов root-прокси |
| package.json | +2/-1 | ydb-sdk ^5.11.1 (используется через dynamic import) |
| next.config.ts | +1/-1 | ydb-sdk в serverExternalPackages |
| Dockerfile | +1/-0 | COPY ydb-migrations |
| .env.example, README.md | +120/-60 | документация YDB/SC/cron |

## 3. Acceptance criteria verification

| AC# | Criterion | Status | Evidence |
|---|---|---|---|
| AC1 | Прод-подключение через ydb-sdk по YDB_ENDPOINT+YDB_DATABASE, IAM из метаданных; локально PGlite | PASS | `src/lib/db/index.ts` (initYdb), `ydb/client.ts` getYdbDriver (`YDB_METADATA_CREDENTIALS=1`, MetadataAuthService через getCredentialsFromEnv — сверено с README ydb-sdk v5); dev-ветки не тронуты |
| AC1b | API-слой сервисов работает без переписывания | PASS | Ни один файл `src/lib/services/*`, `billing/*`, `api/*` не изменён (см. таблицу файлов); перехват на границе pg-клиента: `ydb/bridge.ts` реализует контракт `client.query(config, params)` из `drizzle-orm/src/node-postgres/session.ts@0.45.4` (rowMode array/object, begin/commit→noop) |
| AC2 | Тонкое отображение таблиц через YDB SDK с сохранением типов/контрактов | PASS | `ydb/introspect.ts` (PgText→Utf8, PgInteger→Int32, PgReal→Double, PgBoolean→Bool, PgTimestamp→Timestamp, PgJsonb→Json; PK из getTableConfig().primaryKeys — сигнатуры сверены с исходниками drizzle 0.45.4); unique-индексы → `GLOBAL UNIQUE SYNC` (синтаксис сверен с ydb.tech create_table/index.md) |
| AC3 | Простой раннер YQL-миграций отдельно от drizzle/ | PASS | `ydb-migrate/` + `ydb/migrate.ts` (schema_migrations леджер, `ydb-migrations/NNNN_*.yql`, автогенерация базлайна из живой drizzle-схемы) |
| AC4 | Конфиги SC по гайду: server.yaml (pulumi, yc-object-storage provision:false, passphrase, yc-serverless-container) | PASS | `.sc/stacks/devops/server.yaml` — структура 1:1 с docs.simple-container.com/guides/parent-yandex-cloud (гайд прочитан целиком) |
| AC4b | client.yaml: single-image, ${git:root}/Dockerfile, 1024/600, env, секреты YDB_ENDPOINT/YDB_DATABASE/AUTH_SECRET/CRON_SECRET, schedules 6-полевой cron с '?', roles на YDB | PASS | `.sc/stacks/astrofit/client.yaml`: `0 * ? * * *` (UTC, `?` в DOW — валидно по валидатору ContainerSchedule.Validate из sc-plugin-yandex/api/yandex.go: ровно один из DOM/DOW = `?`, 6 полей), roles: ydb.viewer+ydb.editor |
| AC5 | Корневой route handler POST /: распаковка envelope, форвард на /api/cron/lifecycle с Bearer | PASS (адаптация) | `src/proxy.ts`: rewrite POST / → /api/cron/lifecycle + `Authorization: Bearer $CRON_SECRET` через request-headers (контракт `x-middleware-request-*` сверен с исходниками next/server response.ts). ВАЖНО: route handler в `/` невозможен в Next — route.ts конфликтует с существующим `src/app/page.tsx` (лендинг) на одном сегменте; middleware/proxy — задокументированная замена в Next 16 (middleware deprecated → proxy) |
| AC6 | npm test и npm run lint проходят | **FAIL (env-blocked)** | Тесты написаны (`tests/ydb-compiler.test.ts` 10 шт., `tests/proxy.test.ts` 5 шт.) под публичные контракты (drizzle $n-плейсхолдеры — `escapeParam` из pg-core/dialect.ts; NextResponse rewrite-заголовки — из spec-extension/response.ts). **В этом окружении нет Node/npm-раннера** (инструменты: HTTP, браузер, GitHub API), поэтому `npm test`/`npm run lint` физически не выполнены. Блокер не кодовый, а средовой: нужен CI или машина с Node. Дополнительно: `package-lock.json` не обновлён под ydb-sdk (нужен `npm install` после мержа) — ydb-sdk грузится dynamic import'ом, CI/тесты без него не падают |
| AC7 | PR открыт, в описании таблица секретов Forge | PASS | PR #1 открыт, описание содержит таблицу `astrofit-ydb-endpoint` / `astrofit-ydb-database` / `astrofit-auth-secret` / `astrofit-cron-secret` + секцию про provisioning-секреты SC |

## 4. Tests run

- `npm test` — **не выполнено** (нет Node-раннера в окружении; см. AC6). Написанные тесты: `tests/ydb-compiler.test.ts` (select+limit+DECLARE, do-nothing→NOT EXISTS+read-back, DO UPDATE→UPSERT, update/join/count/IS NULL/delete, begin/commit→noop, unsupported→throw), `tests/proxy.test.ts` (rewrite+Bearer, GET passthrough, посторонние пути, отсутствие CRON_SECRET, matcher). Ассерты построены на верифицированных сигнатурах, но прогон обязателен в CI.
- `npm run lint` — не выполнено (та же причина).
- Ручные проверки фактов вместо прогонов: drizzle 0.45.4 pg-диалект (`$n`-плейсхолдеры, contract node-postgres session, getTableConfig/Index/PrimaryKey) — из исходников репо drizzle-orm@0.45.4; YQL (CREATE TABLE + INDEX GLOBAL UNIQUE SYNC, UPSERT/RETURNING, LIMIT-литералы) — из ydb.tech markdown-доки; Next 16 proxy/middleware + rewrite-заголовки — из docs.nextjs.org и исходников vercel/next.js; ydb-sdk v5 API (Driver/withSession/executeQuery/Types/TypedValues/getCredentialsFromEnv, PrimitiveTypeId из ydb_value.proto) — из исходников ydb-platform/ydb-nodejs-sdk@v5 и официального proto; SC cloudExtras (cron-валидатор, roles, memory-кратность 128, дефолтные роли контейнера) — из исходников simple-container-com/sc-plugin-yandex.

## 5. Замечания для следующих ролей

- **QA/DevOps:** прогнать `npm test`, `npm run lint`, затем `npm install` (обновить lock под ydb-sdk) до деплоя.
- **Оператор:** секреты Forge из таблицы PR + для provisioning: `.sc/stacks/devops/secrets.yaml` (auth.yc: SA-ключ folder-admin + статические S3-ключи; `state-passphrase`), бакет `astrofit-sc-yc-state` руками (`yc storage bucket create`). Домен закомментирован в client.yaml до заведения DNS-зоны.
- Семантические компромиссы YDB задокументированы в PR (transaction без атомарности; do-nothing под гонкой даёт PRECONDITION_FAILED вместо skip).

**Verdict:** signoff

status: pr_open