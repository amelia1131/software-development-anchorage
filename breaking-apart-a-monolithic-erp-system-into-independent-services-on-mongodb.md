# Breaking Apart a Monolithic ERP System into Independent Services on MongoDB

As the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most challenging projects I've undertaken was the transformation of a legacy monolithic ERP system into a modern microservices architecture. The outdated monolith had become costly to maintain and was impeding my client's business innovation.

Fortunately, I've encountered this issue multiple times during our years of offering customized [Software Development Services in Anchorage](https://hybridwebagency.com/anchorage-ak/best-software-development-company/). I understand the difficulties organizations face when encumbered by rigid and hard-to-adapt systems. That's why I'm thrilled to present the step-by-step strategy we employed to overhaul the data model and disassemble the monolith, both in terms of code and the database structure.

Throughout this migration, my team and I unearthed some lesser-known best practices for modeling domain entities in a document database such as MongoDB. These practices are essential for supporting entirely independent microservices. If you've ever wondered how to future-proof your database architecture for the cloud while retaining historical data, you're in for a treat.

By the end of this article, you'll possess a clear plan for migrating your legacy systems to a modern architecture. I'll be sharing plenty of practical insights to help you steer clear of common pitfalls and accelerate the delivery of new features to your customers. Let's commence this journey!

## The Advantages of a Microservices Architecture

Implementing a microservices architecture offers several advantages over a monolithic approach. Services can be deployed independently, enabling the swift development and release of features without disrupting the entire application.

Furthermore, individual services can be developed using different programming languages and frameworks. This enables the selection of the best technologies for each domain. For instance, a recommendation engine might make use of Python's machine learning libraries, while the user interface is built using React.

The separation of concerns allows specialized teams to work autonomously on distinct services. Ideas can be rapidly validated through prototypes before committing to a complete monolithic rewrite. New team members can readily contribute to a specific service aligned with their expertise.

Technologies are constantly evolving. Microservices reduce the risk associated with these shifts, as any replacements affect only small, isolated portions of the system, rather than necessitating a complete rewrite of the entire monolith. Clear interfaces simplify the migration of components to new implementations.

#### Independent Scaling

Resource management is optimized when services can scale autonomously in response to demand. The frontend API gateway routes traffic based on URLs but can be securely deployed behind a load balancer to handle peak loads. For instance, during holiday seasons, only order processing requires additional servers, without impacting the entire system.

```python
ordersAPI.scale(replicas: 5)
```

Horizontal scaling at the service level leads to significant cost savings by precisely adjusting resource allocation. Support microservices that are idle, like user profiles, do not need expensive overprovisioning to manage traffic that does not affect them.

## Analyzing the Data Model
### Comprehending Entity Relationships

The initial step in transitioning to a microservices architecture is to analyze how entities are interrelated within the existing monolithic data model. We thoroughly examined each collection in the MongoDB database to identify areas where domains were clustered and transaction boundaries were defined.

Entities such as Users, Products, and Orders formed the central components of these bounded contexts. The relationships between these core entities became candidates for service decomposition. For example, we observed that Orders contained foreign keys to Users, representing the customers, and Products, representing the items purchased.

To gain a better understanding of cross-dependencies, we printed example documents to visualize connected fields. We noticed that the legacy code amalgamated data that had subsequently become relevant to distinct business capabilities. For instance, shipping addresses replicated user profiles, rather than maintaining a lightweight reference.

Analyzing these relationships exposed instances of questionable tight coupling between modules, resulting in cascading updates. The normalization of redundant data eliminated impediments to the autonomous development of user profiles and shipping namespaces.

Database tools proved invaluable for exploring these connections. With MongoDB Compass, we diagrammed relationships through $lookup pipelines and executed aggregate queries to count references between entities. This process revealed key breakpoints for dividing logic into coherent services.

Understanding these relationships helped establish domain boundaries and guaranteed that services offered clean interfaces. Well-defined contracts facilitated the independent development and deployment of modules as micro Frontends, without blocking each other.

### Identifying Transactional Boundaries

Beyond relationships, we also reviewed transactions within the existing codebase to comprehend the flows of business processes. This exercise identified areas where data modifications needed to occur within a single service to maintain data consistency and integrity.

