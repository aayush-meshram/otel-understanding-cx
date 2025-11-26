### 1. Connectivity Keys

**Context:** Infrastructure expects traffic on Node Host IP, Port 4318 (HTTP).

| Key | Scenario | Value | Internal SDK Behavior | Impact on Coralogix / Ops |
| :--- | :--- | :--- | :--- | :--- |
| **OTEL_EXPORTER_OTLP_ENDPOINT** | **Hardcoded (Correct)** | `http://$(NODE_IP):4318` | Resolves Node IP. Sends HTTP POST. | **Success.** Minimal latency (local traffic). Data reaches collector. |
| | **Absent / Missing** | *(Empty)* | Defaults to `localhost:4318`. | **CRITICAL FAILURE.** App tries to talk to itself (Pod), not the Node. Connection Refused. **100% Data Loss.** |
| | **Wrong Protocol** | `https://...` | Attempts TLS Handshake. | **Failure.** Internal node agent listens on plaintext HTTP. Handshake fails. |
| | **Wrong Port** | `...:4317` | Sends HTTP to gRPC port. | **Failure.** Port 4317 expects HTTP/2 framing. Connection rejected. |
| **OTEL_EXPORTER_OTLP_PROTOCOL** | **Hardcoded (Correct)** | `http/protobuf` | Uses Binary Protobuf over HTTP 1.1. | **Success.** Compatible with Port 4318. High performance. |
| | **Absent / Missing** | *(Empty)* | Varies by Language (Node defaults HTTP, Java/Python may default gRPC). | **High Risk.** If SDK defaults to `grpc` against port 4318, connection hangs. |
| | **gRPC** | `grpc` | Opens HTTP/2 stream. | **Failure.** Requires Port 4317. Will fail against Port 4318. |
| | **JSON** | `http/json` | Uses Text JSON over HTTP. | **Inefficient.** 30-40% higher CPU overhead for serialization. Same data, higher cost. |

---

### 2. Signal Keys

**Context:** Controls which data types are generated and sent.

| Key | Scenario | Value | Internal SDK Behavior | Impact on Coralogix / Ops |
| :--- | :--- | :--- | :--- | :--- |
| **OTEL_[LOGS/METRICS/TRACES]_EXPORTER** | **Hardcoded (Correct)** | `otlp` | Captures signal and pushes to Endpoint. | **Success.** Full observability visibility. |
| | **Absent / Missing** | *(Empty)* | Inconsistent defaults (some SDKs disable logging if not explicit). | **Partial Blindness.** Some apps may show traces but no logs. |
| | **Console** | `console` | Prints signal to Standard Output (`stdout`). | **Disaster.** Creates infinite log loops (Log -> Stdout -> Log). Makes `kubectl logs` unreadable. |
| | **None** | `none` | Disables data collection. | **Blindness.** Zero data sent. Hardcoding prevents accidental disabling. |
| | **Prometheus** | `prometheus` | Starts a pull-based HTTP server. | **Failure.** Our architecture is Push-based. Data will sit in the pod and never be scraped. |

---

### 3. Identity Keys

**Context:** Controls how Coralogix groups and indexes the data.

| Key | Scenario | Value | Internal SDK Behavior | Impact on Coralogix / Ops |
| :--- | :--- | :--- | :--- | :--- |
| **OTEL_SERVICE_NAME** | **Hardcoded (Correct)** | `{{ .Chart.Name }}` | Sets the logical service identity. | **Success.** Maps to **Subsystem**. Allows filtering by microservice. |
| | **Absent / Missing** | *(Empty)* | SDK uses defaults (`unknown_service:java`, etc.). | **Catalog Failure.** All apps merge into one "unknown_service" bucket. Impossible to distinguish logs. |
| | **Static String** | `my-app` | Same name for every deployment. | **Data Pollution.** Logs from different apps mixed together under one name. |
| **OTEL_RESOURCE_ATTRIBUTES** | **Hardcoded (Correct)** | `k8s.namespace=...,` | Adds metadata tags to every packet. | **Success.** `k8s.namespace` maps to **Application**. Enables correct routing/RBAC. |
| | **Absent / Missing** | *(Empty)* | Sends no custom tags. | **Routing Failure.** Logs fall into "Default" application. Hard to search or alert on specific envs. |
| | **Malformed** | `key=val,key2` | SDK drops the malformed string or pair. | **Partial Data Loss.** Namespace tags might be missing, breaking Coralogix views. |
