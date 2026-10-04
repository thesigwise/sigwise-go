# SigWise API SDK for Go

Send events about your objects, get typed answers back.

Covers version 1.0.0 of the API. Full documentation, guides and the API reference:
<https://sigwise.ai/docs>.

## Get your API key and secret

1. Sign up at <https://sigwise.ai/register> (or sign in).
2. In the console, open **API keys** and click **New key**.
3. Copy both values it shows:
   - the **key ID** (for example `7Hx2Qp9LmZ`), which identifies the key;
   - the **signing secret**, which is **shown only once**. Store it like a password.

Pass them to the client, or set them in the environment, where the client
finds them on its own:

```bash
export SIGWISE_API_KEY=your_key_id
export SIGWISE_SECRET=your_secret
```

Lost the secret? **Rotate** the key on the same page to get a new one.

## Install

```bash
go get github.com/thesigwise/sigwise-go
```

## Quick start

```go
import sigwise "github.com/thesigwise/sigwise-go"

// Empty values fall back to SIGWISE_API_KEY and SIGWISE_SECRET.
client := sigwise.New("your_key_id", "your_secret")

// Configure what you want to know about your objects.
_, err := client.Signals.Upsert(ctx, "is_scammer", &sigwise.SignalInput{
	Type:         sigwise.SignalTypeNoul,
	Instructions: "Decide if this user is likely a scammer.",
	Criteria:     map[string]string{"true": "clear scam signals", "false": "legitimate behaviour"},
})

// Send events. Analysis runs in the background.
_, err = client.Events.Ingest(ctx, "user-42", &sigwise.IngestRequest{
	ObjectType: sigwise.String("user"),
	Events: []sigwise.EventInput{
		{Type: sigwise.EventTypeMessage, Content: sigwise.String("is this still available? can I pay by wire?")},
	},
})

// Read the latest answers.
obj, err := client.Objects.Get(ctx, "user-42")
for _, a := range obj.Analysis {
	fmt.Println(a.Key, a.Type)
}

// Or score inline and gate the content before publishing it.
res, err := client.Events.Ingest(ctx, "user-42", &sigwise.IngestRequest{
	Wait:    sigwise.Bool(true),
	Signals: []string{"is_scammer"},
	Events:  []sigwise.EventInput{{Type: sigwise.EventTypeMessage, Content: sigwise.String("pay me by wire and I double it")}},
})
if v := res.ModerationVerdict; v != nil && len(v.Answers) > 0 && v.Answers[0].Noul != nil && *v.Answers[0].Noul > 0.9 {
	// block it
}
```

## Authentication

The client sends the key ID with every request and signs a short-lived HS256
token with the secret, bound to the request's method and path. The secret
itself is never sent, so keep it on your server. Without explicit arguments
the client reads `SIGWISE_API_KEY` and `SIGWISE_SECRET` from the environment.

## Configuration

```go
client := sigwise.New("your_key_id", "your_secret",
	sigwise.WithHTTPClient(&http.Client{Timeout: 10 * time.Second}),
	sigwise.WithMaxRetries(3), // idempotent requests only
)
```

The client talks to the production API (`https://api.sigwise.ai`). Optionally, point it at
another deployment with `sigwise.WithBaseURL(...)` or the `SIGWISE_BASE_URL` environment variable.

Idempotent requests (`GET`, `PUT`, `DELETE`) are retried with exponential
backoff after a network error, a `429` or a `5xx`. Other requests are never
retried, so an event is never ingested twice.

## Errors

An error response raises an error carrying the HTTP status, the stable
machine-readable `code` (`not_found`, `payment_required`, …) and the message.

```go
obj, err := client.Objects.Get(ctx, "user-42")
var apiErr *sigwise.Error
switch {
case errors.As(err, &apiErr):
	fmt.Println(apiErr.StatusCode, apiErr.Code, apiErr.Message) // 404 not_found no data for this object
case err != nil:
	// *sigwise.ConnectionError: network error, timeout or cancelled context
}
```

## Webhooks

Verify every delivery before trusting it. The helper checks the
`X-Webhook-Signature` HMAC against the raw body and rejects timestamps older
than five minutes.

