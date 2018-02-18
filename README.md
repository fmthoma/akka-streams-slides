Streaming for Big Data Applications
================================================================================

Streaming Big Data
--------------------------------------------------------------------------------

* Billions of events per day (Terabytes!)
* (Near) real-time processing
* Fault tolerance

* Bounded: Batch processing
* Unbounded: Stream processing


Why Use Akka Streams?
--------------------------------------------------------------------------------

* Type-safe
* Compositional
* High-level
* Explicit semantics
* Integrates well (Alpakka)
* Fast (fusion & other optimizations!)

<div class="notes">

* Explicit semantics (as opposed to Flink, which tries to figure out
  batching/grouping/sharding by itself)

</div>



Building Blocks of Akka Streams
================================================================================

Introductory Example
--------------------------------------------------------------------------------

```scala
Source(List(1, 2, 3))
    .via(Flow().map(_ + 1))
    .to(Sink.foreach(System.out.println(_)))
    .run()
```

. . .

```bob

                       via                  to
+-----------------------.--------------------.-------------------+
| "Source(List(1, 2, 3))"] "Flow().map(_ + 1)"] "Sink.foreach(_)"|
+-----------------------'--------------------'-------------------+
```


Sources, Sinks and Flows
--------------------------------------------------------------------------------

```bob

                   .--------------------.
             A -->  ] "Flow[-A, +B, Mat]"] --> B
                   '----------+---------'
                              +-> Mat

+-----------------.                       .-----------------+
| "Source[+A, Mat]"] --> A          A -->  ] "Sink[-A, Mat]"|
+---------+-------'                       '-------+---------+
          +-> Mat                                 +-> Mat

```

. . .

```bob

+---------------------+
| "RunnableGraph[Mat]"|
+----------+----------+
           +-> Mat
```

<div class="notes">

* A `Source` emits (produces) items of type `A`
* A `Sink` accepts (consumes) items of type `A`
* A `Flow` accepts items of type `A` and emits items of type `B`.
* A `RunnableGraph` is a black box that neither consumes nor produces items, but
  it still returns a materialized value (and performs side effects!).

</div>


Materialized Values
--------------------------------------------------------------------------------

```bob
               .--------------------+
ByteString -->  ] "FileIO.toPath(_)"|
               '----------+---------+
                          +-> "Future[IOResult]"
```

Each stream element allows to return some information on the items processed.

* `NotUsed` (no information available)
* The number of elements processed
* The result (Success/Failure) of an IO action (`Future[IOResult]`)
* Items collected from the stream (`Future[Seq[A]]`)

<div class="notes">

Note: The materialized value becomes avaialable when the stream is *started*,
not when it is finished. Hence, `Mat` is often a `Future[…]` that will be
completed when the stream is finished.

</div>


Connecting Stream Elements: `via` and `to`
--------------------------------------------------------------------------------

```bob
                     via
+---------------------.----------------------.    +----------------------------.
| "s: Source[A, Mat]"  ] "f: Flow[A, B, Mat2]"] = | "s.via(f): Source[B, Mat]"  ]
+---------------------'----------------------'    +----------------------------'

                     via
.---------------------.----------------------.    .----------------------------.
 ] "f: Flow[A, B, Mat]"] "g: Flow[B, C, Mat2]"] =  ] "s.via(f): Flow[A, C, Mat]"]
'---------------------'----------------------'    '----------------------------'

                     to
.---------------------.-----------------------+   .------------------------------+
 ] "f: Flow[A, B, Mat]"] "t: Sink[B, Mat2]"   | =  ] "f.to(t): Sink[A, Mat]"     |
'---------+-----------'-----------------------+   '--------------+---------------+

                     to
+---------------------.-----------------------+   +------------------------------+
| "s: Source[A, Mat]"  ] "t: Sink[A, Mat2]"   | = | "s.to(t): RunnableGraph[Mat]"|
+---------+-----------'-----------------------+   +--------------+---------------+
```

<div class="notes">

`via` composes the the outlet of a `Source` or `Flow` with another `Flow`,
keeping the materialized value.

`to` connects the the outlet of a `Source` or `Flow` to a `Sink`, keeping the
materialized value.

