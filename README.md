# Rotting Oranges: How Multi-Source BFS Actually Works, Why the -1 Case Trips People Up, and the Bugs That Cause TLE

You open Rotting Oranges expecting a straightforward grid traversal. You write something with a queue, it passes the first example, and then it either returns the wrong number of minutes, or it sits there hitting the time limit because your loop condition never lets go.

LeetCode 994 is a Medium, and it gets solved by thousands of people every week, so the difficulty isn't the code. It's the mental model. Rotting Oranges is the cleanest introduction to multi-source BFS on the platform, and almost everything that goes wrong with it traces back to one of three things: forgetting that all the rotten oranges spread at the same time, mishandling the case where some orange can never be reached, or writing a loop condition that can never terminate.

Here's all of it, in the order it actually goes wrong.

## What the problem is asking for

You get an `m x n` grid. Each cell holds one of three values:

- `0` — empty cell, nothing lives here
- `1` — a fresh orange
- `2` — a rotten orange

Every minute, any fresh orange with a rotten orange directly above, below, left, or right of it turns rotten. Diagonals don't count. You return the minimum number of minutes until no fresh oranges remain, or `-1` if some fresh orange can never be reached.

The three sample cases are worth reading carefully, because each one is testing a different part of your implementation:

| Grid | Output | What it's testing |
| --- | --- | --- |
| `[[2,1,1],[1,1,0],[0,1,1]]` | `4` | Normal spread, including around an empty cell |
| `[[2,1,1],[0,1,1],[1,0,1]]` | `-1` | An orange the rot can never reach |
| `[[0,2]]` | `0` | No fresh oranges exist at minute zero |

That third case is the one that quietly breaks a lot of otherwise correct solutions. If you start `minutes` at `0`, run your BFS, and return `minutes` at the end, you'll return `0` for `[[0,2]]` by accident and feel good about it. Then a grid with a single rotten orange and no fresh ones in a different shape still works, and you assume the logic is fine. It usually is — but only because you got lucky with how you initialized the counter, not because you handled it. Handle it explicitly: count the fresh oranges up front, and if the count is zero, return `0` before the BFS starts.

## The one insight that makes the whole thing click

A single rotten orange spreading rot is just a BFS from one source. Multiple rotten oranges spreading at the same time is a BFS from multiple sources, and the trick is that **each BFS level is exactly one minute**.

That's it. Every cell you reach in the first level away from the starting queue rots at minute 1. Everything at distance two rots at minute 2. The shortest-path distance from the nearest rotten orange *is* the answer for that cell, and BFS is the algorithm that finds shortest-path distances level by level without being told to.

This is also why DFS doesn't work here, and it's worth saying plainly rather than hand-waving at it. DFS commits to one branch and goes deep. If you start from a single rotten orange and walk down one path, you'll count the travel time along that path and completely miss that three other oranges rotted in parallel on a different branch. You'd have to run DFS from every rotten orange and take the minimum per cell, which is more work for a worse answer. The problem is inherently about simultaneous waves, and BFS is the algorithm that models waves.

In the first sample grid there are six fresh oranges, but the answer is `4`, not `6`. At minute 2, two separate oranges rot in the same minute. That's the simultaneity you have to preserve in the code.

## A minute-by-minute run through the sample grid

Grid:


2 1 1
1 1 0
0 1 1


Initial state: queue holds `(0,0)`, `fresh = 6`, `minutes = 0`.

**Minute 1.** Pop `(0,0)`. Its neighbors are `(0,1)` fresh and `(1,0)` fresh. Both rot, both get pushed. Queue is now `[(0,1), (1,0)]`. `fresh = 4`. `minutes = 1`.

**Minute 2.** The level has two oranges, and both are processed before `minutes` moves. From `(0,1)`: `(0,2)` rots, `(1,1)` rots. From `(1,0)`: `(0,0)` is already rotten and `(2,0)` is empty, so nothing new. Queue is `[(0,2), (1,1)]`. `fresh = 2`. `minutes = 2`.

