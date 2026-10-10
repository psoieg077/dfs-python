# dfs python: Build One Traversal Template You Can Reuse on Trees, Graphs, Backtracking, and Cycle Detection

Most people typing "dfs python" into a search box want one of three things. Either they need the actual function — an adjacency list walked recursively in about a dozen lines. Or they already wrote it and it's blowing the stack, or returning nodes in a different order than the recursive version, and they need to know why. Or they've hit a problem that hides the DFS behind unfamiliar clothes: flood fill, cycle detection, topological sort, permutations.

All three are reasonable. Depth-first search is a short algorithm with an unusually long list of variations, and almost every bug in it comes from one of four decisions you make before you write a single loop. Get those right and DFS stops being a topic you memorize and becomes a tool you reach for.

## The version that handles most graphs

If you take one implementation away from this, take this one:

python
def dfs(graph, node, visited):
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)

graph = {
    "A": ["B", "C"],
    "B": ["D"],
    "C": ["D"],
    "D": [],
}

visited = set()
dfs(graph, "A", visited)
print(visited)  # {'A', 'B', 'D', 'C'}


Three things matter here. The adjacency structure is a `dict` of lists, which is the standard Python stand-in for an adjacency list and gives O(1) average lookup per node. `visited` is a `set`, not a list — `if neighbor not in some_list` is a linear scan, and inside a loop that runs once per edge, that turns a linear algorithm into a quadratic one. And the `visited.add(node)` sits at the top of the function, not at the bottom, which is what prevents a cycle from sending you back into a node you're currently exploring.

One structural note: the recursion above assumes the graph is connected and you're starting at a node that reaches everything. For a general graph, wrap it:

python
def dfs_all(graph):
    visited = set()
    for node in graph:
        if node not in visited:
            dfs(graph, node, visited)
    return visited


That outer loop is the part people forget, and it's the difference between "traverses the component containing `start`" and "traverses the graph."

## Recursive DFS has a hard Python problem

CPython's default recursion limit is 1000 frames. A path-shaped graph with a few thousand nodes will raise `RecursionError` long before you run out of memory, and the fix isn't always `sys.setrecursionlimit(100000)` — pushing the limit high enough can crash the interpreter with a real stack overflow, which is a segmentation fault rather than a catchable exception.

This is a well-known enough issue in the Python world that mature libraries avoid recursion entirely. David Eppstein's `DFS.py` in the PADS collection opens with the line that the search must be iterative "in order to avoid Python's low recursion limit," and NetworkX's depth-first module is built on the same iterative pattern.

The iterative translation:

python
def dfs_iterative(graph, start):
    visited = set()
    stack = [start]

    while stack:
        node = stack.pop()
        if node in visited:
            continue
        visited.add(node)
        for neighbor in reversed(graph[node]):
            if neighbor not in visited:
                stack.append(neighbor)

    return visited


Use a plain list as the stack. `list.append` and `list.pop()` on the end are both O(1) amortized, and a list beats `collections.deque` for stack behavior in CPython because of how it caches.

The `reversed()` call is not decoration. Without it, you push neighbors left to right and pop them right to left, so the iterative version visits siblings in the opposite order from the recursive one. Anything that depends on traversal order — a "print the path in this exact shape" test case, for example — will fail in a way that looks like a logic bug when it's really an ordering bug.

## Four decisions that determine whether your DFS is correct

Before typing, answer these. Most wrong answers trace back to one of them.

| Decision | The choice | What changes |
| --- | --- | --- |
| When to mark visited | On push, or on pop | Marking on pop lets the same node sit in the stack multiple times and can duplicate work |
| When to record the node | Before the loop (preorder) or after (postorder) | Postorder is what topological sort and "subtree result" problems need |
| How to track the path | Pass a fresh list down, or mutate one and pop | Mutation is what makes backtracking space-efficient; passing copies is easier to reason about |
| Directed or undirected | Whether the adjacency list has both directions | Undirected edges written once produce a graph half the size you expect |

