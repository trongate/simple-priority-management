# Simple Priority Management — Trongate v2

Reference implementation of click-to-reorder priority management for Trongate v2 modules: two triangles per row (`&#9650;` / `&#9660;`), an AJAX swap with no page reload, no drag-and-drop, no JavaScript of our own (Trongate MX does the wiring).

**This is a reference, not a package.** Nothing here is installable or designed to be used verbatim. It is a worked example of one technique. Copy the pattern out into your own module and adapt it to your columns, your table and your views.

## Using this repository as an AI reference

If you are an AI coding agent asked to add priority management to an existing Trongate module:

**Fastest route: [`PORTING.md`](PORTING.md).** Ten minutes, in order: the schema, the block to copy, the arrow markup, and the four checks that catch real bugs. Come back to this README only for the reasoning behind a step.

If you want to understand the mechanism first:

1. Read §1–§3 to understand the mechanism and the database contract.
2. Read §5.1 (the portable model core) and §5.3/§5.4 (endpoints and view).
3. Follow the adaptation checklist in §7.
4. Read §8 before writing code — it lists the failures that make this feature subtly wrong.

The demo uses an `articles` table on purpose. It has nothing to do with tasks, enquiries or anything else, so nothing in the pattern depends on a specific domain.

---

## 1. The approach

Every record carries an integer `priority`. Records are ordered by it. A "move" swaps the moving record's priority with that of its immediate neighbour above or below. Nothing else changes: **a move touches exactly two rows, no matter how large the list is.**

Priorities are kept normalised to `1..N` with no duplicates and no gaps. That gives three cheap operations:

| Operation | Cost | How |
|---|---|---|
| Order the list | `ORDER BY priority ASC` | Indexed column |
| Move one place up/down | 2 rows touched | Swap the two priority values |
| Insert | 1 row | New record gets `MAX(priority) + 1` (i.e. goes to the bottom) |
| Delete | 1 delete + resequence | Close the gap the delete leaves |

The UI is two triangle triggers per row. Each one fires a Trongate MX request; the endpoint performs the swap and returns **the whole table partial**, which MX swaps in with `mx-swap="outerHTML"`. The browser therefore never rebuilds the list itself — the server remains the single source of truth for both data and markup.

## 2. Invariants

Break any of these and the feature misbehaves in ways that are hard to spot by eye.

1. `priority` is `INT NOT NULL` with a **`UNIQUE` index**. Uniqueness is what makes ordering deterministic. The unique index is also the reason for the negative-placeholder dance in §5.2 — it will reject a naive two-way swap.
2. Stored priorities are **positive integers**. `1..N`, starting at 1. Zero and negative values are never stored, which is what makes `-(priority)` a safe temporary parking value.
3. A move is a **swap with a neighbour**, never a renumbering of the table.
4. Every multi-statement write runs inside a **transaction**.
5. The table name **never appears in a URL**. Only record ids do. The id is untrusted input; the table is not user input at all.
6. The move endpoint returns the **refreshed table partial**, not a status code, not JSON. The server owns the markup.
7. The swap target wrapper (`<div id="articles-table">`) lives **inside the partial**, so an `outerHTML` swap replaces a complete, self-identifying element.
8. Every mutating endpoint calls `$this->validation->run()`. In Trongate v2 that call is the only thing that enforces CSRF.
9. **`priority` is the first column of the table** — first `<th>` in the `<thead>` row and first `<td>` in every row. Lead with it or not at all: do not drop it in beside `title` because that happens to be where the code was easiest to edit. The number is the record's position in the list and the arrows are the feature's one control, so both belong at the left edge, where the eye starts. A priority column in the middle of a row reads as ordinary data and the whole feature looks like an afterthought. This applies to the demo and to every port of it.

## 3. Database

```sql
CREATE TABLE `articles` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL,
  `priority` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_priority` (`priority`)
) ENGINE=InnoDB;
```

Full demo schema plus seed rows: [`sql/articles.sql`](sql/articles.sql).

There are no foreign keys and no extra tables. Any table can take a `priority` column. The only structural decisions are:

- **Uniqueness** — `UNIQUE KEY uniq_priority (priority)`. InnoDB is required for the transactions to behave.
- **Scope** — if priority applies *within* a grouping (per category, per user, per project), the unique index becomes `(category_id, priority)` and every priority query in §5.1 must filter on that scope. See §8.2.

If the table already has rows, add the column, backfill it, and only then add the unique index:

```sql
-- 1. add the column without the constraint (every row starts at 0)
ALTER TABLE `articles` ADD `priority` int(11) NOT NULL DEFAULT 0;
-- 2. renumber to 1..N (see below)
-- 3. add the constraint last
ALTER TABLE `articles` ADD UNIQUE KEY `uniq_priority` (`priority`);
```

Step 2 is the same two-pass problem as a runtime resequence, so the simplest route is to reuse `resequence_priorities()` from the model against the target table before the constraint exists. If you prefer to do it in SQL, avoid ranking inside the `UPDATE` (see §8.1) and use an explicit two-step:

```sql
-- pass 1: unique negatives derived from the ordering you want to keep
UPDATE `articles` SET `priority` = -`id`;

