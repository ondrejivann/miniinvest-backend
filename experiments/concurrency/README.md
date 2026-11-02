# Concurrency experiment: thread pool vs virtual threads vs coroutines

Course module 2 (JVM, Java and concurrency). The experiment measures how three concurrency models handle thousands of blocking calls to a simulated market-service, so that the thread model of invest-core and market-service is chosen from data, not from opinions. The decision is recorded in ADR 0002.

This is a standalone Gradle build. It is not part of any service, it is never deployed and it can be deleted once ADR 0002 is accepted.

## Question

In step S5 of the buy flow (`docs/architecture.md`), invest-core calls market-service synchronously for a price. How many of these calls can one service have in flight at once, how long does a batch of them take, and how does the answer change with the concurrency model?

## Setup

- Kotlin 2.4.21, `kotlinx-coroutines-core` 1.11.0 and Gradle 9.1 to 9.7 with the Kotlin DSL and the `application` plugin. Gradle 9.1 is the first version that runs on Java 25, and 9.7 is the newest one that Kotlin Gradle plugin 2.4.21 is tested with.
- Gradle runs on JDK 25 (the repository's `.sdkmanrc`), but the code is compiled for Java 21 with `kotlin { jvmToolchain(21) }`, so the same build runs on JDK 21 and on JDK 25 (see Pinning). Gradle finds a JDK 21 installed with SDKMAN on its own.
- Run the measurements with the start script that `./gradlew installDist` creates. It uses the JDK in `JAVA_HOME` and takes JVM options from `JAVA_OPTS`.
- `SimulatedMarketClient.price(isin)` blocks the calling thread for a configurable latency with `Thread.sleep`. It stands for a blocking HTTP or JDBC call. Do not use `delay` here: the point of the experiment is a blocking call.
- Command-line parameters: variant, number of calls (default 10 000) and latency (default 100 ms).

## Variants

| Variant | Implementation |
|---|---|
| `fixed-pool` | `Executors.newFixedThreadPool(200)`, the default maximum number of Tomcat threads |
| `virtual` | `Executors.newVirtualThreadPerTaskExecutor()`, one virtual thread per call |
| `coroutines-io` | one coroutine per call on `Dispatchers.IO` |
| `coroutines-io-1000` | one coroutine per call on `Dispatchers.IO.limitedParallelism(1000)` |
| `coroutines-default` | the blocking call on `Dispatchers.Default`, the anti-pattern, for comparison |
| `virtual-limited` | virtual threads with a `Semaphore(50)` in front of the client, the way invest-core protects market-service |
| `virtual-synchronized` | virtual threads, the blocking call inside a `synchronized` block on a lock object per call (see Pinning) |

Slow variants (`coroutines-default`, `virtual-synchronized` on JDK 21) can run with fewer calls, for example 1 000. Say so in the results.

## Measurements

Every run prints:

- wall time and throughput (calls per second),
- p50 and p99 latency of one call, measured from submission to completion, so the time spent waiting in a queue counts,
- the peak number of live platform threads (`ThreadMXBean.getPeakThreadCount()` from `ManagementFactory`),
- used heap after the run.

Run one warm-up round that is not counted, then three measured rounds, and report the median.

## Predict first

Before running anything, write the predicted wall time of every variant into `RESULTS.md` using Little's law: wall time ≈ number of calls × latency ÷ number of calls in flight. Then compare the prediction with the measurement and explain every difference larger than 20 %.

## Pinning on JDK 21 and JDK 25

Run `virtual-synchronized` on JDK 21 and on JDK 25 with a Flight Recorder recording and count the pinning events:

```bash
./gradlew installDist
JAVA_HOME=$(sdk home java <JDK 21 version>) JAVA_OPTS="-XX:StartFlightRecording=filename=pinning-21.jfr" \
  build/install/concurrency/bin/concurrency virtual-synchronized 1000
JAVA_HOME=$(sdk home java <JDK 25 version>) JAVA_OPTS="-XX:StartFlightRecording=filename=pinning-25.jfr" \
  build/install/concurrency/bin/concurrency virtual-synchronized 1000
jfr summary pinning-21.jfr
jfr print --events jdk.VirtualThreadPinned pinning-21.jfr
```

`jfr summary` prints the number of events per type, and `jfr print` adds the stack of each event. The paths assume `rootProject.name = "concurrency"` in `settings.gradle.kts`.

On JDK 21 a virtual thread that blocks inside `synchronized` pins its carrier thread, so the run is limited by the number of carriers. JDK 24 removed this with JEP 491. Record the wall time and the number of events for both JDKs.

## Results

`RESULTS.md` next to this file contains:

1. Environment: JDK vendor and version, CPU cores, memory, OS.
2. A table per variant: limit on calls in flight, predicted wall time, measured wall time (median), throughput, p50, p99, peak platform threads.
3. At least three conclusions that explain the differences between the variants.
4. The pinning comparison.

## Done when

- [ ] `experiments/concurrency` builds and runs on JDK 25 with one command documented in this README.
- [ ] All variants use the same workload and the same parameters.
- [ ] `RESULTS.md` has the predictions, the measurements and at least three conclusions.
- [ ] The pinning comparison on JDK 21 and JDK 25 includes the number of `jdk.VirtualThreadPinned` events.
- [ ] `docs/adr/0002-thread-model.md` records the thread model of invest-core and market-service in the format of ADR 0001: options, decision, business impact, cost and consequences (database connection pool, timeouts and the limit on calls to market-service).
- [ ] Every line of the code can be explained in review.
