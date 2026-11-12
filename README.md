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

The collector YAMLs contain placeholders so replace them with real Dynatrace environment values now (this will be explained in more detail later):

```
source .env
python replace_placeholders.py
```

Open four terminal windows.

In the first terminal, start the stack:

```
./run.sh
```

In a second terminal, start `org1` collector:

```
source .env
docker run --rm --name otelcol-org1 \
  -p 4318:4318 \
  -e DT_ENV1_ID -e DT_ENV1_INGEST_TOKEN \
  -v "$(pwd)/collector/org1-collector.yaml:/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.155.0 --config /config.yaml
```

In a third terminal, start `org2` collector:

```
source .env
docker run --rm --name otelcol-org2 \
  -p 4328:4328 \
  -e DT_ENV2_ID -e DT_ENV2_INGEST_TOKEN \
  -v "$(pwd)/collector/org2-collector.yaml:/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.155.0 --config /config.yaml
```

## Run Traffic

In a fourth terminal, send a request to `serviceA`:

```
curl http://localhost:9000
```

You should see this response, proving the request has hit all 4 services:

```
{"path":["servicea","serviceb","servicec","serviced"]}
```

## View Data in Dynatrace

Open the Dynatrace environment for `organisation 1` and press `ctrl + k`. Search for `distributed traces` and open the `Distributed Tracing` app.

<img width="218" height="120" alt="image" src="https://github.com/user-attachments/assets/b489ca91-d253-4b32-9f88-0520d2569971" />

Optional: Open the search facets (left hand panel) and filter for `servicea` and `serviceb`.

<img width="985" height="117" alt="image" src="https://github.com/user-attachments/assets/53a937a0-386b-4dc0-8178-43948f0d2fa1" />

Open the `servicea` span and notice the transaction flows all the way through `serviceb` (in other words, you have an end-to-end view for `org`).

<img width="1271" height="305" alt="image" src="https://github.com/user-attachments/assets/fce0a828-a922-4411-828e-d77b64d68d89" />

Now highlight the first `client` span in `serviceb` and search for `peer trace url`. You will see the hyperlink to the `organisation 2` environment.

Copy the trace URL by hovering over the right hand side of the URL and under the "three dots" icon you'll find a copy option.

<img width="1271" height="305" alt="image" src="https://github.com/user-attachments/assets/95795148-d086-4214-a481-b1030cf85c25" />
<img width="1271" height="191" alt="image" src="https://github.com/user-attachments/assets/a8136aea-174c-4afc-9c0c-eb895229ff8c" />

Open the `org 2` URL in a new tab and (presuming you have access rights into the `org2` environment) you'll see the distributed traces app pre filtered for your trace and should see the transaction continuing through `servicec` and `serviced`.

<img width="1403" height="449" alt="image" src="https://github.com/user-attachments/assets/5a711c21-44b6-4334-94ae-b8b0d675a2f4" />

### Navigating Backwards

Highlight the first `server` span in `servicec` and search again for `peer trace url` and you'll see a link "back" to `org1`. So no matter whether you're a staff member at `org1` or `org2`, you can each find the correlating traces and navigate easily between environments.

<img width="1177" height="539" alt="image" src="https://github.com/user-attachments/assets/5fe9b5a4-c56c-4320-a4d0-df5f0649694b" />

## How Does it Work?

We already know the spans flow through two different OpenTelemetry collectors. The "magic" happens in those collectors. So let's take a closer look. Open [collector/org1-collector.yaml](collector/org1-collector.yaml). Lines 19 to 23 define a single data pipeline which handles a data type of traces (the collector can also handle `metrics` and `logs`). This pipeline is simple. Traces (spans) are received via OTLP (the services send spans to this collector) then the collector `transform`s the spans in realtime then the modified spans are sent to `organisation 1`s Dynatrace environment.

Let's take a closer look at that `transform` block as it's key to understanding how this all works. The key line is this (split over multiple lines here for readability).

```
- set(span.attributes["peer.trace.url"],
  Concat(["https://def23456.apps.dynatrace.com/ui/apps/dynatrace.distributedtracing/explorer?filter=trace.id+%3D+", span.trace_id.string], ""))
  where span.attributes["http.url"] == "http://localhost:9002/"
```

As spans from `servicea` and `serviceb` flow through this collector, whenever the span has an attribute called `http.url` and it's value is `http://localhost:9002` (which is the case for spans produced when `serviceb` calls out to `services`) then the collector adds a **new** attribute onto that span with a key of `peer.trace.url` and a value which is a concatenation of `org2` Dynatrace environment ID deep linking to the `Distributed Tracing` app pre filtered by the trace ID pulled from this span.

So remember, this logic is applied to `serviceb` spans so users can hyperlink "downstream" into `servicec`. The exact same logic applies in [collector/org1-collector.yaml](collector/org2-collector.yaml) but in reverse (so `servicec` spans are linked "back" to their `serviceb` source).

> [!TIP]
> If you had many rules, maintaining this list could become unweildy. There is another option to use a YAML lookup table but that requires a custom collector build (nowhere near as scary as it sounds) but it does mean slightly more work. The method above will work with almost any standard OpenTelemetry collector distribution (which is why we've chosen it as step 1 above).
> If you would like instructions on how to use the [lookup processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/lookupprocessor) method, please [open an issue](https://github.com/agardnerIT/cross-environment-linking-dt/issues/new).




