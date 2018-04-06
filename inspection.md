# Inspection and Performance Profiling

## v8 flags

Multiple flags and so called _run time functions_ are available to anyone who likes to peek
into the inner workings of v8.

### AST

- `--print-ast` prints the AST generated by v8 to the console

### Byte Code

- `--print-bytecode` prints bytecode generated by ignition interpreter to the console
- provides more info when run with debug build d8, i.e. information about maps created

### Tracing Inline Caches

- `--trace-ic` dumps IC traces to the console
- pipe that output into `./v8/tools/ic-processor` to visualize it

### Optimized Code

- `--print-opt-code` prints the actual optimized code that is generated by TurboFan
- `--code-comments` adds comments to printed optimized code

### Tracing Optimizations

- `--trace-opt` traces lazy optimization
	-  _generic ICs_ are _bad_ as if lots of them are present, code will not be optimized
  - _ICs with typeinfo_ are _good_

### Tracing Map Creation

- `--trace-maps` in combination with `--trace-maps-details` trace map generation into v8.log
- `--expose-gc` allows forcing GC via `gc()` in your code to see which maps are short lived
- the output can be parsed to see what maps _hidden classes_ and transitions into other maps v8
  creates to represent your objects
- a graphical presentation is also available by loading the resulting `v8.log` into
  `/v8/tools/map-processor.htm` (requires v8 checkout)

#### Resources

- [The case of temporary objects in Chrome](http://benediktmeurer.de/2016/10/11/the-case-of-temporary-objects-in-chrome/)

### Runtime Call Stats

- `--runtime-call-stats` dumps statistics about the v8 runtime to the console
- these stats give detailed info where v8 time is spent

**Sample Output** (abreviated)

```

                      Runtime Function/C++ Builtin        Time             Count
========================================================================================
                                      JS_Execution      8.82ms  47.24%         1   0.11%
                              RecompileSynchronous      3.89ms  20.83%         7   0.75%
                                   API_Context_New      2.20ms  11.78%         1   0.11%
                             GC_SCAVENGER_SCAVENGE      0.88ms   4.73%        15   1.60%
                                AllocateInNewSpace      0.51ms   2.71%        71   7.59%
                       GC_SCAVENGER_SCAVENGE_ROOTS      0.38ms   2.04%        15   1.60%
                    GC_SCAVENGER_SCAVENGE_PARALLEL      0.24ms   1.29%        15   1.60%
         GC_SCAVENGER_BACKGROUND_SCAVENGE_PARALLEL      0.18ms   0.94%        15   1.60%
                         GC_Custom_SlowAllocateRaw      0.15ms   0.79%         6   0.64%
                      CompileForOnStackReplacement      0.13ms   0.70%         2   0.21%
                                      ParseProgram      0.13ms   0.70%         1   0.11%
                                   CompileIgnition      0.13ms   0.69%         4   0.43%
                      PreParseNoVariableResolution      0.11ms   0.59%         3   0.32%
                                      OptimizeCode      0.09ms   0.46%         5   0.53%
                                     CompileScript      0.08ms   0.41%         1   0.11%
                      Map_TransitionToDataProperty      0.07ms   0.40%        92   9.84%
                        InterpreterDeserializeLazy      0.07ms   0.36%        28   2.99%
                                  Map_SetPrototype      0.06ms   0.29%       249  26.63%
                                  FunctionCallback      0.05ms   0.27%         3   0.32%
                              ParseFunctionLiteral      0.04ms   0.22%         3   0.32%
                                  GC_HEAP_EPILOGUE      0.04ms   0.19%        15   1.60%
                                [ ...                   ...     ...         ...    ... ]
----------------------------------------------------------------------------------------
                                             Total     18.67ms 100.00%       935 100.00%
```

#### Resources

- [real world performance measurements](http://benediktmeurer.de/2016/12/20/v8-behind-the-scenes-december-edition/#real-world-performance-measurements)

### Memory Visualization

- v8 heap statistics feature provides insight into both the v8 managed heap and the C++ heap
- `--trace-gc-object-stats` dumps memory-related statistics to the console
- this data can be visualized via the [v8 heap visualizer](https://mlippautz.github.io/v8-heap-stats/)
  - make sure to not log to _stdout_ when generating the `v8.gc_stats` file
  - NOTE: when I tried this tool by loading a `v8.gc_stats` generated via
  `node --trace-gc-object-stats script.js  > v8.gc_stats` it errored
  - serving `v8 ./tools/heap-stats` locally had the same result

#### Resources

- [Optimizing V8 memory consumption](https://v8project.blogspot.com/2016/10/fall-cleaning-optimizing-v8-memory.html)

### Array Elements Kinds

- enable native functions via `--allow-natives-syntax`
- then use `%DebugPrint(array)` to dump information about this array to the console
- the `elements` field will hold information about the _elements kinds_ of the array

**Sample Output** (abreviated)

```
DebugPrint: 0x1fbbad30fd71: [JSArray]
 - map = 0x10a6f8a038b1 [FastProperties]
 - prototype = 0x1212bb687ec1
 - elements = 0x1fbbad30fd19 <FixedArray[3]> [PACKED_SMI_ELEMENTS (COW)]
 - length = 3
 - properties = 0x219eb0702241 <FixedArray[0]> {
    #length: 0x219eb0764ac9 <AccessorInfo> (const accessor descriptor)
 }
 - elements= 0x1fbbad30fd19 <FixedArray[3]> {
           0: 1
           1: 2
           2: 3
 }
[…]
```

- `--trace-elements-transitions` dumps elements transitions taking place to the console

**Sample Output**

```
elements transition [PACKED_SMI_ELEMENTS -> PACKED_DOUBLE_ELEMENTS]
  in ~+34 at x.js:2 for 0x1df87228c911 <JSArray[3]>
  from 0x1df87228c889 <FixedArray[3]> to 0x1df87228c941 <FixedDoubleArray[22]>
```

#### Resources

- ["Elements kinds" in V8](https://v8project.blogspot.com/2017/09/elements-kinds-in-v8.html)


## Considerations when Improving Performance

Three groups of optimizations are algorithmic improvements, workarounds JavaScript limitations
and workarounds for v8 related issues.

As we have shown v8 related issues have decreased immensly and should be reported to the v8
team if found, however in some cases workarounds are needed.

However before applying any optimizations first profile your app and understand the underlying
problem, then apply changes and prove by measuring that they change things for the better.

#### Profilers

- different performance problems call for different approaches to profile and visualize the
  the cause
- learn to use different profilers including _low level_ profilers like `perf`

#### Tweaking hot Code

- before applying micro optimizations to your code reason about its abstract complexity
- evaluate how your code would be used on average and in the worst case and make sure your
  algorithm handles both cases in a performant manner
- prefer monomorphism in very hot code paths if possible, as polymorphic functions cannot be
  optimized to the extent that monomorphic ones can
- measure that strategies like _caching_ and _memoization_ actually result in performance
  improvements before applying them as in some cases the cache lookup maybe more expensive than
  performing the computation
- understand limitations and costs of v8, a garbage collected system, in order to choose
  appropriate data types to improve performance, i.e. prefer a `Uint8Array` over a `String`
  when it makes sense

### Resources

- [Maybe you don't need Rust and WASM to speed up your JS - 2018](https://mrale.ph/blog/2018/02/03/maybe-you-dont-need-rust-to-speed-up-your-js.html)