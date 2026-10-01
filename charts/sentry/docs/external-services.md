# External Services Configuration

As the Sentry chart moves away from bundled dependencies (due to upstream deprecations and maintenance overhead), using external services is becoming the standard for production deployments.

This guide outlines how to configure the various external services required by Sentry.

## ClickHouse

**Status: REQUIRED**

The bundled ClickHouse chart has been removed. Sentry 26.9.0 requires an external ClickHouse endpoint running **25.8.16.10001 or newer**; the examples use `25.8.28.10001.altinitystable` from upstream self-hosted.

Snuba connects over HTTP(S) using `externalClickhouse.httpPort` (default `8123`). The optional `snuba.cleanup` job uses the native client on `externalClickhouse.tcpPort` (default `9000`). For TLS, set `externalClickhouse.secure` and the appropriate HTTPS/native TLS ports; configure certificate verification with `externalClickhouse.verify` and `externalClickhouse.ca_certs`.

- [Clustered ClickHouse setup](../../../README.md#external-clickhouse-configuration)
- [Single-node ClickHouse setup](../../../clickhouse-single-install.md)
- [Sentry 26.9.0 upgrade guide](UPGRADE.md#upgrading-to-sentry-2690)

## Kafka

**Status: Recommended**

Sentry relies heavily on Kafka. While a bundled Kafka is available, managed Kafka services (like Confluent or MSK) or a dedicated Kafka operator (like Strimzi) are recommended for production.

See `externalKafka` in `values.yaml` for configuration options.

## PostgreSQL

**Status: Recommended**

Sentry uses PostgreSQL as its primary datastore. A bundled PostgreSQL is available for convenience, but an external database (e.g., RDS, Cloud SQL) is strongly recommended for production data integrity and management.

See `externalPostgresql` in `values.yaml` for configuration options.

## Redis

**Status: Recommended**

Redis is used for caching and queuing.

See `externalRedis` in `values.yaml` for configuration options.
