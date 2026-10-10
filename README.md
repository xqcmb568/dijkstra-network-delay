# network delay time: why BFS gives the wrong answer, and the Dijkstra template that works on any weighted graph

Search "network delay time" and you get a wall of pasted solutions. Almost none of them explain the part people actually get stuck on: the answer is a *maximum* of shortest distances, the graph is weighted, and the -1 case is not a formality you can skip. This is LeetCode 743, and it shows up in interviews often enough that it's worth understanding properly rather than pattern-matching from an editorial.

Here's the whole thing, from the problem statement to the four bugs that keep submissions red.

## What the problem is actually asking

You have `n` nodes labeled `1` to `n`, and a list of directed edges `times[i] = (u, v, w)`: a signal takes `w` time to travel from `u` to `v`. A signal starts at node `k`. Return how long until every node has received it, or `-1` if some node can't be reached.

The confusion is almost always here: why is the answer the maximum? Because the signal doesn't travel one path. It fans out along every possible route simultaneously, so node 4 has its signal the moment the fastest route to node 4 delivers it. Time when the last node hears the signal = the largest of the fastest arrival times. Two ideas stacked: shortest path per node, then the worst of those.

Constraints worth having in your head, because they shape which algorithm is safe:

| Constraint | Value | Why it matters |
| --- | --- | --- |
| Nodes | `1 <= k <= n <= 100` | Small enough that even O(V³) survives |
| Edges | `1 <= times.length <= 6000` | Sparse graph, adjacency list territory |
| Weights | `0 <= wi <= 100` | Non-negative, so Dijkstra is valid |
| Duplicate edges | All `(u, v)` pairs are unique | No need to keep the minimum of parallel edges |
| Self loops | `ui != vi` | No zero-cost cycles to worry about |

That weight constraint is the one that decides the algorithm. `wi` can be 0, which is fine for Dijkstra — it only breaks on *negative* weights, not zero ones.

## The three sample cases, and what each is testing

| Input | Output | What it checks |
| --- | --- | --- |
| `times = [[2,1,1],[2,3,1],[3,4,1]]`, `n = 4`, `k = 2` | `2` | Basic fan-out: 2→1 is 1, 2→3 is 1, 2→3→4 is 2. Max is 2 |
| `times = [[1,2,1]]`, `n = 2`, `k = 1` | `1` | One reachable node, everything connected |
| `times = [[1,2,1]]`, `n = 2`, `k = 2` | `-1` | Node 2 has no outgoing edges, so node 1 never hears anything |

The third case is where most wrong answers live. If you compute the maximum finite distance and return it without checking reachability first, you'll return something like `0` here instead of `-1`.

## Why a plain BFS or DFS breaks

BFS finds the shortest path in an *unweighted* graph, where every edge costs the same. Here edges cost 1, 4, or 100. If you BFS from node 1 and reach node 3 in one hop, you've recorded a distance of 1 — but that hop might cost 4, while a two-hop route through node 2 costs 3.

An important detail: edge count and edge weight aren't correlated. A path with more edges can be cheaper.

