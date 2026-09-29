# AWS
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## Introduction
The Amazon CloudWatch Observability Helm Chart provides easy mechanisms to setup the [Amazon CloudWatch Agent Operator](https://github.com/aws/amazon-cloudwatch-agent-operator) to manage the [CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html) on Kubernetes clusters.

## Getting Started
Full instructions can be found in the [AWS documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Observability-EKS-addon.html)

### Installation
1. You must have Helm installed to use this chart. For more information about installing Helm, see the [Helm documentation](https://helm.sh/docs/).
2. After you have installed Helm, enter the following commands. Replace my-cluster-name with the name of your cluster, and replace my-cluster-region with the Region that the cluster runs in.

```bash
helm repo add aws-observability https://aws-observability.github.io/helm-charts
helm repo update aws-observability
helm install --wait --create-namespace --namespace amazon-cloudwatch amazon-cloudwatch aws-observability/amazon-cloudwatch-observability --set clusterName=my-cluster-name --set region=my-cluster-region
```

By default, the helm chart will enable [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html) enhanced observability with container logging, and [CloudWatch Application Signals](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html). This helps you to collect infrastructure metrics, application performance telemetry, and container logs from the Amazon EKS cluster.

## LLM model-serving telemetry (vLLM / KServe / Knative)

Set under `otelContainerInsights.solutions`. The four metric sources default on; both trace
sources default off, since they need Transaction Search enabled account-wide and an
endpoint set on the sender.

Which sources exist depends on the KServe deployment mode. In **Serverless** mode (KServe's
default) an InferenceService becomes a Knative Service, so all of them are present. In
**RawDeployment** mode there is no queue-proxy and no `knative-serving` namespace, so only
`vllm` and `kserve.controlPlane` find targets; use the HPA metrics from kube-state-metrics
for the autoscaling view. Note that the request-level metrics are emitted by the Knative
queue-proxy rather than by KServe, which is a control plane only and is never in the request
path — hence `knative.dataPlane`, not `kserve`.

### Which vLLM servers are scraped

One scrape job finds both kinds, with nothing to configure:

- **KServe** — the `kserve-container` of any pod carrying the
  `serving.kserve.io/inferenceservice` label, whatever its image.
- **Anything else** — any container whose image name contains `vllm`
  (`vllm/vllm-openai`, or a mirror of it). A server built from an image without `vllm` in
  its name is not detected.

The port is the one the container declares; a container that declares none is scraped on
the server's default, 8080 for KServe and 8000 for `vllm serve`.

Every target carries the resource attribute `aws.service.type`, so a consumer can separate
inference from training without knowing which metric names belong to which. It is
`ai_inference` unless the pod has an `aws.service.type` label, whose value is used instead —
`ai_training` for a vLLM server that belongs to a training pipeline, such as a rollout
generator in an RL loop. That does not make the pipeline collect trainer metrics: its filter
keeps only `vllm:*` and `http_*`.

### The vLLM metric service name

`service.name` on the engine's metrics and spans is resolved in this order, highest
precedence first:

1. the InferenceService name
2. the workload name — Deployment, else StatefulSet, DaemonSet, Job or ReplicaSet
3. the pod name

The InferenceService outranks the Deployment because in KServe Serverless mode the
Deployment name embeds the Knative revision generation
(`<isvc>-predictor-00004-deployment`) and so changes on every redeploy. Spans that arrive
with a `service.name` of their own, such as `OTEL_SERVICE_NAME`, keep it.

The pipeline also promotes the scraped `pod`/`namespace` labels to resource attributes and
runs `k8sattributes` pod association, so `k8s.workload.name` resolves to the owning
Deployment rather than to the ReplicaSet — whose pod-template hash likewise changes on
every redeploy.

### vLLM request traces require Transaction Search

`solutions.vllm.traces` exports spans to the CloudWatch OTLP traces endpoint
(`https://xray.<region>.amazonaws.com/v1/traces`). **That endpoint only accepts spans when
the account's X-Ray trace segment destination is `CloudWatchLogs`** — i.e. when Transaction
Search is enabled. Until it is, every batch is rejected:

```
HTTP Status Code 400, Message=The OTLP API is supported with CloudWatch Logs as a
Trace Segment Destination. Please enable the CloudWatch Logs destination for your
traces using the UpdateTraceSegmentDestination API
```

The agent treats this as a permanent, non-retryable error and drops the spans, so the
symptom is silent absence of traces plus `Exporting failed. Dropping data.` in the agent
log.

This setting is **account-wide per Region**, not per cluster, and switches all X-Ray span
ingestion in that Region into CloudWatch Logs. Enable it in the CloudWatch console under
**Application Signals → Transaction Search**, or with the API:

```bash
# 1. allow X-Ray to write the span log groups (the console does this for you)
aws logs put-resource-policy --policy-name TransactionSearchAccess --policy-document '{
  "Version":"2012-10-17",
  "Statement":[{
    "Sid":"TransactionSearchXRayAccess",
    "Effect":"Allow",
    "Principal":{"Service":"xray.amazonaws.com"},
    "Action":"logs:PutLogEvents",
    "Resource":[
      "arn:aws:logs:<region>:<account>:log-group:aws/spans:*",
      "arn:aws:logs:<region>:<account>:log-group:/aws/application-signals/data:*"],
    "Condition":{
      "ArnLike":{"aws:SourceArn":"arn:aws:xray:<region>:<account>:*"},
      "StringEquals":{"aws:SourceAccount":"<account>"}}}]}'

# 2. switch the destination
aws xray update-trace-segment-destination --destination CloudWatchLogs

# 3. choose how much to index as trace summaries (1% is free)
aws xray update-indexing-rule --name Default \
  --rule '{"Probabilistic":{"DesiredSamplingPercentage":1}}'
```

Step 1 is required — without it step 2 fails with
`AccessDeniedException: XRay does not have permission to call PutLogEvents on the aws/spans
Log Group`. Spans take up to 10 minutes to become searchable. 100% of spans are stored in
the `aws/spans` log group; only the indexed percentage becomes a searchable trace summary in
X-Ray. No IAM change is needed on the agent itself — `CloudWatchAgentServerPolicy` already
grants the endpoint.

Note that the prerequisite is not currently mentioned on the
[OTLP Endpoints](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPEndpoint.html)
page; it is documented on the
[Transaction Search](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Transaction-Search.html)
pages and enforced by the endpoint.

### What lands, and what vLLM does not send

Spans arrive in the `aws/spans` log group in semantic-convention format with W3C trace IDs,
so every attribute vLLM sets stays queryable — there is no indexed subset to declare.

The endpoint also enrols the spans in Application Signals: it injects `aws.local.*` and
`aws.span.kind` server-side, which creates a service entity and emits billed
`ApplicationSignals` metrics (`Latency`, `Error`, `Fault`, `Throttle`, `InputTokens`,
`OutputTokens`). That is not configured by this chart and cannot be turned off from here.

Three limitations come from vLLM itself and are worth knowing before building on this:

- **Errors are not observable.** vLLM never sets span status, and only successfully
  finished requests are traced at all — aborts, timeouts and errors emit no span. So
  Application Signals `Error` and `Fault` stay at zero regardless of what the engine is
  doing. Upstream fix: [vllm#32162](https://github.com/vllm-project/vllm/pull/32162).
- **No model name, no service name.** The V1 engine sets neither `gen_ai.response.model`
  nor `service.name`. The pipeline names the span from the InferenceService, else the
  Deployment, else the pod — the same name the metrics carry, so join on `service.name`.
- **Traces are single-span.** vLLM honours an inbound W3C `traceparent`, but nothing in a
  default Istio/Knative path propagates one, so there is no end-to-end trace and no
  visibility into gateway queueing.

### Enabling tracing on the engine

The chart only opens the receiver. vLLM's tracing is off by default and is enabled on the
model container, which lives on your InferenceService:

```yaml
spec:
  predictor:
    containers:
      - name: kserve-container
        args:
          - --otlp-traces-endpoint=grpc://cloudwatch-agent.amazon-cloudwatch:4319
```

Use `grpc://` or `http://`, never `https://` — the scheme selects TLS, not the protocol, and
the receiver serves plaintext. vLLM's exporter is OTLP/gRPC unless
`OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=http/protobuf` is set, in which case use the HTTP port.
On vLLM 0.11 also set `OTEL_EXPORTER_OTLP_TRACES_INSECURE=true`; 0.26 needs nothing.
`--collect-detailed-traces` is not needed — it does nothing for traces on the V1 engine.

Spans are billed per trace and vLLM emits one per request with no sampling of its own, so on
a high-QPS endpoint set `OTEL_TRACES_SAMPLER` on the engine.

### Application Signals auto-instrumentation on the engine

vLLM creates its spans through the global tracer provider. If Application Signals injects
the ADOT Python SDK into the engine pod, that provider is ADOT's, so `llm_request` is
exported to the Application Signals port (4316) rather than to this receiver: it arrives
renamed to the service name and without the attributes this pipeline adds. Either exclude
the namespace from auto-instrumentation — all four languages under
`manager.applicationSignals.autoMonitor.exclude`, since the others inject an `xray`
propagator that only the Python SDK provides — or keep the SDK and route its traces here:

```yaml
        env:
          - name: OTEL_SERVICE_NAME
            value: <isvc>
          - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
            value: http://cloudwatch-agent.amazon-cloudwatch:4320/v1/traces
```

The second keeps the SDK's HTTP spans and Application Signals `Latency`/`Error`/`Fault`
in the same trace as the engine span. Application Signals then emits no
`InputTokens`/`OutputTokens`, since the endpoint leaves SDK-processed spans to the SDK;
the `vllm:*` token metrics are unaffected. In KServe Serverless mode `autoMonitor` usually
misses new revisions, so opt in with the
`instrumentation.opentelemetry.io/inject-python: "true"` predictor annotation.

### Knative Serving 1.19 renamed every metric

`solutions.knative.*` handles both name sets. Serving ≤ 1.18 emits OpenCensus names
(`revision_*`, `autoscaler_*`); ≥ 1.19 moved to the OpenTelemetry SDK and renamed them with
a `kn` prefix (`autoscaler_desired_pods` → `kn_revision_pods_desired`). On ≥ 1.19,
`solutions.knative.dataPlane` additionally needs Knative told to export request metrics —
they are off by default and the queue-proxy's port 9091 refuses connections until then:

```bash
kubectl -n knative-serving patch cm config-observability --type merge \
  -p '{"data":{"request-metrics-protocol":"prometheus"}}'
```

This is a separate key from `metrics-protocol`, which governs the control plane only.

## Windows Support
CloudWatch DaemonSet on Windows is officially supported only for containerd runtime.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This project is licensed under the Apache-2.0 License.

