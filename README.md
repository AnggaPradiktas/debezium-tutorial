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


##Mendaftarkan konektor untuk memantau database 

Dengan mendaftarkan konektor Debezium MySQL, konektor akan mulai memantau binlog server database MySQL. Binlog mencatat semua transaksi database (seperti perubahan pada baris individual dan perubahan pada skema). Ketika sebuah baris dalam database berubah, Debezium menghasilkan event perubahan.
