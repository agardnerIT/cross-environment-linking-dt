# cross-environment-linking-dt
Imagine 2 organisations: `organisation 1` and `organisation 2`. Both organisations share an end-to-end transaction that flows from `org1` across two services (`serviceA` and `serviceB`) into `org2` (`serviceC` and `serviceD`).

All services are instrumented with OpenTelemetry so the end-to-end flow is well tracked and automatically (thanks to OTel) has a single consistent trace ID across all 4 services.

<img width="817" height="222" alt="image" src="https://github.com/user-attachments/assets/9893839e-8bcd-404a-a08a-1d8e33a55951" />

## The Challenge

* `org1` wants to store **only** their part of the transaction (data sovereignty and cost)
* `org2` also only wants to store their part of the transaction (data sovereignty and cost)
* `org1` users need to be able to "drill down" into `org2` transactions
* `org2` users need to be able to "drill up" into transactions that originated in `org1`


## The Architecture

<img width="817" height="344" alt="image" src="https://github.com/user-attachments/assets/34c4198a-256f-4fea-a0b0-ed4ea0edb04f" />

* `org1` and `org2` each have their own Dynatrace environment
* The spans from `serviceA` and `serviceB` flow through an OpenTelemetry collector and into the `org1` environment
* Spans from `serviceC` and `serviceD` flow through a different OpenTelemetry collector into the `org2` environment
* Importantly, for any given individual transaction, spans for `serviceA`, `serviceB`, `serviceC` and `serviceD` ALL carry the same trace ID meaning Dynatrace can represent all 4 spans as a single end-to-end transaction.


* `serviceA` will be running on `http://localhost:9000`
* `serviceB` will be running on `http://localhost:9001`
* `serviceC` will be running on `http://localhost:9002`
* `serviceD` will be running on `http://localhost:9003`

## The Hands On

First make sure Docker is running: `docker ps` should work.

Now make a note of the `org1` environment ID (eg. `abc12345`) and the `org2` environment ID (eg. `def23456`).

Now grab this code, create a Python virtual environment (to keep your local machine clean) and activate it.

```
git clone https://github.com/agardnerIT/cross-environment-linking-dt
cd cross-environment-linking-dt
python -m venv .venv
source bin/activate
```

> [!IMPORTANT]  
> Dynatrace acts differently here depending on when your account was created. Read this section carefully!

In each environment, create an API token. In Dynatrace, press `Ctrl + k` and search for `api tokens` and you should see the `Access tokens` app. Open it.

<img width="218" height="120" alt="image" src="https://github.com/user-attachments/assets/da22ffdf-8490-4f8d-b78f-bf96c8fb8fcc" />

Create an API token with `openTelemetryTrace.ingest` permissions.

> [!TIP]
> "Access tokens" may not bring back any results.
> If so, go to `https://myaccount.dynatrace.com/platformTokens` and use the permission: `openpipeline:traces:ingest`

Create a `.env` file from the `.env.sample` template and enter your values. Your `.env` should look something like this:


```
export DT_ENV1_ID=abc12345
export DT_ENV2_ID=def23456

// May start with dt0c01 or dt0s16 (either are correct)
export DT_ENV1_INGEST_TOKEN=dt0s16.TODO.SAMPLE
export DT_ENV2_INGEST_TOKEN=dt0s16.TODO.SAMPLE
```

Start the stack:

```
./run.sh
```

Start `org1` collector:

```
source .env
docker run --rm --name otelcol-org1 \
  -p 4318:4318 \
  -e DT_ENV1_ID -e DT_ENV1_INGEST_TOKEN \
  -v "$(pwd)/collector/org1-collector.yaml:/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.155.0 --config /config.yaml
```

Start `org2` collector:

```
source .env
docker run --rm --name otelcol-org2 \
  -p 4328:4328 \
  -e DT_ENV2_ID -e DT_ENV2_INGEST_TOKEN \
  -v "$(pwd)/collector/org2-collector.yaml:/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.155.0 --config /config.yaml
```

