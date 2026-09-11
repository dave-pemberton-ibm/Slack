# Slack Integration — Service Documentation

## `slack:sendToSlack`

**Package:** `Slack-Integration`  
**Namespace:** `slack`  
**Service:** `sendToSlack`  
**Visibility:** Public  
**Stateless:** Yes

### Description

Sends a text message to a Slack channel via an incoming webhook URL. The service sanitises the message, wraps it in the Slack JSON payload format, performs an HTTP POST to the provided webhook, and returns a status string indicating success or failure.

---

### Input Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | `String` | Yes | The text message to send to Slack. Any double-quote (`"`) characters are automatically replaced with single quotes before sending to ensure valid JSON. |
| `slackWebhookURL` | `String` | Yes | The fully-qualified Slack Incoming Webhook URL to POST the message to (e.g. `https://hooks.slack.com/services/...`). |

---

### Output Parameters

| Field | Type | Description |
|---|---|---|
| `status` | `String` | `"success"` if the HTTP POST returned a `200` response; `"Error sending message"` otherwise. |

---

### Flow Logic

```
1. Sanitise message
   MAP → pub.string:replace
     Replace all " (double-quote) characters in `message` with ' (single-quote)
     to prevent malformed JSON in the Slack payload.

2. Build JSON payload — prefix
   MAP → pub.string:concat
     Prepend the string '{"text":"' to `message`.

3. Build JSON payload — suffix
   MAP → pub.string:concat
     Append the string '"}' to `message`.
     Result: {"text":"<sanitised message>"}

4. Set Content-Type header
   MAP
     Set `headers`/`content-type` = "application/json"

5. HTTP POST
   INVOKE pub.client:http
     method  : POST
     url     : slackWebhookURL
     data    : message (the assembled JSON payload)
     loadAs  : stream

6. Evaluate response
   IF header/status != "200"
     → set status = "Error sending message"
   ELSE
     → set status = "success"

7. Pipeline cleanup
   MAP — drops all intermediate pipeline variables:
     url, method, loadAs, data, slackWebhookURL,
     slackMessage, headers, encodedURL, header, body
```

---

### Service Properties

| Property | Value |
|---|---|
| Allowed HTTP Methods | TRACE, HEAD, DELETE, POST, GET, OPTIONS, PUT, PATCH |
| Timeout Period | 60 s |
| Log On | Error only |
| Circuit Breaker | Disabled |
| Retry Attempts | 0 |
| Validate Input / Output | false / false |
| Cache Results | false |
| Auditing | Never |

---

### Usage Example

**Input pipeline:**
```
message         = "Deployment completed successfully"
slackWebhookURL = "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
```

**Output pipeline:**
```
status = "success"
```

---

### Notes

- Double-quote characters in `message` are replaced with single quotes prior to JSON serialisation. If your message must preserve double-quotes, pre-escape them before calling this service.
- The service does **not** re-throw on HTTP errors — it returns `"Error sending message"` as `status` instead. Callers should check the `status` value if error handling is required.
- No authentication beyond the webhook URL itself is applied; keep `slackWebhookURL` stored securely (e.g. in Integration Server configuration variables or an external secret store).
