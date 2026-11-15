# mongo-sharding

## Запуск
```bash
cd mongo-sharding
docker compose up -d
```

## Инициализация шардирования
1. Config server:
```bash
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({_id: "config_server", configsvr: true, members: [{_id: 0, host: "configSrv:27017"}]})
EOF
```
2. Шарды:
```bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({_id: "shard1", members: [{_id: 0, host: "shard1:27018"}]})
EOF
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({_id: "shard2", members: [{_id: 0, host: "shard2:27019"}]})
EOF
```
3. Роутер: добавляем шарды, включаем шардирование и заливаем 1000 документов:
```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", {"name": "hashed"})
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({age: i, name: "ly" + i})
db.helloDoc.countDocuments()
EOF
```

## Проверка
Открыть http://localhost:8080: там видно общее число документов и сколько их в каждом шарде.
Число документов в шарде можно посмотреть и вручную:
```bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```
