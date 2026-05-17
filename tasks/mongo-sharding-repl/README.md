# Шардирование и репликация

## Запуск контейнеров

Запускаем контейнеры:

```
docker compose up -d
```

## Инициализация сервера конфигурации

Подключаемся к нужному инстансу:

```
docker exec -it configSrv mongosh --port 27017
```

Переключаемся на нужную базу данных:

```
use somedb
```

Выполняем инициализацию:

```
rs.initiate(
  {
    _id : "config_server",
    configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
```

Выходим из текущего инстанса:

```
exit()
```

## Инициализация шардов и создание реплик

### Инициализация первого шарда

Подключаемся к любому из инстансов, составляющих первый шард:

```
docker exec -it shard1rep1 mongosh --port 27018
```

Переключаемся на нужную базу данных:

```
use somedb
```

Инициализируем шард, указывая все его реплики:

```
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1rep1:27018" },
        { _id : 1, host : "shard1rep2:27021" },
        { _id : 2, host : "shard1rep3:27022" },
      ]
    }
);
```

Выходим из текущего инстанса:

```
exit()
```

### Инициализация второго шарда

Аналогичным образом производим инициализацию второго шарда:

```
docker exec -it shard2rep1 mongosh --port 27019

use somedb

rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 0, host : "shard2rep1:27019" },
        { _id : 1, host : "shard2rep2:27023" },
        { _id : 2, host : "shard2rep3:27024" },
      ]
    }
);

exit()
```

## Инициализация роутера

Подключаемся к нужному инстансу:

```
docker exec -it mongos_router mongosh --port 27020
```

При добавлении шарда указываем название набора реплик добавляемого шарда, а также хост хотя бы одной из реплик, составляющих добавляемый шард:

```
sh.addShard( "shard1/shard1rep1:27018");
sh.addShard( "shard2/shard2rep1:27019");
```

Включаем шардирование

```
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
```

Наполняем базу тестовыми данными:

```
use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

db.helloDoc.countDocuments()

exit()
```

## Проверка результатов

### Шардирование

Подключаемся к инстансу роутера:

```
docker exec -it mongos_router mongosh --port 27020
```

Переключаемся на нужную базу данных:

```
use somedb
```

Выводим информацию о шардах:

```
sh.status()
```

### Репликация шардов

Подключаемся к любому из инстансов, составляющих первый шард:

```
docker exec -it shard1rep1 mongosh --port 27018
```

Выводим информацию о текущем наборе реплик:

```
rs.status()
```

Подключаемся к любому из инстансов, составляющих второй шард:

```
docker exec -it shard2rep1 mongosh --port 27019
```

Выводим информацию о текущем наборе реплик:

```
rs.status()
```
