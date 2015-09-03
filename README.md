[![Build Status](https://travis-ci.org/dk14/rflows.svg)](https://travis-ci.org/dk14/rflows)
[![codecov.io](http://codecov.io/github/dk14/rflows/coverage.svg?branch=master)](http://codecov.io/github/dk14/rflows?branch=master)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/dk14/rflows?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

# rflows

This is embedded reactive and typesafe Scala-based DSL for declaring flows using functional composition.

![Graph1](/graph.png)

To get started with SBT, simply add the following to your build.sbt file:

```scala
  resolvers += Resolver.sonatypeRepo("snapshots")
  libraryDependencies += "com.github.dk14" %% "rflows" % "0.2-SNAPSHOT"
```

####Flow
sequentially executes several acts, splitters, aggregators or other flows

Define routing (like `Routing1`)
 
To create new application-level flow - just extend `InstrumentedRouting` trait and define your flow inside:

    trait Routing1 extends InstrumentedRouting[Request] with MyService1 with MyService2 {
      val Flow1 = Act("act1", handler1) |> Act("act2", handler2) |> 
        Split("split1", splitter1) |> Aggregate("aggregate1", aggregate) |> Act("act6", handler6)
    }

define flow as composition (`|>`) of acts, splitters and aggregators

`handlerN` - handler for current flow step
`MyServiceN` - service-layer used by handlers. It's recommended to inject those services in mix-in style.

##### Simple Act 
just executes a simple processor

    Act.simple("act1", handler1)

`handler1` takes input message and returns output message

##### Act
executes a processor and moving to next act (or returning result) in reactive way
    
    Act("act1", handler1)

`handler1` takes input message and returns future of output message, so next act in chain (act2) is executed only when future completes. In combination with scala's Futures composition, it gives you a way to acquire asynchronous services and build processing in reactive way.

##### FlowGroup and SubFlows
groups several sub-flows to be used inside one `Split` (sse below)

    implicit object FlowGroup1 extends Group {
      val SubFlow1 = Act("act3", handler3) |> Act("act4", handler4) tagged //do not forget "tagged" word !!
      val SubFlow2 = Act("act5", handler5) tagged
    }


##### Split
allows to split a message into several ones and send each part to its own (dynamically chosen) flow in reactive way. You can also send same message to different flows.

    Split("split1", splitter1)

    def splitter1(in: Data[Request]) = Seq( //same message, different flows
        Future(in) -> SubFlow1, 
        Future(in) -> SubFlow2
    )
 
    //or
 
    def splitter1(in: Data[Request]) = Seq( //different messages, different flows
        Future(in.part1) -> SubFlow1,
        Future(in.part2) -> SubFlow2
    ) 
    
    //or just
    
    Split("split1", simpleSplitter)
    
    def simpleSplitter(in: Data[Request]) = Seq(
        in.part1 -> SubFlow1,
        in.part2 -> SubFlow2
    )
    

`splitter1` function takes message and returns list of directives, each directive is `Future[Message] -> Flow`, which means that it contains a new message (may be just a copy of input and flow to process this message.

>All flows (returned by splitter) should be part of same group (see below). For instance, `SubFlow1` and `SubFlow2` are parts of `FlowGroup1` (see the diagram).

>Otherwise, you get scary compilation error, which is intentional protection from SubFlow Hell

##### Route
route process a message and route a result to dynamically chosen flow (in reactive way)
Same as `Split` but `route1` returns list of only one directive (with flow chosen based on input message) to process. It can be implemented with same Split construction:

    Split("route1", router1)

    def router1(in: Data[Request]) = 
       if (in.isForSubFlow1)  
          Seq(Future(in.data) -> SubFlow1) else Seq(Future(in.data) -> SubFlow2)
    
    //or just
    
    Split.route("route", router) = //note that this one doesn't need aggregation
      if (in.isForSubFlow1)  
          in.data -> SubFlow1 else in.data -> SubFlow2


##### Aggregate
aggregates messages after being split

    Split("split1", splitter1) |> Aggregate("aggregate1", aggregator1)

Shoud be placed right after corresponding `Split`

##### How to run a flow
it's just a function

  Flow1(Seq(incomingMessage1), context)
  
If you specify several incoming messages, they're going to be processed in parallel (it's like a resuming after `Split`). Usually you don't need that.

#### Basic terms


#####Data

Data is a wrapper around message. It contains the message itself, and also its Context and Meta-info

#####Context

Context is an object shared between all Flows(like Flow1). inside one Routing (like Routing1).
You can specify its type when extending InstrumentedRouting trait (see below) and specify an object itself when passing incoming message to a flow, like `Flow(incomingMessage, context)`
Usually the context is current request, so just `Flow(incomingMessage, incomingMessage)`

#####Meta

Meta - is meta-information which contains current executing flow itself (including name), so you can propagate it to handler, services etc. Used for metrics and logging to define the concrete flow (act, splitter or aggregator) where some measured/logged event is actually takes place.

####See also

See `RoutingDSLTest.scala` for complete working example 

TODO: Big Flow Decomposition

TODO: "how to write handler" examples

TODO: Flow and Routing terms

TODO: how to register before, after, failure handlers
