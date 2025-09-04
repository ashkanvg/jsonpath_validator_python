# JSONPath Query Validator — README

Validate and preview **JSONPath** queries against a sample dataset in a lightweight, reproducible Colab/Notebook workflow.

<a href="https://colab.research.google.com/github/ashkanvg/jsonpath_validator_python/blob/main/query_validator.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open in Colab"/></a>

---

## What this notebook does

- Loads a JSON dataset (array) or **NDJSON/JSONL** file.
- Runs a list of JSONPath expressions using **[`jsonpath2`](https://pypi.org/project/jsonpath2/)**.
- Prints, for each query:
  - the query string,
  - the number of matches,
  - up to 5 previewed values (so you don’t flood the console).

It’s ideal for **authoring/validating** JSONPath queries, checking **counts**, and sanity-checking **payload structure** before wiring queries into downstream code.

---

## Requirements

- Python 3 (Colab or local)
- `jsonpath2` (installed in the first cell)

> The notebook currently targets a small **GitHub Archive–style** sample (`github_archive_sample_25.json`) with 25 event records. You can replace it with your own JSON/JSONL.

---

## File layout & key cells

1) **Install & imports**

```python
# Notebook cell
!pip install jsonpath2

import json, random, sys
from jsonpath2.path import Path
```

2) **Data loader** (supports `.json` array or `.jsonl` / NDJSON)

```python
def load_json_or_jsonl(path):
    with open(path, "r", encoding="utf-8") as f:
        first = f.read(1); f.seek(0)
        if first == '[':
            return json.load(f)
        return [json.loads(line) for line in f if line.strip()]
```

3) **Runner**

```python
def run(expr, dataset):
    path = Path.parse_str(expr)
    matches = [m.current_value for m in path.match(dataset)]
    print(f"\n{expr}\n→ {len(matches)} match(es)")
    for v in matches[:5]:
        print("  ", v)
```

4) **Load sample data**

```python
json_path = "./github_archive_sample_25.json"
data = load_json_or_jsonl(json_path)
assert isinstance(data, list), "Expected a top-level array"
print(f"Loaded {len(data)} records")
```

5) **Queries list**  
A curated set of JSONPath patterns (fields, wildcards, recursive descent, filters, unions, slices, function calls like `length()` and `substring()`), e.g.:

```python
queries = [
    '$[*].id',
    '$[*].actor.login',
    '$[*]..created_at',
    '$[*].payload.pull_request._links.*.href',
    '$[*][?(@.payload.pull_request.state = "open")].payload.pull_request["id","number","title"]',
    '$[*].payload.pull_request["head","base"][?(@.repo.language = "Go")].repo["full_name","watchers_count"]',
    '$[*].payload.pull_request._links.*["href"][length()]',
    '$[*].payload.comment.diff_hunk[substring(0, 40)]',
    # ...more examples in notebook
]
```

6) **Execute all queries**

```python
for q in queries:
    run(q, data)
```

---

## Quick start

### Run in Colab
1. Click the **Open in Colab** badge above.
2. Runtime → Run all.

### Run locally
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install jsonpath2
# launch Jupyter or run cells in your editor
```

Place your data file next to the notebook and set:
```python
json_path = "./your_file.json"   # or "./your_file.jsonl"
```

---

## Input data expectations

- The loader expects:
  - **Array JSON** (top-level `[]`) **or**
  - **JSON Lines** / **NDJSON** (one JSON object per line).
- The notebook asserts the in-memory structure is a **list**.

> If your file is a single large object (`{ ... }`), wrap it into a list or adjust the assertion accordingly.

---

## Understanding the output

For each query the runner prints:

```
$[*].repo.url
→ 25 match(es)
   https://api.github.com/repos/unwarysheep/blog
   https://api.github.com/repos/OfficeDev/PnP
   ...
```

- `→ N match(es)`: total number of matches in the dataset.
- Then up to **5** example values to preview structure/content.

---

## JSONPath2 tips & patterns (used here)

- **Basic field access**: `$.field`, `$.obj.nested`, `$[*].arrayField`
- **Wildcards**: `*` for any key/index: `$.obj.*.href`, `$[*]..id`
- **Recursive descent**: `..` to search deeply: `$[*]..created_at`
- **Unions**: `['a','b']` to project multiple keys: `...['html_url','diff_url','patch_url']`
- **Slices**: `[start:end:step]`, `[-1]` (last), e.g. `$[0:5].id`, `$[::5].id`, `$[-1].id`
- **Filters**: `[? ( <predicate> ) ]` with:
  - Equality: `@.state = "open"`
  - Logical OR/AND: `cond1 or cond2`, `cond1 and cond2`
  - Comparisons: `@.repo.open_issues_count > 0`
  - Existence checks: `@.org` or combined `@.org and @.org.login = "Clever"`
- **Functions** (jsonpath2’s extension):
  - `length()` on arrays/strings: `...["href"][length()]` (returns length)
  - `substring(start, len)`: `...diff_hunk[substring(0, 40)]`
  - Regex matches with `=~`: `@.html_url =~ /^https:/`

> Note: `jsonpath2` uses **`=`** for equality in filters (not `==`), and supports `=~` for regex.

---

## Customizing for your data

- Replace `json_path` with your dataset path.
- Replace/extend `queries` with your own JSONPath expressions.
- If your top level is not a list, remove or adapt:
  ```python
  assert isinstance(data, list)
  ```
- Want larger previews? Change:
  ```python
  for v in matches[:5]:  # increase 5 to a bigger number
  ```

---

## Troubleshooting

- **`Expected a top-level array`**  
  Your JSON is probably a single object. Wrap it in a list or remove the assertion.
- **`ValueError: line X:Y ...`** from `jsonpath2`  
  There’s a syntax issue in your JSONPath. Check:
  - Quotes around union keys: `["a","b"]`
  - Filters use `=` (not `==`)
  - Functions spelled correctly: `length()`, `substring(start, len)`
  - Regex needs `/.../` and proper escaping when necessary.
- **Zero matches but field exists**  
  Confirm the path segment by segment. Try a broader version first (e.g., list `_links.*` before drilling into `.href`). Also ensure the dataset actually contains the structure in the sample slice.

---

## Example queries you can copy

```python
# All event IDs
'$[*].id'

# All actor logins
'$[*].actor.login'

# All URLs found anywhere
'$[*]..url'

# Pull request “open” triad (id, number, title)
'$[*][?(@.payload.pull_request.state = "open")].payload.pull_request["id","number","title"]'

# Links in PR payload (any key) → hrefs
'$[*].payload.pull_request._links.*.href'

# Repos where PR language is Go (project name + watchers)
'$[*].payload.pull_request["head","base"][?(@.repo.language = "Go")].repo["full_name","watchers_count"]'

# Preview first 40 chars of a diff hunk
'$[*].payload.comment.diff_hunk[substring(0, 40)]'
```

---

## Repo & notebook

- **Notebook:** `query_validator.ipynb`
- **Repo:** `ashkanvg/jsonpath_validator_python`

If you open the notebook directly on GitHub, use the **Colab badge** at the top to run it interactively.

---

## License

Add your preferred license (e.g., MIT) in the repository root as `LICENSE`.

---

## Changelog (suggested)

- **v0.1** — Initial notebook with loader, runner, sample queries, and output previews.

---

**Questions or ideas?** Open an issue or PR in the repo—happy validating!
