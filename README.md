# DADA3: A bug-fixing team

**Deployed app:** https://bug-fixing-team-vaev.onrender.com/

## Write-up

### Setup

Both systems used `gemini-3.5-flash-lite` with Daytona sandboxes. I ran each system once for the scored comparison; I ran each system once for the scored comparison. I started the notebook’s second-run cell but stopped it partway because of the daily Gemini quota, so it produced no results and is not part of the comparison.
 
Outside the scored runs, I also ran the team's pieces individually while building them in notebooks 01 and 02. I refer to two of those runs below because they show how much the results vary between runs.

### Results

| System | Resolved | Triage | Bugs fixed | False claims | Calls | Tokens | Time |
|---|---|---|---|---|---|---|---|
| Team | 9/10 | 9/10 | 7/7 | 0 | 110 | 1,916,210 | 532 s |
| Baseline | 10/10 | 10/10 | 7/7 | 0 | 44 | 691,765 | 206 s |

### Did the team do better?

No. The baseline resolved 10 of 10 issues and the team resolved 9 of 10, and neither made a false claim. Both fixed all seven bugs.
 
The team's only miss was triage on issue #3, which it labeled as an enhancement instead of not a bug. Because the team correctly left #3 unchanged either way, this cost the team a point on triage and on issues resolved but did not change any code.
 
The 10-point difference is not larger than the variation between runs of the same system. When I ran the team's triage on its own in notebook 02, it labeled #3 correctly as not_a_bug; in the full team run, it labeled it an enhancement. With one run of each system, I cannot say that either system is better at triage, only that they performed the same on fixing bugs.
 
The runs also varied in fixing. In an earlier test run of the fixer and verifier pair in notebook 01, the verifier accepted a fix for #2 that was wrong for negative amounts: a false claim that the hidden tests would have caught. In the scored run, the verifier caught the same mistake and sent the fix back. One run of each system is not enough to show a reliable difference in false claims.

### The handoff

The only handback in the scored team run was on issue #2, `split_amount`. The fixer's first fix gave the extra cents to the first shares but used floor division, which rounds toward negative infinity, so refunds were split with the extra cent on the wrong share.
 
The verifier sent it back with this reason: "The docstring example split_amount(-1000, 3) == [-334, -333, -333] fails in fixed/ because -1000 % 3 is 2 (since integer division rounds towards negative infinity in Python, -1000 // 3 = -334, and -1000 - 3 * -334 = 2)."
 
The reason was right by the docstring, which lists `split_amount(-1000, 3) == [-334, -333, -333]` as an example. Its description of the wrong output was a guess ("[-335, -335, -334] (or adjust first two shares)"), because the verifier is not shown the fix and explained the failure from the docstring and Python's rules rather than from the code.
 
The fixer's second attempt used truncating division for negative amounts. The verifier accepted it, and the hidden tests passed for #2. I also checked the second fix by hand on cases the docstring does not list: `split_amount(-1, 3)` returned `[-1, 0, 0]` and `split_amount(-2, 3)` returned `[-1, -1, 0]`, both correct.

### What did it cost?

The team used 110 model calls and about 1.9 million tokens; the baseline used 44 calls and about 0.69 million tokens. The team cost about 2.5 times as many calls and 2.8 times as many tokens, and took more than twice as long (532 s against 206 s).
 
In this run, the extra cost bought nothing measurable. Both systems fixed all seven bugs with no false claims, and the team's one handback fixed a mistake that the baseline did not make in the first place, because the baseline can run code while it writes a fix and the team's fixer cannot. On these ten issues, the team was not worth the cost. Its design might pay off on harder issues, where a single agent is more likely to fool itself, but these runs do not show that.
