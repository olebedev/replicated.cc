# swarmdb testing plan 

Databases are likely the 
[most](https://www.sqlite.org/testing.html)
heavily
[tested](https://news.ycombinator.com/item?id=18442637)
software products.
Quite fortunately, swarmdb does not do any serious database things.
It only adds CRDT data structures to RocksDB.
That's why it only has four-five kinds of tests:

## Unit tests at `{ron,rdt,db}/test/`

Unit tests minimally check the freshly written code.
Often, tests are written before the code, just to solidify API.
Their task is to make some basic sanity checks and initial debugging.

## ACID (permutation) tests at `rdt/perm-test/`

Replicated Data Types have to be associative, commutative, idempotent.
How do we ensure that? Well, literally.
ACID tests take an input (a sequence of ops) and feed them into a reducer
in arbitrary batches (associativity), arbitrarily reordered (commutativity)
and arbitrarily duplicated (idempotence).
The reducer may not be able to merge every possible subset/permutation,
but: it must handle error conditions correctly!
After each run, we compare the results. They must stay the same.

## Black box tests at `db/bb-test/`

The simplest way of testing a complete system is blackbox testing:
we have a library of inputs and outputs, we feed inputs into the system,
we compare the results.

## Stress tests at `db/stress-test/`

The priority of this part is low, as most of heavylifting is done by RocksDB.
Still, We must ensure that the system may consume high-volume streams of operations,
handle unusually large objects, process larger datasets. 
Importantly, the system must politely reject any above-the-limit inputs.

## Fuzz tests at `{ron,rdt,db}/fuzz-test/`

> Fuzzing or fuzz testing is an automated software testing technique that
> involves providing invalid, unexpected, or random data as inputs to a
> computer program. The program is then monitored for exceptions such as
> crashes, failing built-in code assertions, or potential memory leaks. 

[Fuzz testing](https://en.wikipedia.org/wiki/Fuzzing) is an epic topic.
I started with [AFL](http://lcamtuf.coredump.cx/afl/), although that one seems seems to be EOLed.
[Clang](http://llvm.org/docs/LibFuzzer.html) has a gorgeous fuzzer that plays along with clang sanitizers.
That produces a nice cumulative effect.

## Continuous integration

Once different testing methods get employed in combination, the effect becomes even more cumulative :)
For 5 types of tests, we have ~30 possible combinations and most of them actually make sense, as far as I can see.
Actually, it is even more fun. Consider an example:

1. stress testing generates a large synthetic input
2. fuzz testing takes that as a starting point, increasing coverage by guided mutations
3. the resulting dataset could be fed back into stress testing to increase its coverage

AFAIU, it would be difficult to fuzz and stress at the same time.
That is because stress tests do lots of actual disk writes; they are not stateless.
This kind of a "combo" may solve that.

Overall, any interesting combos will be added to the continous itegration scripts.
Those scripts are ran by this old gentleman, 24x7. We are not Google, we only have one or two of those.
And given the Amazon EC2 prices (c4.2xlarge, dedicated), it will take this device *a month* to break even.

<img src="xeon.jpg" style="width: 100%"/>