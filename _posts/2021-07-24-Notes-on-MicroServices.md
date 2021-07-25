#### Introduction
* Issues with monolithic architecture were specific to the product.
* When we move from monolithic architecture to microservices, we get a new set of problems that are more generic (load balaning, service discovery etc.). These new set of problems, have well documented frameworks/paradigms to tackle them.

#### MicroServices
* **Database per service model**
  * Each service has its own database. 
  * A service cannot connect to the database of another service directly, only through the other service.
* This brings up the question, "*How do we manage data movement between multiple services*"?

#### Communication patterns between microservices
* **Sync communication between services**
  *  If an aggregation service does a series of serialized and blocking requests to other microservices, the entire request (from client to aggregation service) is only as fast as the slowest request.
  * For example, an aggregation service that shows product details of a specific user, will have to connect to `UserService`, `PurchaseService` and ``ProductService`, sequentially.
  * The entire request can fail if one or more of the microservices are unavailable.
* **Event based communication between services**
  *  All services use an event bus to publish and receive events/notifications.
  * Event bus becomes a single point of failure, so event bus needs to be resilient.
  * In our example, an aggregation service publishes an event on the event bus (eg `UserProductQuery` with some data). The event bus routes this event to all microservices that have registered for this event (eg `UserService`, `ProductService` and `PurchaseService`).
  * All the subscribed microservices, when they get the `UserProductQuery` execute their business logic, and emit another event to the event bus. Eg `UserQueryResult`, `ProductQueryResult` and `PurchaseQueryResult`
  * The aggregation service needs to `Map/Reduce` the individual outputs from all results.
  * This way requests to the microservices could be parallelized.
  * This still has similar issues like *Sync* where one micro service going down will affect the entire request.
* **Database sync between services**
  * The ideal scenario is for the aggregation service to have its own database with the relevant fields from the databases of other services (ie `ID->User`  from `UserService`, `Id->ProductID` from `PurchaseService` etc.) so that it can serve requests independent of the other services.
  * That means, every time these microservices get an update, the aggregation service also needs to somehow update it's own database.
  * The way this is managed is as follows:
    * When a microservice updates it's own database, eg `ProductService` adds a new product into it's database, it also sends an event on the event bus, eg `ProductCreatedEvent`.
    * The event bus will route this event to the services that are interested in this event. In our case, the aggregation service can listen for `ProductCreated` and update it's own database.

#### Fault-tolerance and resilient communication patterns
* Communication between microservices can fail if one of the microservices goes down or is busy. The easiest way to handle this problem is to **scale up** the microservices.
* Relication/scale can help when microservice is down, but what if a microservice is slow (lets say because it connects to a slow external DB service).
* **Thread per request**: For a service, each API request should be handled in a separate thread. The reason for having independent thread is that one API workflow could be slow, because it is calling an slow external DB for example, and could affect responses for other APIs that might otherwise have been fast. Having independent threads makes sure that the `main` thread is still be free to handle new queries.
* The issue with the above approach is that multiple "slow" API queries could result in multiple threads being spawned simultaneously. This would result in higher resource consumption by these threads, and will affect the response for "faster" APIs even though we are using independent threads.
* Ideally, a thread per request model should go hand in hand with timeouts. So that no threads is long living. Once the timeout expires, the thread is cleared and an error is returned.
* This only *partly* solves the problem. If say the timeout is 15 sec, and we receive the slow API request once every 5 sec. Then every 15 sec, one thread is being cleared and 3 are getting spawned. Very quickly, we end up having the same issue of too many "slow" API threads hogging system resources.
* The microservice needs to be smart enough to detect the thread timeouts and throttle the queries to the slow external DB; and hence create lesser number of threads for the "slow" API request it gets. ("Deactivate" the problem so that it doesn't affect other downstream components. **Circuit breaker pattern**)
* **When does the circuit untrip**: The slow external DB connection needs to be retried after a certain period of time. Circuit breaker libraries like Netflix's **Hystrix** have this circuit breaker logic that can be used by services using the annotations Hystrix provides.
* **Bulkhead pattern**: Just like bulkheads in a ship prevent a leakage from cascading and drowning the ship, bulkhead pattern prevents a "slow" API to keep creating multiple threads. Each type of API request has a maximum limit of threads that it can create. This way even if "slow" API consumes all of it's threads in the thread pool, the other APIs still have threads to schedule. Hystrix can be used to implement this Bulkhead pattern as well, by setting the `threadPoolProperties` in the `@HystrixCommand` annotation.
* The same issue of resource exhaustion affects services with multiple consumers. A large number of requests originating from one client may exhaust available resources in the service. Other consumers are no longer able to consume the service, causing a cascading failure effect.
