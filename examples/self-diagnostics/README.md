# Basic OpenTelemetry metrics example with custom error handler:

This example shows how to setup the custom error handler for self-diagnostics.

## Custom Error Handling:

A custom error handler is set up to capture and record errors using the `tracing` crate's `error!` macro. These errors are then exported to a collector using the `opentelemetry-appender-tracing` crate, which utilizes the OTLP log exporter over `HTTP/protobuf`. As a result, any errors generated by the configured OTLP metrics pipeline are funneled through this custom error handler for proper recording and export.

## Filtering logs from external dependencies of OTLP Exporter:

The example configures a tracing `filter` to restrict logs from external crates (`hyper`, `tonic`, and `reqwest`) used by the OTLP Exporter to the `error` level. This helps prevent an infinite loop of log generation when these crates emit logs that are picked up by the tracing subscriber.

## Ensure that the internally generated errors are logged only once:

By using a hashset to track seen errors, the custom error handler ensures that the same error is not logged multiple times. This is particularly useful for handling scenarios where continuous error logging might occur, such as when the OpenTelemetry collector is not running. 


## Usage

### `docker-compose`

By default runs against the `otel/opentelemetry-collector:latest` image, and uses `reqwest-client`
as the http client, using http as the transport.

```shell
docker-compose up
```

In another terminal run the application `cargo run`

The docker-compose terminal will display logs, traces, metrics.

Press Ctrl+C to stop the collector, and then tear it down:

```shell
docker-compose down
```

### Manual

If you don't want to use `docker-compose`, you can manually run the `otel/opentelemetry-collector` container
and inspect the logs to see traces being transferred.

On Unix based systems use:

```shell
# From the current directory, run `opentelemetry-collector`
docker run --rm -it -p 4318:4318 -v $(pwd):/cfg otel/opentelemetry-collector:latest --config=/cfg/otel-collector-config.yaml
```

On Windows use:

```shell
# From the current directory, run `opentelemetry-collector`
docker run --rm -it -p 4318:4318 -v "%cd%":/cfg otel/opentelemetry-collector:latest --config=/cfg/otel-collector-config.yaml
```

Run the app which exports logs, metrics and traces via OTLP to the collector

```shell
cargo run
```

### Output:

- If the docker instance for collector is running, below error should be logged into the container. There won't be any logs from the `hyper`, `reqwest` and `tonic` crates.
```
otel-collector-1  | 2024-06-05T17:09:46.926Z    info    LogsExporter    {"kind": "exporter", "data_type": "logs", "name": "logging", "resource logs": 1, "log records": 1}
otel-collector-1  | 2024-06-05T17:09:46.926Z    info    ResourceLog #0
otel-collector-1  | Resource SchemaURL:
otel-collector-1  | Resource attributes:
otel-collector-1  |      -> telemetry.sdk.name: Str(opentelemetry)
otel-collector-1  |      -> telemetry.sdk.version: Str(0.23.0)
otel-collector-1  |      -> telemetry.sdk.language: Str(rust)
otel-collector-1  |      -> service.name: Str(unknown_service)
otel-collector-1  | ScopeLogs #0
otel-collector-1  | ScopeLogs SchemaURL:
otel-collector-1  | InstrumentationScope opentelemetry-appender-tracing 0.4.0
otel-collector-1  | LogRecord #0
otel-collector-1  | ObservedTimestamp: 2024-06-05 17:09:45.931951161 +0000 UTC
otel-collector-1  | Timestamp: 1970-01-01 00:00:00 +0000 UTC
otel-collector-1  | SeverityText: ERROR
otel-collector-1  | SeverityNumber: Error(17)
otel-collector-1  | Body: Str(OpenTelemetry metrics error occurred: Metrics error: Warning: Maximum data points for metric stream exceeded. Entry added to overflow. Subsequent overflows to same metric until next collect will not be logged.)
otel-collector-1  | Attributes:
otel-collector-1  |      -> name: Str(event examples/self-diagnostics/src/main.rs:42)
otel-collector-1  | Trace ID:
otel-collector-1  | Span ID:
otel-collector-1  | Flags: 0
otel-collector-1  |     {"kind": "exporter", "data_type": "logs", "name": "logging"}
```

- The SDK will keep trying to upload metrics at regular intervals if the collector's Docker instance is down. To avoid a logging loop, internal errors like 'Connection refused' will be attempted to be logged only once.