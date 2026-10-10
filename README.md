# Word Break: How to Solve LeetCode 139 (and Word Break II) With One DP Pattern

Most people meet Word Break in the same way: they read the statement, think "this is just string matching," write a loop, and watch it fail on the third test case.

The problem itself is short. Given a string `s` and a dictionary `wordDict`, decide whether `s` can be split into a space-separated sequence of one or more dictionary words. The same word can be reused as many times as you like, and you can assume the dictionary has no duplicates.

That's it. No clever output format, no ordering requirement, just a boolean. And yet it's one of the most-visited DP problems on LeetCode for a reason: the obvious approach is wrong, and the correct approach looks almost trivially simple once you've seen it.

## What the problem actually asks

Three examples cover essentially everything you need to understand about the setup.

| Input | Output | Why |
| --- | --- | --- |
| `s = "leetcode"`, `wordDict = ["leet","code"]` | `true` | Splits as `"leet code"` |
| `s = "applepenapple"`, `wordDict = ["apple","pen"]` | `true` | Splits as `"apple pen apple"` — `"apple"` is reused |
| `s = "catsandog"`, `wordDict = ["cats","dog","sand","and","cat"]` | `false` | `"cat" + "sand" + "og"` leaves `"og"` stranded; `"cats" + "and"` leaves `"og"` as well |

The official constraints are worth memorizing because they tell you which complexity is acceptable: `1 <= s.length <= 300`, `1 <= wordDict.length <= 1000`, each word is at most 20 characters, everything is lowercase English letters, and the words are unique.

300 characters. That number is your hint. It's small enough that an O(n²) DP passes comfortably, and large enough that any exponential recursion without memoization will time out.

## Why greedy fails, and why that matters

The natural first attempt is to walk the string left to right and cut whenever you find a dictionary word. Something like: at each position, try the smallest match, jump past it, keep going. It works on `"leetcode"` and `"applepenapple"`, which is exactly why people submit it.

Then a case like `s = "aaaaaaa"` with `wordDict = ["aaaa","aaa"]` kills it. Grabbing the short match `"aaa"` first consumes six characters and leaves a single `"a"` that isn't in the dictionary, so the greedy walk returns `false`. The correct answer is `"aaaa" + "aaa"`. The right split exists, but a locally reasonable choice closed the door on it.

This is the real lesson of Word Break. Every cut you make is a commitment about the rest of the string, and you can't judge a cut in isolation. Which is, more or less, the definition of when you need dynamic programming instead of a pointer.

There's a second trap that catches people who reach for DP but frame it wrong. Once you think of dictionary words as items and the string as a container, the question becomes: is this the coin-change-style combination problem or the permutation problem? Here order matters — forming `"applepenapple"` from `["apple","pen"]` requires the sequence apple → pen → apple. Word order is part of the answer, so this is a permutation-style formulation, and the traversal order in your loops has to match that. Get it backwards and `dp[13]` stays `false` while the answer is obviously `true`.

## The DP definition that unlocks the problem

Define `dp[i]` as: can the prefix `s[0:i]` be segmented into dictionary words?

Base case: `dp[0] = true`. The empty string is trivially segmentable, and it's what makes the very first word possible — without it, nothing downstream ever becomes true.

Transition: `dp[i]` is true if there exists some `j < i` where `dp[j]` is true **and** `s[j:i]` is in the dictionary. In plain terms, you already know how to split the front of the string, and the remaining chunk is a word.

In Python:

python
class Solution:
    def wordBreak(self, s: str, wordDict: List[str]) -> bool:
        words = set(wordDict)
        n = len(s)
        dp = [True] + [False] * n

        for i in range(1, n + 1):
            dp[i] = any(dp[j] and s[j:i] in words for j in range(i))

        return dp[n]


Java looks the same, with a `boolean[]` of size `length + 1`, `dp[0] = true`, and an inner loop that can `break` as soon as it finds a valid cut:

java
for (int end = 1; end <= length; end++) {
    for (int start = 0; start < end; start++) {
        if (dp[start] && wordSet.contains(s.substring(start, end))) {
            dp[end] = true;
            break;
        }
    }
}


Two things make this click faster than re-reading the recurrence:

- **`dp[j]` is the "already solved" half.** You're never re-checking whether the front of the string can be split; you look it up in O(1).
- **The `break` is free optimization.** Once `dp[i]` is true, more work on that index is wasted.

