Backgroud:
Kudu currently does not support `append` semantics.  
To reach this goal in current implementation we can only make it happen in the application layer. 
It requires three steps. For instance, in java client
Use kuduClient to scan from the table with the given primary key, select visit_times from profile where id = 1, suppose the visit_times = 10 now.
visit_times++ in memory.
Send an update request to set visit_times = 11 where id = 1.

In step 1 and 3, it requires two scans, 
in 1 it scans explicit.
in 3 it scans implicit.

So, implementing in such an approach would introduce a risk if there were multi processes/threads is doing the same thing. E.g.
Process 1 load data with visit_times = 10
Process 2 load data with visit_times = 10
p1 then update it and increment by 5, p2 then update it and increment by 10.
The final visit_times would be 20,  the modification made by process 1 would lose then.

To avoid this risk, it requires to have only one thread to do data ingestion at a time.

Goal 
with append semantics, the requirement to increment/decrement would be implemented more elegantly. 
kuduClient send a append request to server, set visit_times++ where id =1
The cluster handles it and then returns, the client could receive the return status then.

For numeric columns, it could support increment/decrement.
For string columns, it could support increment(string concatenation).

Benefit
as mentioned in section ‘Background’, it reduces scans from twice(1 explicit 1 implicit) into only once implicit scan. And reduce the risk when multi processes/threads are introduced for data ingestion, in other words, it no longer requires the developer to have too much knowledge about kudu before using it, thus less probability for developers to make mistakes.

How
Basic Problem
Before starting coding, I'd rather have a more comprehensive understanding about the problem and kudu itself. The problem would be seperated to, 
How could the client use it?
After the client sends an ‘Append’ request to the server. How does the leader handle it? 
After the leader handles the request, how could the raft handle the ‘Append’ log handle with it. 
Edge cases and others:
Beside making it work, I can still figure some cases that would damage the system if it doesn’t could not be implemented in an appropriate approach.
Unlike update, append is an action, instead of a final state. It would be safe if the system called update twice, the stored value is correct. But it would be a disaster if append isI called it twice. BeingThe idempotent iswould be much more important in the ‘Append’ scenario.
Consider this case, a developer, he implemented a transaction with these steps in the ingestion program,
1. fetch a batch of data from kafka with given start offset, 
2. and then write these data to kudu, 
3. commit the offset back to step 1, as the start offset for next loop
In a kudu cluster without append, he considered merging these steps as a transaction. Benefiting from the deduplicating feature of kudu, his transaction could always work with correct results in these scenarios, the final result could always be correct.
step 1 -> step 2 -> system crashes -> restart -> step 1 -> step 2 -> step 3
step 1 -> step 2 -> system crashes -> restart -> step 1 -> crash -> restart -> step 1 -> step 2 -> step 3
step 1 -> step 2 -> step 3

But considering we introduced ‘Append’, the original purpose was saving him from the risk of race,  but it seems the transaction could no longer work, the ingestion model would be much more complicated. Are we still relieving him? Or would it be possible that this would introduce more trouble for a developer to chase the modification on records. 


Simple think along the dataflow:
In proto and client:
The increment/decrement on numeric can be treated as append (+/-) values, thus only a Append semantic is needed. Increment/decrement methods can be exposed to the developers with this detail hidden. 

message RowOperationsPB {
  enum Type {
    UNKNOWN = 0;
    INSERT = 1;
    UPDATE = 2;
    xxx
    xxx
    xxx    
    APPEND = 13;
….


void increase(int offset) {
    sendRpcToServer (appendRpc(id = 1, field = a_field, offset ) );
}
void decrease(int offset) {
    sendRpcToServer (appendRpc(id = 1, field = a_field, -offset ) );
}


In the Server:
To combat the first problem mentioned in ‘Edge cases and others’, I think it could have two solutions as I learned so far.
Persist the action into the wal. Then it would have some requirement:
Guarantee the wal log wouldn’t be replayed more than once in any situation. Also, there should be more append action specified UNDO/REDO manipulation for edge cases of raft. For example, the leader forces the follower to go back on its wal queue. 
When handling the append rpc, the leader could get the row with the field when checking the presence of the row with the given primary key. Then convert the Append to a normal update semantic in the wal log. Then it could transfer the ‘append semantic log’ from an action to a fixed state. 
I’d rather take the second approach. 
The first is kind of similar to mergeOperator of rocksdb, unlike rocksdb or some lsm implemented kv engine. Kudu has different semantics between ‘update’ and ‘insert’, so kudu has to check the presence of a row when updating/inserting, so it would only bring a little more overhead compared to implementing the update: just to go over all redo (delta memstore and redo blocks) and get the most recent value of value to be append.

The second problem in section ‘Edge cases and others’ may need more consideration. 