The push-versus-pop detail is subtle enough to be worth spelling out. If you push a neighbor and only mark it inside the loop after popping, the same node can be pushed by two different parents and processed twice. On a graph with 10,000 edges that's not a correctness disaster, but on a problem with a time limit it is.

Backtracking is where the path-tracking choice becomes the whole algorithm. The shape is always the same — choose, recurse, un-choose:

python
def subsets(nums):
    result = []

    def dfs(index, path):
        if index == len(nums):
            result.append(path.copy())
            return
        # skip nums[index]
        dfs(index + 1, path)
        # take nums[index]
        path.append(nums[index])
        dfs(index + 1, path)
        path.pop()          # undo the choice

    dfs(0, [])
    return result


The `path.pop()` is not optional bookkeeping. Without it, every branch after the first one in a level inherits the previous branch's choices, and you get a set of results that looks almost right — which is the most annoying kind of wrong.

`path.copy()` on the result list matters too. Append `path` itself and every entry in `result` points at the same list, which will be empty by the time the recursion unwinds.

## Where DFS actually shows up in questions

DFS is rarely asked as "implement depth-first search." It's asked as a problem where DFS is the natural tool. The recognizable shapes:

**Tree traversal.** Rename the adjacency list to `left`/`right` and the same recursion applies. The template stays recognizable across variations, including the "return a value up the tree" form:

python
def find(root, target):
    if root is None:
        return None
    if root.val == target:
        return root
    left = find(root.left, target)
    if left is not None:
        return left
    return find(root.right, target)


That early `return` after the left subtree is where people lose points — recurse into both children unconditionally and you've turned an early-exit search into a full traversal.

**Connected components and flood fill.** Same code, grid instead of dict. The neighbors become `(r-1, c)`, `(r+1, c)`, `(r, c-1)`, `(r, c+1)`, and "visited" is usually handled by overwriting the cell with the new color or a sentinel. Bounds checks are the entire bug surface here.

**Backtracking and combinatorial search.** Subsets, permutations, combination sum, word break, decoding a digit string. All of them are DFS over an implicit tree where the "neighbors" are the next choices available at that depth. This is a category where the recursion template genuinely transfers between problems — the difference between a permutation problem and a combination problem is usually one `used` array, not a different algorithm.

**Cycle detection in a directed graph.** You need three states, not two: unvisited, in-progress, finished. Hit an in-progress node and you've found a back edge, which means a cycle.

python
def has_cycle(graph):
    state = {node: 0 for node in graph}   # 0 white, 1 gray, 2 black

    def dfs(node):
        state[node] = 1
        for neighbor in graph.get(node, []):
            if state[neighbor] == 1:
                return True
            if state[neighbor] == 0 and dfs(neighbor):
                return True
        state[node] = 2
        return False

    return any(state[n] == 0 and dfs(n) for n in graph)


A plain `visited` set cannot distinguish "this node is done" from "this node is on my current path," and that distinction is the entire algorithm.

**Topological sort.** Run DFS, append each node to a list after its children finish, then reverse the list at the end. Course scheduling, build order, dependency resolution. The reverse-postorder property is why DFS does this for free.

## What to say about complexity

Time is O(V + E) with an adjacency list, since each vertex is entered once and each edge is examined once from its source endpoint (twice for undirected graphs). Space is O(V) for the visited set plus the call stack, which in the worst case holds a path of length V.

If the input arrives as an adjacency matrix instead, the loop over neighbors is O(V) per node and the whole thing becomes O(V²) — the correct answer, but only if you actually know which representation you were given.

Worth saying out loud in an interview: recursion in Python costs noticeably more per call than an explicit stack does, so for large inputs the iterative version isn't just safer, it's faster. It also sidesteps the interpreter's recursion ceiling entirely, which is the answer interviewers are fishing for when they ask what happens on a graph with a million nodes.

## A short bug checklist