That's the whole solution. Nine lines in Python, and it's the answer interviewers expect.

## Bottom-up or top-down: pick whichever you can write under pressure

Both formulations are accepted. They differ in how you think, not in what they compute.

| Approach | Shape | Time | Space | When it's easier |
| --- | --- | --- | --- | --- |
| Bottom-up DP | Fill `dp` from index 1 to n, boolean array | ~O(n²) substring checks plus lookup cost | O(n) for the array | You want a short, deterministic loop with no recursion depth worries |
| Top-down memo + hash set | Recursive `dfs(i)` = "can `s[i:]` be segmented?", cache by index | Same asymptotically; often faster in practice because memoization skips unreachable indices | O(n) memo + recursion stack | You're more comfortable writing a recurrence than an iterative table |

A clean top-down version, using a set for O(1) lookups:

python
def dfs(i):
    if i == len(s):
        return True
    if i in memo:
        return memo[i]
    for j in range(i + 1, len(s) + 1):
        if s[i:j] in words and dfs(j):
            memo[i] = True
            return True
    memo[i] = False
    return False


The recursion bottoms out at `i == len(s)`, which is the top-down equivalent of `dp[0]`.

On complexity: the DP table itself costs O(n) space, and the nested loops give roughly O(n²) iterations. Each iteration does a substring slice and a hash lookup, and slicing isn't free — that's where the extra factor comes from, which is why some writeups quote O(n³) for naive `substr`-heavy implementations. Practically, with n ≤ 300, none of these variants is close to a problem.

## Two optimizations that actually show up in interviews

**Only look back as far as the longest word.** No dictionary word is longer than 20 characters, so `s[j:i]` can never be longer than the max word length in `wordDict`. Instead of `j` running from 0 to `i-1`, let it run from `i - maxLen` to `i-1`. That changes the inner loop from O(n) to O(t), where `t` is the maximum word length.

python
max_len = max(map(len, wordDict))
for i in range(1, n + 1):
    for j in range(max(i - max_len, 0), i):
        if dp[j] and s[j:i] in words:
            dp[i] = True
            break


**Convert the dictionary to a set once.** Doing a linear scan over a 1,000-word list inside a nested loop turns a comfortable solution into a slow one. One `set()` call fixes it, and mentioning this out loud in an interview is worth a few points on its own.

## Word Break II: same recurrence, completely different output

LeetCode 140 asks for all possible sentence splits instead of a yes/no answer. The input string is capped much shorter there, and that's deliberate: the number of valid segmentations can grow exponentially, so the output size itself is the bottleneck.

The structure is a memoized DFS that returns lists instead of booleans:

python
def dfs(i):
    if i == len(s):
        return [""]
    if i in memo:
        return memo[i]
    res = []
    for j in range(i + 1, len(s) + 1):
        w = s[i:j]
        if w in words:
            for tail in dfs(j):
                res.append(w if not tail else w + " " + tail)
    memo[i] = res
    return res


The memo is doing something subtly different from 139 here. In 139, caching a `false` prevents repeated dead-end exploration. In 140, caching a list of sentences at index `i` prevents recomputing the same suffix splits over and over — without it, `"aaaaa...a"` style inputs blow up fast.

If you can write 139 but freeze on 140, the gap is almost always this: you're trying to keep the boolean DP shape and bolt output generation onto it. Switch to "return all results from position i" and the recursion writes itself.

## The trie variant, and when it's worth reaching for

Instead of checking every substring against a hash set, build a trie from `wordDict` and walk it forward from each position, recursing whenever you hit an end-of-word marker.

This is worth understanding for two reasons. First, it's the standard setup for Aho-Corasick-style multi-pattern questions, which is where Word Break's cousins live. Second, in an interview it signals that you can compose two structures — trie plus memoization — rather than applying one canned template.

For LeetCode 139 specifically, the trie version is not faster. Hash lookups on short words are already cheap, and the trie adds nodes to build. Reach for it when the dictionary has heavy shared prefixes and you want to avoid slicing the same substrings repeatedly.

## Edge cases and mistakes that cost points

