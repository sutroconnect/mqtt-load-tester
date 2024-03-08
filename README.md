# MQTT Load Tester

A simple MQTT load testing tool.

Installation:

```sh
go install github.com/sutroconnect/mqtt-load-tester@main
```

The tool supports multiple concurrent clients, configurable message size, etc:

```sh
$ ./mqtt-load-tester -h
Usage of ./mqtt-load-tester:
  -broker string
    	MQTT broker endpoint as scheme://host:port (default "tcp://localhost:1883")
  -broker-ca-cert string
    	Path to broker CA certificate in PEM format
  -client-cert string
    	Path to client certificate in PEM format
  -client-key string
    	Path to private clientKey in PEM format
  -client-prefix string
    	MQTT client id prefix (suffixed with '-<client-num>' (default "mqtt-load-tester")
  -clients int
    	Number of clients to start (default 10)
  -count int
    	Number of messages to send per client (default 100)
  -format string
    	Output format: text|json (default "text")
  -insecure
    	Skip TLS certificate verification
  -message-interval int
    	Time interval in seconds to publish message (default 1)
  -password string
    	MQTT client password (empty if auth disabled)
  -payload string
    	MQTT message payload. If empty, then payload is generated based on the size parameter
  -qos int
    	QoS for published messages (default 1)
  -quiet
    	Suppress logs while running
  -ramp-up-time int
    	Time in seconds to generate clients by default will not wait between load request
  -size int
    	Size of the messages payload (bytes) (default 100)
  -topic string
    	MQTT topic for outgoing messages (default "/test")
  -username string
    	MQTT client username (empty if auth disabled)
  -wait int
    	QoS 1 wait timeout in milliseconds (default 60000)
```

> NOTE: if `count=1` or `clients=1`, the sample standard deviation will be returned as `0` (convention due to the [lack of NaN support in JSON](https://tools.ietf.org/html/rfc4627#section-2.4))

Two output formats supported: human-readable plain text and JSON.

## Modifications for Sutro's Use Case

To facilitate load testing, all that's been changed is adding the option for multiple payloads, separated by `^` (because `,` is already used in JSON and I was in a hurry). If multiple payloads are specified (see example of how to do this below), then the script will parse them into a list and randomly alternate between the payloads.

To specify multiple payloads, which the clients will randomly alternate between, run the command like so:

```sh
> mqtt-load-tester --broker tcp://broker.local:1883 --count 100 --clients 10 --qos 1 --topic house/bedroom/temperature --payload "{\"payload1\":true}^{\"payload2\":true}"

...

2024/03/08 12:29:00 Parsed Payloads:
2024/03/08 12:29:00 {"payload1":true}
2024/03/08 12:29:00 {"payload2":true}

...
```

## Suggested Use

To direct load testing messages to their own SQS queue, you can create an AWS IoT rule with a query that targets messages like `$aws/things/hub/loadtest/id:id:id:id/hub2aws`. Notice the `/loadtest/` section of the topic.

The rule might look something like this:

```sql
SELECT topic(5) as hub_id, parse_time("yyyyMMdd-HHmmss z", timestamp() ) as aws_timestamp, * from '$aws/things/hub/loadtest/+/hub2aws'
```

It would then forward messages to the desired queue, which the load testing environment would be configured to pull from.

```sh
go run . \
    --broker tls://broker-address.iot.us-east-2.amazonaws.com:8883 \
    --username "load-tester" \
    --count 9000000 \
    --clients 10 \
    --qos 0 \
    --message-interval 0 \
    --topic "\$aws/things/hub/loadtest/1234:1234:1234:1234/hub2aws" \
    --payload "{\"newBattStatus\":\"charging\",\"oldBattStatus\":\"complete\",\"timeStamp\":\"20240101-000000\",\"aws_timestamp\":\"20240101-000000 UTC\"}^{\"newBattStatus\":\"complete\",\"oldBattStatus\":\"charging\",\"timeStamp\":\"20240101-000000\",\"aws_timestamp\":\"20240101-000000 UTC\"}^{\"fullReading\":\"fullReadingData",\"hub_id\":\"1234:1234:1234:1234\",\"id\":\"5678:5678:5678:5678\",\"timeStamp\":\"20240101-000000\",\"aws_timestamp\":\"20240101-000000 UTC\"}" \
    --broker-ca-cert path/to/mqtt_broker_certs/root-ca.pem \
    --client-cert path/to/mqtt_broker_certs/certificate.pem.crt \
    --client-key path/to/mqtt_broker_certs/private.pem.key \
```

## General Use and Output

```sh
> mqtt-load-tester --broker tcp://broker.local:1883 --count 100 --size 100 --clients 100 --qos 2 --format text
....

======= CLIENT 27 =======
Ratio:               1 (100/100)
Runtime (s):         16.396
Msg time min (ms):   9.466
Msg time max (ms):   1880.769
Msg time mean (ms):  150.193
Msg time std (ms):   201.884
Bandwidth (msg/sec): 6.099

========= TOTAL (100) =========
Total Ratio:                 1 (10000/10000)
Total Runime (sec):          16.398
Average Runtime (sec):       15.514
Msg time min (ms):           7.766
Msg time max (ms):           2034.076
Msg time mean mean (ms):     140.751
Msg time mean std (ms):      13.695
Average Bandwidth (msg/sec): 6.761
Total Bandwidth (msg/sec):   676.112
```

With payload specified:

```sh
> mqtt-load-tester --broker tcp://broker.local:1883 --count 100 --clients 10 --qos 1 --topic house/bedroom/temperature --payload {\"temperature\":20,\"timeStamp\":1597314150}
....

======= CLIENT 0 =======
Ratio:               1.000 (100/100)
Runtime (s):         0.725
Msg time min (ms):   1.999
Msg time max (ms):   22.997
Msg time mean (ms):  6.955
Msg time std (ms):   3.523
Bandwidth (msg/sec): 137.839

========= TOTAL (1) =========
Total Ratio:                 1.000 (100/100)
Total Runtime (sec):         0.736
Average Runtime (sec):       0.725
Msg time min (ms):           1.999
Msg time max (ms):           22.997
Msg time mean mean (ms):     6.955
Msg time mean std (ms):      0.000
Average Bandwidth (msg/sec): 137.839
Total Bandwidth (msg/sec):   137.839
```

Similarly, in JSON:

```sh
mqtt-load-tester --broker tcp://broker.local:1883 --count 100 --size 100 --clients 100 --qos 2 --format json --quiet
```

```json
{
  "runs": [
    {
      "id": 61,
      "successes": 100,
      "failures": 0,
      "run_time": 16.142762197,
      "msg_tim_min": 12.798859,
      "msg_time_max": 1273.9553740000001,
      "msg_time_mean": 147.66799521,
      "msg_time_std": 152.08244221156286,
      "msgs_per_sec": 6.194726700402251
    }
  ],
  "totals": {
    "successes": 10000,
    "failures": 0,
    "total_run_time": 16.153741746,
    "avg_run_time": 15.14702422494,
    "msg_time_min": 7.852086000000001,
    "msg_time_max": 1285.241845,
    "msg_time_mean_avg": 136.4360292677,
    "msg_time_mean_std": 12.816965054355633,
    "total_msgs_per_sec": 681.0374046459865,
    "avg_msgs_per_sec": 6.810374046459865
  }
}
```
