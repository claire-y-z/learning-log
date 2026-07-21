## FROM CLAUDE, FOR SYNTAX REVIEW
# Python / Pandas / R Study Guide
*A reference sheet for key syntax, distinctions, and gotchas*

---

## 1. Core Data Structures

| Structure | What it is | Analogy |
|---|---|---|
| **Series** | One column of labeled data | A single labeled list |
| **DataFrame** | A table — multiple Series sharing the same index | A spreadsheet |
| **Index** | The labels attached to rows (DataFrame) or values (Series) | Row "name tags" |

- A DataFrame is really a bunch of Series stacked together, all sharing the **same index**.
- Pulling one column out of a DataFrame (`df['col']`) gives you back a **Series**, and it keeps the original row labels.
- If you don't set a custom index, pandas auto-assigns `0, 1, 2, 3...` — but these are still **labels**, not guaranteed positions (see `.loc` vs `.iloc` below).

**R equivalent:** a `data.frame` / `tibble` in R plays the same role as a pandas DataFrame. R doesn't really have a separate "labeled index" concept the way pandas does — row names exist but are used far less commonly.

---

## 2. `.loc` vs `.iloc` — the most important distinction

| | `.loc` | `.iloc` |
|---|---|---|
| Selects by | **Label** (the actual name in the index) | **Position** (integer count, starting at 0) |
| Column selector | Column **name** (string) | Column **position** (integer) |
| Slicing | **Inclusive** of end (`'a':'c'` includes `'c'`) | **Exclusive** of end (`0:3` excludes position 3 — same as Python lists) |
| Works with boolean filters | ✅ Yes (`df.loc[df['age'] > 3]`) | ❌ No |

```python
df.loc['a']            # row LABELED 'a'
df.iloc[0]               # row at POSITION 0

df.loc[:, 'age']         # all rows, column NAMED 'age'
df.iloc[:, 1]              # all rows, column at POSITION 1

df.loc['a':'c']          # labels a, b, c   (c INCLUDED)
df.iloc[0:3]               # positions 0, 1, 2 (3 EXCLUDED)
```

**Syntax pattern for both:** `df.loc[rows, columns]` / `df.iloc[rows, columns]`
→ **first slot = rows, second slot = columns**, always.

**Key gotcha:** Even when the index *looks* like plain sequential integers (the pandas default), it's still a **label**, not a guaranteed position. If you filter, sort, or drop rows earlier, labels and positions can diverge — label `5` might no longer sit at position `5`.

- Converting a label's **type** (`int(label)`) does **not** convert it into a **position**. Use `df.index.get_loc(label)` if you genuinely need the current position of a known label.

**R/dplyr equivalent:** dplyr doesn't split into loc/iloc — you typically use `filter()` for condition-based row selection and `select()` for columns, and `slice()` if you need row *position*:
```r
df |> filter(age > 3)          # like df.loc[df['age'] > 3]
df |> slice(1:3)                 # like df.iloc[0:3] (note: R is 1-indexed!)
df |> select(name, age)          # like df.loc[:, ['name','age']]
```
⚠️ **R is 1-indexed, Python is 0-indexed** — this is one of the most common cross-language bugs.

---

## 3. `idxmax()` / `idxmin()` — find the *location*, not the value

| Method | Returns |
|---|---|
| `.max()` / `.min()` | The actual **value** |
| `.idxmax()` / `.idxmin()` | The **label** of the row where that value lives |

```python
df['age'].max()        # 5     → the value
df['age'].idxmax()      # 'b'   → the LABEL of the row where age = 5
```

**Common pattern:** find the label of an extreme value, then use `.loc` to pull the full row/other columns from it:
```python
best_label = (reviews.points / reviews.price).idxmax()
reviews.loc[best_label, 'title']     # .loc, NOT .iloc — idxmax() gives a LABEL
```

- `idxmax()` only returns **one** label, even if there's a tie. To get *all* rows tied for a max:
```python
df[df['age'] == df['age'].max()].index
```

**R equivalent:**
```r
df |> filter(age == max(age)) |> pull(name)     # get the name(s) at max age
which.max(df$age)                                  # position of first max (like idxmax, but POSITION not label)
```

---

## 4. `.value_counts()` vs `.groupby()`

- `.value_counts()` = a **shortcut** for one specific job: count how many times each unique value appears in **one column**.
```python
reviews.country.value_counts()
```
- `.groupby()` = the **general-purpose** tool — can count, but also average, sum, or run custom aggregations, across one or many columns.
```python
reviews.groupby('country').size()                  # same result as value_counts()
reviews.groupby('country')['price'].mean()           # value_counts() CANNOT do this
```

**R/dplyr equivalent:**
```r
df |> count(country)                                  # ~ value_counts()
df |> group_by(country) |> summarize(avg = mean(price), .groups = "drop")   # ~ groupby().mean()
```

**Recall your own note on `summarize()` vs `mutate()`:** `summarize()` **collapses** rows down to one per group; `mutate()` **preserves** the original number of rows while adding a new column.

---

## 5. `DISTINCT` / `unique()` / `distinct()` — same logic, three syntaxes

| Tool | Syntax |
|---|---|
| SQL | `SELECT DISTINCT column FROM table` |
| R (dplyr) | `df |> distinct(column)` |
| Pandas | `df['column'].unique()` |

Same underlying concept (get distinct values from a column) — only the vocabulary changes across tools.

---

## 6. `lambda` — quick, unnamed functions

`lambda` is a **Python** feature (not pandas-specific) for writing small, throwaway functions in one line. Pandas leans on it heavily with `.map()` and `.apply()`.

```python
lambda x: x + 1                       # same as: def f(x): return x + 1

reviews.description.map(lambda desc: 'tropical' in desc)
# for each desc, check if 'tropical' is in it → True/False per row
```

