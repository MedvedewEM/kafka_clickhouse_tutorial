# kafka_clickhouse_tutorial
Репозиторий с настройкой и краткими объяснениями по настройке Apache Kafka и Clickhouse и их интеграции.

Используемые технологии и их документации:
1. [Docker + Docker-compose](https://docs.docker.com/)
2. [Документация кафки](https://kafka.apache.org/documentation/) + самое популярное видео на ютубе по Кафке
3. [Документация кликхауса](https://clickhouse.tech/docs/ru/)

# Теоретическая часть

В данном туториале поднимем кластер Кликхауса, Кафку и настроим их взаимоотношение - создадим таблицу в Кликхаусе (точнее, придется создать несколько таблиц), которая будет консюмить Кафку. Таким образом при отправке сообщения в Кафку будет создаваться соответствующая запись в таблице Кликхауса. Кликхаус делает это "из коробки", вот [документация](https://clickhouse.tech/docs/v20.1/ru/operations/table_engines/kafka/)

Наш кластер Кликхауса будет состоять из 3-х шардов по 2 реплики на каждом шарде.

Первый шард - две реплики с именами clickhouse-01 и clickhouse-06

Второй шард - две реплики с именами clickhouse-02 и clickhouse-03

Третий шард - две реплики с именами clickhouse-04 и clickhouse-05

Все это настраивается в конфигах (файлы `./config/clickhouse_config.xml` и `./config/clickhouse_metrika.xml`).
В том числе имя нашего кластера прописано там же - `cluster`

Зачем нужны шарды и реплики?

Мне помог в этом разобраться вот [этот ответ на гитхабе](https://github.com/ClickHouse/ClickHouse/issues/2161)


# Поднятие кластера Кликхауса
Используем готовый проект - https://github.com/vejed/clickhouse-cluster

Переходим в папку, где будем заниматься настройкой всего проекта. Запускаем там команды


    git clone https://github.com/vejed/clickhouse-cluster.git # Клонируем проект к себе
    cd clickhouse-cluster # Переходим в созданную папку

Нас интересуют только:
1. файл `docker-compose.yaml`
2. папка `config`

Во-первых, почему-то в `docker-compose.yaml` закомментирована строчка в разделе `volumes` у сервиса `clickhouse-02`. Давайте расскомментируем ее, чтобы каждый сервис хранил данные на хост-машине, а не где-то в контейнере.

Во-вторых, попозже добавим сюда сервис Кафки, но пока что запустим кластер отдельно:

    docker-compose up

Возможны "ошибки" вида `clickhouse-01 | Include not found: networks` - это только варнинг, никак нам не мешает.

Для проверки, что все удачно запустилось переходим внутрь докеровского контейнера первого (любого от 01 до 06) шарда с помощью команды

    docker exec -it clickhouse-01 bash

Отсюда возможно перейти в консольного клиента Кликхауса командой (эту команду вызываем откуда угодно, кстати)

    clickhouse-client --port 9000

И, например, выполнить команды

    select * from system.databases; # получим список всех баз данных - по умолчанию должны быть default и system
    
Все будущие запросы будем выполнять на бд `default` - дефолтная (надо же!) бд, на которой выполняются все запросы, если не указать явно другую бд.

#  Добавление сервиса Кафки в `docker-compose`
Сервис Кафки в этом примере будем добавлять в ту же докероскую сеть, где и кластер Кликхауса, но в принципе можно запустить и отдельным `docker-compose`'ом Кафку и корректно указать хост для подключения из одного контейнера к другому

Добавляем сервис Кафки в `docker-compose`:
```
  kafka:
      container_name: test_kafka  # Название контейнера - будет использоваться для подключения к Кафке из контейнеров той же сети
      image: wurstmeister/kafka 
      ports:
          - "9092:9092"  # Снаружи контейнера клиенты должны подключаться по этому порту, то есть localhost:9092
      expose:
          - "11192"  # Изнутри контейнера клиенты должны подключаться по этому порту
      networks:  # В принципе тут излишне иметь две сети, но раз так в репозитории vejed'а, то не будем менять, добавим в обе сети данный контейнер
          - int-net
          - ext-net
      depends_on:
          - "clickhouse-zookeeper"
      environment:  # см. дальше описание для разъяснения параметров и их значений
          KAFKA_ZOOKEEPER_CONNECT: clickhouse-zookeeper:2181
          KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT"
          KAFKA_INTER_BROKER_LISTENER_NAME: "INSIDE"
          KAFKA_ADVERTISED_LISTENERS: "INSIDE://test_kafka:11192,OUTSIDE://localhost:9092"
          KAFKA_LISTENERS: "INSIDE://:11192,OUTSIDE://:9092"
      volumes:
          - /var/run/docker.sock:/var/run/docker.sock
```
Параметры имеют следующий смысл (все вопросы должны уйти после прочтения назначения **всех** параметров)

1. Параметр `KAFKA_ZOOKEEPER_CONNECT` нужен для коннекта к zookeeper'у. В данном случае это название его контейнера + порт, указанный в секции `ports` того же контейнера
2. Параметр `KAFKA_LISTENER_SECURITY_PROTOCOL_MAP` нужен для маппинга `имя листенера` -> `протокол подключения` (см. гугл с описанием возможных протоколов, мы используем `PLAINTEXT`). В следующих параметрах будем использовать имя листенера
3. Параметр `KAFKA_INTER_BROKER_LISTENER_NAME` нужен для указания листенера, отвечающего за внутреконтейнеровское подключение клиентов. Вообще к Кафке клиент (консюмер или продюсер) может подключиться:
    * Из того же контейнера, где развернута Кафка (вот данный параметр показывает, какой листенер отвечает за это подключение)
    * Из другого контейнера той же сети докера
    * С хост-машины, где развернуты эти контейнеры
    * Или вообще с другой машины (этот случай даже не рассматриваем)
4. Параметр `KAFKA_ADVERTISED_LISTENERS` состоит из списка элементов, перечисленных через запятую.
Каждый элемент состоит из `listener_name://host:port`. Вместо `listener_name` можно указывать сразу протокол. Это основной параметр - очень краткое описание следующее:

**Когда клиент (консюмер или продюсер) подключается к Кафке (к брокеру), то в ответ получает урл, который необходимо непосредственно использует для создания нового сообщения или для потребления существующих в брокере.**

Соответственно в нашем случае клиенты из того же контейнера (см. ниже, когда будем генерировать сообщения из самого сервиса `test_kafka`) должны ходить по адресу `INSIDE://test_kafka:11192` (именно 11192 - это порт внутри контейнера Кафки, он `expose` - то есть открыт для **внутреннего** использования).
А клиенты, например, из других контейнеров должны ходить по адресу `OUTSIDE://test_kafka:9092`

5. Параметр `KAFKA_LISTENERS` отвечает за то, какие хост:порт прослушивает Кафка на получение сообщений (лучше загуглить это).

> Точно не разбирался, но похоже нельзя указывать для `INSIDE` и `OUTSIDE` одинаковые порты, поэтому мы вынуждены были писать и `ports`, и `expose` в `docker-compose`'е

Остановим и удалим контейнеры (команда удаляет все запущенные контейнеры), запущенные в предыдущем пункте, чтобы запустить их еще раз уже с сервисом Кафки (либо можно присоединить новый сервис командой `docker-compose up -d --no-deps --build kafka`):

    docker stop $(docker container ps -aq)
    docker rm $(docker container ps -aq)
    
И запустим еще раз с помощью команды `docker-compose up` все сервисы

# Создание необходимых таблиц в Кликхаусе

Подключаемся еще раз к клиенту Кликхауса через любой шард кликхауса.

(**Кстати, подключаемся к любому шарду, но при создании таблиц из-за `ON CLUSTER cluster` таблицы будут создаваться на всех репликах**)

Сейчас будем создавать следующие таблицы *в кластере* для записи в Кликхаус через Кафку:
1. Таблица test_kafka
```
CREATE TABLE test_kafka ON CLUSTER cluster 
(
  `event_created_microtime` UInt64,
  `id` UInt64,`message` String,
  `created_date` DateTime
) ENGINE = Kafka SETTINGS 
  kafka_broker_list = 'test_kafka:11192',
  kafka_topic_list = 'test_topic',
  kafka_group_name = 'test_group',
  kafka_format = 'JSONEachRow';
```
Таблица имеет `engine = Kafka` - [документация](https://clickhouse.tech/docs/v20.1/ru/operations/table_engines/kafka/)
Она представляет собой "поток данных" Кафки. То есть в этой таблице данные не будут сохраняться надолго. Они приходят сюда из Кафки и удалятся, как только их считают.
Название топика и группы можно взять произвольное - но в следующем разделе будем указывать их при отправке сообщений.

2. test
```
CREATE TABLE test ON CLUSTER cluster
(
  `event_created_microtime` UInt64,
  `id` UInt64,
  `message` String,
  `created_date` DateTime
) ENGINE = ReplicatedReplacingMergeTree(
  '/clickhouse/tables/{shard}/city/test',
  '{replica}',
  event_created_microtime
) ORDER BY (id);
```
Таблица, в которой будут храниться все записи, полученные из Кафки. Таблица имеет [движок](https://clickhouse.tech/docs/v20.1/ru/operations/table_engines/replication/) `ReplicatedReplacingMergeTree`, поэтому автоматически Кликхаус будет реплицировать (копировать) данные на реплики того же шарда.

3. test_dist
```
CREATE TABLE test_dist ON CLUSTER cluster
(
  `event_created_microtime` UInt64,
  `id` UInt64,
  `message` String,
  `created_date` DateTime
) ENGINE = Distributed(cluster, default, test, cityHash64(id));
```
По факту таблица не создается (из-за [движка](https://clickhouse.tech/docs/v20.1/ru/operations/table_engines/distributed/))

Суть в том, чтобы можно было заселектить данные со всех шардов. То есть при селекте к этой "распределенной таблице" Кликхаус за нас сходит на все шарды (в зависимости от ключа шардирования, но в худшем случае - на все шарды) и выдаст нам общий результат, как будто эти данные были получены с одной таблицы.

4. test_kafka_view

```
CREATE MATERIALIZED VIEW test_kafka_view
  TO test_dist 
  AS SELECT * FROM test_kafka;
```

Данное представление хранит данные (так как `materialized`). При получении данных из Кафки в таблицу `test_kafka` данные появятся и в этом представлении `test_kafka_view` (а из `test_kafka` удалятся сразу). И затем вставятся в `test_dist`, что приведет к инсертам в таблицу (и все ее реплики) `test` в итоге. Про эту цепную реакцию можно прочитать точнее в движке `Distributed`.
Подробнее про представления в [доке](https://clickhouse.tech/docs/v20.1/ru/operations/table_engines/materializedview/)


Выполнив все эти команды на одном любом шарде, проверьте, что на **каждом** шарде появились эти таблицы (за исключением представления - его нельзя создать на кластере). Представление создадим только на одном шарде (этого нам хватит, но можно и на каждом создать - из-за distributed-таблицы все равно будем все данные объединять).

# Создание сообщения в Кафку

Заходим в контейнер Кафки и переходим в папку установки Кафки - там должны лежать скрипты для работы с кафкой:

    docker exec -it test_kafka bash
    cd opt/kafka/bin

Команда `ls` должна вывести примерно следующее:
```
connect-distributed.sh               kafka-producer-perf-test.sh
connect-mirror-maker.sh              kafka-reassign-partitions.sh
connect-standalone.sh                kafka-replica-verification.sh
kafka-acls.sh                        kafka-run-class.sh
kafka-broker-api-versions.sh         kafka-server-start.sh
kafka-configs.sh                     kafka-server-stop.sh
kafka-console-consumer.sh            kafka-streams-application-reset.sh
kafka-console-producer.sh            kafka-topics.sh
kafka-consumer-groups.sh             kafka-verifiable-consumer.sh
kafka-consumer-perf-test.sh          kafka-verifiable-producer.sh
kafka-delegation-tokens.sh           trogdor.sh
kafka-delete-records.sh              windows
kafka-dump-log.sh                    zookeeper-security-migration.sh
kafka-leader-election.sh             zookeeper-server-start.sh
kafka-log-dirs.sh                    zookeeper-server-stop.sh
kafka-mirror-maker.sh                zookeeper-shell.sh
kafka-preferred-replica-election.sh
```

Для создания сообщения нам нужен скрипт `kafka-console-producer.sh`, который запустим, указав хост:порт подключения к Кафке и название топика (с которым мы создали таблицу `test_kafka`):
    `kafka-console-producer.sh --broker-list test_kafka:9092 --topic test_topic`
Подождать.... появится строка с вводом сообщения `>_`

Ввести туда `{"event_created_microtime":1582099666,"id":1,"message":"test string hello","created_date":"2020-02-18 18:00:01"}` и нажать Enter

Затем в клиенте Кликхауса проверить, что в таблицах `test_dist` и `test_kafka_view` - появилась наша запись!
В таблице `test` на реплике на данном шардинге может не быть соответствующей записи, так как весь флоу был такой (допустим, представление `test_kafka_view` создали на шардинге, где лежит реплика 01):
1. Отправили сообщение в Кафку
2. Таблица `test_kafka` (так как она является консюмером) получила это запись (произошел инсерт туда)
3. Затем представление `test_kafka_view` получила также эту запись (это как раз основная задача представления - "перехватывать" все инсерты из указанной таблицы)
4. Представление отправляет эту запись в `test_dist` - в нашу "распределенную таблицу"
5. Инсерт в "распределенную таблицу" порождает инсерт в таблицу реплики **одного из шардов** (по какому принципу нам не важно, но равномерно, так как в конфиге Кликхауса у нас прописаны одинаковые весы для шардов = `1`)
6. Затем происходит инсерт в таблицу другой реплику того же шарда.

Таким образом, из 6-ти таблиц `test`, что у нас есть, запись будет только на двух из них.
