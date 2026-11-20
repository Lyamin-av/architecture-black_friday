# mongo-sharding

Задание 2. Шардирование MongoDB: два шарда без репликации, один сервер конфигурации, один роутер.

## Состав стенда

| Сервис          | Роль                                   | Порт  |
|-----------------|----------------------------------------|-------|
| `configSrv`     | сервер конфигурации (replica set `config_server`) | 27017 |
| `shard1`        | шард 1 (replica set `shard1`)          | 27018 |
| `shard2`        | шард 2 (replica set `shard2`)          | 27019 |
| `mongos_router` | роутер `mongos`                        | 27020 |
| `pymongo_api`   | приложение, подключается к `mongos_router` | 8080  |

База данных — `somedb`, коллекция — `helloDoc`, ключ шардирования — `name` (hashed).

Команды ниже рассчитаны на bash (Linux, macOS, Git Bash или WSL в Windows) и выполняются из директории `mongo-sharding`.

## 1. Запуск

```shell
docker compose up -d --build
```

Проверьте, что все пять контейнеров запущены:

```shell
docker compose ps
```

## 2. Инициализация сервера конфигурации

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF
```

## 3. Инициализация шардов

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [{ _id: 0, host: "shard1:27018" }]
})
EOF
```

```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [{ _id: 0, host: "shard2:27019" }]
})
EOF
```

Подождите 5–10 секунд, чтобы узлы стали `PRIMARY`.

## 4. Настройка роутера и шардирование коллекции

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "name": "hashed" })
EOF
```

## 5. Наполнение базы тестовыми данными

Данные вставляются через роутер, чтобы они распределились по шардам:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
db.helloDoc.countDocuments()
EOF
```

Ожидаемый результат последней команды — `1000`.

## 6. Проверка

Количество документов в каждом шарде (в сумме должно получиться 1000):

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Распределение по шардам одним запросом через роутер:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.getShardDistribution()
EOF
```

Приложение:

- http://localhost:8080 — общее количество документов и список шардов;
- http://localhost:8080/docs — Swagger.

## Остановка

```shell
docker compose down
```

Чтобы удалить данные и пройти инициализацию заново:

```shell
docker compose down -v
```
