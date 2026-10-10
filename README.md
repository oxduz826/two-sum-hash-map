# two sum javascript: the O(n) hash map solution, three approaches compared, and how to stop freezing when the interviewer changes the question

Two Sum is LeetCode problem number one, which is exactly why it's a trap. Nearly everyone can get a working answer in five minutes. Far fewer can explain why calling `nums.indexOf()` inside the loop quietly turns their "one-pass O(n)" solution back into O(n²), or what their code does when `nums = [3, 3]` and `target = 6`.

The JavaScript version has its own set of landmines, and they're different from the Python version most tutorials are translated from. Let's go through the actual code, the specific mistakes that show up in JS submissions, and what the interviewer is really checking when they hand you this problem as a warm-up.

## What the problem is actually asking

You get an array of integers and a target. Return the indices of the two numbers that sum to the target. The problem guarantees exactly one valid answer, and you can return the indices in any order.


nums = [2, 7, 11, 15], target = 9  →  [0, 1]


The line people skim past is "you may not use the same element twice." That restriction is about indices, not values. If `nums = [3, 3]` and `target = 6`, the answer is `[0, 1]`, because index 0 and index 1 are two different elements that happen to hold the same number. A lot of broken solutions fail on exactly this case.

## The version everyone writes first

js
function twoSum(nums, target) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] === target) return [i, j];
    }
  }
  return [];
}


Two loops, O(n²) time, O(1) extra space. Write it if you're stuck, then say the complexity out loud before the interviewer asks. Starting with brute force and improving it is a normal interview move; presenting brute force as your final answer is where the coding screen usually ends.

One detail worth getting right: `j` starts at `i + 1`, not at 0. Starting at 0 forces you to add a `if (j === i) continue` guard, and it also means you check each pair twice.

## The one-pass hash map, which is the answer they want

js
function twoSum(nums, target) {
  const seen = new Map();

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];

    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }

    seen.set(nums[i], i);
  }

  return [];
}


The idea is to stop searching for a sum and start searching for a difference. At every index you ask: have I already walked past the number I need? If yes, you have both indices. If no, record the current number and move on.

Walking through `nums = [2, 7, 11, 15]`, `target = 9`:

| i | nums[i] | complement (9 − nums[i]) | seen before the check | result |
| --- | --- | --- | --- | --- |
| 0 | 2 | 7 | `{}` (empty) | no match, store `2 → 0` |
| 1 | 7 | 2 | `{2 → 0}` | match, return `[0, 1]` |

Two decisions in that code matter more than they look.

The map stores **value → index**, not index → value. You search by value and return by index, so the value has to be the key.

And the check happens **before** the insert. If you inserted first, then for a target where the number pairs with itself, like `[3, 3]` with `target = 6`, you'd store `3 → 0` and then immediately find it on the same index, returning `[0, 0]`, which is wrong. Checking first means index 0 gets inserted only after you've confirmed nothing matched it yet.

Time is O(n) because each element is visited once. Space is O(n) in the worst case, since you may store most of the array before finding the pair.

## The JavaScript-specific mistakes

**Using a truthy check instead of `has()`.**

js
// broken
if (seen.get(complement)) { ... }


If the matching element sits at index 0, `seen.get(complement)` returns `0`, which is falsy, and the branch gets skipped. The solution either returns `[]` or finds a later, wrong pair. Use `seen.has(complement)` or compare against `undefined` explicitly.

**Reaching for `nums.indexOf()` inside the loop.**

js
// looks linear, is not
for (let i = 0; i < nums.length; i++) {
  const diff = target - nums[i];
  const diffIndex = nums.indexOf(diff);
  if (diffIndex !== -1 && diffIndex !== i) return [i, diffIndex];
}


`indexOf` is itself a linear scan, so you've nested a hidden loop and you're back at O(n²) with worse constants than the brute force version. It also needs the `diffIndex !== i` guard, otherwise `[3, 3]` with `target = 6` at `i = 0` looks up `3`, finds index 0, and eliminates it as a self-match. This is one of the most common wrong answers on Two Sum in JavaScript.

**Using a plain object and forgetting that keys become strings.**

js
const seen = {};
seen[1] = "a";
seen["1"] = "b"; // overwrites, because 1 and "1" are the same key


For a pure-integer array it usually behaves, but `Map` keeps key types distinct, preserves insertion order, and avoids collisions with inherited properties like `constructor` or `__proto__` when you forget `hasOwnProperty`. There's no performance reason to prefer the object here.

**Using a `Set` to store seen values.**

A `Set` holds values, not indices. To return indices you end up calling `indexOf` to recover them, which reintroduces the O(n²) problem. If you need index lookups, use `Map`.

## The three approaches side by side

| Approach | Time | Space | Handles `[3, 3]`? | When to use |
| --- | --- | --- | --- | --- |
| Nested loops | O(n²) | O(1) | Yes, with `j = i + 1` | Baseline answer, tiny inputs, or when you need to show your reasoning first |
| Two-pass hash map | O(n) | O(n) | Yes, with an index guard | Fine, but it iterates twice and needs the `index !== i` check |
| One-pass hash map | O(n) | O(n) | Yes, no guard needed | The answer to give |

Two pointers deserves a mention since it's the follow-up. It only works on a sorted array. Two Sum gives you an unsorted one, and you need original indices, so you'd have to build `[value, index]` pairs, sort them by value, then run the two-pointer scan. That's O(n log n) with more code and more places to make an off-by-one error. Correct, but not better here.

