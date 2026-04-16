# Output Conventions

How every template formats the agent's response. Consistency across templates = lower switching cost for users.

## The 5-section structure

Every agent response follows this exact order:

```
**<Bold headline — one sentence with the key numbers.>**

![chart](<URL from create_chart>)

| Column  | Col     | Numeric |
| ------- | ------- | ------: |
| Value   | Value   |   1,234 |
| Value   | Value   |   5,678 |

```sql
SELECT ... -- or ```DAX EVALUATE ...
```

- **Observation 1** — non-obvious pattern
- **Observation 2** — trend / anomaly / comparison
- **Observation 3** — action-worthy insight
```

Separators are blank lines. No "Here is your analysis" preamble. No closing pleasantries.

## Section rules

### 1. Headline

- One sentence, bold
- Contains the key number(s)
- Action-worthy: "Revenue **tripled** to $46K in Apr 2026" beats "Revenue increased over time"
- Never restate the question

### 2. Chart

- Only embedded if `create_chart` was called
- Markdown image syntax: `![chart](URL)` on its own line
- URL comes directly from the tool response — never fabricated

**When to skip the chart:**
- Answer is a single scalar ("How many customers? → 240")
- User explicitly said "no chart" or "table only"
- Result is 1 row × 1 column

**Default: include a chart.** Target ≥80% of responses.

### 3. Data table

- **Pipe-delimited markdown**, never space-aligned
- Header row + separator row with `---` (alignment markers):
  - `---:` = right-align (use for numeric columns)
  - `:---:` = center
  - `:---` = left (default, usually omitted)
- Max 15 rows. If more, summarize or aggregate instead.

### 4. Query block

- Fenced code block with the language tag (`sql`, `DAX`, `MongoDB aggregate`, etc.)
- The exact final query the agent ran — not a simplified version
- No inline comments unless the query has a non-obvious trick

### 5. Observations

- 2-4 bullets
- **Non-obvious only** — don't restate what the table shows
- Good examples:
  - "June 2025 dip is a single anomaly, not a trend"
  - "Growth rate accelerated ~50% after Q3 2025"
  - "One customer accounts for 18% of revenue — concentration risk"
- Bad examples:
  - "Revenue is $463,145" (already in headline)
  - "The data shows monthly revenue" (filler)

## Number formatting

| Type | Format |
|---|---|
| **Money** | `$1,234` (thousands separator, no decimals) — or `$4.99` if < $10 |
| **Percent** | `12.3%` (one decimal, no space before %) |
| **Date (month)** | `2025-04` |
| **Date (day)** | `2025-04-15` |
| **Large counts** | `1.2M` for ≥1M, `45K` for ≥1K, `234` otherwise |
| **Null/missing** | `—` (em-dash) not `null` or empty |

## Table alignment cheat sheet

```
| Text col | Number | Pct    | Money   | Date       |
| -------- | -----: | -----: | ------: | :--------- |
| Value    |  1,234 |  12.3% | $1,234  | 2025-04    |
```

- Text → left (default, no colons)
- Numbers / percents / money → right (`---:`)
- Dates → left (more readable than right)

## Why strict formatting matters

- **Chat widgets render GFM markdown consistently.** Space-aligned tables break. Pipe tables always work.
- **Users scan, they don't read.** Bold headline → chart → table → details is a descending-attention structure that matches how people actually consume analytics.
- **Consistency reduces "is this working?" anxiety.** When every answer has the same shape, users trust the output.

## Chart type selection

| Data shape | Chart type |
|---|---|
| Time series (grouped by day/week/month/year) | `line` |
| Category comparison (by country, segment, category) | `bar` (use `horizontalBar` if >8 categories) |
| Share of total (≤6 slices) | `doughnut` or `pie` |
| Two numeric dimensions | `scatter` |
| Multi-series with similar magnitudes | grouped/stacked `bar` |
| Multi-series with very different magnitudes | **split into two charts** — don't use dual-axis (LLMs get it wrong) |

## Explicitly NOT in the output

- ❌ "Here's your analysis" / "Sure, let me help you with that"
- ❌ "As an AI, I cannot..." (the agent *can* — refuse crisply if needed, no AI apologetics)
- ❌ Trailing summary after the observations ("In conclusion, sales are growing")
- ❌ Emoji decoration (neutral if user adds them, but don't lead)
- ❌ Tables wrapped in code blocks (breaks markdown rendering)