```go
func handle(w http.ResponseWriter, r *http.Request) {
	body, _ := io.ReadAll(r.Body) // the raw body, exactly as received
	ev, err := sigwise.ParseWebhookEvent(body,
		r.Header.Get("X-Webhook-Signature"), r.Header.Get("X-Webhook-Timestamp"),
		os.Getenv("SIGWISE_WEBHOOK_SECRET"), 0)
	if err != nil {
		http.Error(w, "invalid signature", http.StatusBadRequest)
		return
	}
	if e := ev.AnalysisCompletedEvent; e != nil {
		fmt.Println(e.ObjectID, len(e.Answers))
	}
}
```

## Reference

### me

The authenticated principal and tenant settings.

- `client.Me.Get(ctx context.Context) (*Me, error)`  
  `GET /v1/me`: Get the current principal

### overview

An object is anything you want answers about: a user, a listing, an order.

- `client.Overview.Get(ctx context.Context) (*Overview, error)`  
  `GET /v1/overview`: Get tenant overview

### objects

An object is anything you want answers about: a user, a listing, an order.

- `client.Objects.AnalyzeAll(ctx context.Context) (*AnalyzeAllScheduled, error)`  
  `POST /v1/analyze`: Re-analyze every object
- `client.Objects.List(ctx context.Context, params *ListObjectsParams) (*ObjectList, error)`  
  `GET /v1/objects`: List objects
- `client.Objects.Get(ctx context.Context, objectID string) (*ObjectAnalysis, error)`  
  `GET /v1/objects/{object_id}`: Get an object's analysis
- `client.Objects.Delete(ctx context.Context, objectID string) error`  
  `DELETE /v1/objects/{object_id}`: Delete an object's data
- `client.Objects.GetState(ctx context.Context, objectID string) (*ObjectState, error)`  
  `GET /v1/objects/{object_id}/state`: Get an object's compacted history
- `client.Objects.Analyze(ctx context.Context, objectID string) (*AnalyzeScheduled, error)`  
  `POST /v1/objects/{object_id}/analyze`: Re-analyze an object

### playground

Events and messages are the evidence an object's answers are computed from.

- `client.Playground.Run(ctx context.Context, body *PlaygroundRequest) (*PlaygroundResult, error)`  
  `POST /v1/playground`: Try signals on sample events

### events

Events and messages are the evidence an object's answers are computed from.

- `client.Events.Ingest(ctx context.Context, objectID string, body *IngestRequest) (*IngestEventsResponse, error)`  
  `POST /v1/objects/{object_id}/events`: Ingest events
- `client.Events.List(ctx context.Context, objectID string) (*EventList, error)`  
  `GET /v1/objects/{object_id}/events`: List an object's events

### signals

A signal is a question you ask about every object, such as "is this a scammer?" (`noul`), "how trustworthy is this user?" (`score`) or "what is their buyer intent?" (`choice`).

- `client.Signals.List(ctx context.Context) (*SignalList, error)`  
  `GET /v1/signals`: List signals
- `client.Signals.Upsert(ctx context.Context, key string, body *SignalInput) (*Signal, error)`  
  `PUT /v1/signals/{key}`: Create or update a signal
- `client.Signals.Delete(ctx context.Context, key string) error`  
  `DELETE /v1/signals/{key}`: Delete a signal
- `client.Signals.GetBackfill(ctx context.Context, key string) (*BackfillStatus, error)`  
  `GET /v1/signals/{key}/backfill`: Get backfill estimate and progress
- `client.Signals.StartBackfill(ctx context.Context, key string) (*SignalBackfill, error)`  
  `POST /v1/signals/{key}/backfill`: Start a backfill

### settings

The authenticated principal and tenant settings.

- `client.Settings.Get(ctx context.Context) (*Settings, error)`  
  `GET /v1/settings`: Get tenant settings
- `client.Settings.Update(ctx context.Context, body *SettingsUpdate) (*Settings, error)`  
  `PATCH /v1/settings`: Update tenant settings

### webhooks

Webhook endpoints receive a signed `POST` for the event types they subscribe to: `analysis.completed` each time an analysis completes, and `rule.triggered` when a rule with a webhook action fires.

- `client.Webhooks.List(ctx context.Context) (*WebhookEndpointList, error)`  
  `GET /v1/webhooks`: List webhook endpoints
- `client.Webhooks.Create(ctx context.Context, body *WebhookEndpointCreate) (*WebhookEndpointCreated, error)`  
  `POST /v1/webhooks`: Register a webhook endpoint
