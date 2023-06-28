---
layout: post
title:  "Data synchronisation patterns"
date:   2023-06-27 12:39:37 +0100
categories: jekyll update
---
# Scenario
You want to save some data to a database, usually a transactional db, and take additional actions. i.e. In psuedo code:
```java
TodoItem saveTodoItem(TodoItemSpec spec) {
    // begin transaction
    TodoItem item = todoItemRepo.save(spec)
    alertService.sendNewItemAddedAlert(item) // AMQP
    auditService.recordItemAdded(newAuditEntry(item)) // HTTP
    // end transaction
}
```
If we wrote the above code we have a few problems.
- If the transaction cannot commit for any reason we have made a http call and placed an item on to a queue.
- If the `auditService.recordItemAdded` throws an exception we still have a message on our AMQP broker.

We might be tempted to try and 'fix' the issue by changing the code to:
```java
TodoItem saveTodoItem(TodoItemSpec spec) {
    // begin transaction
    TodoItem item = todoItemRepo.save(spec)
    // end transaction
    alertService.sendNewItemAddedAlert(item) // AMQP
    auditService.recordItemAdded(newAuditEntry(item)) // HTTP
}
```
Possbily by writing the code as above or using commit hooks in our library / framework of choice.  
We still have issues however
- We might commit our transaction then shutdown our app, meaning we have an entry in the database with no audit record or alert sent
- If `alertService.sendNewItemAddedAlert` throws not only do we not create our audit but we probably throw an exception / return and error code implying the `saveTodoItem` failed when in fact it is there in the db
Lets now go over strategies to deal with this.

# Solutions
## Who gives a shit
Its important to note this isn't something that always needs to be solved. All solutions that actually sovle the problem in a reasonable manner increase the complexity. Maybe this code is called every minute doing the exact same thing and missing a run or two doesn't matter. Don't invent requirements of consistency where they do not exist. 
## Verify after publish
The idea behind this is to always publish addition information to queue with a delay. The listening application can then verify the data,
![Diagram of Application publishes to Queue while listener consumes from queue and calls to application to verify](/assets/img/2023-06-20-data-sync-patterns-1.png)
Application
```java
TodoItem saveTodoItem(TodoItemSpec spec) {
    // begin transaction
    TodoItem item = todoItemRepo.save(spec)
    alertService.sendNewItemAddedAlert(toAlert(item)) // AMQP
    auditService.recordItemAdded(newAuditEntry(item)) // AMQP
    // end transaction
}
```
Listener
```java
void handleNewItemAddedAlert(AlertSpec alertSpec) {
    sleep("60seconds")
    Optional<TodoItem> item = todoItemClient.getTodoItem(alertSpec.getTodoItemId());
    if (!item.exists()) {
        return
    }
    // Rest of logic
}
```
### Pros
- Simple to write assuming you are already using queues
- Plays nice with immutable data stores / event sourcing model (i.e. no updates / deletes)

### Cons
- Latency - You don't actually have to wait 60 seconds per message only 60 seconds since message was put on queue but even so, still waiting
- How to decide / set the timeout - In the example above I used 60 seconds but what if the transaction took longer than 60 seconds, messages would be consumed, assumed rolledback and thrown away then the transaction completes. Need to account for this somehow.
- Doesn't work naturally for updates only creates - A create can be verified at a later date, is it there or not, an in place update cannot be reliabley verified after the fact without additional tracking tables.

## State machines
You can use a state machine to track the state and pass data between them. 
Completely made up psuedo code syntax:
```java
void saveTodoItem(TodoItemSpec spec) {
    CreateTodoItem create = createTodoItemState(spec);
    transition (create) {
        case start -> context.record("item", todoItemRepo.save(spec))
        case makeAlert -> context.record("alert", toAlert(context.get("item")))
        case sendAlert -> alertService.sendNewItemAddedAlert(context.get("alert"))
        case sendAudit -> auditService.recordItemAdded(newAuditItem(context.get("item")))
    }
}
```
The key here is each operation is in its own state and some library or framework takes ownership of tracking the states and advancing, retrying where needed and working across app shutdowns / startups.
See Netflix Conductor, Temporal, Spring Statemachine for more info.
The state machine approach can also be done manually with no frameworks if the states are simple and you are using queues.
```java
void saveTodoItemFromQueue(TodoItemSpec spec) {
    // Walk through steps of saving items, createing alert, sending audit
    // Ack message from queue at end
}
```

### Pros
- Battle tested open source tooling exists for this
- Tooling often comes with good observability, ability to see how many in flight state machines at whats states with what data

### Cons
- State machine libraries are often a significant departure from the previous way of writing code. They can come with additional infrastructure requirements, somewhere to queue messages, track states etc.
- The 'state' of your application is now persisted between runs, previously only had to consider data that was stored in databases and on queues when updating, now when changing a statemachine is that a new version of the same state machine or a different machine. This is just some extra complexity to be aware of not a deal breaker.
- Care need to be paid to the invocation of the state machine. None of the patterns on this page can possibly work without idempotence *however* its easy to miss the state machine creation itself also needs to be made idempotent. 

### Notes
- This is a good solution if this is going to be a common problem, or you are generally heading down the path of orchestration over co-ordination.

## Change Data Capture / WAL Log Tailing
As in the example at the top of the page often there is a primary datastore that is being used, be it an RDBMS or NoSQL solution like Cassandra, HBase, Mongo etc. 
A common pattern (though not in every database) is to have a Write Ahead Log (WAL), or Journal that writes are written to. This is used normally used for replication and crash recovery.
This can be used to track the changes made.
Application
```java
TodoItem saveTodoItem(TodoItemSpec spec) {
    // begin transaction
    return todoItemRepo.save(spec)
    // end transaction
}
```
Change Data Capture Service
```java
void receiveWal(Wal wal) {
    List<ChangeEvent> events = parse(wal);
    emitEvents(events);
}
```
Listener
```java
void alertOnTodoItemCreate(ChangeEvent changeEvent) {
    if (changeEvent.type() == CREATE) {
        alertService.createAlert(createAlert(changeEvent.getObject()))
    }
}
```
### Pros
- Handles the transactionality for you in that transactions that dont commit are not in the WAL (in most cases other cases might record the absence of commit)

### Cons
- Requires a database with a journal or wal
- Requires understanding the structure of the journal or wal and keeping up to date with this (handled by open source libraries such as debezium)
- Only information that was stored is available

### Notes
- This is a great fit for problems like replicating data from one store to another for different quering such writing to RDBMS and then replicating to a column store or other form of storage.

## Transactional Outbox

## Queue first / Eventing
