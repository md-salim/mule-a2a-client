# mule-a2a-client

MuleSoft A2A Client — REST facade that connects to the **CI SF Account A2A Agent Server** (`mule-a2a-server`) and exposes clean HTTP endpoints for A2A agent interaction.

## Architecture

```
Consumer (Postman/App)
    ↓  REST  (port 8084)
mule-a2a-client
    ↓  JSON-RPC over HTTP  (port 8083)
mule-a2a-server  (CI SF Account Agent)
    ↓  MCP protocol
CI SF Account MCP Server  (Salesforce CRUD)
```

## Port Allocation

| App | Port |
|-----|------|
| ci-sf-account-sys (SAPI) | 8081 |
| mule-mcp-server | 8082 |
| mule-a2a-server | 8083 |
| **mule-a2a-client** | **8084** |

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/a2a-client/agent-card` | Retrieve the agent card (name, capabilities, skills) |
| `POST` | `/v1/a2a-client/messages` | Send a message to the agent (blocking — waits for task completion) |
| `GET` | `/v1/a2a-client/tasks` | List all A2A tasks |
| `GET` | `/v1/a2a-client/tasks/{taskId}` | Get a specific task by ID |
| `POST` | `/v1/a2a-client/tasks/{taskId}/cancel` | Cancel a running task |
| `POST` | `/v1/a2a-client/tasks/{taskId}/push-notification-configs` | Register a push notification callback URL |
| `GET` | `/v1/a2a-client/tasks/{taskId}/push-notification-configs` | List push notification configs for a task |

## Request / Response Examples

### Send Message
```http
POST http://localhost:8084/v1/a2a-client/messages
Content-Type: application/json

{
  "userMessage": "List all Salesforce accounts"
}
```
**Response:** A2A Task object with `status.state` and `artifacts` containing the Salesforce data.

```json
{
  "id": "task-uuid",
  "contextId": "ctx-uuid",
  "status": {
    "state": "TASK_STATE_COMPLETED",
    "message": {
      "role": "ROLE_AGENT",
      "parts": [{ "text": "Operation completed successfully." }]
    }
  },
  "artifacts": [
    {
      "artifactId": "uuid",
      "name": "get_accounts-result",
      "parts": [{ "data": { "accounts": [...] }, "mimeType": "application/json" }]
    }
  ]
}
```

### Get Agent Card
```http
GET http://localhost:8084/v1/a2a-client/agent-card
Accept: application/json
```

### Get Task by ID
```http
GET http://localhost:8084/v1/a2a-client/tasks/{taskId}?historyLength=10
Accept: application/json
```

### Cancel Task
```http
POST http://localhost:8084/v1/a2a-client/tasks/{taskId}/cancel
Content-Type: application/json
```

## Configuration

Properties in `src/main/resources/config.properties`:

| Property | Default | Description |
|----------|---------|-------------|
| `http.port` | `8084` | Listening port |
| `a2a.agent.host` | `localhost` | A2A server host |
| `a2a.agent.port` | `8083` | A2A server port |
| `a2a.agent.basePath` | `/v1/sf-account-agent` | A2A agent base path |
| `logging.category` | `com.centric.a2aclient` | Log4j2 category |

## Running Locally

```bash
# Prerequisites: mule-a2a-server must be running on port 8083
mvn mule:run -Dmule.env=dev
```

Or use VS Code Run button (`.vscode/launch.json` is pre-configured).

## How It Works

The A2A connector 2.0.0 provides **server-side** operations only. The client communicates with the agent using the **JSON-RPC over HTTP** protocol:

- All operations `POST` to `/v1/sf-account-agent/rpc` with standard JSON-RPC 2.0 envelope
- Agent card is fetched via `GET /v1/sf-account-agent/.well-known/agent-card.json`
- The `A2A-Version: 1.0` header is sent with every request