DFS has the opposite problem — it commits to whatever route it walked first and needs explicit backtracking to find anything better. You can make DFS work (that's essentially what Dijkstra does with a priority queue bolted on), but plain DFS returns wrong distances on weighted graphs.

So the shape of the problem is: single-source shortest path with non-negative weights. That's Dijkstra.

## Dijkstra with a min-heap, step by step

The algorithm is one sentence long: *always finalize the closest unfinalized node next.*

1. Build an adjacency list: `graph[u]` holds `(v, w)` pairs.
2. Set `dist[k] = 0`. Everything else is effectively infinity.
3. Push `(0, k)` onto a min-heap ordered by distance.
4. Pop the smallest entry. If its recorded distance is worse than what you already have for that node, discard it and continue — it's a stale entry from an earlier, suboptimal push.
5. For each neighbor, compute `current + weight`. If that beats the stored distance, update it and push the new pair.
6. Repeat until the heap is empty.
7. If the number of nodes with a finalized distance is less than `n`, return `-1`. Otherwise return the largest distance.

The min-heap is what makes this work: it guarantees that when you pop a node, nothing left in the heap can offer a cheaper route to it. Nodes pop in increasing order of arrival time, which is exactly the order a signal would actually spread through the network.

## The code

python
import heapq
from collections import defaultdict

def networkDelayTime(times, n, k):
    graph = defaultdict(list)
    for u, v, w in times:
        graph[u].append((v, w))

    dist = {k: 0}
    heap = [(0, k)]

    while heap:
        d, node = heapq.heappop(heap)
        if d > dist.get(node, float("inf")):
            continue                      # stale entry, skip it
        for nxt, w in graph[node]:
            nd = d + w
            if nd < dist.get(nxt, float("inf")):
                dist[nxt] = nd
                heapq.heappush(heap, (nd, nxt))

    if len(dist) < n:                     # some node never got the signal
        return -1
    return max(dist.values())


Roughly 20 lines. In Java, the same logic is a `PriorityQueue<int[]>` with a `Comparator.comparingInt(a -> a[0])` and an `int[] dist` of size `n + 1`; in C++, `priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>>`.

## Walking through the 4-node example

Take `times = [[2,1,1],[2,3,1],[3,4,1]]`, `n = 4`, `k = 2`.

- Heap starts as `[(0, 2)]`. Pop `(0, 2)`. Neighbors: node 1 at cost 1, node 3 at cost 1. `dist = {2:0, 1:1, 3:1}`.
- Pop `(1, 1)`. Node 1 has no outgoing edges. Nothing changes.
- Pop `(1, 3)`. Neighbor: node 4 at cost 1 + 1 = 2. `dist[4] = 2`.
- Pop `(2, 4)`. No outgoing edges.
- Four nodes recorded, so no `-1`. `max(0, 1, 1, 2) = 2`.

Node 4 never had a direct edge for the signal to take — it only exists as a two-hop destination, and the heap found it because `2→3→4` was the cheapest remaining frontier.

## Complexity, and when something slower is still fine

Dijkstra with a binary heap runs in **O(E log V)** time and **O(V + E)** space. With `E ≤ 6000` and `V ≤ 100`, that's trivial — the heap rarely holds more than a few hundred entries.

Here's the part worth saying out loud: with `n ≤ 100`, **Floyd-Warshall is also acceptable.** It runs in O(V³) = 10⁶ operations on the worst case, which passes comfortably. So does Bellman-Ford at O(V·E) = 600,000. If you blank on Dijkstra in an interview, say that out loud and implement Bellman-Ford: relax all edges `n - 1` times, then read the maximum. You'll lose style points, not correctness points.

What you can't do is say "O(E log V)" while running a linear scan over an array to find the minimum each iteration. That's the O(V²) version, which also passes here — but only mention it if you know that's what you wrote.

> The bound that actually matters for correctness is the non-negative weight constraint. `0 <= wi` means zero-cost edges are legal, and they don't break Dijkstra. Negative weights would, and then Bellman-Ford becomes the only correct choice among the two.

## Four bugs that show up in real submissions

**1. Returning the max before checking reachability.** The single most common failure. Compute the distances, count how many nodes were reached, and only then take the max. The `-1` answer is a separate code path, not a corner case.

**2. Mixing 1-based labels with 0-based arrays.** Nodes are labeled `1..n`, not `0..n-1`. If you allocate `dist = [inf] * n` and then write `dist[k] = 0` with `k = n`, you're out of bounds or silently wrong. Use arrays of size `n + 1`, or subtract 1 from every label consistently. Don't do both.

**3. Forgetting the stale-entry check.** Without `if d > dist[node]: continue`, you'll re-expand nodes that were already superseded. The answer stays correct, but the heap grows and the constant factor gets ugly. NeetCode's version handles the same issue with a `visited` set instead — also correct, slightly different tradeoff.

**4. Early-return implementations that miss nodes.** Some solutions return as soon as `len(visited) == n`, which is fine, but the "all nodes reached" counter has to be based on nodes actually popped, not nodes pushed into the heap. Pushing a node into the heap doesn't mean it's been reached.

## Follow-up questions interviewers actually ask

Once your solution runs, the extensions come fast:

- **"What if weights could be negative?"** Dijkstra breaks. The heap's core assumption — that a popped distance is final — is false when a later path could subtract cost. Bellman-Ford handles it in O(V·E).
- **"What if you needed all-pairs shortest paths?"** Floyd-Warshall, O(V³), and with `n ≤ 100` it's a legitimate answer rather than a theoretical one.
- **"What if the graph were undirected?"** Add each edge in both directions when building the adjacency list. Everything else is unchanged.
- **"What if node k has no outgoing edges?"** Then only node k itself has received the signal. Return `-1` unless `n == 1`, in which case the answer is 0.
- **"What about duplicate edges between the same pair?"** The constraints rule it out here. In the general case, keep whichever edge is cheaper when you build the list.

## Where this fits in the pattern map

The reason 743 feels hard for people who can write Dijkstra is not the implementation — it's noticing that the problem *is* Dijkstra. The statement talks about signals, networks, and reception times. Nothing says "shortest path."

That's a recognition problem, not a coding problem, and it's the one AlgoMonster is built around. Its decision flowchart routes the question "shortest-path problem? → is the graph weighted?" straight to Dijkstra's algorithm, and this problem is a textbook instance: weighted, directed, non-negative, single source.

Worth knowing what the platform actually is before you consider paying:

- **Text and diagrams, no video.** Every explanation is written, with interactive diagrams you step through. If you learn by watching someone code, this format will feel dry.
- **48 explicit patterns, ~231 lessons and problems.** Far fewer problems than LeetCode's 3,000+, and deliberately so. The bet is that pattern selection — not volume — is the bottleneck.
- **Multi-language templates** in Python, Java, JavaScript, C++, Go, and Haskell, so you can practice in whatever your interviews use.
- **Speedrun mode** for timed review, plus company-specific question banks.
- **Free tier with limited content.** Enough to test whether the teaching style fits before you commit.
- **No monthly billing.** It's annual or lifetime, nothing else.

Several third-party reviews mention a job guarantee tied to completing the curriculum. Check the current terms on the site rather than trusting a review, since conditions like that change.

👉 [Check the current AlgoMonster plans and pricing](https://algo.monster?fpr=li69)

## AlgoMonster pricing: the plans as they stand

| Plan | What you get | Price | Billing | Purchase |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and a subset of problems — enough to judge the format, not enough to prep | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; company question sets updated monthly; member discount on live bootcamps | $99/year (shown as $8.25/month billed annually; listed original price $198) | Annual, renews | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates, one payment | $189 (listed original price $378) | One-time | [Buy Pro Lifetime access](https://algo.monster?fpr=li69) |

Two notes on the numbers. AlgoMonster runs sales constantly, and different promotional entry points have shown different headline prices, so the figure you see at checkout may be lower than the list price above. There's also no monthly option — if you only want six weeks of access, the annual plan is what you'd buy.

Two bonus courses come bundled with either paid plan: a foundation course covering data structures and algorithms from scratch, and a "build an LLM" course going from tokenization through attention and training. A quant interview prep course developed with an external partner is sold separately and isn't included.

## Is it worth paying for, for this problem?

Be honest about what you're buying. You can learn Dijkstra from free material — NeetCode walks through 743, LeetCode has thousands of solutions, and this article covers the algorithm end to end. What you're paying for is the routing layer: knowing that "signal spread" means Dijkstra, and that "shortest path with non-negative weights" means a min-heap rather than BFS.

The pricing math is straightforward. At $189 lifetime, AlgoMonster costs less than two years of LeetCode Premium, and the annual plan at $99 is cheaper than most single-month bootcamp discounts. The curriculum is built to be finished in 4–8 weeks at 1–2 hours a day, which is a real constraint if your interview is already scheduled.

Where it doesn't fit: it's not a problem-volume tool. 231 problems won't build the reflex speed that comes from a few hundred repetitions on LeetCode. AlgoMonster teaches the *why* behind pattern selection; LeetCode provides the reps. Candidates who pass FAANG loops generally use both, in that order.

## Quick answers to the questions people search

**Is network delay time hard?** Medium on LeetCode. The code is short; the recognition step is the hard part.

**Can I solve it with BFS?** No — the graph is weighted. BFS gives shortest paths by edge count, and edge count isn't cost.

**Why does my solution return 0 instead of -1?** You're taking the max of reachable distances without first verifying that all `n` nodes were reached. Count reached nodes, then branch.

**Which algorithm does the editorial use?** Dijkstra with a priority queue, O(E log V) time and O(V + E) space.

**Do I need to handle zero-weight edges specially?** No. Dijkstra is valid for any non-negative weight, zero included. Only negative weights break it.
