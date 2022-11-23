# Chapter 1: Reliable, Scalable, and Maintainable Applications

## Thinking About Data Systems
You can view *data systems* as an umbrella term to represent the tools typically used to manage 
the movement, processing, and storage of data. Typically, these tools can be broken up into further 
categories (databases, message queues, caches, data processing frameworks, etc.). However, it 
makes sense to reason about all of these tools together for now because:

- The traditional line between these tools is blurring (e.g. one tool might do things 
  traditionally reserved for another category of tools)
- The high-level objectives we are trying to achieve with these systems is the same

This chapter discusses three of the most important objectives that such a system aims to achieve:

- **Reliability**: Work correctly in a fault-tolerant manner (e.g. robust to hardware + software 
  failures, user errors)
- **Scalability**: Perform as parameters (e.g. data or traffic volume) grow
- **Maintainability**: Evolve in response to changing requirements, customers, and developers

## Reliability
Faults can either be irreversible (e.g. data leaked to hacker) or reversible (e.g. hardware goes 
down). A reliable system aims to prevent the former and be resilient to the latter (while 
minimizing the probability of such faults occurring as well, but such faults will inevitably 
occur). Strategies to maintain reliability include:

- Thoughtful interfaces, abstractions, and APIs to minimize the likelihood of user error
- Thorough unit and integration testing
- Sandboxed environments for people to interact with the system without affecting production 
  users / data
- Data quality / hygiene checks (e.g. checking that outbound records from upstream service 
  equals incoming records in downstream service)  
- Self-imposed stress tests (e.g. the [Netflix Chaos Monkey](https://netflix.github.io/chaosmonkey/))
- Properly calibrated telemetry (i.e. monitoring and alerting)
- Controlled rollouts of new software  
- Simplified rollback process

## Scalability

### Describing load and performance
Scalability is about designing systems that perform in response to an increased load. This 
requires us to be able to describe the system load (i.e. *load parameters*) and its 
corresponding performance.

Common load parameters include:

- Requests per second
- Ratio of reads to writes in a database
- Concurrent users
- Cache hit rate

Common performance metrics include:

- **Throughput**: Number of records processed per unit of time, relevant for batch / offline 
  processing systems
- **Latency**: Duration that a request is waiting to be handled 
- **Service time**: Duration to process the request. This performance metric can degrade as a 
  service becomes dependent on more and more services, as the overall service time increases if 
  even one dependency begins degrading. This phenomenon is known as *tail latency amplication*. 
- **Response time**: Duration that the user waits to get a response from the request. Differs 
  from service time in that it also includes the effect of queued requests (i.e. *head-of-line 
  blocking*).
  
When designing a system, you need to come up with reasonable expectations on load parameters and 
construct a design that meets performance expectations in response to those parameters. You may 
also want to optimize for different levels in the performance metric distribution - e.g. 
optimizing for the median vs tail-end cases.

### Approaches for coping with load

Two general approaches:

- **Vertical scaling**: Move to a more powerful machine, also known as *scaling up*
- **Horizontal scaling**: Distribute the load across more machines, also known as *scaling out*

In practice, systems usually involve a mixture of both approaches - distributed processing is 
commonplace, but each node may be scaled up to maintain good performance. 

Distributed processing can be further organized based on resource / data sharing:

- **Shared-nothing architecture**: Each node has its own resources. For service-oriented 
  architectures, this also means that each service has its own database. Shared-nothing 
  architectures are more fault-tolerant (any failure is constrained to a specific node, and not 
  the entire application) and are easy to scale horizontally, since a resource need not be 
  shared with any new nodes. However, they require applications to manage load balancing and 
  replication logic.
- **Shared-everything architecture**: Nodes share a common resource, such as hardware or a 
  database. Shared-everything architectures do not need to manage consistency / data replication 
  and can have higher resource utilization, but run the risk of bottlenecks / single points of 
  failure.

### Twitter Case Study
The core operations of Twitter can be summarized as posting tweets and reading tweets. There are 
broadly two approaches for writing and serving tweets:

1. At write time, publish the tweet to a global tweet table. Each user's read request will pull all 
  tweets from the global table that came from authors the user follows. Writes are relatively 
  cheap, and reads are relatively expensive.
1. Maintain a cache of tweets for each user. Once an author posts a tweet, update the caches of 
  all users who follow the author with the tweet. Each user's read request will pull from the 
  appropriate cache. Writes are relatively expensive since one tweet requires many cache updates 
  (known as *fan-out write load*), and reads are relatively cheap. The exact amount of work is 
  determined by the distribution of followers per user, a key load parameter in this case.
  
Let's turn to the data to determine which approach is better:

- 4.6k tweet post requests per second, 12k requests per second at peak
- 300k read requests per second
- A small % of users who have a large number of followers (30MM+)

Given the large read / write request ratio (anywhere from ~25-75), it's preferable to do more 
work at write time than read time. As a result, the second approach scales better as traffic 
increases, as long as the read / write ratio does not meaningfully shrink and the fan-out write 
load per tweet remains "reasonable". However, this approach is not suitable for those users with a 
large following - the fan-out load for tweets from such users might result in an unacceptable 
service time (compounded by the fact that these users are the most important traffic-generating 
power users, and so should have a seamless experience).

In practice, Twitter uses a hybrid of both approaches. Most users' tweets can be pushed to 
user-level caches, as their set of followers does not lead to large fan-out write load. However, 
for those users on the extreme end of the follower distribution, Twitter uses the first approach 
and inserts those tweets via lookup in each user's cache at read time.

## Maintainability

Maintainability is about making systems easy to use and adapt. A maintainable system typically 
involves focusing on three core design principles:

- **Operability**: Make it easy to keep the system running
- **Simplicity**: Reduce a system's functionality to what is needed without any unintended 
  consequences, and rely on abstractions to hide implementation details
- **Evolvability**: Make it easy to change the system in response to changing requirements

In practice, building systems with these qualities is a function of the reliability strategies 
outline above as well as organizational practices.

