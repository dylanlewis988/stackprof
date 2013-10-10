## stackprof

a sampling call-stack profiler for ruby 2.1+

inspired heavily by [gperftools](https://code.google.com/p/gperftools/),
and written as a replacement for [perftools.rb](https://github.com/tmm1/perftools.rb)

### sampling

three sampling modes are supported:

  - cpu time (using `ITIMER_PROF` and `SIGPROF`)
  - wall time (using `ITIMER_REAL` and `SIGALRM`)
  - object allocation (using `RUBY_INTERNAL_EVENT_NEWOBJ`)

samplers have a tuneable interval which can be used to reduce overhead or increase granularity:

  - cpu time: sample every _interval_ microseconds of cpu activity (default: 10000 = 10 milliseconds)
  - wall time: sample every _interval_ microseconds of wallclock time (default: 10000)
  - object allocation: sample every _interval_ allocations (default: 1)

samples are taken using a combination of three new C-APIs in ruby 2.1:

  - signal handlers enqueue a sampling job using `rb_postponed_job_register_one`.
    this ensures callstack samples can be taken safely, in case the VM is garbage collecting
    or in some other inconsistent state during the interruption.

  - stack frames are collected via `rb_profile_frames`, which provides low-overhead C-API access
    to the VM's call stack. no object allocations occur in this path, allowing stackprof to collect
    callstacks in allocation mode.

  - in allocation mode, samples are taken via `rb_tracepoint_new(RUBY_INTERNAL_EVENT_NEWOBJ)`,
    which provides a notification every time the VM allocates a new object.

### aggregation

each sample consists of N stack frames, where a frame looks something like `MyClass#method` or `block in MySingleton.method`.
for each of these frames in the sample, the profiler collects a few pieces of metadata:

  - samples: number of samples where this was the topmost frame
  - total_samples: samples where this frame was in the stack
  - lines: samples per line number in this frame
  - edges: samples per callee frame (methods invoked by this frame)

the aggregation algorithm is roughly equivalent to the following pseudo code:

``` ruby
trap('PROF') do
  top, *rest = caller

  top.samples += 1
  top.lines[top.lineno] += 1
  top.total_samples += 1

  prev = top
  rest.each do |frame|
    frame.edges[prev] += 1
    frame.total_samples += 1
    prev = frame
  end
end
```

this technique builds up an incremental callgraph from the samples. on any given frame,
the sum of the outbound edge weights is equal to total samples collected on that frame
(`frame.total_samples == frame.edges.values.sum`).

### reporting

three reporting modes are supported:
  - text
  - dotgraph
  - source annotation

#### text

`StackProf::Report.new(data).print_text`

```
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
        91  (48.4%)          91  (48.4%)     A#pow
        58  (30.9%)          58  (30.9%)     A.newobj
        34  (18.1%)          34  (18.1%)     block in A#math
       188 (100.0%)           3   (1.6%)     block (2 levels) in <main>
       185  (98.4%)           1   (0.5%)     A#initialize
        35  (18.6%)           1   (0.5%)     A#math
       188 (100.0%)           0   (0.0%)     <main>
       188 (100.0%)           0   (0.0%)     block in <main>
       188 (100.0%)           0   (0.0%)     <main>
```

#### dotgraph

`StackProf::Report.new(data).print_graphviz`

![](http://cl.ly/image/2t3l2q0l0B0A/content)

```
digraph profile {
  70346498324780 [size=23.5531914893617] [fontsize=23.5531914893617] [shape=box] [label="A#pow\n91 (48.4%)\r"];
  70346498324680 [size=18.638297872340424] [fontsize=18.638297872340424] [shape=box] [label="A.newobj\n58 (30.9%)\r"];
  70346498324480 [size=15.063829787234042] [fontsize=15.063829787234042] [shape=box] [label="block in A#math\n34 (18.1%)\r"];
  70346498324220 [size=10.446808510638299] [fontsize=10.446808510638299] [shape=box] [label="block (2 levels) in <main>\n3 (1.6%)\rof 188 (100.0%)\r"];
  70346498324220 -> 70346498324900 [label="185"];
  70346498324900 [size=10.148936170212766] [fontsize=10.148936170212766] [shape=box] [label="A#initialize\n1 (0.5%)\rof 185 (98.4%)\r"];
  70346498324900 -> 70346498324780 [label="91"];
  70346498324900 -> 70346498324680 [label="58"];
  70346498324900 -> 70346498324580 [label="35"];
  70346498324580 [size=10.148936170212766] [fontsize=10.148936170212766] [shape=box] [label="A#math\n1 (0.5%)\rof 35 (18.6%)\r"];
  70346498324580 -> 70346498324480 [label="34"];
  70346497983360 [size=10.0] [fontsize=10.0] [shape=box] [label="<main>\n0 (0.0%)\rof 188 (100.0%)\r"];
  70346497983360 -> 70346498325080 [label="188"];
  70346498324300 [size=10.0] [fontsize=10.0] [shape=box] [label="block in <main>\n0 (0.0%)\rof 188 (100.0%)\r"];
  70346498324300 -> 70346498324220 [label="188"];
  70346498325080 [size=10.0] [fontsize=10.0] [shape=box] [label="<main>\n0 (0.0%)\rof 188 (100.0%)\r"];
  70346498325080 -> 70346498324300 [label="188"];
}
```

#### source annotation

`StackProf::Report.new(data).print_method(/pow|newobj|math/)`

```
A#pow (/Users/tmm1/code/stackprof/sample.rb:11)
                         |    11  |   def pow
   91  (48.4% / 100.0%)  |    12  |     2 ** 100
                         |    13  |   end
A.newobj (/Users/tmm1/code/stackprof/sample.rb:15)
                         |    15  |   def self.newobj
   33  (17.6% /  56.9%)  |    16  |     Object.new
   25  (13.3% /  43.1%)  |    17  |     Object.new
                         |    18  |   end
block in A#math (/Users/tmm1/code/stackprof/sample.rb:21)
                         |    21  |     2.times do
   34  (18.1% / 100.0%)  |    22  |       2 + 3 * 4 ^ 5 / 6
                         |    23  |     end
A#math (/Users/tmm1/code/stackprof/sample.rb:20)
                         |    20  |   def math
    1   (0.5% / 100.0%)  |    21  |     2.times do
                         |    22  |       2 + 3 * 4 ^ 5 / 6
```

### usage

the profiler is compiled as a C-extension and exposes a simple api: `StackProf.run(mode, interval)`.
the `run` method takes a block of code and returns a profile as a simple hash.

``` ruby
profile = StackProf.run(sampling_mode, sampling_interval) do
  MyCode.execute
end
```

this profile data structure is part of the public API, and is intended to be saved
(as json/marshal for example) for later processing. the reports above can be generated
by passing this structure into `StackProf::Report.new`.

the format itself is very simple. it contains a header, and a list of frames. each frame has a unique id and
identifying information such as the name, file, line. the frame also contains sampling data, including per-line
samples, and a list of relationships to other frames represented as weighted edges.

```
{:version=>1.0,
 :mode=>"cpu(1000)",
 :samples=>188,
 :frames=>
  {70346498324780=>
    {:name=>"A#pow",
     :file=>"/Users/tmm1/code/stackprof/sample.rb",
     :line=>11,
     :total_samples=>91,
     :samples=>91,
     :lines=>{12=>91}},
   70346498324900=>
    {:name=>"A#initialize",
     :file=>"/Users/tmm1/code/stackprof/sample.rb",
     :line=>5,
     :total_samples=>185,
     :samples=>1,
     :edges=>{70346498324780=>91, 70346498324680=>58, 70346498324580=>35},
     :lines=>{8=>1}},
```

### advanced usage

the profiler can be started, paused, resumed and stopped manually for greater control.

```
StackProf.start
StackProf.pause
StackProf.paused?
StackProf.resume
StackProf.running?
StackProf.stop
StackProf.results
```
