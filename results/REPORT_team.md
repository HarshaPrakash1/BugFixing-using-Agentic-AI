# huskyutils: the open issues

Fixes for issues #1, #2, #4, #5, #6, #8, and #10 were verified and are ready to review, while no issues still need a person, and issues #3, #7, and #9 were closed without a code change.

| Issue | Triage | What happened |
|---|---|---|
| #1 Slugs have double hyphens and a hyphen at the end | bug | fix verified, 1 verifier round(s) |
| #2 Shared payment loses a cent | bug | fix verified, 2 verifier round(s) |
| #3 word_count says "don't stop" is 2 words | enhancement | closed, no code change |
| #4 Delivery date skips over the weekend wrong | bug | fix verified, 1 verifier round(s) |
| #5 Delivery windows overlap after merging | bug | fix verified, 1 verifier round(s) |
| #6 A 1 kg package is charged $9 shipping | bug | fix verified, 1 verifier round(s) |
| #7 Free shipping for orders over $50 | enhancement | closed, no code change |
| #8 Median on the sales dashboard is wrong for an even number of days | bug | fix verified, 1 verifier round(s) |
| #9 5 kg package should be in the heavy tier | not_a_bug | closed, no code change |
| #10 Product card description runs past the edge of the card | bug | fix verified, 1 verifier round(s) |

## #1: Slugs have double hyphens and a hyphen at the end

**Triage:** bug. The slugify function replaces every non-alphanumeric character with a hyphen individually rather than collapsing runs of them into a single hyphen and stripping leading or trailing hyphens, causing double hyphens and trailing hyphens as reported.

**Fix** (fix verified): Updated slugify to handle accent normalization, collapsing runs of non-alphanumeric characters into a single hyphen, and stripping leading/trailing hyphens as specified in the docstring.

```diff
--- a/huskyutils/text.py
+++ b/huskyutils/text.py
@@ -1,4 +1,6 @@
 """Text helpers for product pages."""
+import unicodedata
+import re
 
 
 def slugify(title):
@@ -13,10 +15,14 @@
         slugify("USB-C Dock (2nd gen)") == "usb-c-dock-2nd-gen"
         slugify("Crème Brûlée Mug") == "creme-brulee-mug"
     """
+    nfkd_form = unicodedata.normalize('NFKD', title.lower())
+    ascii_title = "".join([ch for ch in nfkd_form if not unicodedata.combining(ch)])
     out = []
-    for ch in title.lower():
+    for ch in ascii_title:
         out.append(ch if ch.isalnum() else "-")
-    return "".join(out)
+    slug = "".join(out)
+    slug = re.sub(r'-+', '-', slug)
+    return slug.strip('-')
 
 
 def word_count(text):
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both original and fixed directories. In original/, `test_issue_examples` failed with AssertionError (`'hello--world--t-shirt' != 'hello-world-t-shirt'`), reproducing the issue. In fixed/, all 31 tests passed successfully (including existing tests and new issue tests).

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.text import slugify, word_count


class TestSlugify(unittest.TestCase):
    def test_simple_title(self):
        self.assertEqual(slugify("Wireless Headphones"), "wireless-headphones")

    def test_lowercase(self):
        self.assertEqual(slugify("WEBCAM"), "webcam")

    def test_issue_examples(self):
        self.assertEqual(slugify("Hello, World! T-shirt"), "hello-world-t-shirt")
        self.assertEqual(slugify("Webcam (HD)"), "webcam-hd")
        self.assertEqual(slugify("USB-C Dock (2nd gen)"), "usb-c-dock-2nd-gen")
        self.assertEqual(slugify("Crème Brûlée Mug"), "creme-brulee-mug")


class TestWordCount(unittest.TestCase):
    def test_simple(self):
        self.assertEqual(word_count("Great keyboard, fast shipping"), 4)

    def test_extra_spaces(self):
        self.assertEqual(word_count("  works   great  "), 2)

    def test_empty(self):
        self.assertEqual(word_count(""), 0)
```

</details>

## #2: Shared payment loses a cent

**Triage:** bug. The function `split_amount` simply integer-divides `cents // people` and returns identical shares, failing to distribute remainder cents as required by the docstring.

**Fix** (fix verified): Fixed split_amount to use truncating division for negative amounts so that remainder matches docstring expectation.

```diff
--- a/huskyutils/money.py
+++ b/huskyutils/money.py
@@ -12,5 +12,12 @@
         split_amount(1000, 3) == [334, 333, 333]
         split_amount(-1000, 3) == [-334, -333, -333]
     """
-    share = cents // people
-    return [share] * people
+    share = int(cents / people)
+    remainder = abs(cents) % people
+    shares = [share] * people
+    for i in range(remainder):
+        if cents >= 0:
+            shares[i] += 1
+        else:
+            shares[i] -= 1
+    return shares
```