-- pass 2: negative -> dense rank (MySQL 8+ / MariaDB 10.2+)
UPDATE `articles` a
  JOIN (SELECT `id`, ROW_NUMBER() OVER (ORDER BY `id` ASC) AS rn FROM `articles`) r
    ON r.`id` = a.`id`
   SET a.`priority` = r.rn;
```

Ordering by `id` is only an example: use whatever column reflects the order you want to preserve. Verify with `SELECT COUNT(*), MIN(priority), MAX(priority) FROM articles;` before adding the index — the three numbers must be equal (or `0,0,0` for an empty table).

## 4. Repository contents

```
README.md                         The design, in full
PORTING.md                        The short version: the job in order, for adapters
sql/
  articles.sql                    Demo schema + seed data
  verify_priorities.sql           Is the sequence sane? One query per check
modules/articles/
  Articles.php                    Controller: manage page + 2 MX endpoints + create/delete
  Articles_model.php              Model: the priority core (§5.1) + demo CRUD
  views/
    manage.php                    Page shell (bare HTML - swap for your template)
    table.php                     Table partial: the MX swap target and the arrow triangles
    create.php                    Minimal create form
    delete_conf.php               Minimal delete confirmation
```

The module is self-contained but not installable: `views/manage.php`, `create.php` and `delete_conf.php` are deliberately bare HTML pages so the demo runs in any Trongate v2 app with no template dependency. In a real module those shells get replaced by your own template call (e.g. `$this->templates->admin($data)`).

### 4.1 Running the demo

1. Copy this repository's `modules/articles` directory into a Trongate v2 application's `modules/` directory.
2. Import `sql/articles.sql` into a database and point `config/database.php` at it. **The script drops and recreates `articles`** — run it against a scratch database.
3. In `config/config.php`, set `BASE_URL`, and optionally `DEFAULT_MODULE` to `articles` and `DEFAULT_METHOD` to `manage`.
4. Browse to `/articles/manage`. The seed rows are listed in priority order with working arrows.

No login or session setup is required, because the demo has no authentication (§7, step 7).

## 5. How the code works

### 5.1 Model — the portable core

`modules/articles/Articles_model.php` is split into two blocks. Everything between `Priority ordering core` and `End of priority ordering core` is **portable**: it references only `$this->table_name`, the primary key `id`, and the `priority` column. Copy that block into your own model and change `$table_name`. Do not redesign the algorithm.

| Method | Purpose | Portable |
|---|---|---|
| `fetch_records($limit, $offset)` | One page, `ORDER BY priority ASC, id ASC` | Yes |
| `get_priority_bounds()` | `MIN`/`MAX` in use, so the view can disable arrows at the ends | Yes |
| `get_next_priority()` | `MAX(priority) + 1` for new records | Yes |
| `get_priority($id)` | Stored priority of one record | Yes |
| `move_up($id)` / `move_down($id)` | Public entry points | Yes |
| `move_record($id, $direction)` | The swap itself (§5.2) | Yes |
| `resequence_priorities()` | Renumber to `1..N`, order preserved | Yes |
| `ensure_sequential_priorities()` | Resequence only if `COUNT(*) !== MAX(priority)` | Yes |
| `create_new_record($data)` | Insert with `MAX+1` and a duplicate-key retry | Adapt column list |
| `delete_record($id)` | Delete then resequence | Yes |
| `get_data_from_db($id)`, `count_all()` | Demo helpers | Adapt |

`id ASC` in `fetch_records()` is a tie-breaker that never matters while priority is unique. It costs nothing and keeps ordering stable if somebody later removes the unique index.

`move_record()` finds the neighbour by value, not by id:

```sql
-- direction = up
SELECT id, priority FROM articles WHERE priority < :priority ORDER BY priority DESC LIMIT 1;
-- direction = down
SELECT id, priority FROM articles WHERE priority > :priority ORDER BY priority ASC LIMIT 1;
```

No neighbour row means the record is already at an end, and the method returns `false` without writing. Because the search is by value rather than by `priority ± 1`, it still works on legacy data that has gaps.

### 5.2 The swap, step by step

Say record **A** (priority `4`) is moving up past record **B** (priority `3`). Under a `UNIQUE` index you cannot simply write `A.priority = 3` while `B` still holds `3`, and you cannot set both to the same value "temporarily". The swap is therefore three statements in a transaction:

```sql
START TRANSACTION;
UPDATE articles SET priority = -4 WHERE id = A;   -- 1. park A out of the way
UPDATE articles SET priority =  4 WHERE id = B;   -- 2. B takes A's vacated slot
UPDATE articles SET priority =  3 WHERE id = A;   -- 3. A takes B's old slot
COMMIT;
```

The negative park in step 1 is essential; the `UNIQUE` index would reject any direct two-way assignment. It is collision-free only because stored priorities are always positive (§2.2).

Afterwards `ensure_sequential_priorities()` runs. On a normal list this is a no-op (`COUNT(*) === MAX(priority)`), so the cost is one cheap aggregate. It exists to normalise a list that arrived with holes in it — an import, a manual `DELETE`, another feature writing to the table.

`resequence_priorities()` is the two-pass version of the same problem:

```text
read:  SELECT id FROM articles ORDER BY priority ASC, id ASC   -- intended order
pass 1: park every row on -(rank)        -- unique negatives, no collisions
pass 2: write the final 1..N values
```

The intended order is read first and ranked **in PHP**. Do not try to rank inside the `UPDATE` (`SET @rank := @rank + 1 ... ORDER BY ...`, or a window function): the optimiser is free to reorder the rows it updates, which silently shuffles the list. This has been observed in practice.

### 5.3 Controller — the page and the two endpoints

`modules/articles/Articles.php`:

| Method | URL | Role |
|---|---|---|
| `manage()` | `articles/manage/{page}` | Paginated list in priority order |
| `move_up()` | `articles/move_up/{id}/{page}` | MX endpoint: swap up, return table partial |
| `move_down()` | `articles/move_down/{id}/{page}` | MX endpoint: swap down, return table partial |
| `create()` / `submit()` | `articles/create`, `articles/submit` | Create form; priority assigned automatically |
| `delete_conf()` / `submit_delete()` | `.../delete_conf/{id}`, `.../submit_delete/{id}` | Delete and resequence |

Both endpoints delegate to one private `handle_move($direction)`:

```php
private function handle_move(string $direction): void {
    if (!$this->validation->run()) {
        return;                       // CSRF failure: validation kills the request
    }

    $update_id = segment(3, 'int');

    if ($update_id > 0) {
        ($direction === 'up') ? $this->model->move_up($update_id) : $this->model->move_down($update_id);
    }

    $page = max(1, segment(4, 'int'));

    http_response_code(200);
    $this->view('table', $this->get_table_data($page));   // body IS the partial
}
```

Points that matter:

- **The page number travels in the URL** (`$page = segment(4)`) so the response re-renders the same slice of the list the user is looking at. Moving a row on page 3 must not dump the user back on page 1.
- `get_table_data()` is shared by `manage()` and the endpoints. That is what guarantees the swapped markup is identical to the markup that was already on screen — same rows, same bounds, same pagination, same CSRF token.
- The record id and the page number are the only URL input. A missing or unknown id is not an error: the swap quietly does nothing and the current page is re-rendered, which doubles as the "somebody else deleted this row" recovery path.
- `$this->validation->run()` is called on *every* mutating method, including `submit_delete()`, which has no validation rules. It runs the CSRF check regardless of rules; without it, that form would be unprotected.

### 5.4 View — the partial and the arrow triangles

`modules/articles/views/table.php` is rendered twice: inside the page, and as the entire response body of the endpoints. Three details are load-bearing.

**The wrapper lives in the partial.**

```php
<div id="articles-table">
    ... table, arrows, pagination ...
