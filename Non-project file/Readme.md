If you're a peer reviewing my project *This is not part of a project, this is a file for Module  6 HW link*

Notebook which I use to stream data with red-panda end up with size 206.7 MB make me cant not upload it on my fork of [data-engineering-zoomcamp](https://github.com/TinChung41/data-engineering-zoomcamp/tree/main) even with the use of LFS. 
So I decided to upload it to my own 
## Question 1: Redpanda version
Now let's find out the version of redpandas.

For that, check the output of the command rpk help inside the container. The name of the container is redpanda-1.

Find out what you need to execute based on the help output.

What's the version, based on the output of the command you executed? (copy the entire version)

![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/16449f47-cda5-4c0b-ad03-f4d96a2703a1)


Question 2. Creating a topic
Before we can send data to the redpanda server, we need to create a topic. We do it also with the rpk command we used previously for figuring out the version of redpandas.

Read the output of help and based on it, create a topic with name test-topic

What's the output of the command for creating a topic? Include the entire output in your answer.
![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/057c1c9e-b17e-43a7-9193-7bd6bc57a70b)

## Question 3. Connecting to the Kafka server

We need to make sure we can connect to the server, so
later we can send some data to its topics

First, let's install the kafka connector (up to you if you
want to have a separate virtual environment for that)

```bash
pip install kafka-python
```

You can start a jupyter notebook in your solution folder or
create a script

Let's try to connect to our server:

```python
import json
import time 

from kafka import KafkaProducer

def json_serializer(data):
    return json.dumps(data).encode('utf-8')

server = 'localhost:9092'

producer = KafkaProducer(
    bootstrap_servers=[server],
    value_serializer=json_serializer
)

producer.bootstrap_connected()
```

Provided that you can connect to the server, what's the output
of the last command?
![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/015e737f-3d39-415a-9b6d-72a2ef2359bf)


## Question 4. Sending data to the stream

Now we're ready to send some test data:

```python
t0 = time.time()

topic_name = 'test-topic'

for i in range(10):
    message = {'number': i}
    producer.send(topic_name, value=message)
    print(f"Sent: {message}")
    time.sleep(0.05)

producer.flush()

t1 = time.time()
print(f'took {(t1 - t0):.2f} seconds')
```

How much time did it take? Where did it spend most of the time?

* Sending the messages
* Flushing
* Both took approximately the same amount of time