- `client.Webhooks.Get(ctx context.Context, endpointID string) (*WebhookEndpoint, error)`  
  `GET /v1/webhooks/{endpoint_id}`: Get a webhook endpoint
- `client.Webhooks.Update(ctx context.Context, endpointID string, body *WebhookEndpointUpdate) (*WebhookEndpoint, error)`  
  `PATCH /v1/webhooks/{endpoint_id}`: Update a webhook endpoint
- `client.Webhooks.Delete(ctx context.Context, endpointID string) error`  
  `DELETE /v1/webhooks/{endpoint_id}`: Delete a webhook endpoint

### webhookDeliveries

Webhook endpoints receive a signed `POST` for the event types they subscribe to: `analysis.completed` each time an analysis completes, and `rule.triggered` when a rule with a webhook action fires.

- `client.WebhookDeliveries.List(ctx context.Context, params *ListWebhookDeliveriesParams) (*WebhookDeliveryList, error)`  
  `GET /v1/webhook_deliveries`: List webhook deliveries
- `client.WebhookDeliveries.Replay(ctx context.Context, deliveryID string) (*WebhookDelivery, error)`  
  `POST /v1/webhook_deliveries/{delivery_id}/replay`: Replay a delivery

### rules

A rule fires an action (email, Slack or webhook) when an object's signal values cross a line you care about, such as `is_scammer >= 90`.

- `client.Rules.List(ctx context.Context) (*RuleList, error)`  
  `GET /v1/rules`: List rules
- `client.Rules.Create(ctx context.Context, body *RuleCreate) (*Rule, error)`  
  `POST /v1/rules`: Create a rule
- `client.Rules.Get(ctx context.Context, ruleID string) (*Rule, error)`  
  `GET /v1/rules/{rule_id}`: Get a rule
- `client.Rules.Update(ctx context.Context, ruleID string, body *RuleUpdate) (*Rule, error)`  
  `PATCH /v1/rules/{rule_id}`: Update a rule
- `client.Rules.Delete(ctx context.Context, ruleID string) error`  
  `DELETE /v1/rules/{rule_id}`: Delete a rule

### ruleFirings

A rule fires an action (email, Slack or webhook) when an object's signal values cross a line you care about, such as `is_scammer >= 90`.

- `client.RuleFirings.List(ctx context.Context, params *ListRuleFiringsParams) (*RuleFiringList, error)`  
  `GET /v1/rule_firings`: List rule firings

### apiKeys

Create, rotate and revoke the keys your integrations sign requests with.

- `client.APIKeys.List(ctx context.Context) (*APIKeyList, error)`  
  `GET /v1/api_keys`: List API keys
- `client.APIKeys.Create(ctx context.Context, body *APIKeyCreate) (*APIKeySecret, error)`  
  `POST /v1/api_keys`: Create an API key
- `client.APIKeys.Update(ctx context.Context, keyID string, body *APIKeyUpdate) (*APIKey, error)`  
  `PATCH /v1/api_keys/{key_id}`: Update an API key
- `client.APIKeys.Revoke(ctx context.Context, keyID string) error`  
  `DELETE /v1/api_keys/{key_id}`: Revoke an API key
- `client.APIKeys.Rotate(ctx context.Context, keyID string) (*APIKeySecret, error)`  
  `POST /v1/api_keys/{key_id}/rotate`: Rotate an API key's secret

### billing

Prepaid balance, the billing ledger, and card top-ups.

- `client.Billing.GetBalance(ctx context.Context) (*Balance, error)`  
  `GET /v1/billing`: Get the prepaid balance
- `client.Billing.ListLedger(ctx context.Context, params *ListLedgerParams) (*LedgerPage, error)`  
  `GET /v1/billing/ledger`: List ledger entries
- `client.Billing.CreateCheckout(ctx context.Context, body *CheckoutCreate) (*CheckoutSession, error)`  
  `POST /v1/billing/checkout`: Start a card top-up
- `client.Billing.GetCheckout(ctx context.Context, sessionID string) (*CheckoutStatus, error)`  
  `GET /v1/billing/checkout/{session_id}`: Get a top-up's status

### usage

Analysis volume, spend and token usage over a date range.

- `client.Usage.Get(ctx context.Context, params *GetUsageParams) (*Usage, error)`  
  `GET /v1/usage`: Get usage

