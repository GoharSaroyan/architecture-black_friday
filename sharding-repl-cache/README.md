# sharding-repl-cache

## Запуск
```bash
cd sharding-repl-cache
docker compose up -d
```

## Инициализация
1. Config server:
```bash
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({_id: "config_server", configsvr: true, members: [{_id: 0, host: "configSrv:27017"}]})
EOF
```
2. Репликация shard1 (3 реплики):
```bash
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({_id: "shard1", members: [
  {_id: 0, host: "shard1-1:27018"},
  {_id: 1, host: "shard1-2:27018"},
  {_id: 2, host: "shard1-3:27018"}
]})
EOF
```
3. Репликация shard2 (3 реплики):
```bash
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.initiate({_id: "shard2", members: [
  {_id: 0, host: "shard2-1:27019"},
  {_id: 1, host: "shard2-2:27019"},
  {_id: 2, host: "shard2-3:27019"}
]})
EOF
```
4. Подождать 10–15 секунд, пока выберутся primary. Затем роутер: добавляем шарды, включаем шардирование и заливаем данные:
```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018")
sh.addShard("shard2/shard2-1:27019,shard2-2:27019,shard2-3:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", {"name": "hashed"})
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({age: i, name: "ly" + i})
db.helloDoc.countDocuments()
EOF
```

## Проверка
Открыть http://localhost:8080: там видно общее число документов, количество в каждом шарде и число реплик.
Статус реплик можно посмотреть так:
```bash
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.status().members.map(m => m.name + " " + m.stateStr)
EOF
```

Количество документов в каждом шарде:
```bash
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Количество реплик в каждом шарде:
```bash
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.status().members.length
EOF
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.status().members.length
EOF
```

## Проверка кеширования
На http://localhost:8080 должно быть `"cache_enabled": true`.

Первый запрос к `/helloDoc/users` идёт около 1 секунды, повторные берутся из кеша и выполняются быстрее 100 мс:
```bash
curl -s -o /dev/null -w "%{time_total}\n" http://localhost:8080/helloDoc/users
curl -s -o /dev/null -w "%{time_total}\n" http://localhost:8080/helloDoc/users
```