**Minute 3.** From `(0,2)`: `(1,2)` is empty. From `(1,1)`: `(2,1)` rots. Queue is `[(2,1)]`. `fresh = 1`. `minutes = 3`.

**Minute 4.** From `(2,1)`: `(2,2)` rots. `fresh = 0`, the loop ends, and the answer is `4`.

The detail that matters is at minute 2: you process *the entire current level* before incrementing the minute counter. If you increment `minutes` once per orange instead of once per level, everything downstream is wrong.

## The algorithm, step by step

1. Scan the whole grid once. Count fresh oranges, and push every rotten orange's coordinates into a queue. All rotten oranges are sources in the initial level.
2. If the fresh count is `0`, return `0` immediately.
3. Set `minutes = 0`.
4. While the queue is non-empty **and** fresh oranges still exist: record `levelSize = queue.size()`, then process exactly `levelSize` nodes. For each one, check its four neighbors. If a neighbor is fresh, mark it rotten, decrement the fresh count, and push it into the queue for the next minute.
5. After the level is fully processed, increment `minutes`.
6. Return `minutes` if the fresh count reached zero, otherwise `-1`.

Note the condition in step 4 carefully: the two checks are joined with **and**. This is where the time-limit failures come from, and we'll get to that.

## Python

python
from collections import deque

class Solution:
    def orangesRotting(self, grid):
        rows, cols = len(grid), len(grid[0])
        queue = deque()
        fresh = 0

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] == 2:
                    queue.append((r, c))
                elif grid[r][c] == 1:
                    fresh += 1

        if fresh == 0:
            return 0

        minutes = 0
        directions = ((1, 0), (-1, 0), (0, 1), (0, -1))

        while queue and fresh > 0:
            for _ in range(len(queue)):        # snapshot the level size
                r, c = queue.popleft()
                for dr, dc in directions:
                    nr, nc = r + dr, c + dc
                    if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                        grid[nr][nc] = 2
                        fresh -= 1
                        queue.append((nr, nc))
            minutes += 1

        return minutes if fresh == 0 else -1


The inner `for _ in range(len(queue))` is the level snapshot. Python evaluates `range(len(queue))` once, up front, so newly appended oranges land in the *next* level's loop rather than extending the current one. Appending during the loop is fine; iterating a deque directly while mutating it is not, which is a separate bug we'll cover below.

## Java

java
class Solution {
    public int orangesRotting(int[][] grid) {
        int rows = grid.length, cols = grid[0].length;
        Queue<int[]> queue = new LinkedList<>();
        int fresh = 0;

        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == 2) queue.offer(new int[]{r, c});
                else if (grid[r][c] == 1) fresh++;
            }
        }

        if (fresh == 0) return 0;

        int minutes = 0;
        int[][] dirs = {{1,0}, {-1,0}, {0,1}, {0,-1}};

        while (!queue.isEmpty() && fresh > 0) {
            int levelSize = queue.size();
            for (int i = 0; i < levelSize; i++) {
                int[] cur = queue.poll();
                for (int[] d : dirs) {
                    int nr = cur[0] + d[0], nc = cur[1] + d[1];
                    if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == 1) {
                        grid[nr][nc] = 2;
                        fresh--;
                        queue.offer(new int[]{nr, nc});
                    }
                }
            }
            minutes++;
        }

        return fresh == 0 ? minutes : -1;
    }
}


`int levelSize = queue.size()` must be captured *before* the inner loop. Writing `for (int i = 0; i < queue.size(); i++)` is the classic mistake here: `queue.size()` grows as you enqueue this minute's newly rotten oranges, so the loop keeps chasing its own tail and the level boundaries disappear entirely.

## JavaScript

