---
description: "Use when publishing or consuming Azure Event Grid events, creating event handlers, or defining event schemas"
---

# Event Grid Conventions

## Event Schema (§6.4)

**Subject format:** `{MicroserviceDomainName}/{Entity}`
- Examples: `SalesOrder/Order`, `Inventory/Stock`, `WMS/Receipt`, `Adage/RouteCreated`

**Event type naming:** Past tense — `{ActionHappened}`
- Examples: `SalesOrderCreated`, `InventoryAllocated`, `RoutesCreated`, `TransferReceiptCompleted`
- **Never** imperative: `CreateSalesOrder` ❌

## Publishing Pattern (§6.2)

```csharp
// Publish after successful DB operation
try
{
    await _eventPublisher.PublishAsync(
        subject: "SalesOrder/Order",
        eventType: "SalesOrderCreated",
        data: evt);
}
catch (Exception eventEx)
{
    _logger.LogError(eventEx,
        "CRITICAL: Failed to publish event for {OrderNumber}", order.OrderNumber);
    // DO NOT throw — main operation succeeded
}
```

**Payload rules:**
- Include enough context for consumers to process (order number, customer, type, date)
- **Do not** include entire DTOs — consumers should call the microservice if they need full data
- Keep payloads small and focused

## Consumption Pattern (§6.3)

**The Rule:** Store event first, then process. Return success after storage.

```csharp
[Function("HandleSalesOrderCreated")]
public async Task Run(
    [EventGridTrigger] EventGridEvent evt,
    FunctionContext executionContext)
{
    // STEP 1: Store event for idempotency
    await _eventRepository.SaveEventAsync(new ProcessedEvent
    {
        EventId = evt.Id,
        EventType = evt.EventType,
        EventData = evt.Data.ToString(),
        ReceivedDate = DateTime.UtcNow,
        ProcessingStatus = "RECEIVED"
    });

    // STEP 2: Process event
    try
    {
        var data = JsonConvert.DeserializeObject<SalesOrderCreatedEvent>(evt.Data.ToString());
        await _processor.ProcessAsync(data);
        await _eventRepository.UpdateEventStatusAsync(evt.Id, "COMPLETED");
    }
    catch (Exception ex)
    {
        var logger = executionContext.GetLogger("HandleSalesOrderCreated");
        logger.LogError(ex, "Event processing failed");
        await _eventRepository.UpdateEventStatusAsync(evt.Id, "FAILED", ex.Message);
        // DON'T throw — return success to Event Grid (event is stored)
    }
}
```

**Key rules:**
- Store event **before** processing begins
- Return success to Event Grid **after** storage (even if processing fails)
- Handle processing errors **without throwing** after storage
- Implement **idempotency** — before processing, check whether the `EventId` has already been stored; if it has, skip processing and return success to prevent duplicate side effects

## Service Bus Throttling Pattern (§6.5)

This is one accepted pattern for handling high event volumes — other approaches (e.g., direct WebHook with internal queuing, batching, or back-pressure mechanisms) are also valid depending on the scenario.

Use the Service Bus pattern when an Event Grid topic may publish a high volume of events that the consuming service cannot process at the same rate. Instead of a direct WebHook subscription, Event Grid forwards events to an **Azure Service Bus queue**, and the service pulls messages off the queue at its own pace.

**Infrastructure:**
- Event Grid subscription `endpointType` is `ServiceBusQueue` (not `WebHook`)
- Consumer service polls the queue using `ServiceBusClient` / `ServiceBusProcessor` or an Azure Function `ServiceBusTrigger`
- The queue acts as a buffer — messages accumulate until the consumer has capacity

**Azure Function example:**
```csharp
[Function("HandleSalesOrderCreatedFromQueue")]
public async Task Run(
    [ServiceBusTrigger("salesorder-created-queue", Connection = "ServiceBusConnection")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions,
    FunctionContext executionContext)
{
    try
    {
        var evt = JsonConvert.DeserializeObject<SalesOrderCreatedEvent>(
            message.Body.ToString());
        await _processor.ProcessAsync(evt);
        await messageActions.CompleteMessageAsync(message);
    }
    catch (Exception ex)
    {
        var logger = executionContext.GetLogger("HandleSalesOrderCreatedFromQueue");
        logger.LogError(ex, "Failed to process message {MessageId}", message.MessageId);
        // Dead-letter after max delivery count is exhausted — do not complete
        await messageActions.AbandonMessageAsync(message);
    }
}
```

**Key rules:**
- Configure `maxDeliveryCount` on the queue so poison messages are dead-lettered automatically
- Monitor the dead-letter queue and set up alerts for unprocessed messages
- ARM parameter files must use `ServiceBusQueue` endpoint type — see `arm-parameters` instructions

## Oracle AQ → Event Grid Bridge

Adage PL/SQL code publishes events via `ADAGE.EVENTGRID.EVENTGRID_QUEUE_MSG`:
```sql
ADAGE.EVENTGRID.EVENTGRID_QUEUE_MSG('EventType', 'Subject', 'topic-name', l_json_data);
```
Events use `visibility := DBMS_AQ.on_commit` — they become visible only after the Oracle transaction commits.

## Cross-References

- **Durable Functions:** Orchestrators are often triggered by Event Grid events. See `azure-durable-functions` instructions for orchestrator patterns, retry configuration, and terminating exception handling.
- **ARM deployment:** When adding a new Event Grid subscription, update **all four** ARM parameter files (dev/qa/uat/prod) in the consuming project's `ARMTemplateParameterFiles/` directory. See `arm-parameters` instructions for the subscription JSON structure.
