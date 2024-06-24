# Graphite Exporter

This is a exporter for prometheus. Unlike the official exporter, it queries graphite instead of receiving graphite metrics.

You provide graphite queries to the exporter. If you call the metrics endpoint, it queries graphite and exposes the results.

## Build docker in DockerFile

```bash
 docker build -t esm_graphite-exporter .
```

## Run in Docker

```bash
docker run -d \
-v /path/to/config.yml:/config/config.yml:ro \
-v /path/to/certificate/my-cert:/etc/certs/root.cer \
-p 8080:8080 \
esm_graphite-exporter
```

or use a compose file:

```YAML
version: '3.3'

networks:
  networkname:
    external: true

services:
  graphiteexporter:
    image: esm_graphite-exporter
    networks:
      - networkname
    ports:
      - "9999:8080"
    volumes:
      - ./config.yml:/app/config/config.yml
      - ./certs/my-cert.cer:/etc/certs/root.cer
```

Use docker-compose (`docker-compose up -d`) or a stack deploy to a swarm cluster (`docker stack deploy --compose-file docker-compose.yml STACKNAME`)

## Configuration

**minimal config:**

```YAML
---
graphite:
  - name: local
    url: http://192.168.178.53:2999/

targets:
  - name: foo
    graphite: local
    query: "some.graphite.query.*"
```

**extended config:**

```YAML
---
graphite:
  - name: local
    url: http://localhost:1234/
    namespace: "graphite_exporter"
    offset: 60
  - name: external
    url: https://graphite.instance.com/1234
    ssl:
      credentials: "user:pass"
      certificate: "/etc/certs/root.cer"
      skip_tls: true

server:
  port: 9000
  endpoint: "/metrics"
  log_level: info

targets:
  - name: foo
    graphite: local
    query: "some.graphite.query.*"
    labels:
      - "key: value"
    namespace: "metric_namespace"
  - name: bar
    graphite: local
    query: "sensors.basement.dht22.temperature"
    wildcards:
      - "1: location"
      - "2: sensor_type"
  - name: foobar
    graphite: external
    namespace: foobar
```

**explanation:**

- graphite: A sequence (array) of graphite connections.
  - name: Name of the graphite connection. Give them a unique name.
  - url: URL of the graphite instance.
  - *offset*: Query graphite with the `from` set to `now-offset`, default = 60s
  - *namespace*: Default prefix for this graphite instance. The namespace will be prefixed to the metric name.
  - *labels*: Add fixed labels to your metrics
  - *ssl*: Config for SSL/TLS.
    - *credentials*: When provided, the request to graphite will be send with an Authorization header with `Basic: <token>`. The token will be an base64 encoded string of the credentials.
    - *certificate*: Set an certificate in the http client to graphite. It only supports one certificate file.
    - *skip_tls*: Set the flag `InsecureSkipVerify` for the http client.

- *server*: Settings for the local server to expose the metrics.
  - *port*: On which port to expose the endpoint.
  - *endpoint*: Set the endpoint on which the metrics are available.
  - *log_level*: Set the log level of the app. Available options: Critical > Error > Warning > Notice > Info > Debug.

- targets: Target config for the metrics to query in graphite.
  - name: A unique name of your metric. Cannot contain spaces or hyphens.
  - graphite: The name of the graphite connection to use.
  - query: The graphite query to execute. Wildcard `*` is allowed.
  - *namespace*: Default prefix for this target. The namespace will be prefixed to the metric name.
  - *labels*: Add fixed labels to your metrics. 
  - *wildcards*: Add labels based on your query. Query is split on `.`. Starts counting at 0. See [wildcard labels](#wildcard-labels) below for more info.

keys in *italics* are optional

All spaces in the names and labels will be trimmed and the remaining spaces will be replaced by an `_`

For the labels you need to use an `:` as seperator

You can also use the graphite query wildcard. The query is added to the target label.

## Result

```Go
graphite_exporter_foo{label1="value1", target="some.graphite.query.query1"} 10.0
graphite_exporter_foo{label1="value1", target="some.graphite.query.query2"} 20.0
graphite_exporter_bar{label1="value1", label2="value2", target="some.other.graphite.query"} 42.0
graphite_exporter_external_graphite{target="external.graphite.query"} 65.0
```

### Wildcard labels

For example: query `sensors.attic.dht22.*` returns the following data:

- `sensors.attic.dht22.temperature`: 18
- `sensors.attic.dht22.humidity`: 38

You might want to add some tags based on the values in the query, like: `type: <temperature|humidity>` and `location: attic`.

You can do this by using the `wildcard` setting for a Target. Specify the index and the key-name for the labels you want to create.

config:

```YAML
targets:
  - name: sensor
    graphite: local
    query: "sensors.attic.dht22.*"
    wildcards:
      - "1: location"
      - "2: sensor_type"
      - "3: type"
```

result:

```GO
graphite_exporter_sensors{location="attic", sensor_type="dht22", target="sensors.attic.dht22.humidity",type="humidity"} 36.0
graphite_exporter_pi_sensors{location="attic", sensor_type="dht22", target="sensors.attic.dht22.temperature",type="temperature"} 18.0
```
