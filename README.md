# BigQuery Remote Storage Adapter for Prometheus

[![Build Status](https://github.com/KohlsTechnology/prometheus_bigquery_remote_storage_adapter/actions/workflows/pipeline.yml/badge.svg?branch=master)]((https://github.com/KohlsTechnology/prometheus_bigquery_remote_storage_adapter/actions))
[![Go Report Card](https://goreportcard.com/badge/github.com/KohlsTechnology/prometheus_bigquery_remote_storage_adapter)](https://goreportcard.com/report/github.com/KohlsTechnology/prometheus_bigquery_remote_storage_adapter)
[![Join the chat at https://gitter.im/KohlsTechnology/prometheus_bigquery_remote_storage_adapter](https://badges.gitter.im/KohlsTechnology/prometheus_bigquery_remote_storage_adapter.svg)](https://gitter.im/KohlsTechnology/prometheus_bigquery_remote_storage_adapter?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This is a write adapter that receives samples via Prometheus's remote write protocol and stores them in Google BigQuery. This adapter is based off code found in the official prometheus repo:

https://github.com/prometheus/prometheus/tree/master/documentation/examples/remote_storage/remote_storage_adapter

Billing MUST be enabled on the GCP project with the destination BigQuery tables. This adapter uses the "streaming inserts" API. More information is available here: https://cloud.google.com/bigquery/streaming-data-into-bigquery#before_you_begin

The table schema in BigQuery should be the following format:

| Field name | Type | Mode |
| --- | --- | --- |
| metricname | STRING | NULLABLE |
| tags | STRING | NULLABLE |
| value | FLOAT | NULLABLE |
| timestamp | TIMESTAMP | NULLABLE |

It is recommended that the BigQuery table is partitioned on the timestamp column for performance.

The tags field is a json string and can be easily extracted. Here is an example query:

```
SELECT metricname, tags, JSON_EXTRACT(tags, '$.some_label')
  AS some_label, value, timestamp
  FROM `your_gcp_project.prometheus.metrics_stream`
  WHERE JSON_EXTRACT(tags, '$.some_label') = "\\"target_label_value\\""
```

Consider enabling partition expiration on the destination table based on your data retention and billing requirements (https://cloud.google.com/bigquery/docs/managing-partitioned-tables#partition-expiration).


## Running directly with googleAPIjsonkeypath

```
./bigquery_remote_storage_adapter \
  --googleAPIjsonkeypath=/secret/gcp_service_account.json \
  --googleAPIdatasetID=prometheus \
  --googleAPItableID=metrics_stream
```

## Running directly Google ADC

Reference: Google Application Default Credentials ([ADC](https://cloud.google.com/docs/authentication/production#automatically))

```
GOOGLE_APPLICATION_CREDENTIALS=../../private.key.json ./bigquery_remote_storage_adapter \
  --googleProjectID=<GCP Project ID> \
  --googleAPIdatasetID=prometheus \
  --googleAPItableID=metrics_stream
```

To show all flags:

```
./bigquery_remote_storage_adapter -h
```

## Configuration

You can configure this storage adapter either through command line options or environment variables. The latter is required if you're using our docker image.

| Command Line Flag | Environment Variable | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `--googleAPIdatasetID` | `PROMBQ_DATASET` | Yes | | Dataset name as shown in GCP |
| `--googleAPItableID` | `PROMBQ_TABLE` | Yes | | Table name as showon in GCP |
| `--googleAPIjsonkeypath` | `PROMBQ_GCP_JSON` | No | | Path to json keyfile for GCP service account. JSON keyfile also contains project_id. |
| `--googleProjectID` | `PROMBQ_GCP_PROJECT_ID` | Yes (when -googleAPIjsonkeypath is missing)| | The GCP project_id |
| `--send-timeout` | `PROMBQ_TIMEOUT` | No | `30s` | The timeout to use when sending samples to the remote storage |
| `--web.listen-address` | `PROMBQ_LISTEN` | No | `:9201` | Address to listen on for web endpoints |
| `--web.telemetry-path` | `PROMBQ_TELEMETRY` | No | `/metrics` | Address to listen on for web endpoints |
| `--log.level` | `PROMBQ_LOG_LEVEL` | No | `info` | Only log messages with the given severity or above. One of: [debug, info, warn, error] |
| `--log.format` | `PROMBQ_LOG_FORMAT` | No | `logfmt` | Output format of log messages. One of: [logfmt, json] |

## Configuring Prometheus

To configure Prometheus to send samples to this binary, add the following to your `prometheus.yml`:

```yaml
# Remote write configuration (for Google BigQuery).
remote_write:
  - url: "http://localhost:9201/write"

# Remote read configuration (for Google BigQuery).
remote_read:
  - url: "http://localhost:9201/read"

```

## Performance Tuning

You will need to tune the storage adapter based on your needs. You have several levers available...

### Requests & Limits

When running on a container platform (like Kubernetes), it's important to configure the CPU / memory requests and limits properly. You should be able to get away with just a couple hundred megabytes of RAM (make sure request == limit), but the CPU needs will heavily depend on your environment. Set the CPU requests to the minimum you need to achieve the required performance. We recommend setting the limit higher (keep in mind that anything above the request is not guaranteed). Keep an eye on CPU throttling to help tweak your settings.

### Limit Metrics Stored Long-Term

The amount of data you send to BigQuery can be another big constraint. It is easy to overwhelm the BigQuery streaming engine by throwing millions of records at it. You might run into API quota issues or simply have data gaps. We highly recommend not to go crazy when it comes to scrape intervals (<30s) and be very selective on what gets stored long-term. Depending on your needs, it might make sense to calculate and store only aggregated metrics long-term.
Refer to the Prometheus documentation for [remote_write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write) and [relabel_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) on how to implement this.

### Prometheus Remote Storage (remote_write & queue_config)

Prometheus allows you to tune the write behavior for remote storage. Please refer to their [documentation](https://prometheus.io/docs/practices/remote_write/) for details.
### Example `prometheus.yml`

```
remote_write:
- url: http://localhost:9201/write
  remote_timeout: 2m
  write_relabel_configs:
  - source_labels: [__name__]
    separator: ;
    regex: ALERTS|apiserver_request_.*|kube_namespace_labels
    replacement: $1
    action: keep
  queue_config:
    capacity: 500
    max_shards: 200
    min_shards: 1
    max_samples_per_send: 100
    batch_send_deadline: 5s
    min_backoff: 30ms
    max_backoff: 100ms
remote_read:
- url: http://localhost:9201/read
  remote_timeout: 1m
```

## Building

### Binary

If you just need a local version to test, then the simplest way is to execute:

```
make build
```

### Image

In order to build the docker image, simply execute

```
make image
```

## Releasing

This project is using [goreleaser](https://goreleaser.com). GitHub release creation is automated using Travis
CI. New releases are automatically created when new tags are pushed to the repo.
```
$ TAG=v0.0.2 make tag
```

How to manually create a release without relying on Travis CI.
```
$ TAG=v0.0.2 make tag
$ GITHUB_TOKEN=xxx make clean release
```

## Testing

### Credentials via Google API json key file

Use the `--googleAPIjsonkeypath` argument to indicate the keyfile path (note: the GCP Project ID will be automatically extracted from the specified file).


```
go test -v -cover ./... -args \
  --googleAPIjsonkeypath=XXX \
  --googleAPIdatasetID=XXX \
  --googleAPItableID=XXX \
```

### Credentials via `GOOGLE_APPLICATION_CREDENTIALS`

Use the `--googleProjectID` argument to indicate the GCP Project ID and let _Google Application Default Credentials_ ([ADC](https://cloud.google.com/docs/authentication/production#automatically)) identify the service account from the `GOOGLE_APPLICATION_CREDENTIALS` environment variable.

```
GOOGLE_APPLICATION_CREDENTIALS=../../private.key.json \
go test -v -cover ./... -args \
  --googleProjectID=XXX \
  --googleAPIdatasetID=XXX \
  --googleAPItableID=XXX \
```

### Credentials via default service account

Identify the service account via Google Application Default Credentials ([ADC](https://cloud.google.com/docs/authentication/production#automatically)).

Use the `--googleProjectID` argument to indicate GCP Project ID and let ADC identify the default service account. This will only work if you're running on Google's infrastructure (GCP VMs, GKE, etc.).

```
go test -v -cover ./... -args \
  --googleProjectID=XXX \
  --googleAPIdatasetID=XXX \
  --googleAPItableID=XXX \
```

## Prometheus Metrics Offered

| Metric Name | Metric Type | Short Description |
| --- | --- | --- |
| storage_bigquery_received_samples_total | Counter | Total number of received samples. |
| storage_bigquery_sent_samples_total | Counter | Total number of processed samples sent to remote storage that share the same description. |
| storage_bigquery_failed_samples_total | Counter | Total number of processed samples which failed on send to remote storage that share the same description. |
| storage_bigquery_sent_batch_duration_seconds | Histogram | Duration of sample batch send calls to the remote storage that share the same description. |
| storage_bigquery_write_errors_total | Counter | Total number of write errors to BigQuery. |
| storage_bigquery_read_errors_total | Counter | Total number of read errors from BigQuery |
| storage_bigquery_write_api_seconds | Histogram | Duration of the write api processing that share the same description. | 
| storage_bigquery_read_api_seconds | Histogram | Duration of the read api processing that share the same description. |