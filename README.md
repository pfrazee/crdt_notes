# CRDT notes

Excerpts collected from papers and articles published on the Web.



## Network design

From ["Key-CRDT Stores"](http://run.unl.pt/bitstream/10362/7802/1/Sousa_2012.pdf) (the Swiftcloud paper)

### Replication

#### Pessimistic vs. Optimistic Replication

> In pessimistic replication, the system simulates that there is a unique copy of the object despite all client requests to different replicas (one copy serializability). This method requires synchronization among replicas to guarantee that no stale data is retrieved. 

> In wide area networks this technique may not be adequate because the synchronization requirements would imply a great latency overhead. But, in local area networks, it can be acceptable because the latency to contact another replica is low.

> Optimistic solutions allow replicas to diverge. This means that clients can read/write different values, for the same object, if they contact different replicas. When using such level of consistency the application must be aware of this fact.

> These solutions are more adequate to large scale systems providing good fault-tolerance and responsiveness since every replica can reply to a read or write request without coordinating with others.

#### Active vs. Passive Replication

> In active replication (or synchronous), all replicas are contacted inside a transaction to get their values updated. This replication mechanism can be implemented using a peer-to-peer approach, where all replicas have the same responsibilities and can be accessed by any client. Alternatively, a primary replica may have the responsibility of coordinating all other replicas.

> Passive replication (or asynchronous) model assumes a replica will propagate the updates to the other replicas outside the context of a transaction. As usual, replicas must receive all the updates.

#### Operation Based vs. State Based replication

> There are two fundamental methods to propagate updates among replicas. In State based replication, updates contain the full object state (or in optimized versions, a delta of the state). In operation based, the updates contain the operations that modify the object and must be executed in all replicas. The size of an object is typically larger than the size of an operation. Transmitting the whole state of an object can introduce a large overhead in message size. On the other hand, if the number of operations is high it can be better to transmit the whole state instead of all operations. Also, it can be simpler to update the state of an object than applying the operations on it, as discussed in 2.1.5.

### Correctness Criteria

> Pessimistic replication systems usually impose strong correctness criteria such as linearizability or serializability. The concurrent execution of operations in a replicated shared object is said to be linearizable if operation appear to have executed atomically, in some sequential order that is consistent with the real time at which the operations occurred in the real execution.

> According to the definition, for any set of client operations, there is a virtual canonical execution against a virtual single image of the shared object and each client sees a view of the shared object that is consistent with that single image.

> The concurrent execution of operation in a replicated shared object is said to be serializable if operations appear to have executed atomically in some sequential order. With this definition, we must maintain the order of operations within a transaction, but concurrent transactions can be executed with different interleaves, as long as their outcome conforms to a serial execution.

> For optimistic replication, several weaker correctness properties have been defined:

> **Eventual convergence property**: copies of shared objects are identical at all sites if updates cease and all generated updates are propagated to all sites.

> **Precedence property**: if one update `Oa` causally precedes another update `Ob`, then, at each site, the execution of `Oa` happens before the execution of `Ob`.

> **Intention-preservation**: for any update `O`, the effect of executing `O` at all sites is the same as the intention of `O` when executed at the site that originated it, and the effect of executing `O` does not change the effect of non concurrent operations.

> Assuming eventual consistency, a client can get stale data from replicas but the system will be very responsive since any replica can deliver the request. This criteria is often adopted in systems susceptible to node failures, partition, or long message delays.



## Compensation, Costs, and Benefits

From ["Eventual Consistency Today: Limitations, Extensions, and Beyond"](http://queue.acm.org/detail.cfm?id=2462076)

> Programming around consistency anomalies is similar to speculation: you don't know what the latest value of a given data item is, but you can proceed as if the value presented is the latest. When you've guessed wrong, you have to compensate for any incorrect actions taken in the interim. In effect, compensation is a way to achieve safety retroactively—to restore guarantees to users. Compensation ensures that mistakes are eventually corrected but does not guarantee that no mistakes are made.

> As an example of speculation and compensation, consider running an ATM. Without strong consistency, two users might simultaneously withdraw money from an account and end up with more money than the account ever held. Would a bank ever want this behavior? In practice, yes. An ATM's ability to dispense money (availability) outweighs the cost of temporary inconsistency in the event that an ATM is partitioned from the master bank branch's servers. In the event of overdrawing an account, banks have a well-defined system of external compensating actions: for example, overdraft fees charged to the user. Banking software is often used to illustrate the need for strong consistency, but in practice the socio-technical system of the bank can deal with data inconsistency just as well as with other errors such as data-entry mistakes.

> An application designer deciding whether to use eventual consistency faces a choice. In effect, the designer needs to weigh the benefit of weak consistency B (in terms of high availability or low latency) against the cost C of each inconsistency anomaly multiplied by the rate of anomalies R:

> `maximize B-CR`

> This decision is, by necessity, application- and deployment-specific. The cost of anomalies is determined by the cost of compensation: too many overdrafts might cause customers to leave a bank, while too-slow propagation of status updates might cause users to leave a social network. The rate of anomalies—as seen before—depends on the system architecture, configuration, and deployment. Similarly, the benefit of weak consistency is itself possibly a compound term composed of factors such as the incidence of communication failures and communication latency.

### Compensation by Design

> Compensation is error-prone and laborious, and it exposes the programmer (and sometimes the application) to the effects of replication. What if you could program without it? Recent research has provided "compensation-free" programming for many eventually consistent applications.

> The formal underpinnings of eventually consistent programs that are consistent by design are captured by the CALM theorem, indicating which programs are safe under eventual consistency and also (conservatively) which aren't. Formally, CALM means consistency as logical monotonicity; informally, it means that programs that are monotonic, or compute an ever-growing set of facts (by, e.g., receiving new messages or performing operations on behalf of a client) and do not ever "retract" facts that they emit (i.e., the basis for decisions the program has already made doesn't change), can always be safely run on an eventually consistent store. (Full disclosure: CALM was developed by our colleagues at UC Berkeley). Accordingly, CALM tells programmers which operations and programs can guarantee safety when used in an eventually consistent system. Any code that fails CALM tests is a candidate for stronger coordination mechanisms.

> As a concrete example of this logical monotonicity, consider building a database for queries on stock trades. Once completed, trades cannot change, so any answers that are based solely on the immutable historical data will remain true. However, if your database keeps track of the value of the latest trade, then new information—such new stock prices—might retract old information, as new stock prices overwrite the latest ones in the database. Without coordination between replica copies, the second database might return inconsistent data.

> By analyzing programs for monotonicity, you can "bless" monotonic programs as "safe" under eventual consistency and encourage the use of coordination protocols (i.e., strong consistency) in the presence of non-monotonicity. As a general rule, operations such as initializing variables, accumulating set members, and testing a threshold condition are monotonic. In contrast, operations such as variable overwrites, set deletion, counter resets, and negation (e.g., "there does not exist a trade such that...") are generally not logically monotonic.

> CALM captures a wide space of design patterns sometimes referred to as ACID 2.0 (associativity, commutativity, idempotence, and distributed). Associativity means that you can apply a function in any order:

> `f(a,f(b,c)) = f(f(a,b),c)`

> Commutativity means that a function's arguments are order-insensitive:

> `f(a,b) = f(b,a)`

> Commutative and associative programs are order-insensitive and can tolerate message re-ordering, as in eventual consistency. Idempotence means you can call a function on the same input any number of times and get the same result:

> `f(f(x))=f(x) (e.g., max(42, max(42, 42)) = 42)`

> Idempotence allows the use of at-least-once message delivery, instead of at-most-once delivery (which is more expensive to guarantee). Distributed is primarily a placeholder for D in the acronym (!) but symbolizes the fact that ACID 2.0 is all about distributed systems. Carefully applying these design patterns can achieve logical monotonicity.

> Recent work on CRDTs (commutative, replicated data types) embodies CALM and ACID 2.0 principles within a variety of standard data types, providing provably eventually consistent data structures including sets, graphs, and sequences. Any program that correctly uses these predefined, well-specified data structures is guaranteed to never produce any safety violations.

> To understand CRDTs, consider building an increment-only counter that is replicated on two servers. We might implement the increment operation by first reading the counter's value on one replica, incrementing the value by one, and writing the new value back on every replica. If the counter is initially at 0 and two different users simultaneously initiate increment operations on separate servers, both users may read 0 and then distribute the value 1 to the replicas; the counter ends up with a value of 1 instead of the correct value of 2. Instead, we can use a G-counter CRDT, which relies on the fact that increment is a commutative operation—it doesn't matter in what order the two increment operations are applied, as long as they are both eventually applied at all sites. With a G-counter, the current counter status is represented as the count of distinct increment invocations, similar to how counting is introduced at the grade-school level: by making a tally mark for every increment then summing the total. In our example, instead of reading and writing counter values, each invocation distributes an increment operation. All replicas end up with two increment operations, which sum to the correct value of 2. This works because the replicas understand the semantics of increment operations instead of providing general-purpose read/write operations, which are not commutative.



## BOOM group

"Berkeley Orders Of Magnitude Project"

[Lots of papers and talks](http://boom.cs.berkeley.edu/papers.html)

### CALM Conjecture

> "Consistency as Logical Monotonicity"

> A program has an eventually consistent, coordination-free evaluation strategy iff it is expressible in (monotonic) Datalog.

(Source: Joseph M. Hellerstein talk)

### CRON Conjecture

> "Causality Required Only for Non-Monotonicity"

> Program semantics require causal message ordering if and only if the messages participate in non-monotonic derivations.

(Source: Joseph M. Hellerstein talk)



## Convergence

From ["A comprehensive study of Convergent and Commutative Replicated Data Types"](http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf), summarized:

There are two forms of replication: state-base and operation-based. These create two forms of CRDT: Convergent Replicated Data-Types, as in state-based replication; and Commutative Replicated Data-Types, as in ops-based replication.

A state-based CRDT (CvRDT) must...

 - Have a partial order to the values.
 - "Monotonically increase" in state, meaning a new state only ever succeeds the current state in the value's ordering.
 - Define a merge function ("least upper bound") which is idempotent and order-independent.

An ops-based CRDT (CmRDT) must...

 - Have a partial order to the operations.
 - Have a reliable broadcast channel which guarantees that operations are delivered in the partial order.
 - Define the operation such that concurrent ops commute, meaning they can yield a predictable result without clear precedence.

With these semantics, updates from nodes can converge deterministically.

### Relation between the two approaches

> State-based mechanisms (CvRDTs) are simple to reason about, since all necessary information is captured by the state. They require weak channel assumptions, allowing for unknown numbers of replicas. However, sending state may be inefficient for large objects; this can be tackled by shipping deltas, but this requires mechanisms similar to the op-based approach. Historically, the state-based approach is used in file systems such as NFS, AFS, Coda, and in key-value stores such as Dynamo and Riak.

> Specifying operation-based objects (CmRDTs) can be more complex since it requires reasoning about history, but conversely they have greater expressive power. The payload can be simpler since some state is effectively offloaded to the channel. Op-based replication is more demanding of the channel, since it requires reliable broadcast, which in general requires tracking group membership. Historically, op-based approaches have been used in cooperative systems such as Bayou, Rover, IceCube, Telex.



## Portfolio of basic CRDTs
 
From ["A comprehensive study of Convergent and Commutative Replicated Data Types"](http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf), summarized:

### Op-based Counter

One of the simplest types. The available operations are `increment` and `decrement`, which are guaranteed by the channel to be delivered. Because concurrent increments and/or decrements will commute, this is a correct CmRDT.

 - `inc + inc = incinc`
 - `dec + dec = decdec`
 - `inc + dec = noop`
 - `dec + inc = noop`

### State-based increment-only Counter (G-Counter)

> A state-based counter is not as straightforward as one would expect. To simplify the problem, we start with a Counter that only increments.

> Suppose the payload was a single integer and merge computes max. This data type is a CvRDT as its states form a monotonic semilattice. Consider two replicas, with the same initial state of 0; at each one, a client originates increment. They converge to 1 instead of the expected 2.

> Suppose instead the payload is an integer and merge adds the two values. This is not a CvRDT, as merge is not idempotent.

> We propose instead the construct of Specification 6 (inspired by vector clocks). The payload is vector of integers; each source replica is assigned an entry. To increment, add 1 to the entry of the source replica. The value is the sum of all entries. We define the partial order over two states `X` and `Y` by `X ≤ Y ⇔ ∀i ∈ [0,n−1] : X.P[i] ≤ Y.P[i]`, where `n` is the number of replicas. Merge takes the maximum of each entry. This data type is a CvRDT, as its states form a monotonic semilattice, and merge produces the LUB.

> This version makes two important assumptions: the payload does not overflow, and the set of replicas is well-known. Note however that the op-based version implicitly makes the same two assumptions.

> Alternatively, G-Set (described later, Section 3.3.1) can serve as an increment-only counter. G-Set works even when the set of replicas is not known.

### State-based PN Counter

> It is not straightforward to support decrement with the previous representation, because this operation would violate monotonicity of the semilattice. Furthermore, since merge is a max operation, decrement would have no effect.

> Our solution, PN-Counter basically combines two G-Counters. Its payload consists of two vectors: P to register increments, and N for decrements. Its value is the difference between the two corresponding G-Counters, its partial order is the conjunction of the corresponding partial orders, and merge merges the two vectors.

### Notes on non-negative counters

> Some applications require a counter that is non-negative; for instance, to count the remaining credit of an avatar in a P2P game.

> However, this is quite difficult to do while preserving the CRDT properties; indeed, this is a global invariant, which cannot be evaluated based on local information only. For instance, it is not sufficient for each replica to refrain from decrementing when its local value is 0: for instance, two replicas at value 1 might still concurrently decrement, and the value converges to −1.

> One possible approach would be to maintain any value internally, but to externalize negative ones as 0. However this is flawed, since incrementing from an internal value of, say, −1, has no effect; this violates the semantics required.

> A correct approach is to enforce a local invariant that implies the global invariant: e.g., rule that a client may not originate more decrements than it originated increments. However, this may be too strong.

> Note that one of the Set constructs (described later) might serve as a non-negative counter, using add to increment and remove to decrement. However this does not have the expected semantics: if two replicas concurrently remove the same element, the result is equivalent to a single decrement.

> Sadly, the remaining alternative is to synchronise. This might be only occasionally, e.g., by reserving in advance the right to originate a given number of decrements, as in escrow transactions.

### Registers

> A register is a memory cell storing an opaque atom or object (noted type X hereafter). It supports assign to update its value, and value to query it. Non-concurrent assigns preserve sequential semantics: the later one overwrites the earlier one. Unless safeguards are taken, concurrent updates do not commute; two major approaches are that one takes precedence over the other (LWW-Register), or that both are retained (MV-Register).

### Last-Writer-Wins Register (LWW-Register)

> Last-Writer-Wins Register (LWW-Register) creates a total order of assignments by associating a timestamp with each update. Timestamps are assumed unique, totally ordered, and consistent with causal order; i.e., if assignment 1 happened-before assignment 2 , the former’s timestamp is less than the latter’s. This may be implemented as a per-replica counter concatenated with a unique replica identifier, such as its MAC address.

Note: totally-ordered timestamps are not trivial to implement. Vector clocks, for instance, only provide a partial order, as differing values can be equivalent. (Consider, for instance, `<1,2>` vs `<2,1>`, which signify concurrent events on two nodes.) A weak but simple solution is to use the addresses of the nodes in the ordering (node at "A" precedes the node at "B"). This provides a deterministic answer, and thus a total ordering, but it is not semantically meaningful to the application.

> **State-based LWW-Register**. The type of the value can be any (local) data type X. The value operation returns the current value. The assign operation updates the payload with the new assigned value, and generates a new timestamp. The monotonic semilattice orders two values by their associated timestamp; merge procedure selects the value with the maximal timestamp. Clearly, this data type is a CvRDT.

> **Ops-based LWW-Register**. Operation assign generates a new timestamp at the source. Downstream, the update takes effect only if the new timestamp is greater than the current one. Because of the way timestamps are generated, this preserves the sequential semantics; concurrent assignments commute since, whatever the order of execution, only the one with the highest timestamp takes effect. LWW-Registers are ubiquitous in distributed systems. For instance, in a replicated file system such as NFS, type X is a file (or even a block in a file).

### Multi-Value Register (MV-Register)

> An alternative semantics is to define a LUB operation that merges concurrent assignments, for instance taking their union, as in file systems such as Coda or in Amazon’s shopping cart. Clients can later reduce multiple values to a single one, by a new assignment. Alternatively, in Ficus, merge is an application-specific resolver procedure.

CouchDB also uses an application-specific resolver.

> To detect concurrency, a scalar timestamp (as above) is insufficient. Therefore the state-based payload is a set of `(X, versionVector)` pairs (the op-based specification is left as an exercise to the reader). A value operation returns a copy of the payload. As usual, assign overwrites; to this effect, it computes a version vector that dominates all the previous ones. Operation merge takes the union of every element in each input set that is not dominated by an element in the other input set.

> As noted in the Dynamo article, Amazon’s shopping cart presents an anomaly, whereby a removed book may re-appear. The problem is that, MV-Register does not behave like a set, contrary to what one might expect since its payload is a set.

Clarifying that point: they include a figure which shows a write to `{3}` merging with a concurrent write of `{1,2}` and producing `{1,2,3}`. While the writes are concurrent in the system's view, they do have a total order for the user, and the write of `{3}` succeeds the write of `{1,2}`. Therefore, this would appear to the user as items 1 and 2 coming back into the cart.

This makes the MV-Register a poor choice for a cart; instead, a Set should be used.

### Sets

> Sets constitute one of the most basic data structures. Containers, Maps, and Graphs are all based on Sets.

> We consider mutating operations `add` (takes its union with an element) and `remove` (performs a set-minus). Unfortunately, these operations do not commute. Therefore, a Set cannot both be a CRDT and conform to the sequential specification of a set.

To illustrate the non-commutative property:

 - `add(a) + remove(a) = noop`
 - `remove(a) + add(a) = add(a)`

Therefore, a concurrent `add` and `remove` would be non-deterministic.

### Grow-Only Set (G-Set)

> The simplest solution is to avoid remove altogether. A Grow-Only Set (G-Set) supports operations `add` and `lookup` only. The G-Set is useful as a building block for more complex constructions.

> In both the state- and op-based approaches, the payload is a set. Since `add` is based on union, and union is commutative, the op-based implementation converges; G-Set is a CmRDT. 

> In the state-based approach, `add` modifies the local state by a union. We define a partial order on some states `S` and `T` as `S ≤ T ⇔ S ⊆ T` and the `merge` operation as `merge(S, T) = S ∪ T`. Thus defined, states form a monotonic semilattice and `merge` is a LUB operation; G-Set is a CvRDT.

### 2P-Set

> Our second variant is a Set where an element may be added and removed, but never added again thereafter. This is the Two-Phase Set (2P-Set). It combines a G-Set for adding with another for removing; the latter is colloquially known as the tombstone set.

> **State-based 2P-Set** The payload is composed of local set `A` for adding, and local set `R` for removing. The lookup operation checks that the element has been added but not yet removed. Adding or removing a same element twice has no effect, nor does adding an element that has already been removed. The merge procedure computes a LUB by taking the union of the individual added- and removed-sets. Therefore, this is indeed a CRDT.

> Note that a tombstone is required to ensure that, if a removed element is received by a downstream replica before its added counterpart, the effect of the remove still takes precedence.

> **Op-based 2P-Set** Consider now the op-based variant of 2P-Set. Concurrent adds of the same element commute, as do concurrent removes. Concurrent operations on different elements commute. Operation pairs on the same element `add(e) / add(e)` and `remove(e) || remove(e)` commute by definition; and `remove(e)` can occur only after `add(e)`. It follows that this data type is indeed a CRDT.

Note that `remove` must do a successful local `lookup` before executing to ensure that `remove` always follows an `add`.

> **U-Set** 2P-Set can be simplified under two standard assumptions. If elements are unique, a removed element will never be added again. If, furthermore, a downstream precondition ensures that `add(e)` is delivered before `remove(e)`, there is no need to record removed elements, and the remove-set is redundant. (Causal delivery is sufficient to ensure this precondition.)

> U-Set is a CRDT. As every element is assumed unique, adds are independent. A remove operation must be causally after the corresponding add. Accordingly, there can be no concurrent add and remove of the same element.

### LWW-element-Set

> An alternative LWW-based approach, which we call LWW-element-Set, attaches a timestamp to each element (rather than to the whole set). Consider add-set `A` and remove-set `R`, each containing `(element, timestamp)` pairs. To add (resp. remove) an element `e`, add the pair `(e, now())`, where now was specified earlier, to `A` (resp. to `R`). Merging two replicas takes the union of their add-sets and remove-sets. An element `e` is in the set if it is in `A`, and it is not in `R` with a higher timestamp: `lookup(e) = ∃ t, ∀ t 0 > t: (e,t) ∈ A ∧ (e,t0) / ∈ R)`. Since it is based on LWW, this data type is convergent.

As in the LWW-Register, this timestamp implies a total order. Riak uses a mechanism like this in their ORSWOT Map/Set, but does not have a totally-ordered clock; instead, they rely on a vector clock and some coordination methods, as explained later in this file.

### PN-Set

> Yet another variation is to associate a counter to each element, initially 0. Adding an element increments the associated counter, and removing an element decrements it. The element is considered in the set if its counter is strictly positive. An actual use-case is Logoot-Undo, a (totally-ordered) set of elements for text editing.

> However, as noted earlier, a CRDT counter can go positive or negative; adding an element whose counter is already negative has no effect. For some applications, this may be the intended semantics. for instance, in an inventory, a negative count may account for goods in transit. In others, this may be considered a bug.

> Although the semantics are strange, PN-Sets converge. An alternative construction due to Molli, Weiss and Skaf [private communication] is presented in Specification 14. To avoid the above add anomaly, add increments a negative count of `k` by `|k| + 1`; however this presents other anomalies, for instance where remove has no effect. Both these constructs are CRDTs because they combine two CRDTS, a Set and a Counter.

### OR Set

> The preceding Set constructs have practical applications, but are somewhat counter-intuitive. In 2P-Set, a removed element can never be added again; in LWW-Set the outcome of concurrent updates depends on opaque details of how timestamps are allocated.

> We present here the Observed-Removed Set (OR-Set), which supports adding and removing elements and is easily understandable. The outcome of a sequence of adds and removes depends only on its causal history and conforms to the sequential specification of a set. In the case of concurrent add and remove of the same element, add has precedence (in contrast to 2P-Set).

> The intuition is to tag each added element uniquely, without exposing the unique tags in the interface. When removing an element, all associated unique tags observed at the source replica are removed, and only those.

> If op-based, the payload consists of a set of pairs `(element, unique-identifier)`. A `lookup(e)` extracts element `e` from the pairs. Operation `add(e)` generates a unique identifier in the source replica, which is then propagated to downstream replicas, which insert the pair into their payload. Two `add(e)` generate two unique pairs, but lookup masks the duplicates.

> When a client calls `remove(e)` at some source, the set of unique tags associated with `e` at the source is recorded. Downstream, all such pairs are removed from the local payload. Thus, when `remove(e)` happens-after any number of `add(e)`, all duplicate pairs are removed, and the element is not in the set any more, as expected intuitively. When `add(e)` is concurrent with `remove(e)`, the add takes precedence, as the unique tag generated by add cannot be observed by remove.

> OR-Set is a CRDT. Concurrent adds commute since each one is unique. Concurrent removes commute because any common pairs have the same effect, and any disjoint pairs have independent effects. Concurrent `add(e)` and `remove(f)` also commute: if `e != f` they are independent, and if `e = f` the remove has no effect. We leave the corresponding state-based specification as an exercise for the reader. Since every add is effectively unique, a state-based implementation could be based on U-Set.

### Graphs

> A graph is a pair of sets `(V,E)` (called vertices and edges respectively) such that `E ⊆ V × V`. Any of the Set implementations described above can be used for to `V` and `E`.

> Because of the invariant `E ⊆ V × V`, operations on vertices and edges are not independent. An edge may be added only if the corresponding vertices exist; conversely, a vertex may be removed only if it supports no edge. What should happen upon concurrent `addEdge(u,v) || removeVertex(u)`? We see three possibilities: (i) Give precedence to `removeVertex(u)`: all edges to or from `u` are removed as a side effect. This it is easy to implement, by using tombstones for removed vertices. (ii) Give precedence to `addEdge(u,v)`: if either `u` or `v` has been removed, it is restored. This semantics is more complex. (iii) `removeVertex(u)` is delayed until all concurrent `addEdge` operations have executed. This requires synchronisation. Therefore, we choose Option (i). Our Spec. uses a 2P-Set for vertices (in order to have tombstones) and another for edges (since they are not unique).

> A 2P2P-Graph is the combination of two 2P-Sets; as we showed, the dependencies between them are resolved by causal delivery. Dependencies between `addEdge` and `removeEdge`, and between `addVertex` and `removeVertex` are resolved as in 2P-Set. Therefore, this construct is a CRDT.

### Add-only monotonic DAG

> In general, maintaining a particular shape, such as a tree or a DAG, cannot be done by a CRDT. Such a global invariant cannot be determined locally; maintaining it requires synchronisation.

> However, some stronger forms of acyclicity are implied by local properties, for instance a monotonic DAG, in which an edge may be added only if it oriented in the same direction as an existing path. That is, the new edge can only strengthen the partial order defined by the DAG; it follows that the graph remains acyclic. 

> Add-only Monotonic DAG is a CRDT, because concurrent `addEdge` (resp. `addBetween`) either concern different edges (resp. vertices) in which case they are independent, or the same edge (resp. vertex), in which case the execution is idempotent.

> Generalising monotonic DAG to removals proves problematic. It should be OK to remove an edge (expressed as a precondition on `removeEdge`) as long as this does not disrupt paths between distinct vertices. Unfortunately, this is not live.

They include an illustration which shows that a remove could conflict with a concurrent add. The invariant that "removal does not disrupt paths between distinct vertices" would have to be enforced globally.

### Add-Remove Partial Order data type

> The above issues with vertex removal do not occur if we consider a Partial Order data type rather than a DAG. Since a partial order is transitive, implicitly all alternate paths exist; thus the problematic precondition on vertex removal is not necessary. For the representation, we use a minimal DAG and compute transitive relations on the fly (operation `before`). To ensure transitivity, a removed vertex is retained as a tombstone. Thus, our Spec. uses a 2P-Set for vertices, and a G-Set for edges.

> We manage vertices as a 2P-Set. Concurrent addBetweens are either independent or idempotent. Any dependence between addBetween and remove is resolved by causal delivery. Thus this data type is a CRDT.

This is very similar in practice to a doubly-linked list. In their spec, the edges are effectively defined as `{prev, value, next}`, while the verticies are simply `value`. The verticies control membership; edges are never removed.

### Cooperative text editing

> Peer-to-peer co-operative text editing is a particularly interesting use case of an add-remove order. A text document is a sequence of text elements (characters, strings, XML tags, embedded graphics, etc.). Users sharing a document repeatedly insert a text element (`addBetween`) or remove one (`remove`). Using a CRDT for this ensures that concurrent edits never conflict and converge, even for users who remain disconnected from the network for long periods, as long as they eventually reconnect. Thus, the WOOT data structure for concurrent editing corresponds directly to the Add-Remove Partial Order.

> A Partial Order presents a difficulty, as text is normally sequential, but two concurrent inserts at the same position remain unordered. A total order, or sequence, does not have this drawback, and in addition can be implemented much more efficiently.

> A sequence for text editing (or just sequence hereafter) is a totally-ordered set of elements, each composed of a unique identifier and an atom (e.g., a character, a string, an XML tag, or an embedded graphic), supporting operations to add an element at some position, and to remove an element. We now study two different sequence designs. Such a sequence is a CRDT because it a subclass of add-remove total order.

### Replicated Growable Array

> The Replicated Growing Array (RGA) implements a sequence as a linked list (a linear graph). It supports operations `addRight(v,a)`, to add an element containing atom `a` immediately after element `v`. An element’s identifier is a timestamp, assumed unique and ordered consistently with causality, i.e., if two calls to now return `t` and `t'`, then if the former happened-before the latter, then `t < t'`. If a client inserts twice at the same position, as in `addRight(v,a); addRight(v,b)`, the latter insert occurs to the left of the former, and has a higher timestamp. Accordingly, two downstream inserts at the same position are ordered in opposite order of their timestamps. As in Add-Remove Partial Order, removing a vertex leaves a tombstone, in order to accommodate a concurrent add operation.

They illustrate an example in which timestamps are represented as a pair `(local-clock.client-UID)`.

### Continuous Sequence

> An alternative approach to maintaining a mutable sequence is to place its elements in the continuum. Spec. 20 specifies a sequence based on identifying elements in a dense identifier space such as `R`, i.e., where a unique identifier can always be allocated between any two given identifiers. Adding an element assigns it an appropriate identifier; identifiers are unique and totally ordered (and unrelated by causality).

> As noted above, this data structure is a CRDT because it is a subclass of Add-Remove Partial Order. More directly, concurrent adds commute because they occur at different positions in the continuum. Adding and deleting different elements commute because they are independent operations. Adding an element precedes removing it, and they will be applied downstream in that order, by the U-Set assumption of causal delivery.

> Its performance depends crucially on the implementation of identifiers and of `allocateIdentifierBetween`. Using real numbers would certainly be possible but costly.

> **Identifier tree** Instead, we represent the continuum using a tree. The first element is allocated at the root. Thereafter, it is always possible to create a new leaf `e` between any two nodes `n` and `m`, either to the right of `n` or to the left of `m`. To allocate a node `e` to the right of a node `n`:

>   (i) If `n` has a right sibling `m' ≤ m` and there exists a free unique tag `m''` such that `m < m'' < m'`, allocate `e` as `m''`.
>   (ii) Otherwise, if `n` has no right child, allocate `e` as the right child of `n`.
>   (iii) Otherwise, let `n'` be the leftmost descendant of `n`’s right child; clearly, `n < n'`. Recursively, allocate `e` to the left of `n'`.

> Allocating to the left of `m` is symmetric, substituting left for right and vice-versa.

> **Identifiers** A node identifier is a (possibly empty) sequence of pairs `(d1,u1) • ... • (d,um)`, one per level in the tree. At each level, `d` indicates the direction (0 for left child, 1 for right child), and `u` is a unique integer tag.

> The root node has the empty identifier. A child of some node n has identifier `m = n • (d,u)`. Siblings are ordered by their relative identifiers; thus siblings `m = n • (d,u)` and `m' = n • (d',u')` compare as `m < m' ⇔ d < d' ∨ (d = d' ∧ u < u')`. As the tree is traversed in in-order, a parent `n` is greater than its left children and less than its right children; i.e., n compares with its child `m = n • (d,u)` thus: `n < m ⇔ d = 0`.

> In summary, two identifiers `n` and `n'` compare as follows. Let `j ≥ 0` be the length of their longest common prefix: `n = (d1,u1) • ... • (dj,uj) • (dj+1,uj+1) • ... • (dj+k,uj+k)` and `n' = (d1,u1) • ... • (dj,uj) • (d'j+1,u'j+1) • ... • (d'j+k',u'j+k')`. Then: 

>   (i) If `k = 0` and `k' = 0`, the two identifiers are identical.
>   (ii) If `k = 0` and `k' > 0`, then `n'` is a descendant of `n`. It is a right descendant iff `d'j+1 = 1`, i.e., `n < n' ⇔ d'j+1 = 1`.
>   (iii) Symmetrically, if `k > 0` and `k' = 0` then `n < n' ⇔ dj+1 = 0`.
>   (iv) If `k > 0` and `k' > 0`, then either `n` and `n'` are siblings, or they descend from siblings. In both cases, they are ordered by the siblings’ relative identifiers: `n < n' ⇔ dj+1 < d'j+1` ∨ (dj+1 = d'j+1 ∧ uj+1 < u'j+1)`.

> **Experience** Two tree-based CRDTs designed for concurrent editing are Logoot and Treedoc, differing in the details. Logoot always allocates to the right, thus does not require `d`. Treedoc groups sequential adds from the same source into a compact binary tree with tombstones (no `u` part), and uses a sparse, unique tag for concurrent adds only.

> If the tree is well balanced, the identifier size adjusts to the size of the sequence, and operations have logarithmic complexity. Experiments with text editing show that over time the tree becomes unbalanced. Rebalancing the tree is a kind of garbage collection, which we discuss in the next section.


## Garbage Collection

Requires a guarantee of syncronisation. Once shared state is known, old state can be collected.

One option: vector clocks. Requires causal delivery of messages. Replicas share their current vector clocks (basically a network-wide ACK). Once a message has been acked by all replicas, it can be GCed.



## The ORSWOT in Riak 2.0

[Detailed reading on Riak's CRDTs](https://gist.github.com/russelldb/f92f44bdfb619e089a4d)

Riak uses a novel set type they call ORSWOT. From their code's documentation:

> An OR-Set CRDT. An OR-Set allows the adding, and removal, of elements. Should an add and remove be concurrent, the add wins.

> In this implementation there is a version vector for the whole set. When an element is added to the set, the version vector is incremented and the `{actor(), count()}` pair for that increment is stored against the element as its "birth dot". Every time the element is re-added to the set, its "birth dot" is updated to that of the `{actor(), count()}` version vector entry resulting from the add. When an element is removed, we simply drop it, no tombstones.

> When an element exists in replica A and not replica B, is it because A added it and B has not yet seen that, or that B removed it and A has not yet seen that? Usually the presence of a tombstone arbitrates. In this implementation we compare the "birth dot" of the present element to the clock in the Set it is absent from. If the element dot is not "seen" by the Set clock, that means the other set has yet to see this add, and the item is in the merged Set. If the Set clock dominates the dot, that means the other Set has removed this element already, and the item is not in the merged Set.

The Remove operation in Riak uses a "Context" object, which is a set of Observed tags (per the OR-set design). The client fetches the most recent tags with a get, then executes the remove with the Context attached. There are two purposes to this:

 1. The receiving replica may not have observed the same value-tags yet. If it hadn't observed any tags for the value, it would return an error saying "precondition failed" (not found, basically). Including the Context (Observed tag set) ensures that the replica is able to remove.

 2. The receiving replica may have observed *more* value-tags than the client has. Without the Context, the replica would delete more value-tags than the client had observed, causing a sort of "remove-wins" scenario. Since the OR-set is defined as "add-wins," this is bad behavior: it's better to include the Context to ensure the intended semantics.
