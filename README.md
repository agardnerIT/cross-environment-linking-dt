# cross-environment-linking-dt
Imagine 2 organisations: `organisation 1` and `organisation 2`. Both organisations share an end-to-end transaction that flows from `org1` across two services (`serviceA` and `serviceB`) into `org2` (`serviceC` and `serviceD`).

All services are instrumented with OpenTelemetry so the end-to-end flow is well tracked and automatically (thanks to OTel) has a single consistent trace ID across all 4 services.

<img width="817" height="222" alt="image" src="https://github.com/user-attachments/assets/9893839e-8bcd-404a-a08a-1d8e33a55951" />

## The Challenge

* `org1` wants to store **only** their part of the transaction (data sovereignty and cost)
* `org2` also only wants to store their part of the transaction (data sovereignty and cost)
* `org1` users need to be able to "drill down" into `org2` transactions)
* `org2` users need to be able to "drill up" into transactions that originated in `org1`


## The Architecture

<img width="817" height="344" alt="image" src="https://github.com/user-attachments/assets/34c4198a-256f-4fea-a0b0-ed4ea0edb04f" />

* `org1` and `org2` each have their own Dynatrace environment
* The spans from `serviceA` and `serviceB` flow through an OpenTelemetry collector and into the `org1` environment
* Spans from `serviceC` and `serviceD` flow through a different OpenTelemetry collector into the `org2` environment
* Importantly, for any given individual transaction, spans for `serviceA`, `serviceB`, `serviceC` and `serviceD` ALL carry the same trace ID meaning Dynatrace can represent all 4 spans as a single end-to-end transaction.

## The Hands On

First make sure Docker is running: `docker ps` should work.

* `serviceA` will be running on `http://localhost:9000`
* `serviceB` will be running on `http://localhost:9001`
* `serviceC` will be running on `http://localhost:9002`
* `serviceD` will be running on `http://localhost:9003`

```
git clone https://github.com/agardnerIT/cross-environment-linking-dt
cd cross-environment-linking-dt
```

Start the stack:

```
./run.sh
```