javascript
var orangesRotting = function(grid) {
  const rows = grid.length, cols = grid[0].length;
  const queue = [];
  let head = 0, fresh = 0;

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === 2) queue.push([r, c]);
      else if (grid[r][c] === 1) fresh++;
    }
  }

  if (fresh === 0) return 0;

  const dirs = [[1,0], [-1,0], [0,1], [0,-1]];
  let minutes = 0;

  while (head < queue.length && fresh > 0) {
    const levelSize = queue.length - head;
    for (let i = 0; i < levelSize; i++) {
      const [r, c] = queue[head++];
      for (const [dr, dc] of dirs) {
        const nr = r + dr, nc = c + dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === 1) {
          grid[nr][nc] = 2;
          fresh--;
          queue.push([nr, nc]);
        }
      }
    }
    minutes++;
  }

  return fresh === 0 ? minutes : -1;
};


Using a `head` index instead of `shift()` matters in JS. `Array.prototype.shift()` is O(n), and calling it once per cell turns a linear solution into a quadratic one on any meaningful input.

## The bugs that actually break submissions

**The `||` that becomes an infinite loop.** This one is so common it shows up as its own Stack Overflow question. Writing `while (queue.length !== 0 || fresh > 0)` means the loop keeps running whenever *either* condition holds — including when the queue is empty but fresh oranges are still counted, which is exactly the unreachable-orange case. The loop spins forever doing nothing, and you get a TLE instead of the `-1` you were supposed to return. It has to be `&&`.

**Mutating a deque while iterating it.** In Python, `for cell in queue: ... queue.append(...)` inside that loop raises `RuntimeError: deque mutated during iteration`. Snapshot with `range(len(queue))`, as above.

**Forgetting that you need two pieces of state.** You need the level size *and* a fresh-orange count. Drop the count and you can't distinguish "everything is rotten" from "some orange is stranded," and you'll return a number for a grid whose correct answer is `-1`.

**Off-by-one on the counter.** Incrementing `minutes` inside the per-orange loop instead of after the per-level loop inflates the answer. The other direction — returning before the final level is credited — truncates it by one.

**Never handling `[[0,2]]`.** Covered above, but worth repeating: return `0` when the initial fresh count is `0`, before the BFS runs.

**Marking cells late.** Set `grid[nr][nc] = 2` at the moment you enqueue, not when you dequeue. If you only mark on dequeue, the same fresh orange can be pushed multiple times by different neighbors in the same minute.

## Complexity

Time is `O(m × n)`. Every cell is scanned once during initialization and can be enqueued at most once, since marking it rotten happens at enqueue time and the value `1` check then fails for every later neighbor. Space is also `O(m × n)` in the worst case, because a grid that starts almost entirely rotten can put nearly every cell into the queue. LeetCode's constraints cap the grid at `10 × 10`, so none of this is close to a real limit — which is exactly why TLE on this problem is almost always a loop-condition bug and not a performance problem.

## Edge cases to check before you submit

- All cells empty, no oranges at all → `0`
- Rotten oranges present, zero fresh → `0`
- Fresh oranges present, zero rotten → `-1`
- A fresh orange cut off by empty cells or the grid boundary → `-1`
- A grid that's a single row or single column, where "4-directional" leaves only two neighbors for most cells
- The last minute of spread: make sure it's counted, since that's the level where the fresh count hits zero

## The same pattern wearing a different costume

Multi-source BFS is a family, and Rotting Oranges is the friendliest member of it. Once the level-equals-one-unit-of-time idea is in your head, these become the same problem with different nouns:

- **Walls and Gates (286)** — every gate spreads distance outward simultaneously; the queue starts with all gates.
- **01 Matrix (542)** — the same computation, returning per-cell distances instead of a single maximizing value.
- **Word Ladder (127)** — level count *is* the transformation length.
- **Shortest Path in Binary Matrix (1091)** — single source, but the same level-by-level distance bookkeeping.

The interviewers who ask 994 are usually checking whether you reach for BFS unprompted. If you also mention that a multi-source BFS starting from all sources at once is equivalent to a super-source connected to each of them with a zero-weight edge, you've answered the follow-up before it's asked.