By default, only the materialized value of the upstream flow is kept. There
exist variants (`viaMat`, `toMat`) that allow to keep/merge the materialized
values from both flows.

</div>


Tee Pieces: `alsoTo`
--------------------------------------------------------------------------------

```bob
+--------------------.--------.                     +------------------------------.
| "s: Source[A, Mat]" ] alsoTo ]                  = | "s.alsoTo(t): Source[A, Mat]" ]
+--------------------'-+  +---'                     +------------------------------'
                       |  +-.--------------------+
                       |     ] "t: Sink[A, Mat2]"|
                       +----'--------------------+


.--------------------.--------.                     .------------------------------.
 ]"f: Flow[A, B, Mat]"] alsoTo ]                  =  ]"f.alsoTo(t): Flow[A, B, Mat]"]
'--------------------'-+  +---'                     '------------------------------'
                       |  +-.--------------------+
                       |     ] "t: Sink[B, Mat2]"|
                       +----'--------------------+
```

<div class="notes">

By default, only the materialized value of the upstream flow is kept, but
`alsoToMat` allows to keep/merge the materialized values from the `Sink`.

</div>


Some `Source`, `Flow` and `Sink` examples
--------------------------------------------------------------------------------

```scala
Source[T](xs: Iterable[T]): Source[T, NotUsed]

Sink.ignore: Sink[Any, Future[Done]]

Sink.foreach[T](f: T => Unit): Sink[T, Future[Done]]
// e.g. Sink.foreach(System.out.println(_))

Sink.fold[U, T](zero: U)(f: (U, T) => U): Sink[T, Future[U]]

Flow.fromFunction[A, B](f: A => B): Flow[A, B, NotUsed]

FileIO.fromPath(f: Path): Source[ByteString, Future[IOResult]]
FileIO.toPath(f: Path): Sink[ByteString, Future[IOResult]]

// Akka HTTP client/server is a Flow[HttpRequest, HttpResponse, …]:
Http.outgoingConnection(…): Flow[HttpRequest, HttpResponse, Future[OutgoingConnection]]
Http.bindAndHandle(handler: Flow[HttpRequest, HttpResponse, Any], …)
```


Materialization
--------------------------------------------------------------------------------

`Source`s, `Sink`s and `Flow`s are just *blueprints*. `RunnableGraph.run` builds
and optimizes the actual stream.

```scala
implicit val system : ActorSystem = ActorSystem()
implicit val materializer : Materializer = ActorMaterializer()

val blueprint: RunnableGraph[Future[Int]] =
  Source(List(1, 2, 3)).toMat(Sink.fold(0)(_ + _))(Keep.right)

val result: Future[Int] = blueprint.run
```

<div class="notes">

* `run(implicit materializer: Materializer)`
* Optimization: Fusion!

</div>




Backpressure
================================================================================

What is Backpressure?
--------------------------------------------------------------------------------

```scala
FileIO.fromPath(Paths.get("requests.txt"))                       // <- fast-ish
    .via(Framing.delimiter("\n", 1024))                          // <- fast
    .via(Fold.fromFunction(request => send(request)))            // <- slow
    .to(Sink.foreach(response => System.out.println(response)))  // <- fast
```

Default backpressuring: Only produce/consume as fast as the slowest link in the
chain.


Backpressure Boundaries
--------------------------------------------------------------------------------

What if I can't (or don't want to) control the speed of a `Source`?

```scala
incomingRequests       // <- will turn away requests if they come too fast
    .to(slowSink)
```

```scala
incomingRequests                               // <- won't turn away requests
    .buffer(50, OverflowStrategy.dropNew)      // <- may lose requests
    .to(slowSink)
```

```scala
incomingRequests                                 // <- will turn away requests
    .buffer(50, OverflowStrategy.backpressure)   //    if buffer is full
    .to(slowSink)
```


Throttling
--------------------------------------------------------------------------------

What if a `Sink` chokes if items come in too fast?

```scala
fastSource                                                  // <- fast
    .to(chokingSink)                                        // <- chokes :-(
```