In the context of order processing, we noticed that any updates related to the order itself, payments, inventory levels, and shipment notifications had to occur within a single service to maintain data consistency. This insight informed the delineation of our Order Management service boundary.

By examining both relationships and transactions, we gained invaluable insights for restructuring the data model and logic into independently deployable microservices with well-defined interfaces.

## Refactoring for Microservices

### Standardizing Data Schemas

To support independent services that could deploy to different data stores if necessary, we standardized schemas to eliminate redundancy and include only the essential data required by each service.

For instance, the original Orders schema contained the entire User object. We refactored this to a lightweight reference:

```python
# Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  #...
}

# After  
Orders: {
  userId: 1234
  #...  
}
```

Similarly, product details were removed from the Orders collection and placed in their own collections. This separation allowed these entities to evolve independently over time.

### Embracing Domain-Driven Design

We employed bounded contexts from Domain-Driven Design to logically separate services, such as Order Fulfillment and User Profiles. Interfaces were used to abstract data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

To accommodate the new architecture, queries and commands required refactoring. Previously, services accessed data directly using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```python
# order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

# mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This approach provided the flexibility to migrate databases without altering consumer code.

## Deploying Microservices

### Independent Scaling

In the realm of microservices, autoscaling takes place at the individual service level rather than at the application level. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests specified scaling policies based on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  


      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

Scaling tiers provided reserved capacity and elastic overflow management. The Orders service autonomously monitored itself and spawned or terminated containers as needed:

```python
# orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers such as Nginx and Traefik routed to scaled replica sets. This approach optimized resource utilization, enhanced throughput and reduced latency, all while reducing costs.

### Implementing Resilience

Other resiliency techniques included retry policies, timeouts, and circuit breakers. Rate limiting and throttling prevented cascading failures. The Platform service defined the transience policy for dependent services.

Both homemade and open-source solutions, such as Polly, Hystrix, and Resilience4j, offered additional protection. Centralized logging using Elasticsearch allowed us to trace errors across the distributed applications.

### Bolstering Reliability

Building reliability into microservices involved implementing various techniques to prevent single points of failure. We focused on automating responses to transient errors and overload protection.

The Resilience4J library provided circuit breakers to gracefully handle faults:

```python
# OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting prevented service flooding during periods of high stress:

```python
# RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were employed to abort long-running calls:

```python
# ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

Retry logic was implemented through policies defined at the client level:

```python
# RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return new RetryTemplate(policy);
  }

} 
```

These techniques ensured consistent responses and prevented cascading failures across services.

## Conclusion

The journey of migrating our monolithic ERP system to microservices has been an immense learning experience. More than just a technical migration, it represents an organizational transformation that will enable our client to better serve the evolving needs of their customers.

By dismantling tightly coupled layers and establishing clear boundaries between domains, our development team has unlocked a new level of agility. Features can now be developed and deployed independently based on business priorities, rather than architectural constraints. This ability to experiment and iterate rapidly will keep the application ahead of changing market demands.

Simultaneously, our operations team now enjoys complete visibility and control over each system component. Anomalous behavior can be detected early through improved monitoring of discrete services. Scaling and failovers are automated, replacing manual processes. This heightened resilience will provide a reliable foundation for the client's ongoing growth.

While the benefits of microservices are evident, such a migration is far from trivial. Taking the time to analyze relationships, define interfaces, and introduce abstraction, rather than opting for a crude 'rip and replace' approach, has resulted in a flexible architecture capable of evolving in line with customer needs.

Above all, I am grateful for the opportunity this project provided to work closely with a client on their digital transformation. Sharing our experiences and insights here is a way to pay it forward, enabling others to learn from our successes and mistakes. My hope is that it empowers more businesses to embark on the journey of modernization, with the ultimate rewards being enhanced customer experiences.
## References 

- MongoDB Documentation - Official documentation on data modeling, queries, deployment, and more: [Read more](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles: [Read more](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains: [Read more](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps readily deployable to the cloud: [Read more](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, cloud platforms: [Read more](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps: [Read more](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture: [Read more](https://kubernetes.io/docs/home/)
