---
layout: post
title:  "Pitfalls of API design for data sync"
date:   2023-07-10 23:36:12 +0100
categories: jekyll update
comments: true
---
* Table of contents
{:toc}
# Scenario
## Background
You have a web app for displaying data. Its backed by a REST api. It probably looks something like:
```json
// GET /todo-items
[
    {
        "id" : 1,
        "title" : "Read more blogs",
        "created" : "2023-07-10 23:36:12",
        "updated" : "2023-07-10 23:36:12"
    }
]
```
And supports pagination via `offset` & `limit` (or any equivalent page based approach), perhaps `sort-by` and probably some filtering operations such as `id` and maybe `created-between`, `updated-between`.    
## Request
The request comes in to support integrating this todo app with something like JIRA or Trello. Basically someone wants every `TodoItem` from your webapp into their tool of choice. If your app has 12,345,523 items in it for a user, and the user enabled the integration, the system should sync the items across, resulting in exactly 12,345,523 items and then keep up to date with any changes.     

I'm hopefully going to explain why you should **not** use the existing api but create a new one.
# Issues with basic api for sync

## Offset & Limit

### Performance
Theres a number of articles that cover why offset and limit is bad for performance in detail so I won't rehash here. But think about the scenario where the items that are being synced are high volume. 
 What will the performance of `GET /items?offset=1,000,000&limit=100` be? We have to find and sort 1,000,100 items just for that page, and then find and sort 1,000,200 for the next etc.

### Whats the sort
If you are using some sort of pagination you must have a sort otherwise it doesn't make sense. So whats the sort?

#### **Created**
If the sort is `created` you have 2 issues.
1. The users aren't getting updates unless they resync the entire collection from the beginning every time. Thats not going to work for millions of items.
2. If you allow for `TodoItems` to be deleted your list will shift. For example:  
Assuming we have this data
```json
// GET /todo-items
[
    { "id" : 1, "title" : "Read more blogs" },
    { "id" : 2, "title" : "Buy a cat" },
    { "id" : 3, "title" : "Fix spelling in blog post" },
    { "id" : 4, "title" : "Book flights" }
]
```
We get our first page (page size 2 to make this less tedious)
```json
// GET /todo-items? offset=0 & limit=2
[
    { "id" : 1, "title" : "Read more blogs" },
    { "id" : 2, "title" : "Buy a cat" }
]
```
Now the user decides buying a cat before booking a flight is a bad idea so the user deletes the todo item `2`
```json
// DELETE /todo-items/2
```
Now our syncing software calls for the next page
```json
// GET /todo-items? offset=2 & limit=2
[
    { "id" : 4, "title" : "Book flights" }
]
```
And is only returned `"Book flights"`. Now we have a record that was deleted synced AND we are missing a record.  
We never received `{ "id" : 3, "title" : "Fix spelling in blog post" }` as that moved from the 3rd position to the 2nd.

#### **Updated**
If the sort is `updated` you have 2 issues.
1. If you allow for `TodoItems` to be deleted your list will shift. See above.
2. As the records are updated the sort changes also shifting the offset. For example:
Assuming we have this data
```json
// GET /todo-items
[
    { "id" : 1, "title" : "Read more blogs"             , "updated" : "2000-01-01 00:00:01" },
    { "id" : 2, "title" : "Buy a cat"                   , "updated" : "2000-01-01 00:00:02" },
    { "id" : 3, "title" : "Fix spelling in blog post"   , "updated" : "2000-01-01 00:00:03" },
    { "id" : 4, "title" : "Book flights"                , "updated" : "2000-01-01 00:00:04" }
]
```
We get our first page (page size 2 to make this less tedious)
```json
// GET /todo-items? offset=0 & limit=2 & sort-by=updated
[
    { "id" : 1, "title" : "Read more blogs" , "updated" : "2000-01-01 00:00:01"},
    { "id" : 2, "title" : "Buy a cat"       , "updated" : "2000-01-01 00:00:02"}
]
```
Now the user decides `"Read more blogs"` is a bad item and updates it to `*Write* more blogs`
```json
// PUT /todo-items/1
//Body
{
     "title" : "*Write* more blogs"
}
//Response
200
{ "id" : 1, "title" : "*Write* more blogs" , "updated" : "2000-01-01 00:00:05"}
```
Now our syncing software calls for the next page
```json
// GET /todo-items?offset=2&limit=2&sort-by=updated
[
    { "id" : 4, "title" : "Book flights"       , "updated" : "2000-01-01 00:00:04" },
    { "id" : 1, "title" : "*Write* more blogs" , "updated" : "2000-01-01 00:00:05"}
]
```
We never received `{ "id" : 3, "title" : "Fix spelling in blog post" }` as that moved from the 3rd position to the 2nd.

### Summary
Hopefully now you are persuaded that using offset and limit of a REST api for syncing data is a bad idea.

## Use date filtering
The next logical step is to therefore use date filtering, where we can use the updated date in ranges taking the last value.  
Given the data 
```json
// GET /todo-items
[
    { "id" : 1, "title" : "Read more blogs"             , "updated" : "2000-01-01 00:00:01" },
    { "id" : 2, "title" : "Buy a cat"                   , "updated" : "2000-01-01 00:00:02" },
    { "id" : 3, "title" : "Fix spelling in blog post"   , "updated" : "2000-01-01 00:00:03" },
    { "id" : 4, "title" : "Book flights"                , "updated" : "2000-01-01 00:00:04" }
]
```
We get our first page (page size 3 to make this less tedious)
```json
// GET /todo-items? limit=3 & sort-by=updated
[
    { "id" : 1, "title" : "Read more blogs"             , "updated" : "2000-01-01 00:00:01" },
    { "id" : 2, "title" : "Buy a cat"                   , "updated" : "2000-01-01 00:00:02" },
    { "id" : 3, "title" : "Fix spelling in blog post"   , "updated" : "2000-01-01 00:00:03" }
]
```
Now we take the last updated date to get the next page
```json
// GET /todo-items? limit=3 & sort-by=updated & updated-after=2000-01-01T00:00:03
[
    { "id" : 3, "title" : "Fix spelling in blog post"   , "updated" : "2000-01-01 00:00:03" },
    { "id" : 4, "title" : "Book flights"                , "updated" : "2000-01-01 00:00:04" }
]
```
Is the theory. This also has problems.

