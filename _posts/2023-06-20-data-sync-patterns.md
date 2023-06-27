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
- Plays nice with immutable data stores (i.e. no updates / deletes)

### Cons
- Latency
- How to decide / set the timeout
- Doesn't work naturally for updates only creates

## State machines
## Change Data Capture
## Transactional Outbox