</div>
```

`mx-target="#articles-table"` with `mx-swap="outerHTML"` replaces that element with the response body. If the wrapper were in `manage.php` instead, the first swap would delete it and every subsequent click would have nothing to write into.

**Each arrow carries its own request.**

```php
$move_attr = [
    'class'      => 'priority-arrow',
    'mx-target'  => '#articles-table',
    'mx-swap'    => 'outerHTML',
    'mx-vals'    => $mx_vals           // {"csrf_token":"…","page":1}
];

$up_attr = $move_attr + [
    'title'      => 'Move up',
    'aria-label' => 'Move '.$row->title.' up',
    'mx-post'    => 'articles/move_up/'.$row->id.'/'.$current_page
];

$up_arrow = ($row_priority <= $lowest_priority)
    ? '<span class="priority-arrow is-off" aria-hidden="true">&#9650;</span>'
    : anchor('#', '&#9650;', $up_attr);
```

- `mx-vals` is a JSON object; MX appends each entry to the request body. That is how the CSRF token reaches an element that is not inside a `<form>`. Same session key (`$_SESSION['csrf_token']`) that `form_close()` uses.
- `$lowest_priority` / `$highest_priority` come from `get_priority_bounds()` and decide only *which markup to emit*: a link where the row can move, a `<span>` where it cannot. The comparison uses `<=`/`>=` rather than `===` so it still behaves on data with gaps. Never express this with `disabled`: see the last bullet below.
- The arrows are plain `<a>` elements (`anchor('#', '&#9650;', $attr)`) holding nothing but the triangle entity — not `form_button()`. No button chrome, no icon font, no image. A link is still keyboard-focusable (Enter) and touch-friendly. `href="#"` is never followed, because MX calls `preventDefault()` on any element carrying an `mx-*` attribute. That guarantee depends on MX being loaded: `anchor()` prepends `BASE_URL`, so the real href is the base URL plus `#`, and without JavaScript a click would navigate there rather than do nothing.
- **End of the list.** Where a move cannot happen, the arrow is rendered as `<span class="priority-arrow is-off" aria-hidden="true">&#9650;</span>` — same glyph, same position in the row, dimmed, and with no `mx-post`. `disabled` is not used: it is a no-op on a link, so a "disabled" anchor would still fire.

