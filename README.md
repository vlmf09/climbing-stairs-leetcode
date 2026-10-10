# climbing stairs: LeetCode 70 explained from naive recursion to O(1) space, plus the follow-up questions interviewers actually ask

## What the question is really asking

You are standing at the bottom of a staircase with `n` steps. Each move, you take either one step or two. Count how many different sequences of moves get you to the top. That is LeetCode 70 verbatim, constraints included: `1 <= n <= 45`.

Most people can write down the answer for `n = 1` and `n = 2` without thinking. The interesting part starts at `n = 3`, so it is worth writing the small cases out before touching a recurrence:

| n | Ways | Enumerated |
| --- | --- | --- |
| 1 | 1 | 1 |
| 2 | 2 | 1+1, 2 |
| 3 | 3 | 1+1+1, 1+2, 2+1 |
| 4 | 5 | 1+1+1+1, 1+1+2, 1+2+1, 2+1+1, 2+2 |
| 5 | 8 | — |
| 6 | 13 | — |

Sequence: 1, 2, 3, 5, 8, 13, 21. Every term is the sum of the two before it.

One thing in that table is worth pausing on, because it is where a lot of wrong first attempts come from: `1+2` and `2+1` are counted separately. You are counting **sequences** of moves, not sets of moves. If you treat the problem as "how many 1s and 2s add up to n" you get a much smaller number, and it will be wrong.

Two details from the constraints that are easy to skip and expensive to skip:

- `n <= 45` is small enough that the intended complexity is linear. Saying "O(n) is fine here because n tops out at 45" out loud is a cheap way to show you read the constraints instead of pattern-matching to DP on reflex.
- The answer for `n = 45` is 1,836,311,903, which fits inside a signed 32-bit int (the ceiling is 2,147,483,647). No overflow handling, no big integers, no `long` in Java. If you bring that up unprompted, you avoid the "did they just get lucky?" question.

## The insight, in one line

To land on step `n`, your last move was either a single step from `n-1` or a double step from `n-2`. Those are the only two options, and they can't both describe the same sequence, so the counts simply add:

> ways(n) = ways(n-1) + ways(n-2), with ways(1) = 1 and ways(2) = 2

That is the whole algorithm. Everything below is implementation detail and complexity hygiene.

The reason this is not "just Fibonacci trivia" is that the same reasoning gets reused constantly in 1D dynamic programming: define the state as "number of ways to reach position i", find that the state depends only on the previous one or two states, then decide how much history you actually need to keep. Interviewers care much more about whether you can produce that reasoning under pressure than whether you can recite the sequence.

## Four implementations, and what each one is for

| Approach | Time | Space | What it's for |
| --- | --- | --- | --- |
| Naive recursion | O(2ⁿ) | O(n) call stack | The baseline you name before optimizing |
| Memoization (top-down) | O(n) | O(n) | When the interviewer wants to see DP reasoning |
| Tabulation (bottom-up) | O(n) | O(n) | No recursion, clear evaluation order |
| Two variables | O(n) | O(1) | The answer they're waiting for |
| Matrix exponentiation | O(log n) | O(1) | Correct, and almost always overkill |

**Naive recursion.** Clear, and exponential.

python
def climbStairs(n: int) -> int:
    if n <= 2:
        return n
    return climbStairs(n - 1) + climbStairs(n - 2)


The recursion tree recomputes the same subproblems over and over. `climbStairs(5)` calls `climbStairs(3)` through two different paths, and it gets worse fast. For `n = 45` this is billions of calls, which is why it times out rather than merely being inelegant.

**Memoization.** Same shape, cached results.

python
from functools import cache

@cache
def climbStairs(n: int) -> int:
    if n <= 2:
        return n
    return climbStairs(n - 1) + climbStairs(n - 2)


Each state is computed once, so time drops to O(n). The cache costs O(n), and the recursion stack costs O(n) more.

**Tabulation.** Fill an array from the bottom up. This is the version most people find easiest to explain out loud, because the evaluation order is explicit.

python
def climbStairs(n: int) -> int:
    dp = [0] * (n + 1)
    dp[0] = 1
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]
    return dp[n]


**Two variables.** The recurrence only ever looks two steps back, so the array is unnecessary.

python
def climbStairs(n: int) -> int:
    if n <= 2:
        return n
    two_back, one_back = 1, 2
    for _ in range(3, n + 1):
        two_back, one_back = one_back, two_back + one_back
    return one_back


