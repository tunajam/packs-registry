# Azure Functions

Comprehensive guide for Azure Functions serverless development, triggers, bindings, and deployment.

## When to Apply

- Building serverless applications on Azure
- Choosing triggers and bindings
- Configuring function apps
- Implementing durable functions
- Optimizing cold start and performance
- Deploying and monitoring functions

## Project Structure

### JavaScript/TypeScript (v4 Model)
```
my-function-app/
├── src/
│   └── functions/
│       ├── httpTrigger.ts
│       ├── queueTrigger.ts
│       └── timerTrigger.ts
├── host.json
├── local.settings.json
├── package.json
└── tsconfig.json
```

### Python
```
my-function-app/
├── function_app.py
├── host.json
├── local.settings.json
└── requirements.txt
```

## HTTP Triggers

### TypeScript (v4 Model)
```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";

export async function httpTrigger(
  request: HttpRequest, 
  context: InvocationContext
): Promise<HttpResponseInit> {
  context.log(`Http function processed request for url "${request.url}"`);

  const name = request.query.get("name") || (await request.text()) || "World";

  return {
    status: 200,
    jsonBody: { message: `Hello, ${name}!` }
  };
}

app.http("httpTrigger", {
  methods: ["GET", "POST"],
  authLevel: "anonymous",
  handler: httpTrigger,
});
```

### Python
```python
import azure.functions as func
import json

app = func.FunctionApp()

@app.route(route="hello", methods=["GET", "POST"])
def hello(req: func.HttpRequest) -> func.HttpResponse:
    name = req.params.get("name") or req.get_json().get("name", "World")
    
    return func.HttpResponse(
        json.dumps({"message": f"Hello, {name}!"}),
        mimetype="application/json"
    )
```

### Route Parameters
```typescript
app.http("getUser", {
  methods: ["GET"],
  route: "users/{userId}",
  handler: async (request, context) => {
    const userId = request.params.userId;
    return { jsonBody: { userId } };
  },
});
```

## Queue Triggers

### Storage Queue
```typescript
import { app, InvocationContext } from "@azure/functions";

interface QueueMessage {
  orderId: string;
  items: string[];
}

export async function processOrder(
  queueItem: QueueMessage, 
  context: InvocationContext
): Promise<void> {
  context.log("Processing order:", queueItem.orderId);
  
  // Process the order
  for (const item of queueItem.items) {
    await processItem(item);
  }
}

app.storageQueue("processOrder", {
  queueName: "orders",
  connection: "AzureWebJobsStorage",
  handler: processOrder,
});
```

### Service Bus Queue
```typescript
app.serviceBusQueue("processMessage", {
  queueName: "myqueue",
  connection: "ServiceBusConnection",
  handler: async (message, context) => {
    context.log("Message:", message);
  },
});
```

## Blob Triggers

```typescript
app.storageBlob("processBlob", {
  path: "uploads/{name}",
  connection: "AzureWebJobsStorage",
  handler: async (blob: Buffer, context) => {
    context.log(`Blob name: ${context.triggerMetadata.name}`);
    context.log(`Blob size: ${blob.length} bytes`);
    
    // Process blob content
  },
});
```

## Timer Triggers (Scheduled)

```typescript
app.timer("dailyCleanup", {
  schedule: "0 0 0 * * *", // Midnight UTC daily
  handler: async (myTimer, context) => {
    context.log("Daily cleanup started");
    
    if (myTimer.isPastDue) {
      context.log("Timer is running late!");
    }
    
    await performCleanup();
  },
});
```

### CRON Expressions
```
# Format: {second} {minute} {hour} {day} {month} {day-of-week}

"0 */5 * * * *"     # Every 5 minutes
"0 0 * * * *"       # Every hour
"0 0 0 * * *"       # Daily at midnight
"0 0 9 * * 1-5"     # Weekdays at 9am
"0 0 0 1 * *"       # First of every month
```

## Output Bindings

### Return Queue Message
```typescript
interface OutputBinding {
  queueOutput: string;
}

app.http("createOrder", {
  methods: ["POST"],
  return: app.output.storageQueue({
    queueName: "orders",
    connection: "AzureWebJobsStorage",
  }),
  handler: async (request, context) => {
    const order = await request.json();
    
    // Return value goes to queue
    return {
      body: "Order created",
      jsonBody: order, // This goes to queue
    };
  },
});
```

### Multiple Outputs
```typescript
const queueOutput = app.output.storageQueue({
  queueName: "notifications",
  connection: "AzureWebJobsStorage",
});

app.http("processWithOutput", {
  methods: ["POST"],
  extraOutputs: [queueOutput],
  handler: async (request, context) => {
    const data = await request.json();
    
    // Set extra output
    context.extraOutputs.set(queueOutput, {
      type: "notification",
      message: "Processing complete"
    });
    
    return { jsonBody: { status: "ok" } };
  },
});
```

## Durable Functions

### Orchestrator
```typescript
import * as df from "durable-functions";

const activityName = "processItem";

df.app.orchestration("processOrderOrchestrator", function* (context) {
  const input = context.df.getInput();
  const results = [];
  
  // Sequential execution
  for (const item of input.items) {
    const result = yield context.df.callActivity(activityName, item);
    results.push(result);
  }
  
  return results;
});

// Activity
df.app.activity(activityName, {
  handler: async (input) => {
    // Process single item
    return { processed: input };
  },
});

// HTTP starter
app.http("startOrchestration", {
  methods: ["POST"],
  handler: async (request, context) => {
    const client = df.getClient(context);
    const input = await request.json();
    
    const instanceId = await client.startNew("processOrderOrchestrator", {
      input,
    });
    
    return client.createCheckStatusResponse(request, instanceId);
  },
});
```

