# Upgrade

## Upgrading to Sentry 26.9.0

Review the [self-hosted release notes](https://github.com/getsentry/self-hosted/releases/tag/26.9.0) and [required upgrade stops](https://develop.sentry.dev/self-hosted/releases/) if upgrading from an older Sentry release. The steps below cover the change from 26.8.0 to 26.9.0.

### Before upgrading

1. Back up PostgreSQL, ClickHouse, and persistent file storage, and save your release values. Database migrations may prevent a simple Helm rollback.
2. Upgrade external ClickHouse to **25.8.16.10001 or newer** before running Snuba 26.9.0. The installation examples, CI fixture, and optional cleanup client use `altinity/clickhouse-server:25.8.28.10001.altinitystable`, matching upstream self-hosted. The clustered example also updates Keeper to the matching tag. Follow your ClickHouse operator's upgrade procedure and verify database health first.
3. If span processing is enabled, pause incoming ingestion and let the existing 26.8.0 span pipeline finish, including buffered spans and the `buffered-segments` Kafka backlog. Confirm the old `process-segments` consumer has drained before removing it: the new taskworker path does not consume this backlog.
4. Review custom values against the new defaults. Helm replaces lists such as `sentry.taskBroker.brokers` and `kafka.provisioning.topics`; `--reuse-values` can also preserve obsolete configuration. Keep required credentials, storage settings, and custom routing while applying the changes below.

### Configuration changes

- Sentry, Snuba, Relay, Symbolicator, Vroom, uptime-checker, Taskbroker, and Launchpad use `26.9.0` images from GHCR by default. Remove or update explicit image tag overrides so the components stay aligned.
- Remove `sentry.genericMetricsConsumer`, `sentry.processSegments`, `snuba.genericMetricsCountersConsumer`, and `snuba.subscriptionConsumerGenericMetricsCounters` overrides. These workloads no longer exist upstream.
- Remove `generic-metrics-subscription-results` from custom Taskbroker `kafkaTopics` maps. Its raw task handler, `sentry.snuba.query_subscriptions.run.process_generic_metrics_subscription_from_kafka`, has been removed.
- Segment processing uses `SENTRY_OPTIONS["spans.buffer.process-segments-task-rollout-rate"] = 1.0` and routes the `spans.process_segments` namespace to `taskworker-ingest`. Keep the ingest broker and its workers enabled, and retain this route in custom `config.taskbrokerRoutingYml` configuration. Allow worker capacity for the extra span workload.
- Synchronize custom Kafka topic lists with `values.yaml`, including `ingest-events-backlog`, `ingest-spans-dlq`, `ingest-generic-metrics-dlq`, and `snuba-llm-proxy-cost`. Six obsolete generic metrics sets/distributions/gauges scheduler and commit-log declarations are removed. Remaining generic metrics topics and Snuba storage mappings are retained for bootstrap and historical migrations; existing Kafka topics and ClickHouse tables are not deleted by this upgrade.
- Snuba uses `externalClickhouse.httpPort` for queries, ingestion, and migrations, including custom `CLUSTERS[].port` settings. `CLICKHOUSE_HTTP_PORT` is exported to Snuba. `externalClickhouse.tcpPort` remains the native port for the optional cleanup client. Review network policies and TLS ports accordingly.
- Snuba's production image has no shell or coreutils. The chart uses `python3` for file probes; update any custom Snuba commands or probes that invoke `sh`, `bash`, `test`, `stat`, or `date`.
- Remove `relay.cache.envelopeBufferSize` and custom `cache.envelope_buffer_size` configuration. Relay no longer supports this option or Expect-CT, HPKP, and Expect-Staple security reports.
- Remove custom `organizations:incidents` and obsolete generic metrics feature flags. Experimental flags listed in the release notes remain opt-in.

### Remove obsolete hook resources

With the default `asHook: true`, Helm does not delete old hook Deployments when their templates disappear. After draining the old pipeline, preview the obsolete Deployments for your release:

```bash
SENTRY_NAMESPACE=sentry
SENTRY_RELEASE=sentry
SENTRY_REMOVED_SELECTOR="release=${SENTRY_RELEASE},app.kubernetes.io/component in (sentry-process-segments,generic-metrics-consumer,snuba-generic-metrics-counters-consumer,snuba-subscription-consumer-generic-metrics-counters)"
kubectl -n "$SENTRY_NAMESPACE" get deployments -l "$SENTRY_REMOVED_SELECTOR"
```

Check the namespace, release, and listed workloads, then remove only those obsolete Deployments:

```bash
kubectl -n "$SENTRY_NAMESPACE" delete deployments -l "$SENTRY_REMOVED_SELECTOR"
```

With `asHook: false`, Helm removes the obsolete tracked resources during the upgrade. Custom controllers or GitOps hook handling may require equivalent cleanup. The chart also removes the two dedicated Sentry ServiceAccount templates; remove any manually managed equivalents if unused.

### Run the upgrade

Run `helm upgrade` with the reviewed values and allow the chart's Sentry, Snuba, and applicable Taskbroker migration hooks to finish. If you manage migrations separately, run them before starting the new workloads. Verify worker health, ingestion, queries, and Kafka lag before resuming traffic.

Upstream self-hosted changes its PostgreSQL 14 base image to Debian trixie and reindexes for a glibc collation change. This chart does not change the bundled PostgreSQL image or run that Docker Compose migration script. If you independently change your database's OS or glibc, follow PostgreSQL's collation/reindex procedure and review the duplicate-row warning in the release notes. PostgreSQL 18 is planned for a later Sentry release and is not required for 26.9.0.

## Upgrading from 13.x.x version of this Chart to 14.0.0

ClickHouse was reconfigured with sharding and replication in-mind, If you are using external ClickHouse, you don't need to do anything.

**WARNING**: You will lose current event data<br>
Otherwise, you should delete the old ClickHouse volumes in-order to upgrade to this version.

## Upgrading from 12.x.x version of this Chart to 13.0.0

The service annotions have been moved from the `service` section to the respective service's service sub-section. So what was:

```yaml
service:
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /_health/
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
```

will now be set per service:

```yaml
sentry:
  web:
    service:
      annotations:
        alb.ingress.kubernetes.io/healthcheck-path: /_health/
        alb.ingress.kubernetes.io/healthcheck-port: traffic-port

relay:
  service:
    annotations:
      alb.ingress.kubernetes.io/healthcheck-path: /api/relay/healthcheck/ready/
      alb.ingress.kubernetes.io/healthcheck-port: traffic-port
```

## Upgrading from 11.x.x version of this Chart to 12.0.0

Redis chart was upgraded to newer version. If you are using external redis, you don't need to do anything.

Otherwise, when upgrading to chart version 12.x.x from 11.x.x you need to either run `helm upgrade` with `--force` flag, or prior to upgrade delete statefulsets for redis master and redis slave. Then run upgrade and it will roll out new statefulsets. Your master redis data will not be lost (PVC is not deleted when you delete statefulset). Your redis slave will now be named redis replica and you can delete PVCs that were used by redis slave after the upgrade.

## Upgrading from 10.x.x version of this Chart to 11.0.0

If you were using clickhouse tabix externally, we disabled it per default.

## Upgrading from deprecated 9.0 -> 10.0 Chart

As this chart runs in helm 3 and also tries its best to follow on from the original Sentry chart. There are some steps that needs to be taken in order to correctly upgrade.

From the previous upgrade, make sure to get the following from your previous installation:

- Redis Password (If Redis auth was enabled)
- Postgresql Password
  Both should be in the `secrets` of your original 9.0 release. Make a note of both of these values.

### Upgrade Steps

Due to an issue where transferring from Helm 2 to 3. Statefulsets that use the following: `heritage: {{ .Release.Service }}` in the metadata field will error out with a `Forbidden` error during the upgrade. The only workaround is to delete the existing statefulsets (Don't worry, PVC will be retained):

```shell
kubectl delete --all sts -n <Sentry Namespace>
```

Once the statefulsets are deleted. Next steps is to convert the helm release from version 2 to 3 using the helm 3 plugin:

```shell
helm3 2to3 convert <Sentry Release Name>
```

Finally, it's just a case of upgrading and ensuring the correct params are used:

If Redis auth enabled:

```shell
helm upgrade -n <Sentry namespace> <Sentry Release> . --set redis.usePassword=true --set redis.password=<Redis Password>
```

If Redis auth is disabled:

```shell
helm upgrade -n <Sentry namespace> <Sentry Release> .
```
