# mongo-sharding-repl

Задание 3. Шардирование и репликация MongoDB: два шарда, в каждом три реплики (replica set из трёх узлов), один сервер конфигурации, один роутер.

## Состав стенда

| Сервис          | Роль                                                  | Порт на хосте → в контейнере |
|-----------------|-------------------------------------------------------|------------------------------|
| `configSrv`     | сервер конфигурации (replica set `config_server`)     | 27017 → 27017 |
| `shard1-1`      | шард 1, узел 1 (replica set `shard1`)                 | 27018 → 27018 |
| `shard1-2`      | шард 1, узел 2 (replica set `shard1`)                 | 27021 → 27018 |
| `shard1-3`      | шард 1, узел 3 (replica set `shard1`)                 | 27022 → 27018 |
| `shard2-1`      | шард 2, узел 1 (replica set `shard2`)                 | 27019 → 27019 |
| `shard2-2`      | шард 2, узел 2 (replica set `shard2`)                 | 27023 → 27019 |
| `shard2-3`      | шард 2, узел 3 (replica set `shard2`)                 | 27024 → 27019 |
| `mongos_router` | роутер `mongos`                                       | 27020 → 27020 |
| `pymongo_api`   | приложение, подключается к `mongos_router`            | 8080 → 8080   |

База данных — `somedb`, коллекция — `helloDoc`, ключ шардирования — `name` (hashed).

Внутри сети Docker узлы обращаются друг к другу по имени сервиса и внутреннему порту: `shard1-1:27018`, `shard2-1:27019` и т. д.

Команды ниже рассчитаны на bash (Linux, macOS, Git Bash или WSL в Windows) и выполняются из директории `mongo-sharding-repl`.

## 1. Запуск

Если запущен стенд из задания 2 (`mongo-sharding`), остановите его — он использует те же порты и имена контейнеров:

```shell
cd ../mongo-sharding && docker compose down && cd ../mongo-sharding-repl
```

Запустите стенд:

```shell
docker compose up -d --build
```

Проверьте, что запущены все девять контейнеров:

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

## 3. Настройка репликации шардов

Каждый шард — это replica set из трёх узлов. Инициализация выполняется один раз на любом узле шарда (ниже — на первом); после выборов один узел станет `PRIMARY`, два других — `SECONDARY`.

Шард 1:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1-1:27018" },
    { _id: 1, host: "shard1-2:27018" },
    { _id: 2, host: "shard1-3:27018" }
  ]
})
EOF
```

Шард 2:

```shell
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2-1:27019" },
    { _id: 1, host: "shard2-2:27019" },
    { _id: 2, host: "shard2-3:27019" }
  ]
})
EOF
```

Подождите 10–15 секунд, пока пройдут выборы, и проверьте состояние узлов. В каждом шарде должен быть один `PRIMARY` и два `SECONDARY`:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet --eval 'rs.status().members.forEach(m => print(m.name, m.stateStr))'
docker compose exec -T shard2-1 mongosh --port 27019 --quiet --eval 'rs.status().members.forEach(m => print(m.name, m.stateStr))'
```

## 4. Настройка роутера и шардирование коллекции

Шард добавляется в роутер как replica set: имя replica set и список всех его узлов.

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018")
sh.addShard("shard2/shard2-1:27019,shard2-2:27019,shard2-3:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "name": "hashed" })
EOF
```

## 5. Наполнение базы тестовыми данными

Данные вставляются через роутер, чтобы они распределились по шардам и реплицировались внутри каждого шарда:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
db.helloDoc.countDocuments()
EOF
```

Ожидаемый результат последней команды — `1000`.

## 6. Проверка

Распределение документов по шардам через роутер (в сумме 1000):

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.getShardDistribution()
EOF
```

Количество документов в каждом шарде — на первом узле шарда:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

```shell
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Репликация данных — то же количество на вторичном узле (чтение с `SECONDARY` разрешается через `readPreference`):

```shell
docker compose exec -T shard1-2 mongosh --port 27018 --quiet <<EOF
db.getMongo().setReadPref("secondaryPreferred")
use somedb
db.helloDoc.countDocuments()
EOF
```

Количество реплик в каждом шарде — по списку шардов в роутере:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet --eval 'db.adminCommand({ listShards: 1 }).shards.forEach(s => print(s._id, s.host))'
```

Ожидаемый результат:

```
shard1 shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018
shard2 shard2/shard2-1:27019,shard2-2:27019,shard2-3:27019
```

Приложение:

- http://localhost:8080 — общее количество документов (`≥ 1000`), список шардов с их узлами (по три реплики в каждом);
- http://localhost:8080/docs — Swagger.

## Остановка

```shell
docker compose down
```

Чтобы удалить данные и пройти инициализацию заново:

```shell
docker compose down -v
```
