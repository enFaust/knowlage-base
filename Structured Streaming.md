Это [[Cтруктурированные API|высокоуровневый API]] для потоковой обработки данных. Добавился в версии Spark 2.2. Он позволяет использовать те же операции, которые были в batch через Spark Structured API и запускать их в потоковом режиме. Что позволяет обрабатывать данные инкерментально.

Это работает благодаря инкрементальной обработки данных. 

Для примера возьмем датасет розничных продаж, в котором есть точная дата. Мы будем использовать набор файлов "по дням". Один файл соответствует одному дню.
```cvm
InvoiceNo,StockCode,Description,Quantity,InvoiceDate,UnitPrice,CustomerID,Country
536365,85123A,WHITE HANGING HEART T-LIGHT HOLDER,6,2010-12-01 08:26:00,2.55,17...
536365,71053,WHITE METAL LANTERN,6,2010-12-01 08:26:00,3.39,17850.0,United Kin...
536365,84406B,CREAM CUPID HEARTS COAT HANGER,8,2010-12-01 08:26:00,2.75,17850...
```


Сначала проанализируем данные как статический набор и создадим [[DataFrame]]. Так же, нам нужна schema.

```Scala
val staticDataFrame = spark.read.format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("/data/retail-data/by-day/*.csv")

staticDataFrame.createOrReplaceTempView("retail_data")

val staticSchema = staticDataFrame.schema

```


```Python
staticDataFrame = spark.read.format("csv")\
    .option("header", "true")\
    .option("inferSchema", "true")\
    .load("/data/retail-data/by-day/*.csv")

staticDataFrame.createOrReplaceTempView("retail_data")
staticSchema = staticDataFrame.schema

```

Так как мы работаем с временным рядом, нам надо агрегировать и группировать данные.
Мы посмотрим на часы продаж, в которые конкретный клиент (по `CustomerID`) делает крупные покупки.

Добавим столбец общей стоимости и посмотрим, в какие дни, клиент потратил больше всего. Функция `window` включит в агрегацию все данные за каждый день.

```Scala
import org.apache.spark.sql.functions.{window, column, desc, col}

staticDataFrame
  .selectExpr(
    "CustomerId",
    "(UnitPrice * Quantity) as total_cost",
    "InvoiceDate"
  )
  .groupBy(
    col("CustomerId"),
    window(col("InvoiceDate"), "1 day")
  )
  .sum("total_cost")
  .show(5)

```

```python
from pyspark.sql.functions import window, column, desc, col

staticDataFrame\
  .selectExpr(
    "CustomerId",
    "(UnitPrice * Quantity) as total_cost",
    "InvoiceDate")\
  .groupBy(
    col("CustomerId"),
    window(col("InvoiceDate"), "1 day"))\
  .sum("total_cost")\
  .show(5)
```
Пример вывода:
```text
+----------+--------------------+------------------+
|CustomerId| window             | sum(total_cost)  |
+----------+--------------------+------------------+
|  17450.0 |[2011-09-20 00:00...|          71601.44|
...
|      null|[2011-12-08 00:00...|31975.590000000007|
+----------+--------------------+------------------+
```

`null` в `CustomerId` означает, что для некоторых транзакций нет сustomer Id

Это статический вариант с [[DataFrame]]

Теперь посмортим на код в стриминге. Мы используем `readStream` вместо `read`. И появляется опция `maxFilesPerTrigger`, которая просто задает, сколько файлов надо читать за 1 триггер.

```Scala
val streamingDataFrame = spark.readStream
  .schema(staticSchema)
  .option("maxFilesPerTrigger", 1)
  .format("csv")
  .option("header", "true")
  .load("/data/retail-data/by-day/*.csv")
```

```Python
streamingDataFrame = spark.readStream\
    .schema(staticSchema)\
    .option("maxFilesPerTrigger", 1)\
    .format("csv")\
    .option("header", "true")\
    .load("/data/retail-data/by-day/*.csv")
```

Теперь проверим, является ли `DataFrame` стриминговым

```scala
streamingDataFrame.isStreaming // вернёт true
```