AlgoMonster's published walkthrough for this problem uses a slightly different framing: it keeps `a = 0` and `b = 1` and runs exactly `n` iterations with `a, b = b, a + b`, returning `b`. Same O(n) time and O(1) space, and it removes the `n <= 2` special case, which is its main advantage. The version above is easier to read because `one_back` and `two_back` mean what their names say at every point in the loop.

If you want the exotic option: the recurrence is a linear recurrence, so you can compute `Mⁿ` for the 2×2 matrix `[[1,1],[1,0]]` by binary exponentiation and read the answer off the top-left entry, giving O(log n). For `n <= 45` it buys nothing, but it is a legitimate answer if someone asks how you'd handle a version of the problem where `n` is a billion.

## The two places people actually lose points

**Base cases, and the `dp[0]` argument.** Ask five people what `dp[0]` should be and you'll get three answers. `dp[0] = 1` reads as "there is one way to climb zero steps: do nothing", which feels like a trick until you notice it makes the recurrence start cleanly. `dp[1] = 1, dp[2] = 2` avoids the question entirely for most implementations. Both are correct as long as the recurrence and the bases are internally consistent — the failure mode isn't picking the wrong one, it's mixing conventions halfway through and getting an off-by-one for `n = 1` or `n = 2`.

This is not a hypothetical worry. Interviewers use the `dp[0]` question deliberately: it's a low-cost way to check whether you understand what your state definition means or whether you copied a template that happened to work. Be able to say, in one sentence, what `dp[i]` represents and why the base case follows from it.

**Update order in the two-variable version.** Writing

python
two_back = one_back
one_back = two_back + one_back


silently reuses the already-overwritten value and produces the wrong sequence. Python's tuple assignment `two_back, one_back = one_back, two_back + one_back` evaluates the right-hand side first, which is why the idiomatic form is also the correct form. If you're writing Java or C++ instead, use a temporary and trace `n = 4` by hand before you declare it done.

## What comes after "it works"

Solving Climbing Stairs is roughly a five-minute exercise. The value in it is that it's a staging ground for harder versions, and this is where most of the interview signal lives.

**Min Cost Climbing Stairs (LeetCode 746).** Each step has a cost, you can start at step 0 or step 1, and you want the cheapest route past the last step. Now the state gets a `min` instead of a `+`: `M[i] = cost[i] + min(M[i+1], M[i+2])`, and the answer is `min(M[0], M[1])` because you choose your starting step. The common bug is charging the step you're arriving at instead of the step you're leaving.

**Up to `m` steps at a time.** Instead of 1 or 2, allow any jump up to `m`. The recurrence becomes `dp[i] = sum(dp[i-j])` for `j = 1..m`, which is the counting analogue of an unbounded knapsack. Setting `m = 2` collapses it back to the original problem.

**No two consecutive double steps.** Now the one-dimensional state is insufficient: `dp[i-1]` mixes sequences based on how they arrived, so you can't tell whether adding a 2-step jump is legal. You have to widen the state to `[i, j]`, where `j` records the size of the last jump, giving `dp[i][1] = dp[i-1][2]` and `dp[i][2] = dp[i-2][1] + dp[i-2][2]`. This one is genuinely instructive because it shows what "the state doesn't carry enough information" looks like in practice.

**1, 2, or 3 steps.** Same structure, three-way sum, Tribonacci numbers instead of Fibonacci.

Working through these is what turns a memorized answer into a pattern you can apply to a problem you've never seen. That's also the argument for studying this problem inside a structured curriculum rather than isolating it on its own.

## Where the pattern sits in AlgoMonster

AlgoMonster is a coding-interview prep platform built by ex-Google engineers, and it teaches algorithms as patterns rather than as a problem list — 48 explicit patterns, each with a template, a set of mapped problems, and a decision flowchart for the "which pattern does this problem need?" question. Climbing Stairs is a textbook case of the 1D linear DP / counting-ways pattern, and the platform publishes a walkthrough of it, including a step-by-step trace for `n = 4` where the two sliding variables are printed after each iteration. That trace is part of the freely readable sample content, which is the honest way to check whether their text-and-diagram teaching style matches how you learn:

