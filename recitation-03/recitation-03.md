# CMPS 2200  Recitation 03

**Name (Team Member 1):** Tri Huynh
**Name (Team Member 2):** Kiet Huynh



## Analyzing a recursive, parallel algorithm


You were recently hired by Netflix to work on their movie recommendation
algorithm. A key part of the algorithm works by comparing two users'
movie ratings to determine how similar the users are. For example, to
find users similar to Mary, we start by having Mary rank all her movies.
Then, for another user Joe, we look at Joe's rankings and count how
often his pairwise rankings disagree with Mary's:

|      | Beetlejuice | Batman | Jackie Brown | Mr. Mom | Multiplicity |
| ---- | ----------- | ------ | ------------ | ------- | ------------ |
| Mary | 1           | 2      | 3            | 4       | 5            |
| Joe  | 1           | **3**  | **4**        | **2**   | 5            |

Here, Joe (somehow) liked *Mr. Mom* more than *Batman* and *Jackie
Brown*, so the number of disagreements is 2:
(3 <->  2, 4 <-> 2). More formally, a
disagreement occurs for indices (i,j) when (j > i) and
(value[j] < value[i]).

When you arrived at Netflix, you were shocked (shocked!) to see that
they were using this O(n^2) algorithm to solve the problem:



``` python
def num_disagreements_slow(ranks):
    """
    Params:
      ranks...list of ints for a user's move rankings (e.g., Joe in the example above)
    Returns:
      number of pairwise disagreements
    """
    count = 0
    for i, vi in enumerate(ranks):
        for j, vj in enumerate(ranks[i:]):
            if vj < vi:
                count += 1
    return count
```

``` python 
>>> num_disagreements_slow([1,3,4,2,5])
2
```

Armed with your CMPS 2200 knowledge, you quickly threw together this
recursive algorithm that you claim is both more efficient and easier to
run on the giant parallel processing cluster Netflix has.

``` python
def num_disagreements_fast(ranks):
    # base cases
    if len(ranks) <= 1:
        return (0, ranks)
    elif len(ranks) == 2:
        if ranks[1] < ranks[0]:
            return (1, [ranks[1], ranks[0]])  # found a disagreement
        else:
            return (0, ranks)
    # recursion
    else:
        left_disagreements, left_ranks = num_disagreements_fast(ranks[:len(ranks)//2])
        right_disagreements, right_ranks = num_disagreements_fast(ranks[len(ranks)//2:])
        
        combined_disagreements, combined_ranks = combine(left_ranks, right_ranks)

        return (left_disagreements + right_disagreements + combined_disagreements,
                combined_ranks)

def combine(left_ranks, right_ranks):
    i = j = 0
    result = []
    n_disagreements = 0
    while i < len(left_ranks) and j < len(right_ranks):
        if right_ranks[j] < left_ranks[i]: 
            n_disagreements += len(left_ranks[i:])   # found some disagreements
            result.append(right_ranks[j])
            j += 1
        else:
            result.append(left_ranks[i])
            i += 1
    
    result.extend(left_ranks[i:])
    result.extend(right_ranks[j:])
    print('combine: input=(%s, %s) returns=(%s, %s)' % 
          (left_ranks, right_ranks, n_disagreements, result))
    return n_disagreements, result

```

```python
>>> num_disagreements_fast([1,3,4,2,5])
combine: input=([4], [2, 5]) returns=(1, [2, 4, 5])
combine: input=([1, 3], [2, 4, 5]) returns=(1, [1, 2, 3, 4, 5])
(2, [1, 2, 3, 4, 5])
```

As so often happens, your boss demands theoretical proof that this will
be faster than their existing algorithm. To do so, complete the
following:

a) Describe, in your own words, what the `combine` method is doing and
what it returns.

`combine(left_ranks, right_ranks)` assumes both inputs are already sorted and then it performs a standard merge like in a merge sort. While merging, it also counts “cross” disagreements, which are pairs where an element from the right half is smaller than an element still remaining in the left half. When `right_ranks[j] < left_ranks[i]`, that right value is smaller than every item in `left_ranks[i:]`, so we add `len(left_ranks[i:])` to the count in one step.
It returns two things:

1. the number of cross disagreements between the two halves, and

2. the fully merged, sorted list made from both halves.

b) Write the work recurrence formula for `num_disagreements_fast`. Please explain how do you have this.

Let n be the length of ranks. Each call splits the array into two halves of size about n/2, solves both recursively, then calls `combine`, which scans each element once. The split and tuple bookkeeping are constant time. So:

*  Base: `W(1) = Θ(1), W(2) = Θ(1)`

*  Recurrence for n > 2: `W(n) = W(⌊n/2⌋) + W(⌈n/2⌉) + Θ(n)`
For analysis we use the balanced form: `W(n) = 2W(n/2) + c n`.

c) Solve this recurrence using any method you like. Please explain how do you have this.

Use the Master Theorem with a = 2, b = 2, and f(n) = c n. We have `n^{log_b a} = n^{log_2 2} = n`. Since f(n) = Θ(n), this is the regular case where `W(n) = Θ(n log n)`.
Intuition: the merge costs add up to n at each level, and there are log n levels.


d) Assuming that your recursive calls to `num_disagreements_fast` are done in parallel, write the span recurrence for your algorithm. Please explain how do you have this.

Span measures the length of the critical path. The two recursive calls on the halves can run at the same time, so their spans do not add. We take the maximum of the two, which is S(n/2), then we must run combine after both finish. combine is linear. So:

* Base: `S(1) = Θ(1)`, `S(2) = Θ(1)`

* Recurrence: `S(n) = S(n/2) + c n`.

e) Solve this recurrence using any method you like. Please explain how do you have this.

Unfold:

`S(n) = c n + c(n/2) + c(n/4) + ... + c(1)`
This is a geometric series bounded by `2 c n`. Therefore `S(n) = Θ(n)`.

f) If `ranks` is a list of size n, Netflix says it will give you lg(n) processors to run your algorithm in parallel. What is the upper bound on the runtime of this parallel implementation? (Hint: assume a Greedy Scheduler). Please explain how do you have this.

Let `T1` be total work and `T∞` be span. From above: `T1 = Θ(n log n)`, `T∞ = Θ(n)`. Netflix gives `P = lg n` processors. Under a Greedy Scheduler,
`T_P ≤ T1 / P + T∞ = Θ( (n log n) / log n ) + Θ(n) = Θ(n) + Θ(n) = Θ(n)`.
So the parallel runtime with `lg n` processors is `O(n)`. This matches the idea that `T∞` is already linear, and dividing the work term by `log n` also yields a linear term.