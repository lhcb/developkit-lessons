# New Counters

{% objectives "Learning Objectives" %}
- "Learn about upgrading to the new counters in Gaudi
{% endobjectives %}


## Old counters

The old counters looked like this:

```cpp
++counter("# valid banks"); // Add one
counter("skipped Banks") += tBanks.size(); // Add a number
++counter("nInAcceptance"); // binomial here !
```

This approach as a plethora of issues. First, counters are looked up by string. If one does not exist with that string, a new one is created.
There are special name patterns that indicate different sorts of counters, such as "acc" for binomial. This is all protected by a mutex, which locks on every counter access.
And there are five doubles that are tracked for all counters, such as the mean, even if all you need is a simple increment counter.

All together, this meant that 33% of the time in `createUTLiteCluster` was spent on counters alone!

{% discussion "Half way there" %}

This removes the name lookup, but still is not ideal, either for timing or design:

```cpp
class ... {
    mutable StatEntity m_validBanks{this, "# valid banks"};
};
```

{% enddiscussion %}

## New counters

To replace this system, a new counter object was built in Gaudi. Usage looks like this:

```cpp
class ... {
    mutable Counter<> m_validBanks{this, "# valid banks"};
};
```

This is allowed to be mutable, since it locks internally (just like an atomic or mutex member of a class can be mutable). The string here is only used for
print out purposes. And there is a family of counters, depending on what you want:

| Name | Usage details |
|------|-----------------|
| `Counter` | A pure counter, counts number of entries (`++` only).  |
| `AveragingCounter` |  Keeps count and sum, can compute mean. (`+=` also)  |
| `SigmaCounter` | Also keeps sum of squares, can compute variance, standard deviation.   |
| `StatCounter` | Also keeps min/max values.  |
| `BinomialCounter` | Dedicated to binomial law, compute efficiency. |

There are also two template parameters, with the following defaults:

```cpp
Counter<double, atomicity::atomic>
```

The first template parameter is the type (float is usually fine), and the second is the atomicity,
with `atomic` and `none` (non-atomic operations, for local counters). If you use `atomicity::none`,
the member should not be marked `mutable`!

## Temporary counters

This helped improve the speed, partially by using atomics instead of mutex locks, and partially by reducing the number of operations when
not needed. But the biggest improvement came from buffered local counters. If you create a buffer counter from a counter, you can update it
locally, and then when the object goes out of scope, the counter it was created from updates. You can now do this:

```cpp
AveragingCounter<float> counter{this, "main counter"};
{ // Entering a local scope
    auto buf = counter.buffer();
    for (int i = 0; i < 10000; i++)
        buf += i; // No atomics here, pure local!
} // single atomic update of main counter
```

This allows you to loop over tracks, or some other per-event structure, and still only update your counters once per event (which is much, much faster). This removed
the rest of the measurable time spent in counters in `createUTLiteCluster`, giving a 33% speed up!



{% discussion "Design for experts" %}

* Counters are only accumulators + printing
* Many basic accumulator exists
    - count, sum, square, min, max, true, false
* You can easily extend the set, based on `GenericAccumulator` class
    - Templated by type, atomicity, transform (identify, square), `ValueHandler` (sum, maximum, ...)
* They are then merged using `AccumulatorSet`

```cpp
struct AveragingAccumulator :
    AccumulatorSet<Arithmetic, Atomicity, CountAccumulator, SumAccumulator>
```

{% enddiscussion %}