👉 [Read the free Climbing Stairs walkthrough and see the n = 4 trace](https://algo.monster?fpr=li69)

The paid side is broader than one problem. Current official plan details list 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus unlimited AI assistant usage, unlimited Speedrun (timed pattern drills), monthly-updated questions collected from top tech companies, and a member discount on their live bootcamps. Two bonus courses are bundled in: a CS Foundation course and a Build an LLM course. Courses developed with external partners, such as Quant Interview Prep, are sold separately.

What it doesn't have matters just as much, and reviewers are consistent about it: there are no video walkthroughs anywhere in the product, system design coverage is introductory and not sufficient for senior-level loops, there are no live mock interviews, and the problem bank is a few hundred problems rather than LeetCode's few thousand. AlgoMonster is a pattern-teaching layer, not a replacement for volume practice. Most people who use it pair it with LeetCode for reps.

## Full plan comparison

| Plan | What you get | Price | Billing | Buy |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems; enough to evaluate the teaching style | $0 | — | [Start with the free content](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company-collected questions updated monthly; 500+ lessons / 1,000+ illustrations / 3,000+ editorials; member discount on live bootcamps; 12 months of access | $99/year, shown as $8.25/month billed annually (about $0.28/day) | Billed annually, cancel or upgrade anytime | [Check the current Yearly price](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan plus lifetime updates; one payment, no renewals | $189 | Billed once | [Check the current Lifetime price](https://algo.monster?fpr=li69) |

Three notes before you decide, because pricing on this platform moves around and the review sites haven't kept up:

1. **There is no monthly plan.** Annual or lifetime, nothing in between. If you have three weeks before an onsite, the yearly plan is the only option and it's an expensive three weeks.
2. **The figures above come from the official subscribe page.** Coupon pages describe them as a standing 50% discount off list prices of $198/year and $378 one-time, and some advertise an extra ~10% stackable code on top. Codes rotate and expire, so treat any code you find as unverified until the checkout total actually changes — the number on the checkout page is the only one that counts.
3. **Other reviews quote different numbers** — $89/year, $119 to $299/year, $459 lifetime — because promotional pricing changes and older articles never got updated. If a figure in a comparison post doesn't match the subscribe page, trust the subscribe page.

## Who the lifetime plan is actually for

Do the arithmetic on your own plans instead of on the marketing copy. Lifetime at $189 pays for itself against yearly at $99 after about two years of use, so the question is whether you'll still want this content in 2027 and 2028. If you interview once now and then stay put for four years, the yearly plan wins and it isn't close. If you expect to move again — a promotion loop, a return to the market after a break, teammates asking you to run loops — the lifetime plan is the cheaper side of the bet.

The case against buying anything is also worth stating plainly. If you're two weeks out from an interview, the curriculum takes four to eight weeks to work through, so there isn't enough runway for retention. If you're still shaky on basic data structures, the material assumes coding fluency and will feel like it's skipping steps. If you learn from video, this is text and diagrams only, and no amount of good writing fixes that mismatch.

The case for it is narrower than the marketing suggests and more durable than it sounds: the decision flowcharts and pattern templates. Grinding 300 problems without naming the patterns leaves you with 300 answers and no method. Learning 48 patterns first, then using LeetCode for volume, is the more efficient order — and a problem like Climbing Stairs is exactly where that order starts paying off.

## A 45-minute drill for this one problem

If you want to actually own Climbing Stairs rather than recognize it, this sequence gets you there:

1. Write the recurrence and the base cases on paper, with no code, and justify both in one sentence each.
2. Hand-trace `n = 5` all the way through, including the two-variable version. This catches update-order bugs before they appear in an editor.
3. Re-implement the naive, memoized, tabulated and O(1)-space versions from memory in one sitting. If the O(1) version takes more than two minutes, redo step 2.
4. Solve Min Cost Climbing Stairs (746) and note exactly which line changed. It's one operator.
5. Sketch the "up to m steps" generalization. Don't code it — just write the recurrence.
6. Then try the constrained version where consecutive double steps are banned. Expect to get it wrong, and notice that the failure is about state definition, not about DP.

Step 6 is the one that transfers to other problems. The gap between "I can solve climbing stairs" and "I can recognize a counting problem whose state needs a second dimension" is where interviews get decided, and it's a gap you can close in an afternoon if you deliberately walk into it.

👉 [Work through the pattern-based version of this problem on AlgoMonster](https://algo.monster?fpr=li69)