- Mark nodes visited when you push them, not when you pop them.
- Use a `set` for `visited`, not a list.
- Loop over all nodes, not just the start node, when the graph may be disconnected.
- `reversed()` your neighbor list if the iterative version needs to match recursive order.
- `copy()` the path before appending it to results.
- Test on a two-node cycle before testing on the actual example.
- For directed graphs, use three states if the question mentions cycles.

## Where a structured template library saves time

Writing DFS once is easy. Writing the correct variant under time pressure — knowing whether this particular question wants postorder, whether the `visited` check goes before or after the recursion, whether cycles are even possible — is where preparation actually gets tested. That's a recognition problem more than a coding problem.

AlgoMonster is built around that gap. It's a pattern-based interview prep platform written by engineers who came out of Google, and instead of handing you a few thousand problems to sort through, it organizes the material into templates — DFS on trees, DFS on graphs, backtracking with and without pruning, topological sort — with Python among the supported languages. Each pattern comes with a written explanation, a reusable code skeleton, and then a handful of problems that stress the same skeleton in different directions.

The current subscription page lists 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus a decision flowchart set that walks you from "is this a tree or a graph?" to the pattern most likely to fit. There's also a Speedrun mode for timed review, an AI assistant for hints, and company-tagged question sets. Two caveats worth knowing before you pay: everything is text and diagrams, so there are no video walkthroughs, and the system design material is introductory rather than something you'd rely on for a senior-level design round.

If a template-driven approach to DFS and its neighbors sounds like your learning style:

👉 [See AlgoMonster's current plans and pricing](https://algo.monster?fpr=li69)

## AlgoMonster plans at a glance

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems — enough to judge the teaching style | $0 | — | [Start with the free plan](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons, and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; member discount on live bootcamps | $99/year at the standard promo price, and roughly $5.84/month ($69.99 billed annually) when the deeper sale is running | Billed annually, 12 months of access | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, and access to any courses added later | $189 billed once, and $89 during the deeper promotional windows | One-time payment | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

The pricing deserves one honest note. AlgoMonster runs its paid plans at a discount almost all the time, but the size of that discount rotates — the subscribe page has shown Pro Yearly at both $99/year and $69.99/year, and Pro Lifetime at both $189 and $89, at different points. There's no monthly plan, and no trial beyond the free tier. Third-party coupon sites advertise stackable codes, but the discount you can actually count on is the one already applied on the checkout page, so check that number rather than trusting a blog post — this one included.

## Which plan makes sense

AlgoMonster's own framing is that the curriculum is meant to be worked through in a matter of weeks rather than months, at an hour or two a day. Take that at face value and the math is simple: if you're preparing for one interview loop in the next couple of months, the yearly plan covers it with room to spare, and paying nearly double for permanent access is money you may never use again. If you expect to interview again in two or three years — a normal pattern for engineers who move between companies — the lifetime plan stops looking like an upsell.

The free tier is genuinely useful for one decision: whether you learn well from text with step-by-step diagrams. If you need someone talking through a solution on video, that's not a preference you can train away in a week, and no discount fixes it.

As for DFS itself, the algorithm really is short. What takes practice is picking the right variation fast and getting the `visited` bookkeeping right on the first try. Whether you get that from a template library or from writing the same twenty lines forty times is a personal call — both work, and neither one is free.

## Quick answers

**Does DFS in Python need to be iterative?** Not always, but recursion breaks somewhere past a thousand frames deep, so anything that could be a long chain — a path graph, a linked list treated as a graph, a deep tree — should use an explicit stack.

**Recursive or iterative for interviews?** Recursive is shorter and easier to explain. If the interviewer mentions deep inputs or Python's recursion limit, switch to the stack version and mention `reversed()` for ordering.

**Why does my iterative DFS visit nodes in a different order?** Because a stack reverses whatever order you push in. Reverse the neighbor list before pushing.

**Where does DFS beat BFS?** Anywhere you need the path structure rather than the shortest path: cycle detection, topological sort, backtracking, connected components. BFS wins on unweighted shortest paths and level-by-level traversal.