**Verifier:** Ran tests in original/ and fixed/ using `python -m unittest discover -s tests -v`. In original/, test_docstring_examples and test_issue_example failed with AssertionError ([333, 333, 333] != [334, 333, 333]). In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.money import split_amount


class TestSplitAmountIssue(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(split_amount(900, 3), [300, 300, 300])
        self.assertEqual(split_amount(1000, 3), [334, 333, 333])
        self.assertEqual(split_amount(-1000, 3), [-334, -333, -333])

    def test_issue_example(self):
        self.assertEqual(split_amount(1000, 3), [334, 333, 333])
```

</details>

## #3: word_count says "don't stop" is 2 words

**Triage:** enhancement. The docstring explicitly states that punctuation stays attached to the word and 'don't' is counted as one word. The code correctly implements this behavior, making the reporter's expectation an enhancement request rather than a bug.

## #4: Delivery date skips over the weekend wrong

**Triage:** bug. The function add_business_days simply adds the number of days to the start date and skips weekend/holiday days after, failing to count the correct number of business days as specified.

**Fix** (fix verified): Fixed add_business_days to correctly count `days` business days rather than just adding calendar days and skipping trailing weekends/holidays.

```diff
--- a/huskyutils/dates.py
+++ b/huskyutils/dates.py
@@ -13,9 +13,10 @@
         add_business_days(date(2026, 11, 6), 1) == date(2026, 11, 9)                             # Monday
         add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}) == date(2026, 11, 10)
     """
-    if days == 0:
-        return start
-    result = start + timedelta(days=days)
-    while result.weekday() >= 5 or result in holidays:
+    result = start
+    added = 0
+    while added < days:
         result += timedelta(days=1)
+        if result.weekday() < 5 and result not in holidays:
+            added += 1
     return result
```

**Verifier:** Ran unit tests on original and fixed repositories with the newly created tests/test_issue.py. In original/, test_thursday_plus_three_business_days failed with AssertionError: datetime.date(2026, 11, 9) != datetime.date(2026, 11, 10). In fixed/, all 27 tests passed successfully (exit code 0).

<details><summary>The verifier's test</summary>

```python
import unittest
from datetime import date

from huskyutils.dates import add_business_days


class TestIssue4(unittest.TestCase):
    def test_thursday_plus_three_business_days(self):
        # Issue #4 description:
        # An order placed on Thursday 2026-11-05 with 3 business days of handling
        # shows a delivery date of Monday 2026-11-09. Thursday plus 3 business days
        # is Friday, Monday, Tuesday, so it should be Tuesday 2026-11-10.
        self.assertEqual(add_business_days(date(2026, 11, 5), 3), date(2026, 11, 10))

    def test_docstring_examples(self):
        self.assertEqual(add_business_days(date(2026, 11, 6), 1), date(2026, 11, 9))
        self.assertEqual(add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}), date(2026, 11, 10))
```

</details>

## #5: Delivery windows overlap after merging

**Triage:** bug. The docstring states that intervals can be in any order, but the code processes them sequentially without sorting first, causing overlapping windows when given unordered input.

**Fix** (fix verified): Sort the input intervals by start time before merging so that unordered inputs are correctly handled and merged.

```diff
--- a/huskyutils/intervals.py
+++ b/huskyutils/intervals.py
@@ -13,8 +13,11 @@
         merge_intervals([[1, 3], [2, 6], [8, 10]]) == [[1, 6], [8, 10]]
         merge_intervals([[1, 4], [4, 5]]) == [[1, 5]]
     """
+    if not intervals:
+        return []
+    sorted_intervals = sorted(intervals, key=lambda x: x[0])
     merged = []
-    for start, end in intervals:
+    for start, end in sorted_intervals:
         if merged and start <= merged[-1][1]:
             merged[-1][1] = max(merged[-1][1], end)
         else:
```

**Verifier:** Ran tests in original/ and fixed/ using python -m unittest discover -s tests -v. In original/, test_issue_unsorted failed as expected (reproducing Issue #5). In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.intervals import merge_intervals


class TestIssueIntervals(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(merge_intervals([[1, 3], [2, 6], [8, 10]]), [[1, 6], [8, 10]])
        self.assertEqual(merge_intervals([[1, 4], [4, 5]]), [[1, 5]])

    def test_issue_unsorted(self):
        self.assertEqual(merge_intervals([[8, 10], [1, 3], [2, 6]]), [[1, 6], [8, 10]])
```

</details>

## #6: A 1 kg package is charged $9 shipping

**Triage:** bug. The docstring states that packages up to and including 1 kg cost $5.00, but the code checks `if weight_kg < 1:` returning $9.00 for exactly 1 kg instead of `if weight_kg <= 1:`.

