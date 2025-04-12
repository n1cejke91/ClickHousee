# Развертывание ClickHouse с использованием Docker Compose

Этот проект демонстрирует:

- Развёртывание БД ClickHouse
- Импорт тестового набора данных
- Выполнение SQL-запросов
- Оценка скорости выполнения

---

## 1.Развёртывание ClickHouse с помощью Docker Compose

Создать файл `docker-compose.yml`:

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse
    ports:
      - "8123:8123"   # HTTP-интерфейс
      - "9000:9000"   # TCP клиент
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    volumes:
      - ./data:/var/lib/clickhouse
```

Запуск:

```bash
docker-compose up -d
```

Подключение к серверу:

```bash
docker exec -it clickhouse clickhouse-client
```

---

## 2.Создание таблицы `trips`

Выполнить SQL-запрос:

```sql
CREATE TABLE trips
(
    trip_id UInt32,
    vendor_id Enum8('1' = 1, '2' = 2, '3' = 3, '4' = 4, 'CMT' = 5, 'VTS' = 6, 'DDS' = 7, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14, '' = 15),
    pickup_date Date,
    pickup_datetime DateTime,
    dropoff_date Date,
    dropoff_datetime DateTime,
    store_and_fwd_flag UInt8,
    rate_code_id UInt8,
    pickup_longitude Float64,
    pickup_latitude Float64,
    dropoff_longitude Float64,
    dropoff_latitude Float64,
    passenger_count UInt8,
    trip_distance Float64,
    fare_amount Float32,
    extra Float32,
    mta_tax Float32,
    tip_amount Float32,
    tolls_amount Float32,
    ehail_fee Float32,
    improvement_surcharge Float32,
    total_amount Float32,
    payment_type Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),
    trip_type UInt8,
    pickup FixedString(25),
    dropoff FixedString(25),
    cab_type Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),
    pickup_nyct2010_gid Int8,
    pickup_ctlabel Float32,
    pickup_borocode Int8,
    pickup_ct2010 String,
    pickup_boroct2010 String,
    pickup_cdeligibil String,
    pickup_ntacode FixedString(4),
    pickup_ntaname String,
    pickup_puma UInt16,
    dropoff_nyct2010_gid UInt8,
    dropoff_ctlabel Float32,
    dropoff_borocode UInt8,
    dropoff_ct2010 String,
    dropoff_boroct2010 String,
    dropoff_cdeligibil String,
    dropoff_ntacode FixedString(4),
    dropoff_ntaname String,
    dropoff_puma UInt16
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(pickup_date)
ORDER BY pickup_datetime;
```

---

## 3.Импорт тестовой БД

Для загрузки данных использовать команду:

```sql
INSERT INTO trips
SELECT * FROM s3(
    'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{1..2}.gz',
    'TabSeparatedWithNames',
    'trip_id UInt32,
     vendor_id Enum8('1' = 1, '2' = 2, '3' = 3, '4' = 4, 'CMT' = 5, 'VTS' = 6, 'DDS' = 7, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14, '' = 15),
     pickup_date Date,
     pickup_datetime DateTime,
     dropoff_date Date,
     dropoff_datetime DateTime,
     store_and_fwd_flag UInt8,
     rate_code_id UInt8,
     pickup_longitude Float64,
     pickup_latitude Float64,
     dropoff_longitude Float64,
     dropoff_latitude Float64,
     passenger_count UInt8,
     trip_distance Float64,
     fare_amount Float32,
     extra Float32,
     mta_tax Float32,
     tip_amount Float32,
     tolls_amount Float32,
     ehail_fee Float32,
     improvement_surcharge Float32,
     total_amount Float32,
     payment_type Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),
     trip_type UInt8,
     pickup FixedString(25),
     dropoff FixedString(25),
     cab_type Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),
     pickup_nyct2010_gid Int8,
     pickup_ctlabel Float32,
     pickup_borocode Int8,
     pickup_ct2010 String,
     pickup_boroct2010 String,
     pickup_cdeligibil String,
     pickup_ntacode FixedString(4),
     pickup_ntaname String,
     pickup_puma UInt16,
     dropoff_nyct2010_gid UInt8,
     dropoff_ctlabel Float32,
     dropoff_borocode UInt8,
     dropoff_ct2010 String,
     dropoff_boroct2010 String,
     dropoff_cdeligibil String,
     dropoff_ntacode FixedString(4),
     dropoff_ntaname String,
     dropoff_puma UInt16'
) SETTINGS input_format_try_infer_datetimes = 0;
```

> После загрузки таблица содержит более 2 миллионов строк.

---

## 4.Выполнение запросов и оценка производительности

Примеры SQL-запросов и их среднее время выполнения:

| SQL-запрос                                                                                                 | Описание                                | Время выполнения |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------|------------------|
| `SELECT count(*) FROM trips`                                                                               | Количество строк                        | ~100–150 мс      |
| `SELECT payment_type, avg(tip_amount) FROM trips GROUP BY payment_type`                                   | Средняя сумма чаевых по способу оплаты | ~150–300 мс      |
| `SELECT cab_type, count(*) FROM trips GROUP BY cab_type`                                                  | Кол-во поездок по типу такси           | ~120 мс          |
| `SELECT pickup_ntaname, avg(total_amount) FROM trips GROUP BY pickup_ntaname ORDER BY avg(total_amount) DESC LIMIT 10` | ТОП-10 районов по средней стоимости     | ~500–600 мс      |

---

## 5.Дополнительная тестовая база данных с рецептами

ClickHouse предоставляет примерную базу данных с рецептами. Для её загрузки выполнить следующие шаги:

### 5.1 Загрузка данных

```sql
CREATE TABLE recipes
ENGINE = MergeTree
ORDER BY id AS
SELECT *
FROM url('https://datasets.clickhouse.com/recipes/recipes.csv.gz')
SETTINGS input_format_allow_errors_num = 10,
         input_format_allow_errors_ratio = 0.01;
```

### 5.2 Примеры запросов

| SQL-запрос                                                                                       | Описание                                     | Время выполнения |
|--------------------------------------------------------------------------------------------------|----------------------------------------------|------------------|
| `SELECT count(*) FROM recipes`                                                                   | Общее количество рецептов                   | ~10–20 мс        |
| `SELECT category, count() FROM recipes GROUP BY category ORDER BY count() DESC LIMIT 10`         | ТОП-10 категорий                            | ~20–40 мс        |
| `SELECT name FROM recipes WHERE ingredients ILIKE '%chocolate%' LIMIT 5`                         | Рецепты с шоколадом                         | ~15–30 мс        |
| `SELECT arrayJoin(keywords) AS kw, count() FROM recipes GROUP BY kw ORDER BY count() DESC LIMIT 10` | Самые популярные ключевые слова             | ~50–100 мс       |

### 5.3 Вывод

- Таблица с рецептами демонстрирует работу с CSV-данными напрямую из URL
- Загрузка и агрегация происходят почти мгновенно
- Удобно для демонстрации полнотекстового поиска и работы с массивами

---

## Общий вывод

- ClickHouse легко разворачивается с Docker Compose
- Быстрая загрузка данных из внешних источников (например, S3)
- Мгновенное выполнение аналитических SQL-запросов
- Отличное решение для OLAP и realtime-аналитики
