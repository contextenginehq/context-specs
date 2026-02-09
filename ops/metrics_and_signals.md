# Metrics and Signals

## Core Metrics

| Metric              | Description                              | Component |
|---------------------|------------------------------------------|-----------|
| Context hit rate    | Cache hit percentage                     | Paid      |
| Cache freshness     | Time since last rebuild                  | Paid      |
| Token utilization   | Budget usage efficiency                  | Paid      |
| Drift detection     | Source/cache divergence                  | Paid      |
| Cost attribution    | Resource usage per context               | Paid      |

## Open Metrics

The following metrics are available in the open core:

- Cache build duration
- Document count
- Cache size

## Signal Requirements

- All signals must be exportable in standard formats (Prometheus, OpenTelemetry)
- Signals must not require runtime overhead when disabled
- Historical data retention is a paid feature
