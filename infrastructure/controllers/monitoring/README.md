# Monitoring Controller Notes

For the two-day lab, metrics are exposed through `/metrics` and can be scraped manually or by
Managed Service for Prometheus if installed on GKE.

Avoid committing a `PodMonitor` until the Prometheus Operator CRDs exist; otherwise Flux will
fail reconciliation. In production, install the monitoring stack first, then add service-level
scrape resources with low-cardinality labels only.