(Don't remove `time.sleep` when answering this question)

![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/827b8d3b-4321-46af-927a-bd60c18da54b)

## Reading data with `rpk`

You can see the messages that you send to the topic
with `rpk`:

```bash
rpk topic consume test-topic
```

Run the command above and send the messages one more time to 
see them
![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/3460960f-fe03-4d0e-b3af-4cf1211e59cb)


## Sending the taxi data

Now let's send our actual data:

* Read the green csv.gz file
* We will only need these columns:
  * `'lpep_pickup_datetime',`
  * `'lpep_dropoff_datetime',`
  * `'PULocationID',`
  * `'DOLocationID',`
  * `'passenger_count',`
  * `'trip_distance',`
  * `'tip_amount'`

Iterate over the records in the dataframe

```python
for row in df_green.itertuples(index=False):
    row_dict = {col: getattr(row, col) for col in row._fields}
    print(row_dict)
    break

    # TODO implement sending the data here
```
```python
topic_name = 'green-trips'

for row in df_green.itertuples(index=False):
    row_dict = {col: getattr(row, col) for col in row._fields}
    print(row_dict)


    #TODO implement sending the data here
    producer.send(topic_name, value=row_dict)
    print(f"Sent: {row_dict}")

producer.flush()

t1 = time.time()
print(f'took {(t1 - t0):.2f} seconds')
```
Note: this way of iterating over the records is more efficient compared
to `iterrows`


## Question 5: Sending the Trip Data

* Create a topic `green-trips` and send the data there
* How much time in seconds did it take? (You can round it to a whole number)
* Make sure you don't include sleeps in your code


## Creating the PySpark consumer

Now let's read the data with PySpark. 

Spark needs a library (jar) to be able to connect to Kafka, 
so we need to tell PySpark that it needs to use it:

```python
import pyspark
from pyspark.sql import SparkSession

pyspark_version = pyspark.__version__
kafka_jar_package = f"org.apache.spark:spark-sql-kafka-0-10_2.12:{pyspark_version}"

spark = SparkSession \
    .builder \
    .master("local[*]") \
    .appName("GreenTripsConsumer") \
    .config("spark.jars.packages", kafka_jar_package) \
    .getOrCreate()
```

Now we can connect to the stream:

```python
green_stream = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "green-trips") \
    .option("startingOffsets", "earliest") \
    .load()
```
![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/f7a70859-d92b-4a21-b61e-a09e98f26392)

In order to test that we can consume from the stream, 
let's see what will be the first record there. 

In Spark streaming, the stream is represented as a sequence of 
small batches, each batch being a small RDD (or a small dataframe).

So we can execute a function over each mini-batch.
Let's run `take(1)` there to see what do we have in the stream:

```python
def peek(mini_batch, batch_id):
    first_row = mini_batch.take(1)

    if first_row:
        print(first_row[0])

query = green_stream.writeStream.foreachBatch(peek).start()
```

You should see a record like this:

```
Row(key=None, value=bytearray(b'{"lpep_pickup_datetime": "2019-10-01 00:26:02", "lpep_dropoff_datetime": "2019-10-01 00:39:58", "PULocationID": 112, "DOLocationID": 196, "passenger_count": 1.0, "trip_distance": 5.88, "tip_amount": 0.0}'), topic='green-trips', partition=0, offset=0, timestamp=datetime.datetime(2024, 3, 12, 22, 42, 9, 411000), timestampType=0)
```

Now let's stop the query, so it doesn't keep consuming messages
from the stream

```python
query.stop()
```

## Question 6. Parsing the data

The data is JSON, but currently it's in binary format. We need
to parse it and turn it into a streaming dataframe with proper
columns.

Similarly to PySpark, we define the schema

```python
from pyspark.sql import types

schema = types.StructType() \
    .add("lpep_pickup_datetime", types.StringType()) \
    .add("lpep_dropoff_datetime", types.StringType()) \
    .add("PULocationID", types.IntegerType()) \
    .add("DOLocationID", types.IntegerType()) \
    .add("passenger_count", types.DoubleType()) \
    .add("trip_distance", types.DoubleType()) \
    .add("tip_amount", types.DoubleType())
```

And apply this schema:

```python
from pyspark.sql import functions as F

green_stream = green_stream \
  .select(F.from_json(F.col("value").cast('STRING'), schema).alias("data")) \
  .select("data.*")
```

How does the record look after parsing? Copy the output. 
```
24/03/23 21:01:12 WARN ResolveWriteToStream: Temporary checkpoint location created which is deleted normally when the query didn't fail: /tmp/temporary-46ab4b1c-7e48-4305-be0b-99d7d0fa2c6e. If it's required to delete it under any circumstances, please set spark.sql.streaming.forceDeleteTempCheckpointLocation to true. Important to know deleting temp checkpoint folder is best effort.
24/03/23 21:01:12 WARN ResolveWriteToStream: spark.sql.adaptive.enabled is not supported in streaming DataFrames/Datasets and will be disabled.
24/03/23 21:01:12 WARN AdminClientConfig: These configurations '[key.deserializer, value.deserializer, enable.auto.commit, max.poll.records, auto.offset.reset]' were supplied but are not used yet.
                                                                                
Row(lpep_pickup_datetime='2019-10-01 00:26:02', lpep_dropoff_datetime='2019-10-01 00:39:58', PULocationID=112, DOLocationID=196, passenger_count=1.0, trip_distance=5.88, tip_amount=0.0)
```
![image](https://github.com/TinChung41/data-engineering-zoomcamp-project/assets/98845918/7140b69d-419b-4057-8a1a-6e48eaaf0ecd)



### Question 7: Most popular destination

Now let's finally do some streaming analytics. We will
see what's the most popular destination currently 
based on our stream of data (which ideally we should 
have sent with delays like we did in workshop 2)


This is how you can do it:

* Add a column "timestamp" using the `current_timestamp` function
* Group by:
  * 5 minutes window based on the timestamp column (`F.window(col("timestamp"), "5 minutes")`)
  * `"DOLocationID"`
* Order by count

You can print the output to the console using this 
code

```python
query = popular_destinations \
    .writeStream \
    .outputMode("complete") \
    .format("console") \
    .option("truncate", "false") \
    .start()

query.awaitTermination()
```

Write the most popular destination, your answer should be *either* the zone ID or the zone name of this destination. (You will need to re-send the data for this to work)
