# architecture-black_friday

Финальное решение: приложение pymongo-api + MongoDB с шардированием (2 шарда) и репликацией (по 3 реплики в каждом шарде) + кеширование в Redis.
Решение находится в папке `sharding-repl-cache`.

## Запуск
```bash
cd sharding-repl-cache
docker compose up -d
```

## Инициализация MongoDB
```bash
# 1. Config server
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({_id: "config_server", configsvr: true, members: [{_id: 0, host: "configSrv:27017"}]})
EOF

# 2. Репликация shard1
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({_id: "shard1", members: [{_id: 0, host: "shard1-1:27018"}, {_id: 1, host: "shard1-2:27018"}, {_id: 2, host: "shard1-3:27018"}]})
EOF

# 3. Репликация shard2
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.initiate({_id: "shard2", members: [{_id: 0, host: "shard2-1:27019"}, {_id: 1, host: "shard2-2:27019"}, {_id: 2, host: "shard2-3:27019"}]})
EOF

# 4. Подождать 10–15 секунд, затем роутер: шарды, шардирование, 1000 документов
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018")
sh.addShard("shard2/shard2-1:27019,shard2-2:27019,shard2-3:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", {"name": "hashed"})
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({age: i, name: "ly" + i})
EOF
```

## Проверка
- http://localhost:8080 показывает общее число документов, шарды с репликами и `"cache_enabled": true`.
- Число документов в шардах и число реплик:
```bash
docker compose exec -T shard1-1 mongosh --port 27018 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments()'
docker compose exec -T shard2-1 mongosh --port 27019 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments()'
docker compose exec -T shard1-1 mongosh --port 27018 --quiet --eval 'rs.status().members.length'
docker compose exec -T shard2-1 mongosh --port 27019 --quiet --eval 'rs.status().members.length'
```
- Кеш: повторный запрос выполняется быстрее 100 мс.
```bash
curl -s -o /dev/null -w "%{time_total}\n" http://localhost:8080/helloDoc/users
curl -s -o /dev/null -w "%{time_total}\n" http://localhost:8080/helloDoc/users
```

## Структура
- `mongo-sharding`: задание 2, шардирование
- `mongo-sharding-repl`: задание 3, репликация
- `sharding-repl-cache`: задание 4, кеширование (финальное решение)
- `Task1`–`Task6`: схемы draw.io. **Итоговая схема:** `Task6/sharding-repl-cache-gateway-cdn.drawio`