```scala
fastSource                                                  // <- fast
    .throttle(elements = 5, per = 1 second, mode = shaping) // <- slow down!
    .to(chokingSink)                                        // <- doesn't choke
```

<div class="notes">

e.g.: Web server that can only handle a certain amount of requests

</div>

Backpressure Boundaries (II)
--------------------------------------------------------------------------------

```scala
fastSource
    .alsoTo(slowSink)       // <- slows everything down :-(
    .to(fastSink)
```

```scala
fastSource
    .alsoTo(Flow()
        .buffer(50, backpressure)  // <- Tries to buffer
        .to(slowSink))             //    before slowing everything down
    .to(fastSink)
```

Particularly useful if the source produces at irregular intervals.


Batching
--------------------------------------------------------------------------------

Another way to connect a fast `Source` to a slow `Sink`:

```scala
fastSource
    .batch(max = 10, seed = List(_))(_ :+ _)   // <- backpressures only if
                                               //    batch size is exceeded
    .to(slowBatchSink)                         // <- consumes batches faster
                                               //    than individual elements
```



Graph DSL for More Complex Streams
================================================================================

Talk About Shapes
--------------------------------------------------------------------------------

```bob
+----------------.   .------------------.   .----------------+
| "SourceShape[A]"]   ] "FlowShape[A, B]"]   ] "SinkShape[B]"|
+----------------'   '------------------'   '----------------+

.------------------------------------------------.
 ]  in            "FlowShape[A, B]"           out ]
'------------------------------------------------'

           .-------------------------------------.
          /                               "out(1)"]
.--------'                                       '
 ]  in          "UniformFanOutShape[A, B]"       :
'--------.                                       .
          \                               "out(n)"]
           '-------------------------------------'
```


Shapes & Graphs
--------------------------------------------------------------------------------

`Flow[A, B, Mat]`

is just a decorator around

`Graph[FlowShape[A, B], Mat]`

<div class="notes">

Most of the time, you can use them interchangeably. Else, just use
`Flow`/`Source`/`Sink.fromGraph`.

</div>


Some Useful Shapes
--------------------------------------------------------------------------------

* `SourceShape[A]` (`Source[A, Mat]`)
* `SinkShape[A]` (`Sink[A, Mat]`)
* `FlowShape[A, B]` (`Flow[A, B, Mat]`)
* `ClosedShape` (`RunnableGraph[Mat]`)
* `FanOutShape2[A, B1, B2]`, `FanInShape2[A1, A2, B]` (up to 22 inlets/outlets)
* `BidiShape[In1, Out1, In2, Out2]`


Constructing Graphs from Shapes
--------------------------------------------------------------------------------

Balancing between workers:

```scala
def balanced[S, T, Mat >: Any](workers: Seq[Graph[FlowShape[S, T], Mat]]): Flow[S, T, NotUsed] =
  Flow.fromGraph(GraphDSL.create() { implicit builder =>

    import GraphDSL.Implicits._

    val n = workers.length
    val balance: UniformFanOutShape[S, S] = builder.add(Balance[S](n))
    val merge: UniformFanInShape[T, T] = builder.add(Merge[T](n))

    for (i <- 0 until n) {
      balance.out(i) ~> workers(i).async ~> merge.in(i)
    }

    FlowShape(balance.in, merge.out)
  })
```

Also one way to speed up slow `Flow` elements!



Connecting to the World
================================================================================

Akka Streams Connectors
--------------------------------------------------------------------------------

* Akka HTTP
* Slick (JDBC, Functional Relational Mapping)
* Apache Kafka
* Apache Camel
* AWS (S3, Kinesis, …)
* Have a look at [Alpakka][] for more connectors

[Alpakka]: https://developer.lightbend.com/docs/alpakka/current/



Thank you!
================================================================================

Questions?
--------------------------------------------------------------------------------

. . .

Slides on Github: [fmthoma/akka-streams-slides](https://github.com/fmthoma/akka-streams-slides)

[fmthoma](https://github.com/fmthoma) on Github

[fmthoma](https://keybase.io/fmthoma) on keybase.io

[franz.thoma@tngtech.com](mailto://franz.thoma@tngtech.com)
