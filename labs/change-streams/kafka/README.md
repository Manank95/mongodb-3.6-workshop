# MongoDB Change Streams - Node with Apache Kafka Lab

This example application uses the new MongoDB 3.6 change streams feature to send messages to a Kafka broker. These messages are consumed and displayed by a separate web application. The application does the following:

- Inserts time-series stock ticker data into a MongoDB collection
- Listens to change stream events on the collection using `collection.watch()`
- Sends messages to a Kafka broker
- Consumes messages with a Kafka consumer
- Displays the stock price information in a web application running on http://localhost:3000/

## Requirements

This application was built using versions of the following software:

- [MongoDB version 3.6.x (or later)](https://www.mongodb.com/download-center#production)
- [MongoDB Node.js driver version 3.x (or later)](https://www.npmjs.com/package/mongodb)
- [Kafka version 1.x](https://kafka.apache.org/downloads) (or later) or [Confluent Open Source 4.x](https://www.confluent.io/download/) (or later)
- [Node.JS](https://nodejs.org) version 8.9.0 (or later)
- NPM version 5.5.1 (or later)

## Setup & Run

Install Node.js packages:

```npm install```

[Start Zookeeper (optional) and Kafka](https://kafka.apache.org/quickstart) with Kafka listening on `localhost:9092`. You may also use Confluent with the default command `confluent start`.

If necessary, edit configuration options in [config.js](config.js) s

1. Make sure [MongoDB 3.6+](https://www.mongodb.com/download-center#production) is installed and your machine and the MongoDB installation folder (containing tools such as `mongo` and `mongod`) is added to your local path.
1. From a Terminal/Bash console, navigate to the `shell` directory using `cd labs/change-streams/kafka`.
1. Run `sh startRS.sh` to start a test, single-node replica set (in the `/data/rs1/db` sub-folder).
1. If you've started your replica set for the first time, run `sh initiateRS.sh` (in a separate bash console) to initialize your replica set.

In the `bin` folder of your Kafka installation, run the following to create the `market-data` topics in 3 partitions (one for each of the 3 stocks we're tracking - MSFT, GOOG and FB):

    kafka-topics.sh --create --zookeeper localhost:2181 --topic market-data --replication-factor 1 --partitions 3

If you use Confluent, you should type instead:

    kafka-topics --create --zookeeper localhost:2181 --topic market-data --replication-factor 1 --partitions 3


Start the Kafka Producer:

```npm run producer```

Start the Consumer web application:

```npm run server```

Visit http://localhost:3000 to watch data.

## Code Overview

The file [loadFiles.js](loadFiles.js) reads from JSON data files and inserts into a MongoDB collection at a given interval. Because this is time-series data, each document is structured in a nested format to optimize retrieval. The `_id` key is the combination of the stock symbol and the current day.

```json

{
    "_id" : {
        "symbol" : "MSFT",
        "day" : ISODate("2017-11-16T00:00:00Z")
    },
    "hours" : {
        "11" : {    // Hour
            "57" : {    // Minute
                "58" : {    // Second
                    "305" : {   // Millisecond
                        "open" : "78.5800",
                        "high" : "78.6200",
                        "low" : "78.5400",
                        "close" : "78.5800",
                        "volume" : "236900",
                        "date" : "2017-11-16T11:57:58.305"
                    }, ...
                }, ...
            }, ...
        }, ...
    }
}

```

The change stream documents from MongoDB take the following format. We will use the `symbol` from the `documentKey._id` to map to a Kafka partition, where each stock symbol has its own partition. We will parse the `updatedFields` as the body of the message sent to Kafka, which is later consumed by our web application.

```json

{
  "_id": {
    "_data": "gloN1/UAAAAGRkZfaWQARjxzeW1ib2wAPE1TRlQAeGRheQB4gAABX8IgCAAAAFoQBOHWRLjzyEvutTsXq0MfFjsE"
  },
  "operationType": "update",
  "ns": {
    "db": "market",
    "coll": "prices"
  },
  "documentKey": {
    "_id": {
      "symbol": "MSFT",
      "day": "2017-11-16T00:00:00.000Z"
    }
  },
  "updateDescription": {
    "updatedFields": {
      "hours.11.57.58.305": {
        "open" : "78.5800",
        "high" : "78.6200",
        "low" : "78.5400",
        "close" : "78.5800",
        "volume" : "236900",
        "date" : "2017-11-16T11:57:58.305"
      }
    },
    "removedFields": []
  }
}

```

The following excerpt from [kafkaProducer.js](kafkaProducer.js) uses change streams to send messages to a Kafka broker. The function `getMessageFromChange`, parses the change stream event into a message for Kafka. This includes the partition of the symbol, the key (date), and value (stock symbol and closing price).

```javascript

    producer.on('ready', function() {

        // Create change stream that responds to updates, inserts, and replaces.
        collection.watch([{
            $match: {
                operationType: { $in: ['update', 'insert', 'replace'] }
            }
        }]).on('change', function(c) {

            // Parse data and add fields
            getMessageFromChange(c).then(message => {

                // Send to Kafka broker
                producer.send([{
                    topic: config.kafka.topic,
                    partition: message.partition,
                    messages: new kafka.KeyedMessage(message.key, JSON.stringify(message.value, null, 2))
                }], function (err, data) {
                    if (err) console.log(err);
                });

            }).catch(err => {
                console.log("Unable to parse change event: ", err);
            });

        });
    });

```

Credits go to [Louis Williams](https://github.com/louiswilliams) for the [original version](https://github.com/louiswilliams/mongodb-kafka-changestreams) of this demo application.