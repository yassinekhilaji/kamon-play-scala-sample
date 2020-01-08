# Play REST API

This is the example project for [Making a REST API in Play](http://developer.lightbend.com/guides/play-rest-api/index.html).

## Appendix

### Required Configuration

To get started the following settings in the `/kamon-play-scala-sample/conf/application.conf` will need to be configured by environment variables.

1. To send trace data to New Relic using the Kamon New Relic Span Reporter you must provide an `nr-insights-insert-key`:

```yaml
kamon.newrelic {
    # A New Relic Insights API Insert Key is required to send trace data to New Relic
    # https://docs.newrelic.com/docs/apis/get-started/intro-apis/types-new-relic-api-keys#insert-key-create
    nr-insights-insert-key = ${?INSIGHTS_INSERT_KEY}
}
```

2. To run a Play Framework application in production mode you must provide an application secret key:

```yaml
# https://www.playframework.com/documentation/2.7.x/ApplicationSecret
play.http.secret.key = ${?APPLICATION_SECRET}
```

3. To send data to Kamon APM using the Kamon APM Reporter you must provide an `apm.api-key`:

```yaml
kamon.apm {
  # API Key. You can find it in the Administration section in Kamon APM https://ingestion.kamino.io/
  api-key = ${?KAMON_APM_API_KEY}
  ...
}
```

### Run without Docker

You need to download and install sbt for this application to run.

Once you have sbt installed, from the root project directory, compile the program. 

```bash
sbt compile
```

Then start up the app in development mode:

```bash
sbt run
```

Play will start up on the HTTP port at <http://localhost:9000/>.   You don't need to deploy or reload anything -- changing any source code while the server is running will automatically recompile and hot-reload the application on the next HTTP request.

### Usage

If you call the same URL from the command line, you’ll see JSON. Using [httpie](https://httpie.org/), we can execute the command:

```bash
http --verbose http://localhost:9000/v1/posts
```

and get back:

```routes
GET /v1/posts HTTP/1.1
```

Likewise, you can also send a POST directly as JSON:

```bash
http --verbose POST http://localhost:9000/v1/posts title="hello" body="world"
```

and get:

```routes
POST /v1/posts HTTP/1.1
```

### Ports

Overview of what data is exposed on what ports (docker syntax `HOST_PORT:CONTAINER_PORT`).

Port `5266`          = Kamon status page  
Ports `9000`/`9001`  = Kamon Play app  
Port `9095`          = Prometheus text format generated by kamon-prometheus reporter  
Port `9411`          = Port where the Zipkin Server is running. Trace data is sent here.  

### Run with Docker

Generate a valid [secret key](https://www.playframework.com/documentation/2.7.x/ApplicationSecret) to run the play app in production mode in the docker container:

Option 1: Generate a valid key and update the key in`/kamon-play-scala-sample/conf/application.conf`:

```bash
sbt playUpdateSecret
```

Option 2: Only generate a key:
 
```bash
sbt playGenerateSecret
```

---

Stage your app so you can run it locally without having the app packaged:

```bash
sbt stage
```

---

Once it's been staged there should be an executable at:

```bash
./kamon-play-scala-sample/target/universal/stage/bin/play-scala-rest-api-example
```

---

Generate Dockerfile at `/kamon-play-scala-sample/target/docker/stage/Dockerfile`: 

```bash
sbt docker:stage
```

---

Build and publish docker image to local registry based on Dockerfile:

```bash
sbt docker:publishLocal
```

If you run `docker images` you should see a newly created one named `play-scala-rest-api-example` with the tag `2.7.x`.

---

Run the Kamon Play sample app docker container. Note: You must set the Play `APPLICATION_SECRET`, `KAMON_APM_API_KEY`, and `INSIGHTS_INSERT_KEY` environment variables:

```bash
docker run -i -t --name kamon_play_app --rm \
  -p 5266:5266 -p 9000:9000 -p 9001:9001 -p 9095:9095 \
  -e APPLICATION_SECRET -e KAMON_APM_API_KEY -e INSIGHTS_INSERT_KEY \
  play-scala-rest-api-example:2.7.x
```

---

Attach to a bash terminal in the docker container:

```bash
docker ps
```

```bash
docker exec -it <CONTAINER_ID> /bin/bash
```

### Prometheus Metrics

[Prometheus](https://github.com/prometheus/prometheus) is a monitoring system that collects metrics at a configurable interval.

When using the [kamon-prometheus](https://github.com/kamon-io/kamon-prometheus) reporter it will expose metrics in Prometheus text format for scraping at http://localhost:9095/

```nrql
SELECT count(*) FROM Metric  WHERE clusterName = 'kamon-play-scala-sample' facet metricName since 20 minutes ago  limit 1000 
```

#### Running Locally

https://docs.newrelic.com/docs/integrations/prometheus-integrations/prometheus-docker/new-relic-prometheus-openmetrics-integration-docker

In the `config.yml` set the host you want to scrape Prometheus metrics from:

```yaml
targets:
  - description: Our sick sample app yo
    urls: ["http://localhost:9095/"]
```

```bash
docker run -it --rm \
    --name nri-prometheus \
    -e $LICENSE_KEY \
    -e VERBOSE=true \
    -v "$(pwd)/config.yaml:/config.yaml" \
    newrelic/nri-prometheus:1.1
```

### Zipkin Traces (Spans)

This section covers some Docker networking configuration for running docker containers locally with the goal of running Zipkin slim in a container listening on port `9411` and reporting Kamon trace data there.

Create create a user-defined bridge network, then connect both containers to this network. Then each container can use the other’s host name to access its ports. An additional benefit is that entities outside the network cannot access these ports. 

```bash
docker network create kamon_network
``` 

Start Zipkin Docker container on the network

```bash
docker run -d -p 9411:9411 --rm --name zipkin_slim --network kamon_network openzipkin/zipkin-slim
``` 

Start the Play app Docker container on the same network. Note: You must set the Play `APPLICATION_SECRET`, `KAMON_APM_API_KEY`, and `INSIGHTS_INSERT_KEY` environment variables:

```bash
docker run -it --rm --name kamon_play_app --network kamon_network \
  -p 9000:9000 -p 9001:9001 -p 9095:9095 -p 5266:5266 \
  -e APPLICATION_SECRET -e KAMON_APM_API_KEY -e INSIGHTS_INSERT_KEY \
  play-scala-rest-api-example:2.7.x
``` 

Inspect the network and verify that both containers are running on it

```bash
docker network inspect kamon_network
```

You should see output similar to the following

```bash
"Containers": {
    "d7fd1e81e48af8466dfd9216043b104bc69c3720d203c88869cc5d89214d4dda": {
        "Name": "kamon_play_app",
        "EndpointID": "3ad4b8b15615c6da6ce562f3683743a30c4b7d3c78760fd0e6b0d422d379c693",
        "MacAddress": "02:42:ac:19:00:03",
        "IPv4Address": "172.25.0.3/16",
        "IPv6Address": ""
    },
    "e06b0d1cac6ae3e9dff30d2f7869d13a70da8874f91d9899903a694033039a12": {
        "Name": "zipkin_slim",
        "EndpointID": "28de62f54e90cde35ebe1f4fcca289fd64f5622899f381e6b7feb49f9ec9d0a2",
        "MacAddress": "02:42:ac:19:00:02",
        "IPv4Address": "172.25.0.2/16",
        "IPv6Address": ""
    }
}
```

### Load Testing

The best way to see what Play can do is to run a load test.  We've included Gatling in this test project for integrated load testing.

Start Play in production mode, by [staging the application](https://www.playframework.com/documentation/2.5.x/Deploying) and running the play script:

```bash
sbt stage
cd target/universal/stage
./bin/play-scala-rest-api-example -Dplay.http.secret.key=some-long-key-that-will-be-used-by-your-application
```

Then you'll start the Gatling load test up (it's already integrated into the project):

```bash
sbt ";project gatling;gatling:test"
```

alternatively,

```bash
sbt project gatling
sbt gatling:test
```

For best results, start the gatling load test up on another machine so you do not have contending resources.  You can edit the [Gatling simulation](http://gatling.io/docs/2.2.2/general/simulation_structure.html#simulation-structure), and change the numbers as appropriate.

Once the test completes, you'll see an HTML file containing the load test chart:

```bash
./kamon-play-scala-sample/target/gatling/gatlingspec-1472579540405/index.html
```

That will contain your load test results.
They will be useful.
