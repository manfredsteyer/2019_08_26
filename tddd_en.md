# Tactical Domain-Driven Design with MonoRepos?

Business and industrial applications are usually long-lived. These applications include customer-facing applications layered with complex backend services and systems. Many of these applications are now implemented with web frontends using JavaScript. 

So how can we build and maintain such software systems?

**Domain Driven Design (DDD)** provides the answer! Best of all, it is especially suitable for complex Angular solutions. 

In this article we will learn:

- How **DDD** helps to organize our application source code as smaller, manageable, coherent parts.
- How **Monorepos** help implementing them
- How **Facades** help with isolating your domain logic
- How client-side DDD paves the way to micro frontends

> All these topics are also part of our advanced [Angular Architecture workshop](https://www.softwarearchitekt.at/schulungen/advanced-angular-enterprise-anwendungen-und-architektur/). 

As always, the examples used can be found in my [GitHub account](https://github.com/manfredsteyer/angular-ddd).


> Big thanks to [Thomas Burleson](https://twitter.com/ThomasBurleson) who deeply reviewed this article and contributed to it.

## Domain Decomposition

Let's consider a consumer travel web application we are building. We have several domains that we can identify: 

![image](domains.png)

To manage complexity of these domains, we should leverage **Domain-Driven Design** and best practices from the Angular community. 

Domain-driven design recommends subdividing an entire system into several small, possibly self-contained domains. Each domain should be modeled separately and receives its own entities which best reflect the respective domain's business area. 


## Domain Complexity

#### So how can we manage complexity in an application with many domains?

We need to apply concepts and techniques from both parts of DDD:

* **[Strategic Domain Design](https://www.softwarearchitekt.at/aktuelles/sustainable-angular-architectures-1/)**
* **Tactical Design**

I've already written about the use of **[Strategic Design](https://www.softwarearchitekt.at/aktuelles/sustainable-angular-architectures-1/)** in Angular applications. Readers are encouraged to read that article for foundational understandings. Hence, here, I'll focus on Tactical Domain Design.


### Domain Organization using Layers

Our domain identification approach leads to column subdivisions ("swimlanes") for domains. Those domains can be organized with layers of libraries which leads to row subdivisions:

![](domains-layer.png)

Note how each layer could consists one (1) or more **libraries**

>While using layers is quite a traditional way of organizing an domain, there are also alternatives like hexagonal architectures or clean architecture.

#### What about shared functionality?

For those aspects that are to be *shared* and used across domains, an additional ``Shared`` swimlane is used. Shared domains can be quite useful. Consider - for example - a shared domain with libraries for authentication or logging.

>  Note: the `shared` column corresponds to the Shared Kernel proposed by DDD and also includes technical libraries to share.

#### How to prevent high coupling?

Access rules can be defined to constrain which libraries can use/depend upon other libraries. Typically, each layer is only allowed to communicate with underlying layers. Cross-domain access is allowed only with the ``shared`` area. The benefit of using these restrictions can result in loose coupling and thus increased maintainability. 

> To prevent too much logic to be put into the ``shared`` area, the approach presented here also uses APIs that publish building blocks for other domains. This corresponds to the idea of Open Services in DDD.

Regarding the shared part, one can see the following two characteristics:

*  As the grayed-out blocks indicate, most ``util`` libraries are in the ``shared`` area, especially as aspects such as authentication or logging are used across systems. The same applies to general UI libraries that ensure a system-wide look and feel.
* UI features contained in the `shared` library are usually global UI, custom UI component libraries, etc.

#### What about feature-specific functionality?

Notice that domain-specific `feature` libraries, however, are not in the shared area. Feature-related code should be placed within its own domain. 

While developers may elect to share feature code (between domains), this practice can lead to shared responsibilities, more coordination effort, and breaking changes. Hence, it should only be shared sparingly.

----

### Code Organization 

Based on Nrwl.io's [Enterprise MonoRepository Patterns](https://go.nrwl.io/angular-enterprise-monorepo-patterns-new-book), I distinguish between five (5) categories of layers or libraries:

Category | Description | Exemplary content
--------- | ------------ | ---------------------
feature | Contains components for an use case. | search-flight component 
ui | Contains so-called "dumb components" that are use-case agnostic and thus reusable. | datetime-component, address-component, adress-pipe
api | Exports building blocks from the current subdomain for others. | Flight API
domain   | Contains the **domain models** (classes, interfaces, types) that are used by the domain (swimlane)
util | Include general utility functions | formatDate

This complete architectural matrix is initially overwhelming. But after a brief review, almost all developers I've worked with agreed that the code organization facilitates code reuse and future features. 

----

### Isolate the Domain

To isolate the domain logic, one should consider issues of state management and push-based architectures. Two excellent articles are available for deep dives into those areas:

* [Ngrx + Facades: Better State Management](https://medium.com/@thomasburlesonIA/ngrx-facades-better-state-management-82a04b9a1e39)
* [Push-based Archtectures with RxJS](https://medium.com/@thomasburlesonIA/push-based-architectures-with-rxjs-81b327d7c32d)

>  These approaches - which encapsulate domain logic and manage changes to state - leverage the concepts of Facades.

![Isolating the Domain Layer](https://i.imgur.com/sjxc4we.png)


While Facades are currently quite popular in the Angular environment, this idea also correlates wonderfully with DDD (where they are called application services). 

It is important to architecturally separate *infrastructure requirements* from the actual domain logic. 

In an SPA, infrastructure concerns are -- at least most of the time -- asynchronous communication with the server and data exchanges. Maintaining this separation results in three additional layers: 

* the api layer with Facades, 
* the actual domain layer, and 
* the infrastructure layer.

Of course, these layers can now also be packaged in their own libraries. For the sake of simplicity, it is also possible to store them in a single library, which is subdivided accordingly. This decision to use a single grouping library makes sense if these layers are usually used together and only need to be exchanged for unit tests.


### Implementations in a Monorepos

Once the components of our architecture have been determined, the question arises of how they can be implemented in the world of Angular. A very common approach also used by Google itself is the monorepo. It is a code repository that contains all the libraries of a software system.

While a project created with the Angular CLI can nowadays be used as a monorepo, the popular tool [Nx](https://nx.dev/) offers some additional possibilities which are especially valuable for large enterprise solutions. These include the previously discussed ways to introduce [access restrictions between libraries](https://www.softwarearchitekt.at/aktuelles/sustainable-angular-architectures-2/). This prevents each library from accessing each other, resulting in a highly coupled overall system.

To create a library in a monorepo, one instruction is enough:

```
ng generate library domain --directory boarding
```

> As you just use `ng generate library` instead of `ng generate module` there is not more effort for you. However, you get a cleaner structure, improved maintainability, and less coupling.

The switch ``directory`` provided by Nx specifies an optional subdirectory where the libraries are to be put. This way, they can be **grouped by domain** within the library file system:

![Monorepo with domains](folder.png)

The names of the libraries also reflect the layers. 
If a layer has several libraries, it makes sense to use these names as a prefix. This results in names such as ``feature-search`` or ``feature-edit``.

In order to isolate the actual domain model, the example shown here divides the domain library into the three further layers mentioned:

![](folder-domain.png)

#### Builds within a Monorepo

By looking at the git commit log, Nx can also identify which libraries are *affected by the latest code changes*. 

This change information is used to recompile only the **affected** libraries or just run their **affected** tests. Obviously, this saves a lot of time on large systems that are stored in a repository as a whole.

<br/>

----

### Entities and your Tactical Design

Tactical Design provides many ideas for structuring the domain layer. At the center of this layer, there are **entities** the reflecting real world domain and constructs.  

The following listing shows an enum and two entities that conform to the usual practices of object-oriented languages such as Java or C#.
 

```Java
public enum BoardingStatus {
  WAIT_FOR_BOARDING,
  BOARDED,
  NO_SHOW
}

public class BoardingList {

  private int id;
  private int flightId;
  private List<BoardingListEntry> entries;
  private boolean completed;

  // getters and setters

  public void setStatus (int passengerId, BoardingStatus status) {
    // Complex logic to update status
  }

}

public class BoardingListEntry {

  private int id;
  private boarding status status;

  // getters and setters
}
```

As usual in OO-land, these entities use information hiding to ensure that their state remains consistent. You implement this with private fields and public methods that operate on them. 

These entities not only encapsulate data, but also business rules. At least the method ``setStatus`` indicates this circumstance. Only for cases where business rules can not be meaningfully accommodated in an entity, DDD defines so-called domain services.

> Entities that only represent data structures are frowned upon in DDD. The community calls them devaluing [bloodless (anemic)](https://martinfowler.com/bliki/AnemicDomainModel.html).

### Tactical DDD with Functional Programming

From an object-oriented point of view, the previous approach makes sense. However - with languages such as JavaScript and TypeScript - object-orientation is less important. 

Typescript is a multi-paradigm language in which Functional Programming is particularly important. Books dealing with Functional DDD can be found here:

* [Domain Modeling Made Funcitonal](https://pragprog.com/book/swdddf/domain-modeling-made-functional),
* [Functional and Reactive Domain Modeling](https://www.amzn.com/1617292249).

With functional programming, the previously considered entity model would therefore be separated into a data part and a logic part. [Domain-Driven Design Distilled](https://www.amzn.com/0134434420) which is one of the standard works for DDD and primarily relies on OOP, also admits that this rule change is necessary in the world of FP.

Consider the following example:

```Typescript
export type BoardingStatus = 'WAIT_FOR_BOARDING' | 'BOARDED' | 'NO_SHOW' ;

export interface BoardingList {
    readonly id: number;
    readonly flightId: number;
    readonly entries: BoardingListEntry [];
    readonly completed: boolean;
}

export interface BoardingListEntry {
    readonly passengerId: number;
    readonly status: BoardingStatus;
}
```

```Typescript
export function updateBoardingStatus (
                   boardingList: BoardingList,
                   passengerId: number,
                   status: BoardingStatus): Promise <BoardingList> {

        // Complex logic to update status

}
```
 
Here the entities also use public properties. This practice is quite common in FP; the excessive use of getters and setters, which only delegate to private properties, is often ridiculed!

Much more interesting, however, is the question of how the Functional world avoids inconsistent states. The answer is amazingly simple: data structures are preferably **immutable**. The keyword ``readonly`` in the example shown emphasizes this.

A part of the program that wants to change such objects has to clone it, and if other parts of the program have first validated an object for their own purposes, they can assume that it remains valid.

> A wonderful side-affect of using immutable data structures is that change detection performance is optimized. No longer are *deep-comparisons* required. Instead a *changed* object is actually a new instance and thus the object references will no longer be the same.

### Tactial DDD with Aggregates

To keep track of the components of a domain model, Tactical DDD combines entities into aggregates. In the last example, ``BoardingList`` and ``BoardingListEntry`` form such an aggregate.
 
The state of all components of an aggregate must be consistent as a whole. For example, in the example outlined above, one could specify that ``completed`` in `` BoardingList`` may only be set to ``true`` if no ``BoardingListEntry`` has the status `` WAIT_FOR_BOARDING`` .

In addition, different aggregates may not reference each other through object references. Instead, they can use IDs. This should prevent unnecessary coupling between aggregates. Large domains can thus be broken down into smaller groups of aggregates.

> [Domain-Driven Design Distilled](https://www.amzn.com/0134434420) suggests making aggregates as small as possible. First of all, consider each entity as an aggregate and then merge aggregates that need to be changed together without delay.

<br/>

----

## Facades

**[Facades](https://go.nrwl.io/angular-enterprise-monorepo-patterns-new-book)** (aka Applications Services) are used:

* encapsulate complexity
* provide a simplified API 
* provide an API for specific use cases. 
 
Independent of DDD, this idea has been very popular in the world of Angular for some time. 

For our example, we could create the following Facade:
 

```typescript
@Injectable ({providedIn: 'root'})
export class FlightFacade {

    private notifier = new BehaviorSubject<Flight[]>([]);    
    public  flights$ = this.notifier.asObservable();

    constructor(private flightService: FlightService) { }

    search(from: string, to: string, urgent: boolean): void {
        this.flightService
            .find(from, to, urgent)
            .subscribe (
              (flights) => this.notifier.next(flights),
              (err) => console.error ('err', err);
            );
    }
}
```

Note the use of RxJS and observables in the facade. This means that the facade can auto-deliver updated flight information when conditions change. Another advantage of Facades is the ability to transparently introduce Redux and ``@ngrx/store`` later when needed. This can be done without affecting any of the external application components. 

> For the consumer of the Facade it is not relevant whether it manages the state by it self or by delegating to a state management library.

Developers are encouraged to read/watch Thomas Burleson's work about this topic:

* Blog article [Push-based Architectures using RxJS + Facades](https://medium.com/@thomasburlesonIA/push-based-architectures-with-rxjs-81b327d7c32d), or 
* YouTube video [Push-based Architectures with RxJS](https://www.youtube.com/watch?v=HnNytR32Otk&t=1s)

### Stateless Facades

While it is a good practice to make server-side services stateless, often this goal is not performant for services in web/client-tier. 

A web SPA has state and that's what makes it user-friendly!

To avoid UX issues, Angular applications do not want to reload all the information from the server over and over again. Hence, the Facade outlined holds the loaded flights (within the observable discussed  before).

----

## Domain Events

Besides performance improvements, using Observables provide a further advantage. Observables allow further decoupling, since the sender and the receiver do not have to know each other directly. 

This also perfectly fits to DDD, where the use of **domain events** are now part of the architecture. If something interesting happens in one part of the application, it sends a domain event and other application parts can react to it. 

In the shown example, a domain event could indicate that a passenger is now BOARDED. If this is interesting for other parts of the system they can execute specific logics. 

> For Angular developers familiar with Redux or Ngrx: Domain events can be represented as *dispatched actions*!

## Domain-Driven Design and Micro Frontends?

The ideas of Domain-Driven Design are known to pave the way for micro-service architectures. Hence, client-side DDD can be used as a basis for micro frontends.

Whether a deployment monolith, micro frontends, or anything in between is created, depends on the use of the monorepo. If the monorepo gets an own application for each domain, a big step towards micro frontends is taken:

![DDD on the Client as the Basis for Micro Frontends](https://i.imgur.com/keOLDU3.png)

The access restrictions discussed above ensure a loose coupling and even allow a later split to multiple repositories. Then you can talk about micro frontends in the classic sense of a micro-service architecture. However, in this case the team has to take care of versioning and distributing shared libraries which is common with micro-services.

More thoughts about this topic can be found in my blog article [It’s about cutting your domain, not (first and foremost) about micro frontends!](https://www.softwarearchitekt.at/aktuelles/its-about-cutting-your-domain-not-first-and-foremost-about-micro-frontends/).

## Conclusion

Modern single page applications (SPAs) are often more than just recipients of data transfer objects (DTOs). They often contain significant domain logic; which adds complexity. Ideas from DDD help developers manage and scale with the resulting complexity.

Due to the object-functional nature of TypeScript and prevailing customs, a few rule changes are necessary. For instance, we usually use immutables and separate data structures from the logics operating on them.

The implementation outlined here bases upon the following ideas:

* The use of monorepos with multiple libraries grouped by domains helps building the basic structure.
* Access restrictions between libraries prevent coupling between domains. 
* Facades prepare the domain model for individual use cases and take care of maintaining the state. 
* If needed, Redux can be used behind the facade without the rest of the application noticing. 

Besides, a team also creates the conditions for micro frontends by using client-side DDD.

> If you want to see all these topics in action, checkout our [Angular Architecture workshop](https://www.softwarearchitekt.at/schulungen/advanced-angular-enterprise-anwendungen-und-architektur/).

 