## Follow-up questions that actually get asked

Interviewers rarely stop at "got it, O(n)." The follow-ups are where the round is decided:

- **What if the array is sorted?** Two pointers, O(n) time and O(1) space. This is Two Sum II and it's worth having memorized.
- **What if there are multiple valid pairs?** Return all of them, and switch the early `return` to pushing into a results array.
- **What if there's no solution?** The `return []` at the bottom handles it, but you should decide whether the API should signal that differently.
- **What if the input is streamed and doesn't fit in memory?** You can't do the two-pass version. The one-pass map is the only one that still works, and you'd need to talk about memory limits.
- **What if the array has duplicates?** Already handled by the check-before-insert order, but say why.

The pattern underneath all of this, trading O(n) extra space to turn a lookup into an O(1) operation, is the same move behind Subarray Sum Equals K, Group Anagrams, Longest Consecutive Sequence, and a long list of others. Two Sum is a pattern named "hash map for lookups" wearing a beginner's costume.

## Why solving Two Sum isn't the same as passing the round

Here's the honest version: if you can write the hash map solution from memory and explain the complexity, you've cleared the bar for this one problem. That's a small bar. The problem that decides your interview is the one you've never seen, where you have to look at an unfamiliar prompt and work out which of the roughly forty recurring patterns applies within the first ten minutes.

That gap, recognizing the pattern rather than implementing it, is the thing AlgoMonster was built around. It's a DSA prep platform put together by ex-Google engineers, and the format is deliberately different from LeetCode: instead of a bank of thousands of unsorted problems, it teaches 48 named patterns, each with a written explanation, a decision flowchart for when to apply it, template code in several languages, and a set of practice problems mapped to that specific pattern.

For a Two Sum reader, the parts that matter most:

- **JavaScript is a first-class language.** Templates and solutions ship in JavaScript alongside Python, Java, C++, Go, and Haskell, so you're not translating Python idioms by eye.
- **The format is text plus interactive diagrams.** Roughly 700 illustrations across the current curriculum, no video. Reviewing a pattern before an interview means skimming and skimming is fast. If you learn by watching someone else code, this format will frustrate you.
- **Speedrun mode** compresses all 48 patterns into a single review pass, which is the thing you actually want the week of a scheduled interview.
- **An embedded AI assistant** explains concepts, gives hints when you're stuck, and debugs submitted code. It also tracks which patterns you keep missing.
- **Bonus courses** ship with a subscription at no extra cost, currently a CS foundation course and a "build an LLM from scratch" course.

Where it doesn't hold up, and you should know this before paying:

- The problem count is around 230. That's enough to internalize patterns and nowhere near enough for volume. Most people who do well in FAANG screens use a pattern course plus LeetCode for raw reps.
- System design coverage is introductory. If you're targeting L5 or above, budget separately for that round.
- There's no live interview simulation. Practicing at your own pace with no timer and no one asking follow-ups does not replicate someone watching you and asking what happens if the dataset doesn't fit in memory.

## AlgoMonster plans and current pricing

AlgoMonster runs on a small number of plans, and paid pricing moves with promotions, so check the page rather than trusting anything cached.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems, enough to test the teaching style | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Monthly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company question banks updated monthly, Solutions in multiple languages | $18.99/mo | Billed monthly, cancel anytime | [See the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | Everything in Pro, 12 months of full access, bonus Foundation and LLM courses, member discount on live bootcamps | $99/year ($8.25/mo billed annually) | Billed annually | [Check the yearly price](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates on all future content | $189 one-time | Billed once | [See lifetime access pricing](https://algo.monster/subscribe?fpr=li69) |

Two things worth flagging about that table. First, sales are frequent and they're not small: snapshots of AlgoMonster's own subscribe page have shown Pro Yearly at $69.99/year and Pro Lifetime at $89 during deeper promotions, so the numbers above are the resting state, not the floor. Second, third-party coupon pages currently advertise codes like SCHELLEY (an extra 10% stacked on an active sale) and FAANG20. Verify whatever code you plan to use at checkout before entering payment details; they expire without notice.

There's no team or enterprise plan. This is built for individuals.

If your interviews are two or three months out, the yearly plan is the sensible one. If you're someone who changes jobs every couple of years and re-reads the patterns each time, the lifetime plan stops being a gamble. 👉 [Browse the pattern library and current offers on AlgoMonster](https://algo.monster?fpr=li69) before you decide which one fits your timeline.

## Quick answers

**Is Two Sum worth doing in JavaScript specifically?** Yes, because the JS version exposes mistakes that the Python version hides. Falsy checks on index 0, string coercion in plain objects, and `indexOf` inside a loop all show up in JS submissions and rarely in Python ones.

**Should I memorize the solution?** Memorize the reasoning: look for the complement, check before you insert, store value to index. The ten lines of code follow from that. Memorizing the code without the reasoning is why people rewrite the same bug under pressure.

**Do I still need LeetCode if I buy a pattern course?** If your interview is more than a month out, yes. Patterns give you the framework; volume gives you speed. The two aren't substitutes.

**Does AlgoMonster cover JavaScript solutions?** Yes. JavaScript is one of the supported languages for templates and editorials, which is not something every prep platform can say.

Two Sum takes fifteen minutes to learn and about two minutes to break. The difference between those two states is almost never the algorithm. It's whether you checked `has()` instead of trusting a truthy value, and whether the pattern behind the problem is something you recognized or something you happened to remember.
