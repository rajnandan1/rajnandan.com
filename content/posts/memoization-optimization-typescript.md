+++
title = "Memoization: The Art of Teaching Your Code to Remember"
date = 2025-12-11
description = "How a simple caching technique transforms exponential algorithms into blazing-fast solutions, with practical TypeScript examples."
[taxonomies]
tags = ["typescript", "algorithms", "performance", "optimization"]
+++

## The Core Insight

Memoization is elegantly simple: cache a function's results by its inputs. When the same input appears again, return the cached value instead of recomputing. You're trading memory for speed—replacing repeated work with instant lookups.

## The Problem: Counting Paths Up Stairs

Imagine a staircase where you can climb 1, 3, or 5 steps at a time. How many distinct ways can you reach step $n$?

Let $f(n)$ be the count of ways. The recurrence relation is:

$$f(n) = f(n-1) + f(n-3) + f(n-5)$$

With base cases:

-   $f(0) = 1$ — one valid way: do nothing
-   $f(n) = 0$ when $n < 0$ — invalid path

## Naive Recursion: Clean but Catastrophically Slow

```typescript
function stepCount(n: number): number {
    if (n < 0) return 0;
    if (n === 0) return 1;
    return stepCount(n - 1) + stepCount(n - 3) + stepCount(n - 5);
}
```

This reads like the mathematical definition. Beautiful, right?

**The problem**: identical subproblems explode across branches. For `n = 10`, `stepCount(5)` gets called multiple times. The call tree grows exponentially, and we keep solving the same problems over and over.

Try `stepCount(35)` and watch your CPU fan spin up. For `n = 50`, you might as well go make coffee.

## Memoization: Add a Memory

Store results by input. On repeat calls, return the stored value instantly.

```typescript
function memoSteps(n: number, cache: Map<number, number> = new Map()): number {
    if (n < 0) return 0;
    if (n === 0) return 1;

    if (cache.has(n)) {
        return cache.get(n)!;
    }

    const result =
        memoSteps(n - 1, cache) +
        memoSteps(n - 3, cache) +
        memoSteps(n - 5, cache);

    cache.set(n, result);
    return result;
}
```

Now `n = 35` returns instantly. Even `n = 100` is trivial because each distinct value of `n` is computed exactly once.

## A Reusable Memoization Utility

Rather than manually threading caches, build a generic memoizer:

```typescript
function memoize<T extends (...args: any[]) => any>(fn: T): T {
    const cache = new Map<string, ReturnType<T>>();

    return ((...args: Parameters<T>): ReturnType<T> => {
        const key = JSON.stringify(args);

        if (cache.has(key)) {
            return cache.get(key)!;
        }

        const result = fn(...args);
        cache.set(key, result);
        return result;
    }) as T;
}

// Usage
const stepCount = memoize((n: number, steps: number[] = [1, 3, 5]): number => {
    if (n < 0) return 0;
    if (n === 0) return 1;
    return steps.reduce((sum, step) => sum + stepCount(n - step, steps), 0);
});
```

## LRU Cache: Bounded Memory

For long-running applications, unbounded caches are dangerous. Implement a Least Recently Used cache:

```typescript
class LRUCache<K, V> {
    private cache = new Map<K, V>();

    constructor(private maxSize: number) {}

    get(key: K): V | undefined {
        if (!this.cache.has(key)) return undefined;

        // Move to end (most recently used)
        const value = this.cache.get(key)!;
        this.cache.delete(key);
        this.cache.set(key, value);
        return value;
    }

    set(key: K, value: V): void {
        if (this.cache.has(key)) {
            this.cache.delete(key);
        } else if (this.cache.size >= this.maxSize) {
            // Evict oldest (first) entry
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        this.cache.set(key, value);
    }

    has(key: K): boolean {
        return this.cache.has(key);
    }
}

function memoizeWithLRU<T extends (...args: any[]) => any>(
    fn: T,
    maxSize: number = 1000
): T {
    const cache = new LRUCache<string, ReturnType<T>>(maxSize);

    return ((...args: Parameters<T>): ReturnType<T> => {
        const key = JSON.stringify(args);

        if (cache.has(key)) {
            return cache.get(key)!;
        }

        const result = fn(...args);
        cache.set(key, result);
        return result;
    }) as T;
}
```

## Why It Works

Memoization transforms a branching recursion tree into a directed acyclic graph. Each unique input becomes a single node. Compute once, reuse forever.

Think of counting items in a box. You write "8" on a sticky note. When you add one more, you don't recount—you read "8" and add "1". Memoization is that sticky note.

## When to Reach for Memoization

**It shines when you have:**

1. **Pure functions** — output depends only on input, no side effects
2. **Overlapping subproblems** — the same inputs recur across different execution paths
3. **Manageable state space** — unique inputs fit comfortably in memory

**Common applications:**

-   **Dynamic programming**: Fibonacci, coin change, edit distance, longest common subsequence
-   **Graph algorithms**: path counting, shortest paths with repeated states
-   **Game engines**: transposition tables in chess cache board evaluations to skip re-analyzing identical positions
-   **API responses**: cache expensive computations or external calls

## The Bottom-Up Alternative

Sometimes you can flip the script—compute iteratively from base cases up:

```typescript
function stepCountDP(n: number, steps: number[] = [1, 3, 5]): number {
    const dp: number[] = new Array(n + 1).fill(0);
    dp[0] = 1;

    for (let i = 1; i <= n; i++) {
        for (const step of steps) {
            if (i - step >= 0) {
                dp[i] += dp[i - step];
            }
        }
    }

    return dp[n];
}
```

-   **Time**: $O(n \cdot |\text{steps}|)$
-   **Space**: $O(n)$
-   **No recursion**: avoids stack overflow for large $n$

## Choosing Your Approach

| Approach         | Best For                                                                         |
| ---------------- | -------------------------------------------------------------------------------- |
| **Memoization**  | Natural recursive problems, lazy evaluation, when not all subproblems are needed |
| **Bottom-up DP** | Clear state space, tight memory control, avoiding recursion limits               |
| **LRU caching**  | Long-running services, memory-constrained environments                           |

## Practical Gotchas

1. **Memory growth**: Unbounded caches can leak memory. Use LRU or clear caches periodically.

2. **Cache key design**: Keys must be immutable and capture all inputs that affect the result. Use `JSON.stringify` for simplicity, but beware of object key ordering.

3. **Break-even point**: For tiny inputs, cache overhead can exceed computation cost. Profile before optimizing.

4. **Cache invalidation**: If the underlying behavior changes (config updates, time-dependent logic), version your cache or clear it.

## The Takeaway

Memoization is a small change with outsized impact. Cache results by input, reuse them on repeat calls. For problems with overlapping subproblems, it transforms exponential blowups into linear-time solutions.

Design your keys carefully, bound your memory when needed, and you'll have code that's both elegant and fast.
