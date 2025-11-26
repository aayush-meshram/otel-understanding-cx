# 🔭 OpenTelemetry Integration Guide

**Target Systems:** Java, Node.js, Python  
**Observability Backend:** Coralogix  
**Infrastructure:** Kubernetes (Helm Deployed)

## 📖 Overview
We enforce a **"Zero-Config" policy** for application developers regarding telemetry infrastructure. 
We utilize a **DaemonSet Architecture** where an OpenTelemetry Collector Agent runs on every Kubernetes node. Application Pods send telemetry to their specific host node, which then forwards compressed data to Coralogix.

---

## 🏗️ DevOps Guide: The Immutable Configuration

**⚠️ WARNING:** The following configuration is **hardcoded** in the Base Helm Chart (`deployment.yaml`). 
Developers **cannot** override these values via `values.yaml`. This ensures consistent data flow and guarantees that logs, metrics, and traces are enabled by default.

### Hardcoded Environment Variables
The following variables are injected automatically at deployment time using the Kubernetes Downward API.

| Variable | Value / Logic | Purpose |
| :--- | :--- | :--- |
| `NODE_IP` | `status.hostIP` | Captures the physical Node IP. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://$(NODE_IP):4318` | Sends data to the local node agent. |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf` | Uses efficient binary encoding over HTTP. |
| `OTEL_LOGS/METRICS/TRACES_EXPORTER` | `otlp` | Enables all signals. |
| `OTEL_SERVICE_NAME` | `{{ .Chart.Name }}` | Maps to Coralogix **Subsystem**. |
| `OTEL_RESOURCE_ATTRIBUTES` | `k8s.namespace=...,deployment.environment=...` | Maps to Coralogix **Application**. |

### Header Propagation
To ensure distributed tracing works across services, the following headers are captured automatically by the infrastructure configuration:

*   `x-platform-id`
*   `x-regid`
*   `x-request-id`
*   `x-session-id`

**Exceptions:**
For applications requiring `x-user-crn`, add the following to the specific application's `values.yaml`:
```yaml
otel:
  captureUserCrn: true
  ```
## 👩‍💻 Developer Guide: Instrumentation

**Your Responsibility:** You do **not** need to configure endpoints, IPs, or ports. You only need to **install the libraries** in your Dockerfile.

### ☕ Java Applications

Java integration is handled via the Java Agent JAR.

**Dockerfile:**

```
FROM openjdk:17-slim
WORKDIR /app
COPY target/my-app.jar app.jar

# 1. Download the Agent
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar

# 2. Add -javaagent flag
ENTRYPOINT ["java", "-javaagent:/app/opentelemetry-javaagent.jar", "-jar", "/app/app.jar"]
```

### 🟢 Node.js Applications

Node.js requires the auto-instrumentation packages and the --require flag.

**1. Install Dependencies (package.json):**

```
npm install @opentelemetry/api @opentelemetry/auto-instrumentations-node
```

**2. Update Dockerfile:**
```
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install --production

# 3. Use --require to load instrumentation
CMD ["node", "--require", "@opentelemetry/auto-instrumentations-node/register", "app.js"]
```

### 🐍 Python Applications

Python requires the OTel distro and the instrument wrapper.

**1. Update requirements.txt:**

```
opentelemetry-distro
opentelemetry-exporter-otlp
opentelemetry-instrumentation-logging
```

**2. Update Dockerfile:**

```
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

# 3. Install hooks
RUN opentelemetry-bootstrap -a install

COPY . .

# 4. Wrap command (Logs Exporter is mandatory for Python)
CMD ["opentelemetry-instrument", "--logs_exporter", "otlp", "python", "app.py"]
```

----------

## 🔗 References

-   [OpenTelemetry Specification: OTLP via HTTP](https://www.google.com/url?sa=E&q=https%3A%2F%2Fopentelemetry.io%2Fdocs%2Fspecs%2Fotel%2Fprotocol%2Fexporter%2F)
    
-   [OpenTelemetry Java Agent](https://www.google.com/url?sa=E&q=https%3A%2F%2Fgithub.com%2Fopen-telemetry%2Fopentelemetry-java-instrumentation)
    
-   [Coralogix OpenTelemetry Integration](https://www.google.com/url?sa=E&q=https%3A%2F%2Fcoralogix.com%2Fdocs%2Fopentelemetry-using-opentelemetry-collector%2F)
