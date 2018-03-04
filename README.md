A helper script to count the ratio of unique branching point values in
[american fuzzy lop](http://lcamtuf.coredump.cx/afl/) instrumented
programs. It is meant to give a quick overview if the amount of
instrumentation in the program is likely too much to be effective when
fuzzed with `afl-fuzz` command.

## Usage

```
$ count-afl-calls afl-instrumented-binary
Total: 8061
Unique: 7558
Repeating: 479
$ count-afl-calls afl-instrumented-library.so
Total: 12067
Unique: 11040
Repeating: 968
```

This script uses [GNU bash](https://www.gnu.org/software/bash/) to run
itself, [objdump](https://www.gnu.org/software/binutils/) to do its
main task, and some
[GNU coreutils](https://www.gnu.org/software/coreutils/coreutils.html)
programs to calculate the final result.

## Background

American fuzzy lop instrumentation works inherently in probabilistic
fashion where each program branch point gets a random value assigned
by the compiler at compilation time. At each branch point the current
branch point value and the previous branch point value are combined
together and the resulting value is registered in a map. This map
registers the amount of times that the program visits certain branch
points pairs.

The space where these branch point values are getting values
corresponds to the size of the map where the count of visited branch
points are registered. This limited space means that due to the
[birthday paradox](https://en.wikipedia.org/wiki/Birthday_problem) it
is probable that two different branching points get the same
value. This probability rises quite fast when the program size
increases. This alone is not enough to cause collisions in the map
that collects the results. As the map size is as big as the space
where branch points get their values, the likelihood of different
taken branches resulting in the same map value also increases.

## Approaches to collision probability reduction

By default the map size is 16 bits that results in 65536 unique values
that can be registered. This is fine for most of the small programs
with a very small likelihood of collision, but medium and large
programs will naturally tend to suffer from this.

First approach that is independent of american fuzzy lop is to reduce
the part of the program that is fuzzed, usually called fuzz
target. This means that if your fuzz target first does decompression
and then actual data processing, you should create a separate fuzz
targets for both decompression and data processing. This then leads
into more effective fuzzing of both parts. This script can result in
warnings about big binaries especially in programs that do many types
and formats of media processing in the same binary. These may or may
not be detrimental for fuzzing efficiency and the final verdict should
be done by looking at the `map coverage` section of
[afl-fuzz status screen](http://lcamtuf.coredump.cx/afl/status_screen.txt).

American fuzzy lop provides two approaches to reduce the probability
of collisions in the compiled program; `AFL_INST_RATIO` environmental
variable at runtime and american fuzzy lop's
[`MAP_SIZE` value](https://github.com/mirrorer/afl/blob/master/config.h#L310)
at compilation time.

Changing the value of `AFL_INST_RATIO` makes the program to have
smaller instrumentation coverage and all branches do not get any
coverage at all. If you want to have the probability that your program
is more thoroughly covered, you need to compile your program multiple
times and run different afl instrumented compilations of it in
different `afl-fuzz` instances that can see each other.

Changing `MAP_SIZE` value (65536 by default) at american fuzzy lop's
`config.h` file to bigger one makes the collision probability smaller
by increasing the space where these random values for instrumentation
come from. It also makes each `afl-fuzz` run slower, as now there is a
bigger map to go through after each fuzzing iteration. This also
requires that you compile american fuzzy lop yourself, which makes
this only viable when you are willing to invest more than a quick look
at fuzzing with american fuzzy lop.