- **Forgetting `dp[0] = true`.** Without it, no transition ever fires and everything returns `false`.
- **Assuming a word can only be used once.** Reuse is explicitly allowed, and `"applepenapple"` depends on it.
- **Overwriting instead of OR-ing.** `dp[i]` should be set to true by any valid cut; don't let a later failing `j` reset it.
- **Assuming a word dictionary entry longer than the remaining string.** Slice bounds handle this in Python and Java silently, but be explicit about it in C++ and Go.
- **Trying to return the actual split in 139.** It's a boolean question. If you need the words back, you're solving 140 or you need to reconstruct via parent pointers.
- **Assuming `wordDict` is sorted or unique.** Uniqueness is guaranteed, ordering isn't — sorting buys you nothing here.

If you want to pressure-test your implementation, the follow-ups worth doing after 139 are Palindrome Partitioning (same "partition the string" shape, different validity test), Concatenated Words, and Letter Combinations of a Phone Number for the pure backtracking version of the same muscle.

## Practicing the pattern instead of memorizing this one problem

Here's the honest problem with Word Break: the solution is short enough that you can memorize it in one sitting and still fail a variation six weeks later. The thing you're actually building is judgment — recognizing "every cut is a commitment about the rest of the string, therefore DP over prefixes."

That's the argument behind AlgoMonster, which organizes its curriculum around reusable patterns rather than a chronological problem list. Two entry points are useful for this specific problem:

- A free editorial for LeetCode 139 with the DP walkthrough and Python/Java implementations: 👉 [Read AlgoMonster's Word Break editorial](https://algo.monster/liteproblems/139?fpr=li69)
- The paid track, which adds the structured pattern lessons, decision flowcharts, and timed Speedrun practice — 👉 [See AlgoMonster's plans and current pricing](https://algo.monster?fpr=li69)

The paid side bundles unlimited Speedrun sessions, unlimited AI assistant usage, 500+ lessons across six courses, 1,000+ illustrations, and 3,000+ editorials, plus company-tagged questions refreshed monthly. There are also two bonus courses included with a subscription — a fundamentals course for beginners and a "build an LLM from scratch" course — and lifelong access to anything added later.

Third-party write-ups consistently put the curriculum at roughly 48 patterns and a few hundred curated problems, which is a deliberate contrast with LeetCode's problem count in the thousands. A couple of reviews also mention a job-offer guarantee; that's a third-party claim, so check the current terms on the site yourself rather than taking it from a blog.

## AlgoMonster plans and pricing

AlgoMonster doesn't sell a monthly plan. Free, yearly, or lifetime are the only three options on the subscribe page, and the prices below are what the page currently displays.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge the lesson format, not enough to prep seriously | $0 | No card needed | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited Speedrun and AI assistant; 500+ lessons, 1,000+ illustrations, 3,000+ editorials; company questions updated monthly; member discount on live bootcamps | $5.84/mo, billed annually at $69.99/year | Annual, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates — all future content included | $89 | One-time payment | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

One caveat you should know before you budget: AlgoMonster's pricing moves a lot. The site runs frequent sales, and third-party price trackers have quoted materially higher figures at other points in time, including annual rates around $99 and lifetime rates around $189. The number at checkout is the only one that's binding, so don't treat any blog table — including this one — as fixed.

## Which one to pick

The lifetime-to-yearly gap is the whole decision. At $89 one-time versus $69.99 for twelve months, the lifetime option costs about $19 more than a single year of access. If there's any chance you'll come back to this material before a second job search, the math is not close.

Which raises the real question: will you? If you're doing a focused six-week push for one interview and you're confident you'll never want the reference material again, the annual plan is fine and you keep $19. If you're early in your career, expect to interview again in two or three years, or just want the pattern library sitting there when you need it, lifetime is the obvious call at this spread. Someone at a senior level who values a written reference over video courses gets more out of the lifetime tier than the annual one, since the recurring value is the lookup, not the first read-through.

What AlgoMonster won't do is replace everything else. The curriculum is text-and-diagram based with no video walkthroughs, the problem bank is curated rather than enormous, and the system design coverage is introductory. If your gaps are mock interviews or deep system design, budget for something else on top.

For Word Break specifically, the free 139 editorial is genuinely enough to learn the DP formulation. The paid tier makes sense when you want the same treatment applied across the pattern family — Palindrome Partitioning, trie-plus-memo, and the other string DP problems that show up in the same interview loop.

If you'd rather start with the pattern than the problem, 👉 [take a look at what's actually included before picking a plan](https://algo.monster?fpr=li69).
