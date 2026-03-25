# mqtt

MQTT is deployed here using the Bitnami `rabbitmq` Helm chart with the RabbitMQ MQTT plugin enabled.

Current defaults:

- Single-node RabbitMQ deployment
- Internal-only service (`ClusterIP`)
- Longhorn-backed persistence (`8Gi`)
- MQTT listener enabled on TCP `1883`
- RabbitMQ management UI kept internal on TCP `15672`
- AMQP still available internally on TCP `5672`
- Credentials sourced from the sealed secret `rabbit-mq`

## Credentials

- RabbitMQ username is set directly in values as `admin`
- The password comes from the unsealed Secret created by `sealedsecret.yaml`
- Expected Secret name: `rabbit-mq`
- Expected password key: `RABBITMQ_DEFAULT_PASS`
- Expected Erlang cookie key: `RABBITMQ_ERLANG_COOKIE`

The sealed key `RABBITMQ_DEFAULT_USER` is not used by the chart.

## Exposure

This is internal-only by default. If you want LAN devices to connect over MQTT, the next step is to change the RabbitMQ service to `LoadBalancer` and pin a free MetalLB IP for port `1883`.