**Fix** (fix verified): Changed `if weight_kg < 1:` to `if weight_kg <= 1:` to correctly include packages weighing exactly 1 kg in the $5.00 shipping tier as stated in the docstring.

```diff
--- a/huskyutils/shipping.py
+++ b/huskyutils/shipping.py
@@ -12,7 +12,7 @@
         shipping_cost(0.4) == 500
         shipping_cost(5.2) == 1100
     """
-    if weight_kg < 1:
+    if weight_kg <= 1:
         return 500
     if weight_kg <= 5:
         return 900
```

**Verifier:** Ran tests in original/ repository: test_shipping_cost_one_kg failed with AssertionError (900 != 500), reproducing the issue. Ran tests in fixed/ repository: all 27 tests passed successfully (including the new test verifying that a 1 kg package costs 500 cents).

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.shipping import shipping_cost


class TestShippingIssue(unittest.TestCase):
    def test_shipping_cost_docstring_examples(self):
        self.assertEqual(shipping_cost(0.4), 500)
        self.assertEqual(shipping_cost(5.2), 1100)

    def test_shipping_cost_one_kg(self):
        self.assertEqual(shipping_cost(1), 500)
```

</details>

## #7: Free shipping for orders over $50

**Triage:** enhancement. The shipping_cost function calculates shipping based on weight, while the issue requests a feature for free shipping based on order total. This is a new capability not promised by the docstring.

## #8: Median on the sales dashboard is wrong for an even number of days

**Triage:** bug. The median function for an even list length simply picks the upper middle element instead of averaging the two middle values, violating the docstring and mathematical definition.

**Fix** (fix verified): Fixed median function to correctly calculate the average of the two middle values for an even number of items, matching the docstring and mathematical definition.

```diff
--- a/huskyutils/stats.py
+++ b/huskyutils/stats.py
@@ -11,4 +11,8 @@
         median([4, 1, 3, 2]) == 2.5
     """
     ordered = sorted(values)
-    return ordered[len(ordered) // 2]
+    n = len(ordered)
+    if n % 2 == 1:
+        return ordered[n // 2]
+    else:
+        return (ordered[n // 2 - 1] + ordered[n // 2]) / 2
```

**Verifier:** Ran unit tests in original/ and fixed/ using `python -m unittest discover -s tests -v`. In original/, test_median_even and test_docstring_examples in test_issue.py failed as expected (reproducing Issue #8), while in fixed/, all tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.stats import median


class TestIssue8(unittest.TestCase):
    def test_median_even(self):
        self.assertEqual(median([10, 20, 30, 40]), 25)

    def test_docstring_examples(self):
        self.assertEqual(median([3, 1, 2]), 2)
        self.assertEqual(median([4, 1, 3, 2]), 2.5)
```

</details>

## #9: 5 kg package should be in the heavy tier

**Triage:** not_a_bug. The docstring says packages over 5 kg cost $9.00 plus $2.00 for each started kilogram over 5, and up to and including 5 kg costs $9.00. A 5.0 kg package costs $9.00 as specified.

## #10: Product card description runs past the edge of the card

**Triage:** bug. The function truncate() should return a string of at most limit characters, but the implementation simply slices text[:limit] and appends '...', ignoring the complex word-boundary requirements stated in the docstring.

**Fix** (fix verified): Implemented truncate according to docstring requirements so it respects word boundaries and character limits.

```diff
--- a/huskyutils/cards.py
+++ b/huskyutils/cards.py
@@ -15,4 +15,20 @@
     """
     if len(text) <= limit:
         return text
-    return text[:limit] + "..."
+
+    words = text.split()
+    current = ""
+    for word in words:
+        if current:
+            candidate = current + " " + word
+        else:
+            candidate = word
+        if len(candidate) + 3 <= limit:
+            current = candidate
+        else:
+            break
+
+    if current:
+        return current + "..."
+    else:
+        return text[:limit - 3] + "..."
```

**Verifier:** Ran python -m unittest discover -s tests -v in both original/ and fixed/. In original/, the tests in tests/test_issue.py failed as expected, reproducing Issue #10. In fixed/, all 28 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.cards import truncate


class TestTruncateIssue10(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(truncate("Wireless headphones", 30), "Wireless headphones")
        self.assertEqual(truncate("Wireless noise-cancelling headphones with case", 30), "Wireless noise-cancelling...")

    def test_len_within_limit(self):
        s = truncate("Wireless noise-cancelling headphones with case", 30)
        self.assertLessEqual(len(s), 30)

    def test_whole_words_and_ellipsis(self):
        self.assertEqual(truncate("Wireless noise-cancelling headphones with case", 30), "Wireless noise-cancelling...")
```

</details>