**The whole table plus pagination is re-rendered.** Search filters, per-page selectors and anything else that shapes the list must be re-applied by `get_table_data()`, and its state must travel with the request (§8.4).

## 6. Flow of one click

```
browser: click "move down" on record 7 (page 1)
  POST articles/move_down/7/1
  body: csrf_token=…, page=1
      │
      ├─ validation->run()             CSRF ok → continue (bad token: 403 for MX, 302 otherwise)
      ├─ model->move_down(7)           START TRANSACTION → 3 UPDATEs → COMMIT
      │                                ensure_sequential_priorities() → no-op if 1..N
      ├─ get_table_data(page 1)        rows + bounds + pagination + fresh csrf token
      └─ view('table', …)              echo the partial, HTTP 200
      │
browser: mx-swap="outerHTML" replaces #articles-table with the response body
         → new arrows, new bounds, new pagination, no page reload
```

## 7. Adapting this to your own module

**What to change:**

| Item | Demo value | Change to |
|---|---|---|
| Table | `articles` | Your table |
| Display column | `title` | Your column |
| `$table_name` | `'articles'` | Your table (model, one property) |
| Insert payload | `['title' => $title]` | Your columns, without `priority` |
| Controller | `Articles`, `articles/...` URLs | Your module and endpoint names |
| `$default_limit` | `10` | Your page size (or remove pagination) |
| `views/table.php` | Priority / Title / Actions | Your columns; keep the priority cell **as the first column** and the wrapper |
| Page shells | `manage.php`, `create.php`, `delete_conf.php` | Your admin template |
| Schema | 3 columns | Yours + `priority INT NOT NULL` + `UNIQUE` |

**Checklist:**

1. **Schema.** Add `priority int(11) NOT NULL` and `UNIQUE KEY uniq_priority (priority)`. Renumber existing rows first (§3). Verify: `SELECT COUNT(*), MAX(priority) FROM your_table;` — both numbers must match, and `MIN` must be 1.
2. **Model.** Copy the portable block (§5.1) into your model. Point `$table_name` at your table. Leave `move_record()`, `resequence_priorities()` and `ensure_sequential_priorities()` untouched.
3. **Insert.** Route every insert through `create_new_record()`, so nothing ever lands in the table without a priority. If your module also creates records elsewhere, fix those paths too.
4. **Delete.** Route every delete through `delete_record()` so the gap gets closed.
5. **Controller.** Add `move_up()` / `move_down()` (or one `move()` taking a direction), both calling `handle_move()`-style logic. Share one `get_table_data()` between the page method and the endpoints.
6. **View.** Keep the wrapper div in the partial, keep `mx-target`/`mx-swap`/`mx-vals` consistent with it, render a `<span>` instead of a link at the bounds, and put every list-shaping parameter into `mx-vals`. Priority is the **first column**, in the header and in every row — see invariant 9.
7. **Auth.** The demo has none. Wrap the endpoints in whatever check protects the rest of your module.

