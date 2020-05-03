Title: Structuring Data Access
Published: 3/5/2020
Tags: General
---

<!-- # Structuring Data Access -->

## Background  

Best practices in software development dictate that data storage and retrieval should be treated as an implementation detail. In practice it is easier said than done though - there are endless debates about how much abstraction is needed, whether the Repository Pattern is still relevant and if yes how to implement it, are ORMs good or evil, Micro ORMs and so on.  

Here are some solutions with their strengths and weaknesses.

## Full Encapsulation  

"Full Encapsulation" is inspired by Uncle Bob's [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) - have an abstraction with simple data structures going in and out. Internally the data access implementation will map these simple data structures to storage entities which suit best the libraries or frameworks used. In terms of total decoupling from any storage technology this is the best approach. It's strict abstraction gives a lot of flexibility - i.e. a team can work on the data access details or multiple data access implementations can be switched or [Polyglot Persistence](https://en.wikipedia.org/wiki/Polyglot_persistence) strategy can be utilized. Some implementations guidelines:

- Organize abstractions and implementations to be in different assemblies (packages, modules) ([Separated interface](https://www.martinfowler.com/eaaCatalog/separatedInterface.html)). I have often seen both in the same place - from practical point of view this is not a bad thing but if you think about different implementation you will need to depend on all the concrete stuff along with the abstractions.
  
- Prefer persistence ignorant abstractions (i.e. do not expose a `Save()` method) - each is atomic operation by itself. It is possible to have multiple abstractions to act as atomic action by introducing a variation of [Unit Of Work](https://www.martinfowler.com/eaaCatalog/unitOfWork.html). Personally I will not go this way since it introduces significant amount of complexity - individual abstractions will loose their autonomy and probably will have implicit dependencies, additional authority (the Unit of Work) should be managed and ultimately it will complicate usage from consumer perspective.
  
- Consider some form of [Specification](https://en.wikipedia.org/wiki/Specification_pattern) for retrieving data. It does not have to be a strict implementation of the pattern, rather it's intent is to provide a way for declaring some aspect of the data we are interested in. This will simply allow for grouping similar data access patterns together i.e. if we have orders we can specify time range, customer, product, etc. I would not go too far with this trying to define every possible case. Instead when complexity grows beyond certain point I would split in different abstractions i.e. customer's orders, product's orders, etc. - each having appropriate "specification".  
- There is no need the abstraction to be about a single entity - it can represent a series of actions serving specific business case, i.e. storing related data in multiple database tables or pulling data from multiple sources to generate a report.  

- Prefer having a single public method. This will keep the focus on single functionality and make it simple to use.  

- Sometimes having abstraction for input and output data may be an overkill - than [DTOs](https://en.wikipedia.org/wiki/Data_transfer_object) can be used instead. In both cases I would not add any behavior to them.  

Working on the data access in this setup is absolute joy - being shielded from other parts of the system gives you freedom. You can use whatever libraries/frameworks, technologies and data sources you need. This is the place where choice actually matters - you will choose a technology/framework/library because you'll pit to use it's unique features. At the same time the ["leakiness"](https://en.wikipedia.org/wiki/Leaky_abstraction) of the abstraction can be minimal. Testing strategy is also clear - integration tests for the implementation and mocks for unit tests.  

Unfortunately such strict abstraction also has it's downsides:

- The number of abstractions, implementations and mappings will grow in time since usually each business case will become to have it's own unique requirements. Striking the balance between reusability and simplicity may become very tricky and hard to recognize.  

- A business case will become "stretched" through layers which may lead to having hard time investigating issues or trying to get the "big picture". (Which is true for each layered system btw.) 

- The tradeoff for encapsulation is mapping - not only mapping the defined contract in and out the data access but also the consumer will have to do the same to it's own representation. This truly shines when storage model differs drastically from business models but when they are alike it becomes a burden since there is little benefit to be seen. 
I have observed a few times that in early stages of system implementation domain and storage entities are 1:1 thus mapping only gets in the way and gets rejected because it "slows down the work". It is only after the system goes live and becomes successful when subtle domain problems have to be solved - then domain model starts to deviate from the storage model.  

- If the system is mostly [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)-ish this can turn to annoyance very quickly - imo it is even not applicable from practical point of view in such case.  


So what if "Full Encapsulation" does not work for you - maybe your system won't benefit from it or maybe you are working in a context of microservice where deliberately reducing the levels of indirection? 

## ORM Without Abstraction

Let's suppose data access is mostly about talking to a database. Then using [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping) framework makes sense in most cases. The big question here is whether putting an abstraction layer over another abstraction is necessary? The ORM will probably already implement the patterns you need like [Identity Map](https://www.martinfowler.com/eaaCatalog/identityMap.html), Unit Of Work, Specification, etc. and building abstraction over it will probably mimic the framework of choice. On the other hand letting a dependency on a framework deeply in domain implementation is not a light decision to take. It is definitely worth thinking it through.  

Personally I will byte the bullet and use the ORM directly without trying to abstract it. In this way I will be able to use it to full extent without restricting myself to abstraction of common ORM features. I will still try to keep data access separate from business logic as much as possible. I can take comfort in the fact that I have an abstraction for the popular relational databases out there (if I ever need to switch). Databases and their supporting technologies have long lifecycles (compared to frontend ones for example). Most of them have reached maturity and will stay around for the foreseeable future.

Testing strategy is not so clear in this case. I will mock the ORM for unit tests and try to extract business logic so it can be tested separately. Some will also unit test data access by making the ORM work with in-memory data set - personally I don't think there is any value in it because there is too much difference with the real thing. Integration tests for the data access are the way to go, though if there is a lot of mixing with business logic there no clear cut what each test should cover. Some will argue that data access is also business logic but I guess it depends on the point of view.     

There are cases when this will not be possible though - company policies, strong opinions or something already in place.

## Repository Pattern  

The [Repository Pattern](https://www.martinfowler.com/eaaCatalog/repository.html) is the most popular approach for abstracting data access. It is also the most controversial. In my opinion its flaw is that it tries to oversimplify data access by pretending it is an in-memory collection. I often dislike that it is built around single entity and depending on implementation can enforce same interface for all entities.  

There are some nuances in implementations worth noting:
- Usually the Repository will be accompanied by Unit of Work and Specification implementations.

- Persistence ignorance vs persistence awareness: basically the choice whether the consumer does not care about persistence details and will let infrastructure deal with saving, transactions handling, etc. or the consumer will have to explicitly manage them (i.e. call `Save()` method).

- Level of abstraction when using ORM: the promise of ORM is that it will let you persist your business/domain objects in database seamlessly. In practice this is not happening - business objects are now "serving two masters" and they have to comply with the rules of the ORM which in turn can contradict the domain goals (mainly structuring and encapsulation). Another problem is that ORM abstractions are "leaky". An example from [Entity Framework](https://docs.microsoft.com/en-us/ef) - the concept navigation property. You never know whether it was eagerly loaded and there is no data, it is not loaded and you have to do it explicitly or it will be lazy loaded given lazy loading is enabled. And how to translate `.Include()` as a meaningful domain concept? Another example is the `.AsNoTracking()` - how can your domain know that an entity is obtained in a special way so that changes to it will not be persisted? One solution to this is to have separate business objects from database entities and provide mapping between them (or even introduce intermediary DTOs). This will make the domain "pure" but unfortunately it adds additional complexity and as described in "Full Encapsulation" it can be quite a burden for simple cases.  

- Usually the Repository implementation will cover the most of the cases. For the non-trivial cases (i.e. batch processing, weird queries) I would introduce separate abstractions instead of trying fit everything in the Repository implementation. 

### Generic Repository Implementation  

If generic implementation is feasible/possible with the tools at hand I would go for it. Implementation will probably be shaped around the underlying framework since it will probably support generics too. Unit of Work, Specification and a way of defining projections (get only the data that you'll need) are mandatory for successful usage. Usually such implementations are quite compact and will save a lot of coding. Some will argue that this is too much of generalization and it does not convey meaningful domain concepts - I tend to agree with this to some extent.  

I prefer using generic repository with domain services, each service "orchestrating" execution of specific use case (or domain concept). It will build specifications declaring the data it needs, delegate execution to the repository, then use the result to apply business logic. It can also dispatch the execution of complex domain logic to specialized classes. 

This approach makes very readable and maintainable use case implementations. The declarative nature of specifications leads to service "owning" it's data access. The ability to create and reuse named specifications allows defining of consistent and meaningful definition of queries. Projections give more insight of what data exactly is needed for a particular case.  

Consideration for testing are pretty much the same as if ORM is used directly. We can go one step further by unit testing the Specifications ensuring we use declare correct input. This will not replace integration test though. 

### Non Generic Repository Implementation  

Non generic repository implementation can become really close to "Full Encapsulation". Unfortunately the same downsides are valid also. The number of abstractions and implementations will grow in time. I would recommend figuring out some form of Specification (which appears to be recurring theme in this post). Otherwise you'll end up with methods clustered around some entity where each case will have it's own method. I have seen such abstractions with more than 30 methods having absurd names like `GetOrdersOverTotalPriceTresholdForPeriodForCustomerForProductIncludingShippingInfoAndInvoiceNumber`. Even worse is a method with more general name having 15 parameters with 12 of them optional. At some point no one will look at them and will just add another method or parameter that he needs leading to massive code duplications.  

Implemented with care non generic repository can become a good abstraction. Testing is also clear - mocks for repositories and tests for specifications in unit tests, integration tests for implementations.   

## More...  

There are more things to be considered for data access, which usually come up along the way: 
batch processing, caching, performance, transactions, compensating actions, concurrency issues, retry policies to name a few. I have not touched upon [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) and [CQRS](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation#Command_query_responsibility_segregation) since I look at them as a bit more specialized solutions to specific problems.

## Conclusion 

If you are more confused now than before reading this post it is probably a good thing. Data access should be an implementation detail but it does not mean it will be easy. The only advice I can give is to take a pragmatic approach, not a dogmatic one. Do what works best for you, your team and the software you build. And if something does not work well - don't be afraid to change it.