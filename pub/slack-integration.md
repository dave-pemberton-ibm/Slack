# Service Documentation: slack:sendToSlack

## Purpose
Sends a notification or message to a designated Slack channel using an incoming Slack Webhook URL. The service sanitizes quotation marks, constructs the JSON payload structure, executes an HTTP POST request to the webhook endpoint, and evaluates the response status.

## Service Signature

### Inputs
| Parameter | Type | Constraints | Description |
|---|---|---|---|
| `message` | `String` | — | The plain text message content to be sent to Slack. Any double quotes inside the string are sanitized to single quotes. |
| `slackWebhookURL` | `String` | — | The target Slack Incoming Webhook URL endpoint. |

### Outputs
| Parameter | Type | Description |
|---|---|---|
| `status` | `String` | The execution status of the delivery (`success` if HTTP status 200 is returned, or `Error sending message` for any non-200 response). |

## Flow Diagram
```mermaid
flowchart TD
    %% Styling
    classDef startEnd  fill:#e8f5e9,stroke:#388e3c,color:#1b5e20
    classDef mapStep   fill:#e3f2fd,stroke:#1976d2,color:#0d47a1
    classDef invoke    fill:#e8eaf6,stroke:#3949ab,color:#1a237e
    classDef decision  fill:#fff9c4,stroke:#f9a825,color:#4e342e

    %% Start
    A([Start: sendToSlack]):::startEnd

    %% Transformations
    A --> B[TRANSFORM pub.string:replace -- replace double quotes with single quotes]:::invoke
    B --> C[TRANSFORM pub.string:concat -- prepend JSON wrapper]:::invoke
    C --> D[TRANSFORM pub.string:concat -- append JSON wrapper]:::invoke
    D --> E[MAP: Set Content-Type header to application/json]:::mapStep

    %% HTTP Invoke
    E --> F[INVOKE pub.client:http -- POST payload to Slack Webhook URL]:::invoke

    %% Conditional Check
    F --> G{IF: header/status != '200'?}:::decision

    subgraph IF_BLOCK["IF: header/status != '200'"]
        H[MAP: Set status = 'Error sending message']:::mapStep
    end

    subgraph ELSE_BLOCK["ELSE"]
        I[MAP: Set status = 'success']:::mapStep
    end

    G -->|true| H
    G -->|false| I

    %% Cleanup and Finish
    H --> J[MAP: Drop temporary variables]:::mapStep
    I --> J
    J --> K([End: sendToSlack]):::startEnd
```

## Usage Examples

### Example 1 — Successful Notification
**Input:**
| Parameter | Value |
|---|---|
| `message` | `"Deployment completed successfully."` |
| `slackWebhookURL` | `"https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"` |

**Output:**
| Parameter | Value |
|---|---|
| `status` | `"success"` |

### Example 2 — Delivery Error
**Input:**
| Parameter | Value |
|---|---|
| `message` | `"Failed job alert"` |
| `slackWebhookURL` | `"https://hooks.slack.com/services/invalid/webhook/url"` |

**Output:**
| Parameter | Value |
|---|---|
| `status` | `"Error sending message"` |

## Implementation Details
1. **Sanitization:** Uses `pub.string:replace` to convert double quotes (`"`) in the message to single quotes (`'`) to ensure valid JSON formatting.
2. **JSON Assembly:** Sequences two `pub.string:concat` transforms to build the payload wrapper `{"text":"<message>"}`.
3. **Header Initialization:** Initializes the `headers` record with `content-type = "application/json"`.
4. **HTTP Dispatch:** Invokes `pub.client:http` using HTTP `POST` and streaming mode to send the JSON string payload to `slackWebhookURL`.
5. **Response Evaluation:** Evaluates `header/status`. If the response code is `200`, sets `status` to `"success"`; otherwise, sets `status` to `"Error sending message"`.
6. **Pipeline Hygiene:** Drops intermediate variables (`url`, `method`, `loadAs`, `data`, `slackWebhookURL`, `slackMessage`, `headers`, `encodedURL`, `header`, `body`).

## Error Handling
The service checks the returned HTTP status code from the Slack endpoint via branching (`IF (header/status!="200")`). Network-level connection failures or unreachable hosts during `pub.client:http` will raise standard service exceptions.

## Related Services
| Service | Purpose |
|---|---|
| `pub.string:replace` | Replaces double quotes with single quotes |
| `pub.string:concat` | Concatenates JSON prefix and suffix around message text |
| `pub.client:http` | Dispatches the HTTP POST request to the webhook URL |

## Notes
- All inputs and outputs conform to webMethods `String` pipeline conventions.
- Outbound network access to Slack webhook endpoints is required.
