## Very low-impact tracing
#### of high-performance
#### C++ application


_Maciej Gajewski, October 2021_

Note:

I'm really sorry for the long title :)

====

#### About me

* Maciek Gajewski [maciej.gajewski0@gmail.com](mailto:maciej.gajewski0@gmail.com)

<img src="img/maciek.jpg" height="200"/>

Note:

* 35 years of programming, 25 years of C++
* 2000-2010 Wrocław (Pruftechnik, Tieto)
* 2010-2018 London, Amsterdam, 
* HFT, teaching (Tibra, Optiver)

====

<!-- .slide: data-background-iframe="https://auros.global/" -->

Note:

* Currently with a Hong Kong - based Crypto trading firm.
* End-to-end trading system
* Majority of the code in Python, but the fast bits in C++
* This presentation is about data solving actual production problem

====

## The problem

<img src="img/app-loop.png" />

Note:

* This is the main loop of the application
* In each loop cycle a number of work items is executed
* Sometimes these are timer or socket events,
  but mostly work items reading data from an SHM object and processing it.
* At the end of every cycle we measure the duration and collect some statistics.

====

## The problem

<img src="img/app-loop-w-time.png" />

Note:

* After each loop we measure the duration
* The duration mostly depends on number of configured inputs
* For this particular application the median duration was around 400us
* This means that the latency introduced was up to 400us, 200us on average
* This was chosen as an acceptable value
* But sometimes the loop duration spiked to few ms, and this is not acceptable!
* The spike could happen several times per second.

====

## Searching for solution

* Sampling profiles are useless
* I was unable to find existing tracing tool that would work
* LTTng, frysk, minitrace, easy_profiler
* Estimated 1 probe per microsecond!

Note:

* Sampling profiles are worthless for inspecting a rare case
* Existing tracers take a lot of time and are too invasive.
* Maybe there exist a tool that would be useful, but I could't find one

====

### Requirements

* Absolutely minimal impact on running application
* Should allow for running application in conditions similar to the one in production, 
* for a prolonged period of time

Note:

The design principles of the tracing system:

* The impact on the application must be minimal,
* As the application takes some time to start up, it should allow for arbitrary long runs
* So no fixed-size storage!

====

### Operation

<img src="img/app-loop-collect-discard-store.png" />

Note:

The idea is to:
* collect events continuously
* evaluate each loop cycle, and if max busy time exceeded - store
* otherwise discard
* The data has to be processed off line, or otherwise outside of the app, to not affect the operation

====

### Tracing event

```cpp
enum class EventType : uint64_t { Enter, Exit };

struct Event {
    EventType type; // type of event
    uint64_t addr;  // code location address
    uint64_t tsc;   // timestamp from the TSC register
};
```

Note:

* The event has to be small, fixed-size record, easy and fast to store.
* The event type can be a simple enum (using 8 bytes, as they would be used by the
padding anyway, and maybe they will be useful)
* To identify the function, a code address is used. It has fixed size and is cheap to get. The function name can be obtained form it using DWARF debug info
* Time comes from the TSC register. This is the fastest way

====