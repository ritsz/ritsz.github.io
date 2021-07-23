### MicroServices
* **Database per service model**
  * Each service has its own database. 
  * A service cannot connect to the database of another service directly, only through the other service.
* This brings up the question, "*How do we manage data movement between multiple services*"?

#### Communication patterns between microservices
* **Sync communication between services**
  *  If an aggregation service does a series of serialized and blocking requests to other microservices, the entire request (from client to aggregation service) is only as fast as the slowest request.
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
  * The ideal scenario is for the aggregation service to have its own database with the relevant fields from the databases of other services (ie `ID->User`  from `UserService`, `Id->ProductID` from `PurchaseServicce` etc.)
  * That means, every time these microservices get an update, the aggregation service also needs to somehow update it's own database.