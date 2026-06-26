Design and implement a multi-tenant Prometheus Remote Write proxy for Grafana Mimir

Background

We are building a multi-tenant observability platform on OpenShift.

Our users deploy applications into Kubernetes namespaces. Each namespace represents one tenant.

We already use OpenShift User Workload Monitoring (UWM) to scrape Prometheus metrics from applications.

Today the metrics can be remote-written directly into Grafana Mimir.

However, Mimir multitenancy is request-based, not sample-based.

Each HTTP remote_write request must contain exactly one

X-Scope-OrgID

header.

A single Prometheus remote_write request cannot contain metrics belonging to multiple Mimir tenants.

Unfortunately OpenShift User Workload Monitoring scrapes metrics for all namespaces together and therefore sends mixed batches.

Because of this, we need a service that sits between Prometheus and Mimir.

---

High-level architecture

Current architecture

OpenShift User Workload Monitoring
            │
            ▼
         Grafana Mimir

Desired architecture

OpenShift User Workload Monitoring
            │
            │ Prometheus Remote Write
            ▼
     Multi-Tenant Write Proxy
            │
            │ Prometheus Remote Write
            ▼
         Grafana Mimir

Only the remote_write URL configured in OpenShift changes.

Instead of writing directly to Mimir, Prometheus writes to our proxy.

The proxy becomes responsible for assigning the correct tenant.

---

Goals

The proxy must:

- receive Prometheus remote_write requests
- decode the protobuf payload
- inspect every time series
- determine the namespace for every series
- group series by namespace
- create one remote_write request per namespace
- send every group to Mimir
- set

X-Scope-OrgID = <namespace>

for every outgoing request.

The proxy must be transparent.

Prometheus should not require any configuration changes except changing the remote_write URL.

---

Namespace == Tenant

The namespace is our tenant identifier.

Example

namespace = customer-a

becomes

X-Scope-OrgID: customer-a

Another example

namespace = production-tenant

becomes

X-Scope-OrgID: production-tenant

There is a 1:1 mapping.

No external database lookup is required.

---

Cluster metrics

Some metrics are cluster-level.

Examples include:

- kube-apiserver
- etcd
- scheduler
- controller-manager
- node-exporter
- kubelet
- machine metrics

These metrics either have no namespace label or should not belong to a customer.

Those metrics must be sent using

X-Scope-OrgID: cluster-system

This creates a dedicated Mimir tenant for platform metrics.

---

Processing algorithm

For every incoming request:

1. Decode Prometheus Remote Write protobuf.

2. Iterate over every TimeSeries.

3. Read all labels.

4. Determine the namespace.

Priority:

- namespace
- kubernetes_namespace
- exported_namespace

(if multiple labels exist, define one canonical rule)

5. If namespace exists:

tenant = namespace

6. If namespace does not exist:

tenant = cluster-system

7. Group all TimeSeries by tenant.

For example

Incoming request

customer-a
customer-a
customer-b
customer-b
customer-b
cluster metric
cluster metric

becomes

Batch A
customer-a

Batch B
customer-b

Batch C
cluster-system

Each batch is then independently encoded back into Prometheus Remote Write format.

Each batch is sent independently to Mimir.

---

HTTP forwarding

Every outgoing request should contain

Content-Encoding: snappy

Content-Type: application/x-protobuf

X-Prometheus-Remote-Write-Version: 0.1.0

X-Scope-OrgID: tenant-name

Everything else should remain identical.

---

Performance

Avoid copying data unnecessarily.

The proxy should:

- decode once
- group references where possible
- encode once per tenant
- reuse buffers
- support concurrent forwarding

This service will receive a high metric ingestion rate.

---

Reliability

The proxy should behave similarly to Mimir Distributor.

Requirements:

- configurable retries
- configurable timeout
- exponential backoff
- connection pooling
- keep-alive
- request metrics
- structured logging

If forwarding one tenant fails, it should not affect the forwarding of other tenant batches from the same incoming request.

---

Observability

Expose Prometheus metrics for the proxy itself.

Useful metrics include:

incoming_requests_total

incoming_timeseries_total

outgoing_requests_total

outgoing_timeseries_total

tenant_batches_total

tenant_batch_size

failed_requests_total

forward_latency_seconds

decode_duration

encode_duration

active_tenants

unknown_namespace_total

---

Security

The proxy should never trust user-provided tenant headers.

Ignore any incoming

X-Scope-OrgID

header.

The proxy is solely responsible for assigning tenants.

---

Configuration

Configuration should include

Mimir endpoint

HTTP timeout

Retry count

Worker concurrency

Maximum request size

Maximum tenants per request

Compression settings

TLS settings

---

Future extensibility

The implementation should allow replacing

tenant = namespace

with

tenant = lookup(namespace)

later.

The routing logic should therefore be isolated behind a TenantResolver interface.

Current implementation:

TenantResolver(namespace):

if namespace exists:
    return namespace

return cluster-system

Future implementations may query Kubernetes labels, CRDs, or external APIs.

---

OpenShift integration

OpenShift User Workload Monitoring should continue scraping exactly as today.

The only platform change should be:

Current

remote_write
    url = https://mimir/api/v1/push

New

remote_write
    url = https://mimir-write-proxy/api/v1/push

Everything else remains unchanged.

The proxy should be completely transparent to Prometheus.

---

Deliverables

Produce:

1. Architecture diagram.
2. Component design.
3. Go implementation (preferred language).
4. Unit tests.
5. Integration tests using Prometheus Remote Write payloads.
6. Kubernetes Deployment.
7. Service.
8. Helm chart.
9. Configuration examples.
10. Documentation explaining the routing algorithm and operational considerations.