### Computers are fast
How many things can a computer do in the precision of time we are using? In the examples above we are returning dates with second precision. So how many `TodoItem`s could be updated in a second?  
What happens if its thousands? 
```json
[
    { "id" : 4, "title" : "Book flights"                , "updated" : "2000-01-01 00:00:04" },
    ...
    { "id" : 12495, "title" : "Stop creating TodoItems" , "updated" : "2000-01-01 00:00:04" }
]
```
In the example above we have many items all with the same updated date, maybe it was a bulk operation such as searching an email inbox and doing a 'select all' then 'mark as read'. It doesn't really matter how it can happen. Now when we attempt our paging we get either stuck in a loop if we include equality, or worse if we use a `>` operation we skip thousands of records. We could use `offset` & `limit` but see above for problems with that.  
**Solution**  
The solution is to add a secondary stable sort. For example:
```json
// GET /todo-items?limit=3&sort-by=(updated,id)
[
    { "id" : 1, "title" : "Read more blogs"             , "updated" : "2000-01-01 00:00:01" },
    { "id" : 2, "title" : "Buy a cat"                   , "updated" : "2000-01-01 00:00:02" },
    { "id" : 3, "title" : "Fix spelling in blog post"   , "updated" : "2000-01-01 00:00:03" }
]
```
Now we take the last updated date and its id to get the next page
```json
// GET /todo-items?limit=3&sort-by=(updated,id)&after=(2000-01-01T00:00:03, 3)
[
    { "id" : 4, "title" : "Book flights"                , "updated" : "2000-01-01 00:00:04" }
]
```
This way when thousands of records share the same updated date we are sorting and filtering on `id`

### Dates are hard
We now have a pattern for sorting and retrieving by datetime, handling duplicate datetimes. This will work for the bulk of our problem but we run into issues when we get to the end of our list of items.    
Dates are famously hard to synchronize across services and many applications are not doing so reliably. This can cause clocks to go backwards when they are synchronized. Consider this sequence of events:    
On the server:
```java
// Time on server is "2000-01-01 00:00:10"
createTodoItem(title="Times are hard" , updated="2000-01-01 00:00:10")
```

API is called by integration:
```json
// GET /todo-items?sort-by=updated & updated-after=2000-01-01T00:00:00
[
    { "id" : 42, "title" : "Times are hard"   , "updated" : "2000-01-01 00:00:10" }
]
```

On the server:
```java
// NTP runs to sync the clocks and finds this clock was too fast. So the time moves backwards.
// Time on server is "2000-01-01 00:00:08" (clock has gone backwards 2 seconds)
createTodoItem(title="Clocks can go back", updated="2000-01-01 00:00:08")
```

API is called by integration:
```json
// GET /todo-items?sort-by=updated & updated-after=2000-01-01T00:00:10
[]
```
There is no data returned because the last updated date seen is greater than the date that was subsequently inserted.

**Solution**  
We have a few options here.
1. Ensure that your clocks don't go backwards. This is hard, though it has been solved. Make sure to use this approach everywhere you are determining dates for this API.
2. Assume that clocks can go backwards and pad your calls, handling duplicate data. i.e. Subtract say 1 minute from the last seen record date when polling for the latest. This is bad for performance, and obviously only handles the clock going backwards by less than 1 minute. 

Option 1 is the only real choice but our problem is still not solved. Read on.

### Transactional visibility
We now have an API that allows compound sorting on updated and id, with accurate clocks that never go backwards. We can still run into issues when polling for the latest records.
```java
beginTx1();
createTodoItem(title="TodoItem transaction 1", updated="2000-01-01 00:00:01")

beginTx2()
createTodoItem(title="TodoItem transaction 2", updated="2000-01-01 00:00:02")

commitTx2()

//API call here

commitTx1()
```

If we have 2 parallel transactions adding `TodoItem`s we probably don't by default have any guarantee that they will commit in order. This means if the integration polls the api after tx2 commits but before tx1 commits (assuming read-committed or above) only the 2nd `TodoItem` will be returned.
```json
// GET /todo-items?sort-by=updated
[
    { "id" : 2, "title" : "TodoItem transaction 2"   , "updated" : "2000-01-01 00:00:02" }
]
```
The next poll will want data AFTER `"id" : 2` and will never see `"id" : 1`  
**Solution**  
Strictly order inserts so that the insert of item 2 cannot complete before item 1 either commits or does a rollback. Possibly by using a single thread or locking the table on write.

### Summary
In summary if we want to use date filtering to synchronize our data to a third party all we have to do is:
1. Support compound sorting and filtering
2. Use dates that never go backwards
3. Only allow 1 write at a time to our database to serialize all events
4. Get all of the above correct for the entire lifetime of the application.

Hopefully now you are persuaded that using date filtering on a REST api for syncing data is a bad idea.

# Closing Thoughts
An API that you have designed for retrieving a users data to be presented to a web app, mobile app or thick client is a poor API for data synchronization. Do yourself and your users a favour and write a sync api. 