## 8. Integration considerations

### 8.1 Things that silently break

- **Renumbering the whole table on every move.** Correct-looking, but O(N) writes per click, and it fights the unique index. A move is a two-row swap.
- **Ranking rows inside an `UPDATE`** with `@rank` or a window function. The optimiser can reorder the updated rows; the list comes out shuffled. Read the order, rank in PHP, write it back — that is what `resequence_priorities()` does.
- **Skipping the negative placeholder** in a swap. The unique index rejects the intermediate state and the request dies with a DB error; wrap everything in a transaction so a failed swap cannot leave the list half-written.
- **Skipping the transaction.** A failure between statements leaves a duplicate or a missing priority, and the list then has two rows competing for a slot.
- **Returning JSON and rebuilding the table in JavaScript.** The response must be the same markup the server renders, or the two implementations drift and the pagination/CSRF state goes stale.
- **`mx-swap="innerHTML"` with the wrapper inside the partial.** You end up with nested duplicate ids; the next click targets the stale outer one. `outerHTML` and the wrapper travel together.
- **Forgetting the page number.** Moving a row on page 3 reloads page 1 and looks like the move failed.
- **Trusting a table name or sort column from the request.** Ids only, in the URL. The table is a constant in your model.

### 8.2 Scoped ordering (priority within a group)

If records are ordered within a category, project or owner:

```sql
UNIQUE KEY uniq_priority_per_scope (`category_id`, `priority`)
```

and then **every** priority statement must carry the scope predicate: the neighbour lookup, the bounds query, `get_next_priority()` (add `WHERE category_id = :id`), `fetch_records()`, and both passes of `resequence_priorities()`. Missing it in one place is the classic bug — the swap silently moves a row in another category. Add the scope column as a parameter to the five affected methods rather than reaching for a property.

### 8.3 Concurrency

Two users moving rows at the same time each get a valid transaction; the happy path is safe because the unique index makes the "park then assign" sequence atomic. What is not protected is *intent*: user A's swap can be based on a list that user B has just reordered, so the last write wins. For a small admin list that is acceptable. If you need more, add a `version` column to the row and reject the move when the posted version no longer matches.

### 8.4 Pagination, search and filters

The table partial re-renders whatever `get_table_data()` returns, so:

- The endpoints must receive the **current page** (URL segment) and **any active filters** (`mx-vals`), and apply exactly the same `WHERE` clause as the page method.
- Keep the pagination root pointing at the manage method (`articles/manage`), so pagination links rebuild correctly after a swap.
- A move can legitimately shift a row across a page boundary. Re-render the requested page; do not try to follow the row.

### 8.5 Accessibility and UX

Because reordering is two links rather than drag-and-drop, keyboard and touch users get the feature for free. Keep `title` and `aria-label` on the arrows (they name the record), render a non-link `<span>` at the ends of the list so nothing useless can be clicked, and do not add drag-and-drop on top: it needs its own keyboard equivalent and it makes concurrent edits harder to reason about.

## 9. Verification checklist

Run these against your adapted implementation. Each has one expected result.

| Check | Expected |
|---|---|
| Move the second row up | It swaps with the first; no duplicate priorities; list still `1..N` |
| Move the first row up | Dimmed `<span>` at the top, not a link; if forced, server does nothing and returns the same list |
| Move the last row down | Same, at the bottom |
| Move a row on page 2 | Response is page 2, not page 1 |
| Create a record | It appears at the bottom with `MAX(priority) + 1` |
| Delete a record | Remaining priorities renumber to `1..N`, order preserved |
| `SELECT COUNT(*), MAX(priority) FROM t` | Equal, `MIN` = 1, after any sequence of the above |
| POST a move without a CSRF token | Rejected (403 for MX/JSON requests, 302 for plain navigation) |
| Empty table | Manage page renders "no records", no PHP warnings |
| Legacy data with a gap, then any move | Gap closed, list normalised |

## 10. Compatibility

Written and tested against **Trongate v2 (2.2026.1028)** with PHP 8.2+ and MySQL/MariaDB InnoDB.

Requires: Trongate MX (`mx-post`, `mx-target`, `mx-swap`, `mx-vals`), the `validation` module for CSRF, and the `pagination` module (drop `pagination_data` and the `Modules::run('pagination/display', …)` call if your list is not paginated).
