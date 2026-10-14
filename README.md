# MQ Client ACE Project

A simple IBM App Connect Enterprise 13 demo application that exposes a REST API to **put** and **get** plain-text messages on an IBM MQ queue over a TLS-secured connection.

---

## Project structure

```
mqclient-ace/
├── MQClientAPI/                    # REST API project (ACE 13)
│   ├── .project
│   ├── .settings/
│   │   └── org.eclipse.core.resources.prefs
│   ├── restapi.descriptor
│   ├── openapi.json                # OpenAPI 3.0 definition
│   ├── MQClientAPI.esql            # ESQL compute modules
│   ├── putMessage.subflow          # POST /messages — puts to MQ queue
│   ├── getMessage.subflow          # GET  /messages — gets from MQ queue
│   └── gen/
│       └── MQClientAPI.msgflow     # Generated main message flow
│
└── MQClientPolicy/                 # ACE Policy project
    ├── .project
    ├── .settings/
    │   └── org.eclipse.core.resources.prefs
    ├── policy.descriptor
    └── MQEndpoint.policyxml        # MQ connection policy (TLS, no mutual auth)
```

---

## API operations

| Method | Path                   | Description                                  |
|--------|------------------------|----------------------------------------------|
| POST   | `/mqclientapi/v1/messages` | Put a plain-text message onto `DEV.QUEUE.1`  |
| GET    | `/mqclientapi/v1/messages` | Get the next message from `DEV.QUEUE.1`      |

### POST `/mqclientapi/v1/messages`

**Request**
```
Content-Type: text/plain

Hello from ACE
```

**Response 200**
```json
{ "status": "Message put successfully" }
```

### GET `/mqclientapi/v1/messages`

**Response 200** (message available)
```json
{ "message": "Hello from ACE" }
```

**Response 204** — queue is empty, no body returned.

---

## Configuration

### MQ connection — `MQClientPolicy/MQEndpoint.policyxml`

Update the placeholder values before deploying:

| Attribute              | Placeholder  | Replace with                     |
|------------------------|--------------|----------------------------------|
| `destinationQueueManagerName` | `QM1`  | Your queue manager name          |
| `queueManagerHostname` | `mqserver`   | Hostname or IP of the QM         |
| `listenerPortNumber`   | `1414`       | MQ listener port                 |
| `channelName`          | `DEV.APP.SVRCONN` | Server-connection channel name |
| `SSLProtocol`          | `ANY_TLS12_OR_HIGHER` | Keep or adjust cipher spec |

### TLS trust store

ACE uses the JVM trust store to validate the queue manager's self-signed certificate.  
Import the queue manager certificate into the ACE keystore before running:

```bash
# Export the QM self-signed certificate from the MQ server
# e.g. using runmqakm or the MQ Console, save it as qm.crt

# Import into the ACE integration server's JVM trust store
keytool -importcert \
  -alias qm1-cert \
  -file qm.crt \
  -keystore <ACE_WORKDIR>/config/iib-truststore.jks \
  -storepass <password>
```

On OpenShift, mount the certificate as a secret and configure the ACE IntegrationServer CR to reference it.  
See: [ACE on Containers — TLS configuration](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=app-connect-in-containers-continuous-delivery)

---

## Running locally

### Prerequisites
- IBM App Connect Enterprise 13 installed
- IBM MQ 9.x / 10.x queue manager accessible on the network
- Queue `DEV.QUEUE.1` defined on the queue manager
- Channel `DEV.APP.SVRCONN` defined with `SSLCAUTH(OPTIONAL)` (one-way TLS)
- ACE integration server initialised

### 1 — Import projects into ACE Toolkit

1. Open ACE Toolkit.
2. **File → Import → General → Existing Projects into Workspace**.
3. Select the `mqclient-ace/` folder — both `MQClientAPI` and `MQClientPolicy` will appear.
4. Click **Finish**.

### 2 — Build the BAR file

```bash
source "/opt/ibm/ace-13/server/bin/mqsiprofile"

ibmint package \
  --input-path mqclient-ace/MQClientAPI \
  --input-path mqclient-ace/MQClientPolicy \
  --output-bar-file MQClientAPI.bar
```

### 3 — Deploy

```bash
# Start an integration server (if not already running)
IntegrationServer --name TEST_SERVER --work-dir /tmp/TEST_SERVER &

# Deploy
curl -sk -X POST https://localhost:7600/apiv2/deploy \
  -H "Content-Type: application/octet-stream" \
  --data-binary @MQClientAPI.bar
```

### 4 — Test

**Put a message**
```bash
curl -X POST http://localhost:7800/mqclientapi/v1/messages \
  -H "Content-Type: text/plain" \
  -d "Hello from ACE"
```

**Get a message**
```bash
curl http://localhost:7800/mqclientapi/v1/messages
```

---

## Running on OpenShift 4.18

Deploy using the IBM App Connect Operator.  
Create an `IntegrationServer` custom resource referencing the BAR file from a ConfigMap or object storage.

Minimal CR example:
```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: IntegrationServer
metadata:
  name: mqclient-ace
spec:
  license:
    accept: true
    license: L-KSBM-C87FU2
    use: CloudPakForIntegrationNonProduction
  barURL:
    - "https://<storage>/MQClientAPI.bar"
  configurations:
    - mqclient-policy-config
  replicas: 1
  version: "13.0"
```

For the TLS trust store, create a `Configuration` CR of type `truststore` containing the queue manager certificate.

---

## Security notes

- TLS is **one-way** (server authenticates to the client only) — no mutual authentication.
- Only self-signed certificates are used in this demo.
- The `SSLProtocol` is set to `ANY_TLS12_OR_HIGHER` to require a minimum of TLS 1.2.
- No MQ user credentials are configured in the policy — add a `securityIdentity` and the corresponding ACE credentials if the channel requires authentication.
# ace-mq-client-app