## Practicing multi-source BFS without grinding blind

Rotting Oranges is a good test of whether you recognize the pattern, and the awkward part of preparation is that recognition is the one skill raw problem volume doesn't build on its own. Solving 500 grids in random order teaches you to solve grids; it doesn't teach you to look at a problem statement and immediately know which of the four or five traversal families it belongs to.

[👉 See how AlgoMonster breaks the Rotting Oranges solution down pattern by pattern](https://algo.monster?fpr=li69)

AlgoMonster's approach is explicit pattern mapping. Instead of shipping a solution per problem and letting you infer, it teaches the pattern first, then attaches the problems that exercise it. The platform has a dedicated page for 994 with multi-language implementations alongside its broader graph BFS material, and it's built by ex-Google engineers with 48 coding patterns as the organizing spine. Two things worth knowing before you buy: the format is text and interactive diagrams with no video at all, and the problem bank is in the low hundreds, not thousands. Reviews from multiple independent write-ups land on the same conclusion — it's a fast way to build pattern recognition over roughly a 4-to-8 week window, and a poor substitute for LeetCode if what you actually need is volume or contest practice.

### Current plans

Pricing on the site moves with sales, so treat the checkout page as the source of truth.

| Plan | What you get | Price | Billing |
| --- | --- | --- | --- |
| Free | Limited access to a sample of patterns and problems | $0 | No charge |
| Pro Monthly | All patterns, lessons and problems, unlimited AI assistant and Speedrun, company question banks | $18.99/mo | Billed monthly |
| Pro Yearly | Everything in Pro Monthly, plus 12 months of access and member pricing on live bootcamps | $99/yr (about $8.25/mo) | Billed annually |
| Pro Lifetime | Everything in the annual plan, with lifetime updates and no renewal | $189 | Billed once |

Plan links:

- [👉 Start with the free tier and see if the teaching style fits](https://algo.monster?fpr=li69)
- [👉 Subscribe to Pro Monthly](https://algo.monster?fpr=li69)
- [👉 Subscribe to Pro Yearly](https://algo.monster?fpr=li69)
- [👉 Get Pro Lifetime access](https://algo.monster?fpr=li69)

One thing to be aware of while you shop around: several third-party review sites still quote a $459 lifetime price and claim there's no monthly option. That doesn't match what the official subscribe page currently shows, which lists a monthly plan at $18.99 and lifetime at $189. Promo codes circulate too — a partner code published by a former Meta engineer, last marked verified in September, is reported to stack an extra 10% on top of the site's sale pricing. Verify anything like that at checkout rather than trusting a blog post, including this one.

If your interview is six weeks out and you can already write correct code in one language, the annual plan is the sensible pick. Lifetime only makes sense if you expect to repeat this process in a few years, which plenty of engineers do.

## FAQ

**Why is the answer 4 and not 6 for the sample grid?**
Because rotting happens in parallel. Each BFS level is one minute, and multiple oranges can rot in the same level. Counting oranges instead of levels gives you too large a number.

**Can I solve this with DFS?**
You can, by running DFS from every rotten orange and taking the minimum distance per cell, then taking the maximum of those minimums. It's more code, more recursion, and no benefit. The simultaneous-spread structure is exactly what BFS models for free.

**What does `-1` actually mean?**
At least one fresh orange is unreachable — it has no path of 4-directionally adjacent cells connecting it to any rotten orange. Empty cells don't count as connections, and diagonals never do.

**Why does my solution time out instead of returning `-1`?**
Almost certainly a loop condition joining two checks with `||` instead of `&&`, so the loop keeps running after the queue empties. It's the single most common failure mode on this problem.

**Which language should I write it in for an interview?**
Whichever you can write bug-free under pressure. The algorithm is identical in all of them; the language-specific traps are the `shift()` cost in JavaScript and the mutated-deque error in Python.

Get the level boundary right, count your fresh oranges before you start, and return `-1` honestly when an orange is stranded. Those three things are the whole problem.
