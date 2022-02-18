# Debezium Tutorial

Tutorial ini menunjukkan bagaimana menggunakan Debezium untuk memonitor database MySQL. Saat data dalam database berubah, Anda akan melihat aliran event yang dihasilkan.

Dalam tutorial ini Anda akan memulai Debezium, menjalankan server database MySQL dengan contoh database sederhana, dan menggunakan Debezium untuk memantau perubahan database.

Prerequisites
- Docker diinstal dan dijalankan.


## Debezium

Debezium adalah platform terdistribusi yang mengubah database Anda yang ada menjadi aliran events, sehingga aplikasi dapat melihat dan merespons dengan segera setiap perubahan tingkat baris dalam database.

Debezium dibangun di atas Apache Kafka dan menyediakan konektor yang kompatibel dengan Kafka Connect yang memantau sistem manajemen basis data tertentu. Debezium mencatat riwayat perubahan data di log Kafka, dari mana aplikasi Anda menggunakannya. Ini memungkinkan aplikasi Anda untuk dengan mudah menggunakan semua acara dengan benar dan lengkap. Bahkan jika aplikasi Anda berhenti tiba-tiba, itu tidak akan melewatkan apa pun: ketika aplikasi dimulai ulang, itu akan melanjutkan mengkonsumsi acara di mana ia tinggalkan.

Debezium mencakup beberapa konektor. Dalam tutorial ini, Anda akan menggunakan konektor MySQL.


## Start the services

1. Zookeeper

`docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.8`

2. Kafka

`docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.8`

3. MySQL

Start MySQL services:
`docker run -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.8`

Start MySQL shell:
`docker run -it --rm --name mysqlterm --link mysql --rm mysql:8.0 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'`

Run commands:
```
use inventory;

show tables;

SELECT * FROM customers;
```

4. Kafka Connect

Start service:
`docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:1.8`

Check status:
`curl -H "Accept:application/json" localhost:8083/`

Check connector status:
`curl -H "Accept:application/json" localhost:8083/connectors/`


## Mendaftarkan konektor untuk memantau database 

Dengan mendaftarkan konektor Debezium MySQL, konektor akan mulai memantau binlog server database MySQL. Binlog mencatat semua transaksi database (seperti perubahan pada baris individual dan perubahan pada skema). Ketika sebuah baris dalam database berubah, Debezium menghasilkan event perubahan.

Configuration example:
```
{
  "name": "inventory-connector",  
  "config": {  
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",  
    "database.hostname": "mysql",  
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",  
    "database.server.name": "dbserver1",  
    "database.include.list": "inventory",  
    "database.history.kafka.bootstrap.servers": "kafka:9092",  
    "database.history.kafka.topic": "schema-changes.inventory"  
  }
}
```

Configuration for this tutorial:
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d \
'{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "dbserver1",
    "database.include.list": "inventory",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "dbhistory.inventory"
  }
}'
```

To review above connector task, we can run
`curl -i -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector`


5. Watch the topic

`docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:1.8 watch-topic -a -k dbserver1.inventory.customers`



6. Update row

`UPDATE customers SET first_name='Anne Marie' WHERE id=1004;`


Topic output
```
{
  "schema": {...},
  "payload": {
    "before": {  
      "id": 1004,
      "first_name": "Anne",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "after": {  
      "id": 1004,
      "first_name": "Anne Marie",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "source": {  
      "name": "1.8.1.Final",
      "name": "dbserver1",
      "server_id": 223344,
      "ts_sec": 1486501486,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 364,
      "row": 0,
      "snapshot": null,
      "thread": 3,
      "db": "inventory",
      "table": "customers"
    },
    "op": "u",  
    "ts_ms": 1486501486308  
  }
}
```

**op**
berisi nilai string yang menjelaskan jenis operasi. Nilai untuk konektor MySQL adalah c untuk membuat (atau menyisipkan), u untuk pembaruan, d untuk menghapus, dan r untuk membaca (dalam kasus snapshot).

**before**
berisi status baris sebelum peristiwa terjadi. Strukturnya akan dijelaskan oleh skema dbserver1.inventory.customers.Value Kafka Connect, yang digunakan konektor dbserver1 untuk semua baris dalam tabel inventory.customers.

**after**
berisi status baris setelah peristiwa terjadi. Strukturnya dijelaskan oleh skema dbserver1.inventory.customers.Value Kafka Connect yang sama dengan yang digunakan sebelumnya.

**source**
berisi struktur yang menjelaskan metadata sumber untuk acara, yang dalam kasus MySQL, berisi beberapa bidang: nama konektor, nama file binlog tempat acara direkam, posisi dalam file binlog tempat acara muncul, baris dalam acara (jika ada lebih dari satu), nama database dan tabel yang terpengaruh, ID utas MySQL yang membuat perubahan, apakah acara ini merupakan bagian dari snapshot, dan, jika tersedia, ID server MySQL, dan stempel waktu dalam hitungan detik.

**ts_ms**
berisi waktu (menggunakan jam sistem di JVM yang menjalankan tugas Kafka Connect) saat konektor memproses acara.


