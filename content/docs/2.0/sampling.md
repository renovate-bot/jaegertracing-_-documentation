---
title: Sampling
hasparent: true
---

Sampling is essential when handling large volumes of traces. Sampling helps to control both the overhead on the applications producing traces, and the storage and processing costs for trace data. Ideally, sampling aims to discard less interesting data and keep the data needed to diagnose issues.

Jaeger supports multiple types of sampling strategies.

## Head Based Sampling 

Head based sampling is when the sampling decision is made at the beginning of a trace. The support for head based sampling is built into the OpenTelemetry SDKs. The topic is covered in the [OpenTelemetry documentation on head based sampling](https://opentelemetry.io/docs/concepts/sampling/#head-sampling).


## Tail Based Sampling

Tail based sampling allows for the sampling decisions to be made after the trace is complete and all spans have been collected. This provides more granular control over which traces and kept and which are discarded. The downsides to this are (a) the runtime overhead on the application that must record and export all traces, and (b) the higher memory and processing costs for the Jaeger backend. You can learn more about tail based sampling in the [OpenTelemetry documentation](https://opentelemetry.io/docs/concepts/sampling/#tail-sampling).

Jaeger supports tail based sampling via a [tail sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor).
  * This [configuration allows for the sampling of everything observed](https://github.com/jaegertracing/jaeger/blob/main/cmd/jaeger/config-tail-sampling-always-sample.yaml) by Jaeger with debug logging to diagnose problems.
  * The second configuration of [samples for a specific service name](https://github.com/jaegertracing/jaeger/blob/main/cmd/jaeger/config-tail-sampling-service-name-policy.yaml).

## Remote Sampling

Remote sampling is a form of head-based sampling where the configuration for the sampling strategy is retrieved by the SDKs from the Jaeger backend. It allows consolidating the sampling policies in one central location and simplifying the management of the SDKs running in large scale production systems.

Jaeger's `remove_sampling` extension supports [Jaeger's remote sampling protocol][remote-sampling-api] that defines sampling strategies as distinct sampling rates for each service and endpoint. The strategies can be generated by Jaeger in two different ways: [periodically loaded from a file](#file-based-sampling-configuration) or [dynamically calculated based on traffic](#adaptive-sampling).

These are two basic types of head samplers that are used by the remote sampling: 

* **Probabilistic** sampler makes a random sampling decision with a pre-configured probability. For example, with `probability=0.1` approximately 1 in 10 traces will be sampled.
* **Rate Limiting** sampler uses a leaky bucket rate limiter to ensure that traces are sampled with a certain constant rate. For example, when `rate=2.0` it will sample requests with the rate of 2 traces per second.

[remote-sampling-api]: ../apis/#remote-sampling-configuration-stable

### File-based Sampling Configuration

The `remote_sampling` extension in Jaeger can be configured with a pointer to a file that describes how sampling strategies should be generated for different services. The file will be automatically reloaded if its contents change.

```yaml
remote_sampling:
  file:
    path: ./cmd/jaeger/sampling-strategies.json
```

If no configuration is provided, Jaeger will return the default probabilistic sampling policy with probability 0.001 (0.1%) for all services.

Example `strategies.json`:
```json
{
  "service_strategies": [
    {
      "service": "foo",
      "type": "probabilistic",
      "param": 0.8,
      "operation_strategies": [
        {
          "operation": "op1",
          "type": "probabilistic",
          "param": 0.2
        },
        {
          "operation": "op2",
          "type": "probabilistic",
          "param": 0.4
        }
      ]
    },
    {
      "service": "bar",
      "type": "ratelimiting",
      "param": 5
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.5,
    "operation_strategies": [
      {
        "operation": "/health",
        "type": "probabilistic",
        "param": 0.0
      },
      {
        "operation": "/metrics",
        "type": "probabilistic",
        "param": 0.0
      }
    ]
  }
}
```

`service_strategies` element defines service specific sampling strategies and `operation_strategies` defines operation specific sampling strategies. There are 2 types of strategies possible: `probabilistic` and `ratelimiting` which are described above (NOTE: `ratelimiting` is not supported for `operation_strategies`). `default_strategy` defines the catch-all sampling strategy that is propagated if the service is not included as part of `service_strategies`.

In the above example:

* All operations of service `foo` are sampled with probability 0.8 except for operations `op1` and `op2` which are probabilistically sampled with probabilities 0.2 and 0.4 respectively.
* All operations for service `bar` are rate-limited at 5 traces per second.
* Any other service will be sampled with probability 0.5 defined by the `default_strategy`.
* The `default_strategy` also includes shared per-operation strategies. In this example we disable tracing on `/health` and `/metrics` endpoints for all services by using probability 0. These per-operation strategies will apply to any new service not listed in the config, as well as to the `foo` and `bar` services unless they define their own strategies for these two operations.

### Adaptive Sampling

Another way to configure `remote_sampling` extension is to use Adaptive Sampling, which works by observing received spans and recalculating sampling probabilities for each service/endpoint combination to ensure that the volume of collected traces matches `target_samples_per_second`. When a new service or endpoint is detected, it is sampled with `initial_sampling_probability` until enough data is collected to calculate the rate appropriate for the traffic going through the endpoint.

Adaptive sampling requires a `sampling_store` storage backend to store the observed traffic data and computed probabilities. At the moment `memory` (for all-in-one deployment), `cassandra`, `badger`, `elasticsearch` and `opensearch` are supported as sampling storage backends. This [sample configuration](https://github.com/jaegertracing/jaeger/blob/main/cmd/jaeger/config.yaml) illustrates the use of adaptive sampling.

Read [this blog post](https://medium.com/jaegertracing/adaptive-sampling-in-jaeger-50f336f4334) for more details on adaptive sampling engine.