- **`lambda` ≠ "pure function."** Whether a function is pure (same input → same output, no side effects) depends on *what's inside it*, not on whether it's written as `lambda`. Most pandas lambdas happen to be pure because that's the natural fit for `.map()`/`.apply()` — but it's not guaranteed by the syntax itself.
- **Naming collision warning:** In *Mathematica*, "Pure Function" means something different — it's just Wolfram's name for an anonymous function (their version of `lambda`), unrelated to the side-effect-free meaning.

**R equivalent:** R has its own anonymous function syntax, including a shorthand introduced in R 4.1+:
```r
sapply(df$age, function(x) x + 1)     # traditional
sapply(df$age, \(x) x + 1)              # shorthand (like lambda)
```

**`.map()` counting pattern** (turns True/False into counts via `.sum()`, since `True == 1` and `False == 0`):
```python
t = reviews.description.map(lambda desc: 'tropical' in desc).sum()
```

---

## 7. Boolean filtering with `.loc`

```python
reviews.loc[reviews.country == 'Italy']
reviews.loc[(reviews.country == 'Italy') & (reviews.points >= 90)]
```

- Parentheses around a **single** condition are optional (just grouping/readability).
- Parentheses are **required** when combining conditions with `&` (and) / `|` (or), due to Python operator precedence.

**R/dplyr equivalent:**
```r
df |> filter(country == "Italy")
df |> filter(country == "Italy" & points >= 90)     # dplyr doesn't require the extra parens
```

---

## 8. First-look / exploration methods

| Method | What it tells you |
|---|---|
| `df.shape` | `(rows, columns)` — size of the dataset |
| `df.head(n)` | First `n` rows (default 5) — quick look at the data |
| `df.describe()` | Summary stats (count, mean, std, min/max, quartiles) |

**R equivalent:**
```r
dim(df)          # ~ .shape
head(df, 5)        # ~ .head()
summary(df)        # ~ .describe()
```

---

## 9. Class & Object fundamentals (Python OOP basics)

| Term | Meaning |
|---|---|
| **Class** | A blueprint/template (e.g., `class Dog:`) |
| **Object / instance** | An actual thing built from the blueprint (e.g., `my_dog = Dog("Rex", 3)`) |
| **`self`** | Refers to "this particular object" — how methods access that object's own data |
| **Attribute** | Data stored on the object (`self.name`) |
| **Method** | A function defined inside a class, using `self` |
| **`__init__`** | Special method that runs automatically when an object is created, to set up starting data |

```python
class Dog:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def bark(self):
        print(f"{self.name} says Woof!")

rex = Dog("Rex", 3)
rex.bark()
```

A class bundles **data + the functions that act on that data**, so each object carries its own info around without needing to re-pass it into every function call.

---

## 10. Loops & comprehensions — common bug patterns to avoid

- **`return` inside a loop** exits the *entire function* immediately — don't use it if you're trying to accumulate results across iterations. Use `.append()` (or a comprehension) instead, and `return` only after the loop finishes.
- **`for...else`**: the `else` runs only if the loop completes *without* hitting `break` — it does **not** mean "if the condition was never True." Easy to misuse.
- **Off-by-one indexing**: when comparing `list[i]` to `list[i+1]`, loop while `i < len(list) - 1`, not `i < len(list)`, to avoid an out-of-range error on the last step.
- **List comprehension vs manual loop** — same result, different mechanics:
```python
[x > thresh for x in L]              # comprehension: bookkeeping (create list, append) is implicit

result = []                             # manual loop: bookkeeping is explicit
for x in L:
    result.append(x > thresh)
```
- **`()` vs `[]`**: `[]` builds an actual list right away; `(x for x in L)` builds a **generator** — lazy, one value at a time, not stored in memory upfront.

**R equivalent (functional style, often preferred over explicit loops in R):**
```r
sapply(L, \(x) x > thresh)          # ~ list comprehension
map_lgl(L, \(x) x > thresh)          # purrr version, more common in tidyverse-style R
```

---

## 11. Function argument order matters (position, not name)

```python
def word_search(doc_list, keyword):   # expects doc_list FIRST, keyword SECOND
    ...

word_search(doc_list, keyword)          # ✅ correct — matches definition order
word_search(keyword, doc_list)          # ❌ wrong — arguments get swapped by POSITION, not by name
```

Python matches arguments to parameters by **position**, left to right — it doesn't know or care what you *named* your variable when calling the function. Mismatched order can cause confusing errors (e.g., calling `.lower()` on a list instead of a string).

---

## Quick-Reference Cheat Sheet

| Task | Pandas | R (dplyr) |
|---|---|---|
| First N rows | `df.head(n)` | `head(df, n)` |
| Dimensions | `df.shape` | `dim(df)` |
| Filter rows | `df.loc[condition]` | `df \|> filter(condition)` |
| Select columns | `df.loc[:, cols]` | `df \|> select(cols)` |
| Row by label | `df.loc[label]` | `df \|> filter(row_names(df) == label)` |
| Row by position | `df.iloc[pos]` | `df \|> slice(pos)` *(1-indexed!)* |
| Unique values | `df['col'].unique()` | `df \|> distinct(col)` |
| Count occurrences | `df['col'].value_counts()` | `df \|> count(col)` |
| Group + aggregate | `df.groupby('col').mean()` | `df \|> group_by(col) \|> summarize(...)` |
| Add column, keep rows | `df['new'] = ...` | `df \|> mutate(new = ...)` |
| Label of max value | `df['col'].idxmax()` | `which.max(df$col)` *(gives position, not label)* |
| Anonymous function | `lambda x: ...` | `\(x) ...` or `function(x) ...` |