### Fan-out/Fan-in Pattern
```typescript
df.app.orchestration("parallelProcessing", function* (context) {
  const items = context.df.getInput();
  
  // Fan-out: start all in parallel
  const tasks = items.map(item => 
    context.df.callActivity("processItem", item)
  );
  
  // Fan-in: wait for all
  const results = yield context.df.Task.all(tasks);
  
  return results;
});
```

### Human Interaction Pattern
```typescript
df.app.orchestration("approvalWorkflow", function* (context) {
  const input = context.df.getInput();
  
  // Send approval request
  yield context.df.callActivity("sendApprovalRequest", input);
  
  // Wait for external event (up to 7 days)
  const approved = yield context.df.waitForExternalEvent(
    "ApprovalEvent",
    new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  );
  
  if (approved) {
    yield context.df.callActivity("processApproved", input);
  } else {
    yield context.df.callActivity("processRejected", input);
  }
});
```

## Configuration

### host.json
```json
{
  "version": "2.0",
  "logging": {
    "logLevel": {
      "default": "Information",
      "Host.Results": "Error",
      "Function": "Information"
    },
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20
      }
    }
  },
  "extensions": {
    "queues": {
      "batchSize": 16,
      "maxPollingInterval": "00:00:02",
      "visibilityTimeout": "00:00:30",
      "maxDequeueCount": 5
    },
    "serviceBus": {
      "prefetchCount": 100,
      "messageHandlerOptions": {
        "maxConcurrentCalls": 32
      }
    }
  },
  "functionTimeout": "00:10:00"
}
```

### local.settings.json
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "ServiceBusConnection": "Endpoint=sb://...",
    "CosmosDBConnection": "AccountEndpoint=..."
  }
}
```

## Environment Variables

```typescript
// Access settings
const apiKey = process.env.API_KEY;
const connectionString = process.env.AzureWebJobsStorage;

// In Azure Portal or CLI
// az functionapp config appsettings set --name <app-name> --resource-group <rg> --settings "API_KEY=secret"
```

## Error Handling

```typescript
app.http("robustFunction", {
  methods: ["POST"],
  handler: async (request, context) => {
    try {
      const data = await request.json();
      const result = await processData(data);
      
      return { jsonBody: result };
    } catch (error) {
      context.error("Processing failed:", error);
      
      if (error instanceof ValidationError) {
        return {
          status: 400,
          jsonBody: { error: error.message }
        };
      }
      
      // Re-throw for retry (queue triggers)
      throw error;
    }
  },
});
```

## Performance Optimization

### Cold Start Reduction
```typescript
// Keep connections warm
import { CosmosClient } from "@azure/cosmos";

// Initialize outside handler (reused across invocations)
const cosmosClient = new CosmosClient(process.env.CosmosDBConnection);
const container = cosmosClient
  .database("mydb")
  .container("items");

app.http("queryItems", {
  handler: async (request, context) => {
    // Reuses warm connection
    const { resources } = await container.items
      .query("SELECT * FROM c")
      .fetchAll();
    
    return { jsonBody: resources };
  },
});
```

### Premium Plan Benefits
- Pre-warmed instances (no cold start)
- VNet integration
- Unlimited execution duration
- More powerful instances

## Deployment

### Azure CLI
```bash
# Create function app
az functionapp create \
  --name my-function-app \
  --resource-group my-rg \
  --storage-account mystorageaccount \
  --consumption-plan-location eastus \
  --runtime node \
  --runtime-version 18 \
  --functions-version 4

# Deploy
func azure functionapp publish my-function-app

# Set app settings
az functionapp config appsettings set \
  --name my-function-app \
  --resource-group my-rg \
  --settings "API_KEY=secret"
```

### GitHub Actions
```yaml
name: Deploy Function App

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
      
      - name: Install and Build
        run: |
          npm ci
          npm run build
      
      - name: Deploy to Azure
        uses: Azure/functions-action@v1
        with:
          app-name: my-function-app
          package: .
          publish-profile: ${{ secrets.AZURE_FUNCTIONAPP_PUBLISH_PROFILE }}
```

## Monitoring

### Application Insights
```typescript
import { app } from "@azure/functions";

app.http("trackedFunction", {
  handler: async (request, context) => {
    // Automatic tracking
    context.log("Processing request");
    
    // Custom telemetry
    context.trace({
      message: "Custom trace",
      severityLevel: "Information"
    });
    
    return { jsonBody: { status: "ok" } };
  },
});
```

### Query Logs (KQL)
```kusto
// Function invocations
requests
| where cloud_RoleName == "my-function-app"
| where timestamp > ago(1h)
| summarize count() by name, resultCode

// Failures
exceptions
| where cloud_RoleName == "my-function-app"
| where timestamp > ago(1h)
| order by timestamp desc

// Duration analysis
requests
| where name == "httpTrigger"
| summarize avg(duration), percentile(duration, 95), max(duration) by bin(timestamp, 5m)
```

## Common Gotchas

### Timeout Limits
| Plan | Default | Maximum |
|------|---------|---------|
| Consumption | 5 min | 10 min |
| Premium | 30 min | Unlimited |
| Dedicated | 30 min | Unlimited |

### Concurrency
- HTTP triggers: Concurrent by default
- Queue triggers: Controlled by `batchSize` and `maxDequeueCount`
- Use Durable Functions for complex workflows

### Idempotency
- Functions may retry on failure
- Design handlers to be idempotent
- Use unique IDs to detect duplicates

## References

- [Azure Functions Documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)
- [Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/)
- [Best Practices](https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices)
- [Triggers and Bindings](https://learn.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings)
