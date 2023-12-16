# Hexagonal Component Architecture

## Context

Typical enterprise software systems are built as service-oriented software systems.

It consists of many independent components interconnected with each other. Most of the components within the architecture are middleware components that serve data from data storage and pass it to the frontend application or other middleware.

Multiple development teams implement and maintain those components and have an independent release lifecycle.

With time, when software receives more and more functional requirements to satisfy, the components become more complex in maintenance. It is necessary to split the components into multiple ones for:

- preserving BC boundaries
- ensuring SPR, Low Coupling, High Cohesion principles on the architecture level

## Problem

How do we structure the code modules of various software system components to ensure the following:

- the domain logic is separated and independent from the technological flavors and constraints: inter-communicational protocols, external dependency technologies, data storage technologies, etc
- the module composition structure, project, and solution structure are standardized across numerous codebases in ST
- the component can be easily split into multiple components, and, as a consequence, code can be easily separated into distinct codebases

## Solution

To achieve the aforementioned goals while designing modules for middleware components in SOA-compliant systems, we encourage you to apply Alistar Cockburn's [**Ports and Adapters Architecture** pattern (aka Hexagonal Architecture)](https://alistair.cockburn.us/hexagonal-architecture).

This architecture can be legitimately used for frontline UI applications as well. Still, we are not enforcing it in there, given the mix of programming paradigms typically used in modern front-end applications.

### Main Concepts

The following diagram depicts the main pattern concepts and how they are connected.
![Main Concepts Diagram](../../.attachments/Hexagon%20Component%20Architecture%20-%20Main%20Concepts.png "Main Concepts")

#### Application Core

This concept unifies **domain logic**, **primary and secondary ports**.

Everything outside the application core is considered external and thus cannot be trusted without validation and checks before using it in domain logic space.

##### Domain Logic

> This is the reason why the particular application exists.

And

> This concept embodies the code related to single (microservice) or multiple (service) domain models.

The pattern doesn't enforce any specific DDD pattern to use or the way to structure code, but essential rules while developing the Domain Logic are:

speak and put model terms from ubiquitous domain language into the code as precisely as possible

don't create synthetic model terms to patch the holes. Ask domain experts the questions to extract missing terms and relations, get a deeper understanding of the field

use all necessary technical names and patterns in addition to domain names to make the code compliant to **Quality Attribute Requirements**: modifiability, testability, etc.

The **Use Cases** term describes the implementations of **Primary Ports** - this is simply a class that implements the interface in practice. While this term is used by many authors on the internet, from our viewpoint, it is insignificant and doesn't require any specific treatment inside the codebase.

The application core may reference shared libraries where the developers can put the domain logic, which is reusable across different applications. The smaller the "service" in your overall architecture - the more shared libraries with domain logic from different bounded contexts you can expect to have.

##### Primary Ports (Driving Ports)

These are abstract _entry points_ to the application core.

**Primary Adapters** invoke them and are usually implemented as an interface in the code.

Primary Ports are expressed using **domain model terms** and language.

Essential parts of the ports are inbound and outbound DTOs.

Consider the following example:

    public UpdateSettingsResultDto UpdateUserSettings(UserSettingsDto dto);

**Primary (Driving) Inbound DTOs** (UserSettingsDto in the example) - the _mandatory_ part of the pattern, a container for untrusted, unvalidated data passed to the Application Core through this port forth from the Primary Adapter as a parameter. This DTO cannot be used inside domain logic until it is validated and the domain entity or value object is constructed from it. DTO cannot have any behavior or be used in any other way than bringing data as a part of the port.

**Primary (Driving) Outbound DTOs** (UpdateSettingsResultDto in the example) - this is an _optional_ DTO in the pattern because the port can also return the domain entity or value object to the adapter. If you want to prevent exposure of your domain code entities outside the application core - you may use DTOs as a return value. Otherwise, you must return a **domain entity or value object** from a method.

##### Secondary Ports (Driven Ports)

These are abstract _exit points_ from the application core. They are used when the Application Core needs additional data from an external source or when the Application Core needs to provide an external party with the information back.

Secondary ports separate the communication protocols, storage technologies, platform APIs, and everything the **Application Core** does not control from itself.

**Application Core** invokes them and usually they are implemented in the form of an interface in the code.

> Secondary Ports are expressed using **domain model terms** and language.

Essential parts of the ports are inbound and outbound DTOs.

Consider the following example:

    public UserViewsDto GetUserViews(ViewsRequestDto dto);

**Secondary (Driven) Outbound DTOs** (UserViewsDto in the example) - the _mandatory_ part of the pattern, a container for untrusted, unvalidated data that is passed to the Application Core through this port back from the Secondary Adapter as a return value. This DTO cannot be used inside domain logic until it is validated and the domain entity or value object is constructed from it. DTO cannot have any behavior or be used in any other way than bringing data as a part of the port.

**Secondary (Driven) Inbound DTOs** (ViewsRequestDto in the example) - this is an _optional_ DTO in the pattern because the port can also pass the **domain entity or value object** to the adapter. If you want to prevent exposure of your domain code entities outside the application core - you may use DTOs as a parameter. Otherwise, you must pass a **domain entity or value object** into a method.

