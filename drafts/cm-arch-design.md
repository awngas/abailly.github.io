------------
title: Anatomy of a Haskell-based Startup
author: Arnaud Bailly 
date: 2015-10-26
------------

This is the first post of a series I am planning to write about my experience developing software at
[Capital Match](http://www.capital-match.com), a Singapore-based startup providing a peer-to-peer lending marketplace for Small and
Medium Businesses and private and corporate investor. 

This post is about the design and architecture of the system itself, the choices and tradeoffs that were made, whether good or bad. In
the conclusion I try to provide an assessment of the current situation and reflect on those choices.

# Fundamental Design Choices

## Haskell

Basing Capital Match's tech stack on Haskell was an obvious choice for me from the onset, even if I had had limited professional experience with Haskell before that:

* Haskell is a [very mature language](http://research.microsoft.com/en-us/um/people/simonpj/papers/history-of-haskell/) with a
  [state-of-the-art compiler](https://www.haskell.org/ghc/) that receives constant attention from a bunch of extremely bright people
  and thus keeps improving and evolving,
* Haskell's tools for building robust web-based applications is not as mature as what you can find in Java or .Net worlds but it is
  evolving quickly as the platform is gaining traction thanks to efforts from both a vibrant 
  [community](https://wiki.haskell.org/Haskell_Communities_and_Activities_Report) and few but dedicated
  [private bodies](https://github.com/commercialhaskell),
* Haskell developers are few and far between, but their number is growing and they are more often than not passionate and talented,
* I have been programming for fun and small side projects in Haskell since 2002 and I always have wanted to know how it would feel
  to build a whole system with it. Now I know,
* Because it enforces a strict separation of pure and effectful code, Haskell incentivizes the growth of
  a [Hexagonal Architectures](http://alistair.cockburn.us/Hexagonal+architecture)
  aka. [Ports and adapter](http://c2.com/cgi/wiki?PortsAndAdaptersArchitecture): A pure domain kernel which interacts with the
  outside world through *adapters*.

## Event Sourcing

The system was designed from the onset as an [event-sourced](http://martinfowler.com/eaaDev/EventSourcing.html) application: The
source of truth in the system is a sequence of *events* where each event defines a transition between two states. At any point in
time, the state of the system is thus whatever state the current sequence of events leads to. Among the motivations behind using ES
are:

* Having fun and explore this corner of the design space instead of going for the more traditional RDBMS-based web app,
* Auditability and traceability of all actions impacting data on the platform, a property which is highly-desirable in a banking-like
  system. I have had previous exposure to finance software and they all end up implementing some journalling system to trace
  users actions and data changes,
* Reluctance to add the operational burden of maintaining a RDBMS as part of the system. We could have used SaaS relational (or
  non-relational) database to remove that burden but this implies using yet another tool, learning some other piece of technology,
  using some set of drivers with specific bugs and requirements,
* Personal bias against RDBMS used as runtime storage[^6],
* Simplicity of implementation, at least when you don't require HA, partition tolerance or more generally fault-tolerance: A single
  file to which all events are appended is enough, and this is exactly what we do,
* Avoiding languages impedance mismatch. There is the tradional
  [Object-relational Impedance Mismatch](http://c2.com/cgi/wiki?ObjectRelationalImpedanceMismatch)  although
  [some have argued](http://blog.jooq.org/2015/08/26/there-is-no-such-thing-as-object-relational-impedance-mismatch/) it is not
  where we usually think it is. As argued in the latter I think the real issue is in SQL: SQL is (probably) great for writing
  complex queries[^1] but not so much for inserting data.

# Architecture

The main interface to the system is a REST-like API providing various resources and actions over those resources. Most exchanges
with the outside world are done using JSON representation of resources, with some CSV. The *User Interface* is merely a client of
the API and is (morally if not to the letter) a single page application. There is also a command-line client which offers access to
the complete API and is used for administrative purpose.

## Models ##

The core of the application is purely functional and made of several loosely coupled `BusinessModel` instances (think
Aggregates in [DDD](https://en.wikipedia.org/wiki/Domain-driven_design) parlance) that each manage a specific
sub-domain: `Accounting` manages accounts and transactions, 
`Facility` manages facilities lifecycle, `Investor` and `Borrower` manage profiles and roles-dependent data, `User` manages basic
registration, authentication and settings for users (e.g. features)...

A `BusinessModel` is defined as:

~~~~~~~~~ {.haskell}
class BusinessModel a where
  data Event a :: *
  data Command a :: *
  init :: a
  act :: Command a -> a -> Event a
  apply :: Event a -> a  -> a
~~~~~~~~~


* A type of events this model generates
* A type of commands this model can process
* An initial value for the model
* A pair of functions describing which event is generated by a command *acting* on the model and how an event changes the model when
  it is *applied* to it

The state of each BusinessModel instance is computed upon application startup by loading all the events and applying each stored
event to an `init`ialised model. Models are then kept in memory while events are stored persistently. This initial startup process
takes a couple of seconds given the small scale at which we operate.

Each model demarcates transactional boundaries and is the unit of consistency within the system. Commands and events on a single model are
assumed to occur *sequentially*.

## Services ##

Above `BusinessModel`s are `Service`s which provides the interface to the system. Services orchestrate the interactions of one or
more Models. At the simplest level, a `Service` simply consists in the  application of a single `Command` on some `BusinessModel`,
but it can be more complex, synchronizing application of commands over several models. Based on the ideas exposed in
[Life Beyond Distributed Transactions](http://adrianmarriott.net/logosroot/papers/LifeBeyondTxns.pdf), a `Service` represents the
state of the interaction between a single user of the system, e.g. a request, and one or more piece of data maintained by the
system.

Because they are the agents of the outside world in the system, `Service`s operates in an impure context, hence in a dedicated `Monad`
called `WebM`. Services typically return some representable type, when they are queries, or an `Event` denoting
the outcome of the request. `WebM` is actually an instance of a monad transformer `WebStateM` over IO, hence it is impure. It has
access to 2 pieces of state.

Here is the definition of `WebStateM`:

~~~~~~~~~ {.haskell}
newtype WebStateM shared local m a = WebStateM { runWebM :: TVar shared -> local -> m a }

type WebM a = forall s . EventStore s => WebStateM SharedState LocalState s a
~~~~~~~~~

This is simply a `Reader` monad with two different pieces of data:

* `LocalState` is filled with information relevant to a single query (e.g. user id, request id, time...),
* `SharedState` is a `TVar` (transaction variable living in `STM` monad) that is shared across all requests,
* The `EventStore` constraint means we need the underlying monad to provide access to persistent storage.

The vast majority of services use the generic `applyCommand` function which is the critical part of the system. This function is
responsible for:

*  applying the command and updating the stored Model,
*  persist the event in the "database",
*  dispatch the event to interested components.

## Web

The REST interface is provided by [scotty](https://github.com/scotty-web/scotty) which is a simple framework based on
[WAI](https://github.com/yesodweb/wai) and [warp](https://github.com/yesodweb/wai)[^7]. Most action handlers are pretty simple:

* They extract some parameters or JSON data from the body of the request,
* They invoke some service,
* They provide an HTTP response according to the result returned:
    * Queries simply serialize the result to JSON or other requested media type,
    * Actions look at the returned `Event` to provide meaningful answers.

On top of REST endpoints sit some `Middleware`s which check or apply transformations to requests and/or responses:

* Provide a `Request-Id` header,
* Authorisation and authentication,
* Sanity checks (e.g. sizes of payloads),
* Logging,
* Caching and static data service.

## Lost in Translation

Executing a user-triggered action is in a sense a series of  translations occuring between different *level of languages*:

* From REST to `WebM` we use `inWeb :: WebStateM CapitalMatchState LocalState m a -> ActionT e (WebStateM CapitalMatchState
  LocalState m) a`,
* From `WebM` to `STM Model` we use `liftIO . atomically`,
* ... and finally in `Model` we reach the pure kernel of the business domain!

Conceptually, we have this hierarchy of monads, expressed in types:

~~~~~ {.haskell}
model :: Command -> StateT STM Model (Event Model)
service :: WebM (Event Model)
web :: ActionT ()
~~~~~~

This hierarchy of monads delineates, somewhat obviously, the following languages:

* Language of *Models* expresses atomic (e.g. often CRUDesque) changes to a Model, like `RegisterTransaction`, `UpdateProfile` or `CloseFacility`,
* Language of *Services* expresses either direct changes to Models or more complex interactions like `addPledge` or `acceptFacility`
  which require more than one command to complete, 
* Language of *Web* manages HTTP Requests and responses, JSON structures and delegate work to services. It is the language of
  representation of things. 

# Cross-cutting Concerns

## Concurrency ##

Concurrency is mostly handled at the REST layer through Warp and Scotty: Each request is handled concurrently by separate threads
which are [very lightweight in Haskell/GHC](https://ghc.haskell.org/trac/ghc/wiki/LightweightConcurrency). On top of that we have a
couple more threads in the application: 

* One logging thread per handler (currently 2), which handle logging messages,
* A storage thread which handle low-level read/write requests to events file,
* A driver thread which handle events storage[^3],
* A *Heartbeat* thread periodically checks other threads and notifies health.

We used to run directly threads with `forkIO` and friends but finally moved to something simpler and much more robust: The
[async](http://hackage.org/packages/async) package. Concurrent updates to the model are handled through
[Software Transactional Memory](http://hackage.org/packages/stm): A `TVar` (transactional variable) holds a reference to the state
and all operations on the state are thus transactional.

The initial goal was to enforce a strict separation between the various *Business Models* with an eye towards being able to deploy
them as independent services exchange messages. But it happened this rule was broken more than once and a few months later we ended
up having built a monolith with uncontrolled dependencies across domains and layers. We then undertook the necessary refactoring
steps to get back to a "saner" state where `STM` transactions operate at the level of a single `Command` through the `applyCommand`
function. 

## Persistence and Storage ##

Persistence is managed through a dedicated event bus: `Event` are first packaged into a more opaque `StoredEvent` object containing
metadata useful for traceability of the system:

* An event version (more on this later)
* An event type which encodes at the value level the actual type of event
* Timestamp (in UTC)
* ID of user generating the event
* ID of original request 
* SHA1 of commit, e.g. version of the code (thanks to [Geoffroy Couprie](http://geoffroycouprie.com/) for the suggestion)
* `ByteString` payload containing serialized version of the event

Then `StoredEvent`s are pushed to a dedicated `Driver` thread which stores events in the underlying events file. Physical `Storage`
is a simple append-only file which contains sequence of applied events serialized to some binary format
([Kafka](http://kafka.apache.org)-like). We are in the process of moving to a much more robust storage solution: 

* externalize data store to another process/host,
* replicate it to increase fault-tolerance,
* provide strong consistency through distributed consensus.

### Event Versioning

Events are stored with a *version* number which is a monotonically increasing number. This version number is bumped each time we
change the structure of our events, e.g. adding a field to some record, changing a field's type... When an event is persisted, it
contains the *current version* number that's defined in the application at that time. When the event is read back
(i.e. deserialized) to rebuild the state of the system, this version number is used to select the correct read function.

Hence modifying the structure of events always entails the following steps in development:

* Write deserialization test for current version number[^8],
* Bump version number,
* Write new code,
* Write *migration code* to adapt old events to new version.

This mechanism adds some interesting properties to our underlying storage:

* Stored events are immutable hence storage system is append-only: We never need to rewrite past events, 
* It is possible to rebuilt state of the system (code *and* data) at any point in the past.

## User Interface ##

UI code still lives partly in the server and partly as pure client-side code:

* HTML code is generated and served by the server using standard endpoints according to `Accept` header in request. We use
  [blaze-html](https://hackage.haskell.org/package/blaze-html) combinators to describe the pages in Haskell,
* Static assets are served by [wai-middleware-static](https://hackage.haskell.org/package/wai-middleware-static)
  
But the grunt of UI work is done on the client with [Om](https://github.com/omcljs/om/). Om is a
[clojurescript](http://github.com/clojure/clojurescript) interface to Facebook's [React](http://facebook.github.io/react/)[^4]. We
treat the UI-backend interaction as a pure client-server: UI maintains its own state and interact with server through usual Ajax
calls, updating state accordingly. The interface a user sees is a single-page application.

## Logging and Monitoring ##

There is a `Log` structure which is a queue consuming logging events and handling it according to some function. We log all
queries, all commands issued and various other events occuring in the system: application startup/stop, heartbeat, I/O errors,
storage events... In order to prevent sensitive data to leak to logging, we have a `redact` function that rewrites commands to
remove those data before passing it to logging system. 

We currently have two different log backends:

* One log handler outputs JSON-formatted events to `stdout`,
* One log handler outputs some events to [Riemann](http://riemann.io), using [riemann-hs](https://github.com/tel/riemann-hs). Those
events are then used for notification and monitoring of infrastructure (more on this in a later post).

At startup of application we also notify dev team by sending an email with the configuration. This is useful to check startup of
production environment, as this email contains among other things the version of the application and the command line with which
is has been started. 

# Reflection

It's been a bit over a year since I have started working on Capital Match's platform. I -- we -- have made mistakes, not everything
went as smoothly as we would have liked and it is still just the beginning of a hopefully long adventure. One year is a good time to
stop - or slow down a bit - and reflect on what's been accomplished, what went wrong and what went well. In the next sections I try
to provide a more or less objective assessment of the architecture we have put in place, and what would be our next steps.

## The Good, the Bad and the Ugly

We have been live since March 2015, serving more than S$ 3 millions - and counting - in facilities for SMEs in Singapore without any
major interruption of service. This in itself is an extremely positive fact: It works and it supports continuous improvements in a
smooth way[^9].

Here are some benefits I see in our approach, encompassing both the technology used (Haskell, Om/Clojurescript) and the
architecture:

* Strong and expressive types greatly improves confidence in the code[^10]. Among the
  [many features](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/ghc-language-features.html) supported by GHC, here
  are the ones, mostly simple and straightforward, we use routinely:
    * `newtype`s are cheap in Haskell, being unpacked at runtime, but they are enforced during compilation and makes for very
      expressive signatures: No more `String`-based programs but `UserId`, `PassportNr` or `EMail`,
    * Phantom types is a simple technique to distinguish between various objects with equivalent representations, like encoding of a
      `ByteString`, 
    * Type-classes are very useful to define interfaces, possibly with default implementations. Most of the time we use
      `MultiParamTypeClasses` to express relations between different part of the systems,
    * Existential types are useful to *pack* related things under a common opaque type for e.g. serialization or logging, where you
      don't care about the details, 
    * But most of the time using a simple type with a set of constructors is clearer.
* We use `-Wall -Werror` for compiling code which catches things like unused variables (dead code), variable names overriding
  (potential troubles), incomplete pattern matching (runtime failure ahead...),
* Event Sourcing greatly simplifies storage management and removes all the hassle of having to manage data types mapping to
  a relational model, not speaking of managing migration between versions or the burden of operating a DBMS, 
* Because data is stored as a flat file of versioned events, querying the system can be done directly in Haskell: Retrieve event
  stream, build in-memory state from it then use GHCi or Emacs REPL to manipulate state[^11] at your will. Over time we have written
  a number of *scripts* that are small Haskell programs implementing complex queries (e.g. user funnel) or programmatic one-shot
  transformations of data,
* Scotty, WAI and Warp makes it easy to quickly develop and maintain REST interfaces, both for endpoints and middlewares.

Here are some mistakes I made:

* Too many use of partial functions like `fromJust` or even `head`: This makes things simple at first but of course blows up at
  runtime. Lesson learned: Always use **total functions**,
* Too many typeclasses: This might come from my Java background where using an interface is usually a good idea to abstract
  details. Over the time, we have created new typeclasses to answer to express new behaviour, which clutters types, 
* Not taking enough care to keep compilation time low: Our application has grown over time, and I have not taken care of splitting it early enough to
  prevent bloat. Compilation time has grown over time to the point where it is now a problem. Lesson learned: Aggressively split
  code as early as possible, and don't be afraid of having packages with one or two files,
* Too much reliance on `DeriveGeneric` based JSON serialization: Generating `Generic` instance for large types dramatically
  increases compilation time. Lesson learnt: Use more `TemplateHaskell` derivation or custom `ToJSON/FromJSON` instances which provide
  better flexibility[^12]
* Having data accessible only through Haskell implies non-tech people either need to learn it or go through the dev team to
  get data. This is not too much of a trouble in a very small team but could quickly become a problem as the company grows. This is
  where [CQRS](http://martinfowler.com/bliki/CQRS.html) will nicely complement our Event Sourced system: It is a rather simple
  matter to build one or more relational models from our data and ensure the RDBMS is updated on a regular basis,
* Separation of concerns among the various business models has been sloppier over time, leading to too much coupling among the
  models,
* Events are not invertible which means that one cannot travel back and forth in an event stream easily to select required state,
* Generating HTML on the server-side: At the onset of the project, we had a static HTML file. We moved to server-side HTML
  generation using Blaze because we wanted to be able to control the structure of the HTML:
    * To cope for dev/prod environment on the front-end: Code is compiled and optimized differently in the two modes and this
      requires importing a different set of scripts,
    * To manage *feature toggles* which allow us to provide a different UI for different users according to their account's
      settings. This was very useful to handle gracefully migration of our UI,
    * However this strategy entails a number of problems:
        * You sometimes have to recompile server when working on the UI, e.g. when changing structure of pages or handling of features, 
        * It's hard to collaborate with front-end designers and developers, 
        * It makes UI and server more coupled,
* REST interface seems to be growing too quickly. There are some query abstractions that should be developed out of the raw
  endpoints we are exposing.

## What's Next? ##

Within the span of a single year, much has happened in the Haskell ecosystem and things that were experimental or unwieldy one
year ago are now mature and could easily make their way to production: [ghcjs](https://github.com/ghcjs/ghcjs) is now much easier to
build and work with, there are more mature solutions in the front-end like [reflex](https://github.com/ryantrinkle/reflex), build
has never been easier thanks to [stack](https://github.com/commercialhaskell/stack/),
[GHC 7.10](https://downloads.haskell.org/~ghc/7.10.1/docs/html/users_guide/release-7-10-1.html) has brought a number of improvements
(and controversial breaking changes like the TAP proposal)... Gabriel Gonzalez maintains a
[State of Haskell Ecosystem](http://www.haskellforall.com/2015/08/state-of-haskell-ecosystem-august-2015.html) page that provides
interesting overview of what's hot and what's not in the Haskell world.

Here are some major challenges that lie ahead of us to improve our system:

* **Microservices**: In spite of good initial intention we still have built a monolith, albeit a small one and one that will be not
  too hard to split. We now want to increase robustness, resilience and scalability of our development process and our system by
  breaking the monolith into components services. We are in the process of splitting the application into smaller constituents along
  the following lines: 
  * Glue code to wire things together in a single app, 
  * Support code,
  * One component per group of related services,
  * One component per "Model", possibly clustered,
  * Ideally each component should be deployable independently or alongside other components in the same process depending on needed
    granularity and redundancy. This can be achieved easily through configuration at the level of the glue code according to some
    topology configuration.
* **Performance**: I tried to follow this simple development principle:
  [Make it, make it right, make it fast](http://c2.com/cgi/wiki?MakeItWorkMakeItRightMakeItFast). We mostly are done with the first
  part and are quite advanced on the second, so making it fast will be our next challenge especially as user base and data set grow
  in size. There are quite a few areas of improvement on the front: Caching computations, better data structures, improve strictness
  in key areas... But of course the first step will be to measure and set goals for those performance improvements.
* **Robustness**: No system is ever safe from failure but it depends on us what the impact of a failure is. This is definitely a
  must-have and something we can improve using standard replication and redundancy techniques. Splitting the system in finer-grained
  components is a first step towards that goal but we need specific components to ensure consistency in presence of failure.

# Conclusion

This article is already quite long yet it is only a brief overview of our system: It is hard to summarize one year of intense work!
In future installments of this blog post series I plan to address other aspects of the system that were not covered here:
Development and production infrastructure, user interface development from the point of view of a backend developer, development
process.

As a final note, my highest gratitude goes to the following persons without the help of whom this adventure would not have been possible: Pawel
Kuznicki, Chun Dong Chau, Pete Bonee, Willem van den Ende, Carlos Cunha, Guo Liang "Sark"
Oon, Amar Potghan, Konrad Tomaszewski and all the great people at [Capital Match](http://www.capital-match.com). I also would like
to thank Corentin Roux-dit-Buisson, Neil Mitchell, Joey Hess for their support and feedback.

[^1]: Although one could argue that there exists "languages" like Excel that allow you to write complex queries and explore data in
a very sophisticated way without the use of SQL

[^3]: This thread is pretty much redundant with storage thread for the moment. The plan is to use it for serialising
`applyCommand` operations on the models 

[^4]: It looks like our Om will be soon superceded by [om.next](https://github.com/omcljs/om/wiki/Quick-Start-%28om.next%29) 

[^5]: Being a startup in the B2B space, it does not make that much sense for Capital Match to talk about *infinite scaling* as we
don't expect our workload to expand beyond the limit of a single-server in the foreseeable future. 

[^6]: I have been using RDBMS since the 90's, developed a point-of-sale application in Access, have been using PostgreSQL through
its various versions since 1998, and recently worked on integrating DB migration process into a very large system. I am not
an expert but I have had quite an extensive experience of relational databases over a significant number of years and I have always
found that *writing* to DB quickly became a painful things.

[^7]: [Servant](http://haskell-servant.github.io/) is definitely on our roadmap.

[^8]: We use [QuickCheck](https://hackage.haskell.org/package/QuickCheck) to generate a bunch of events for the type of interest.

[^9]: I plan to provide more insights on our development and operations process in another blog post, but to give rough ideas we
have deployed our application about a hundred times in the past 6 months.

[^10]: Having a strong type system is no replacement for a decent test suite however, because obviously a lot of bugs happen at the
boundaries of the system, e.g. when invoking REST API.

[^11]: Obviously, this works as long as your data fits in memory. My bet is this will be the case for
[quite a long time](http://yourdatafitsinram.com/). Shall this ever become a
[problem](https://gettingreal.37signals.com/ch04_Scale_Later.php), we will most probably be in a position to handle it.

[^12]:  This is the approach I took in [hdo](https://github.com/capital-match/hdo/blob/master/src/Network/DO/Types.hs) because
external representation was already defined, and in the end it makes encoding more explicit and easier to work with 