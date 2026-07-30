# Домашнее задание Сартисона Евгения N7 #

Домашнее задание
DCS

Цель:
развернуть отказоустойчивые кластеры Etcd и Consul и проверить их поведение при сбоях узлов.

Описание/Пошаговая инструкция выполнения домашнего задания:
Разворачиваем кластер Etcd любым способом. Проверяем отказоустойчивость


Делаем установку отказоустойчивого кластера ETCD только.



## Настройка кластера ETCD в Докере ## 


docker-compose.yml  
```
version: '3.8'

services:
  etcd-00:
    image: quay.io/coreos/etcd:v3.5.0
    hostname: etcd-00
    command:
      - etcd
      - --name=etcd-00
      - --data-dir=data.etcd
      - --advertise-client-urls=http://etcd-00:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-advertise-peer-urls=http://etcd-00:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-cluster=etcd-00=http://etcd-00:2380,etcd-01=http://etcd-01:2380,etcd-02=http://etcd-02:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=etcd-cluster-1
    volumes:
      - etcd-00vol:/data.etcd
    networks:
      - etcd
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "5"
    restart: always

  etcd-01:
    image: quay.io/coreos/etcd:v3.5.0
    hostname: etcd-01
    command:
      - etcd
      - --name=etcd-01
      - --data-dir=data.etcd
      - --advertise-client-urls=http://etcd-01:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-advertise-peer-urls=http://etcd-01:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-cluster=etcd-00=http://etcd-00:2380,etcd-01=http://etcd-01:2380,etcd-02=http://etcd-02:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=etcd-cluster-1
    volumes:
      - etcd-01vol:/data.etcd
    networks:
      - etcd
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "5"
    restart: always

  etcd-02:
    image: quay.io/coreos/etcd:v3.5.0
    hostname: etcd-02
    command:
      - etcd
      - --name=etcd-02
      - --data-dir=data.etcd
      - --advertise-client-urls=http://etcd-02:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-advertise-peer-urls=http://etcd-02:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-cluster=etcd-00=http://etcd-00:2380,etcd-01=http://etcd-01:2380,etcd-02=http://etcd-02:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=etcd-cluster-1
    volumes:
      - etcd-02vol:/data.etcd
    networks:
      - etcd
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "5"
    restart: always

  nginx:
    image: nginx:alpine
    hostname: nginx-etcd
    volumes:
      - type: bind
        source: ./nginx/nginx.conf
        target: /etc/nginx/nginx.conf
    networks:
     - etcd
    ports:
      - 2379:2379
    depends_on:
      - etcd-00
      - etcd-01
      - etcd-02
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "5"
    restart: always

volumes:
  etcd-00vol:
    driver: local
  etcd-01vol:
    driver: local
  etcd-02vol:
    driver: local

networks:
  etcd:
    driver: bridge
```


Старт кластера и проверка статуса
```
student:~/Consul/docker-etcd-cluster-master$ sudo docker-compose up -d
Pulling etcd-00 (quay.io/coreos/etcd:v3.5.0)...
v3.5.0: Pulling from coreos/etcd
4b376c64dfe4: Pull complete
1813d21adc01: Pull complete
6e96907ab677: Pull complete
444ed0ea8673: Pull complete
0fd2df5633f0: Pull complete
8cc22b9456bb: Pull complete
7ac70aecd290: Pull complete
Digest: sha256:28759af54acd6924b2191dc1a1d096e2fa2e219717a21b9d8edf89717db3631b
Status: Downloaded newer image for quay.io/coreos/etcd:v3.5.0
Creating docker-etcd-cluster-master_etcd-00_1 ... done
Creating docker-etcd-cluster-master_etcd-02_1 ... done
Creating docker-etcd-cluster-master_etcd-01_1 ... done
Creating docker-etcd-cluster-master_nginx_1   ... done
student:~/Consul/docker-etcd-cluster-master$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED         STATUS         PORTS                                                 NAMES
89e624943991   nginx:alpine                 "/docker-entrypoint.…"   6 minutes ago   Up 6 minutes   80/tcp, 0.0.0.0:2379->2379/tcp, [::]:2379->2379/tcp   docker-etcd-cluster-master_nginx_1
3cff8112b3ba   quay.io/coreos/etcd:v3.5.0   "etcd --name=etcd-02…"   6 minutes ago   Up 6 minutes   2379-2380/tcp                                         docker-etcd-cluster-master_etcd-02_1
8d8a5cee69ff   quay.io/coreos/etcd:v3.5.0   "etcd --name=etcd-00…"   6 minutes ago   Up 6 minutes   2379-2380/tcp                                         docker-etcd-cluster-master_etcd-00_1
99ca87147df1   quay.io/coreos/etcd:v3.5.0   "etcd --name=etcd-01…"   6 minutes ago   Up 6 minutes   2379-2380/tcp                                         docker-etcd-cluster-master_etcd-01_1
```



## Проверка отказоустойчивости ## 
Проверка текущего лидера
```
student:~/Consul/docker-etcd-cluster-master$ docker exec -it 8d8a5cee69ff  etcdctl endpoint status --cluster -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://etcd-02:2379 | 9772fa935aee2fcb |   3.5.0 |   20 kB |     false |      false |         2 |          8 |                  8 |        |
| http://etcd-01:2379 | dc6856a29b285c1c |   3.5.0 |   20 kB |     false |      false |         2 |          8 |                  8 |        |
| http://etcd-00:2379 | f9bb7dc32e4429fd |   3.5.0 |   20 kB |      true |      false |         2 |          8 |                  8 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
Текущий лидер под 8d8a5cee69ff и endpoint etcd-00.


останавливаем POD с лидером
```
student:~/Consul/docker-etcd-cluster-master$ docker stop 8d8a5cee69ff
8d8a5cee69ff
```

Проверяем еще раз статус
```
student:~/Consul/docker-etcd-cluster-master$ docker exec -it 3cff8112b3ba etcdctl endpoint status --cluster -w table
{"level":"warn","ts":"2026-07-30T16:40:28.235Z","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc0002d8a80/#initially=[127.0.0.1:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Failed to get the status of endpoint http://etcd-00:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://etcd-02:2379 | 9772fa935aee2fcb |   3.5.0 |   20 kB |     false |      false |         3 |          9 |                  9 |        |
| http://etcd-01:2379 | dc6856a29b285c1c |   3.5.0 |   20 kB |      true |      false |         3 |          9 |                  9 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
Лидер переехал на etcd-01.

Все отработало!