#### Primary Adapters (Driving Adapters)

This concept describes the technology and code that provides access to your **Application Core** to external components or humans, for example:

- REST API Endpoints
- Command / Event Handlers
- Graphical User Interfaces
- Command Line Interfaces

If you are working within OOP and using OOD, the secondary adapters are responsible for the following:

- setup of IoC container where developers bind:
  - use-cases to primary ports
  - secondary adapters to secondary ports
- calling specific primary ports (not necessarily one - as many as needed within workflow)
- processing result of primary port invocation back to the emitter of the workflow

The cross-cutting concerns such as authentication and authorization are also part of the Application Core, often implemented as elements of Aspect-Oriented Design - **AOD** (attributes, annotations), decorating the methods of primary adapters. Don't let syntax sugar stumble you - they are legitimate places to call your primary ports the same way if it would be done inside the other non-AOD-based method.

##### Recommended: Call Primary Ports from Primary Adapters

The recommended way for Primary Adapters to communicate with the rest of the application code is to call Application Core ports and provide data in the form of DTO, which contains untrusted and unvalidated data from the core perspective. The Application Core must first validate the inputs, create domain entities or value objects from those DTOs, and use them downstream.

##### Rare Exception: Call Secondary Ports from Primary Adapters

There might be rare cases when application workflows don't have control flow and domain logic - it happens when the application, for some reason, serves as a pure intermediary in the distributed workflow, for example, to hide the downstream communication protocol from the upstream workflow requestor.

Suppose this happens in the application that conforms to hexagonal architecture. In that case, the particular secondary port can be called directly from the primary adapter, but, again - this is a very rare case and serves rather than an "exception that proves the rule" as normal usage.

##### DONT: Calling Secondary Adapters from Primary Adapters

In the hexagonal architecture, calling secondary adapters directly from secondary ports means that abstractions are not used at all, and it is a strong **NO-GO** for the application where the pattern is already applied.

The only justification for using it is in quick-to-implement and soon-to-recycle nano-service applications.

#### Secondary Adapters (Driven Adapters)

Secondary Adapters are implementations of **Secondary Ports**. They conform to the port contract and hide all communication protocol-oriented, storage technology-oriented, and other interoperability and portability-oriented logic inside.

Do not be confused; secondary and primary ports are supposed to contain logic; control flow instructions can be very complex and must be covered by tests - the same way the **Application Core** code requires to be covered. Just here, you are free to focus on the efficiency of the aforementioned technical mechanisms and are free from domain model terms and language.

The output of Secondary Adapters is passed back to the Application Core in the form of DTO, which contains untrusted and unvalidated data from the core perspective. This means that the code of secondary adapters may be implemented with reusability in mind to fit multiple applications. You can achieve this by extracting all generic and reusable code into the shared library and injecting it into the adapter with additional code to fit into a specific secondary port contract.

The following diagram describes the recommended allocation of architectural concepts to specific compilable libraries:
![Libraries Diagram](../../.attachments/Hexagon%20Component%20Architecture%20-%20Libraries.png "Libraries")

### Hexagonal Architecture and Dependency Inversion Principle

As you can see, the hexagonal architecture explicitly emphasizes one of the SOLID principles - Dependency Inversion by separating Secondary Ports into a separate library and restricting its references from other libraries.

> It is crucial to prevent library with secondary adapters to reference libraries with domain logic and domain model terms implementation. Adapters should depend ONLY on it's ports: interfaces, DTOs.

And

> It is crucial to prevent libraries with domain logic and domain model terms implementation to reference library with secondary adapters and violate COMPLETE separation of domain and infrastructural concerns.

And

> Usually you dont need to separate primary ports into separate library because it increases number of references for primary adapters libary but if your application becomes big enough you can certainly do so.

And

> Primary Adapters with OOD typically reference all other libraries to setup IoC Container and start the required workflow.

### Example of the structure for the code solution following Hexagonal Architecture

The following picture demonstrates the example of a .NET solution structured following the pattern principles and guidelines:
![Libraries Example Diagram](../../.attachments/Hexagon%20Component%20Architecture%20-%20Libraries%20Example.png "Libraries Example")

### FAQ

#### DTO and Model Mapping Cheatsheet

|Source Model|Target Model|Owned By|
|-|-|-|
Primary Adapter DTO|Application Core Primary Port DTO|Primary Adapter|
|Application Core Primary Port DTO|Primary Adapter DTO|Primary Adapter|
|Application Core Primary Port DTO|Application Core Model|Application Core|
|Application Core Model|Application Core Primary Port DTO|Application Core|
|Application Core Model|Application Core Model|Application Core|
|Application Core Model|Application Core Secondary Port DTO|Application Core|
|Application Core Secondary Port DTO|Application Core Model|Application Core|
|Secondary Adapter DTO|Application Core Secondary Port DTO|Secondary Adapter|
|Application Core Secondary Port DTO|Secondary Adapter DTO|Secondary Adapter|

## Links

[Hexagonal Architecture resources](https://hexagonalarchitecture.org) \
[Ports and Adapters Architecture](https://herbertograca.com/2017/09/14/ports-adapters-architecture)
