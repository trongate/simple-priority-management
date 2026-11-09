# Porting priority management — the short version

Ten minutes, not two hours. This page is the whole job in order. The demo
module in `modules/articles/` is the answer key: every file named below is
there, running, with the portable parts marked.

Read the README only when something here surprises you.

## 0. The shape of it

- One `priority int` column, `1..N`, `UNIQUE`.
- A move swaps two rows inside a transaction, parking on a negative value.
- Each arrow is a link with `mx-post`; the response is **the whole list
  partial**, swapped in with `mx-swap="outerHTML"`.
- The page method and both endpoints share **one** `get_table_data()`.

## 1. Names first

Write down, before touching a file: the **table**, the **module** (URLs), the
**display column**, and whether priority is global or per-group. Most of the
time lost on this feature is a half-renamed copy.

## 2. Schema

```sql
ALTER TABLE `your_table` ADD `priority` int(11) NOT NULL DEFAULT 0;
-- renumber the existing rows to 1..N first (README §3), then:
ALTER TABLE `your_table` ADD UNIQUE KEY `uniq_priority` (`priority`);
```

Per-group list? The key is `(group_id, priority)` and every query in the
model core needs the group predicate — README §8.2.

## 3. Model — copy the block, not the algorithm

`modules/articles/Articles_model.php` is split at
`Priority ordering core` / `End of priority ordering core`. Copy that block
verbatim, change `$table_name`, add your own CRUD. Then close the two back
doors:

```bash
grep -rn -- '->insert(' modules/your_module    # each hit must go through create_new_record()
grep -rn -- '->delete(' modules/your_module    # each hit must go through delete_record()
```

## 4. Controller

Copy the five methods from `modules/articles/Articles.php`: `manage()`,
`move_up()`, `move_down()`, `handle_move()`, `get_table_data()`, plus
`get_csrf_token()`. Rename the module in the two `mx-post` URLs and in
`pagination_root`. Every mutating method keeps its `$this->validation->run()`
— in Trongate v2 that call *is* the CSRF check.

## 5. View

Copy `modules/articles/views/table.php`. Three things are load-bearing:

- The `#your-table` wrapper lives **inside** the partial.
- `mx-target` names that wrapper; `mx-swap` is `outerHTML`.
- `mx-vals` carries the CSRF token **and the page number**.

And one thing that is not load-bearing but is the house rule: **priority is the
first column**, first `<th>` in the header row and first `<td>` in every row.
The number is the record's position in the list and the arrows are the
feature's only control, so they sit at the left edge as you would expect. Keep
the priority cell first when you rename the columns; do not let it drift into
the middle of the row.

The arrows themselves:

```php
$move_attr = [
    'class' => 'priority-arrow',
    'mx-target' => '#your-table',
    'mx-swap' => 'outerHTML',
    'mx-vals' => $mx_vals
];

$up_attr = $move_attr + [
    'title' => 'Move up',
    'aria-label' => 'Move '.$row->title.' up',
    'mx-post' => 'your_module/move_up/'.$row->id.'/'.$current_page
];

// A plain link holding nothing but a triangle. Not form_button():
// no button chrome, no icon font, no image.
$up_arrow = ($row_priority <= $lowest_priority)
    ? '<span class="priority-arrow is-off" aria-hidden="true">&#9650;</span>'
    : anchor('#', '&#9650;', $up_attr);
```

The `span` is what happens at the ends of the list. **Do not use
`disabled`** — it is a no-op on a link, so the arrow would still fire.

## 6. CSS

Copy the `<style>` block from `modules/articles/views/manage.php` into the
stylesheet the page already loads. Never inline styles in the partial: a swap
stamps another copy into the document on every click.

## 7. Verify

`sql/verify_priorities.sql` answers "is the sequence sane?" in one query per
check. Then the four checks that actually catch bugs:

| Check | Expected |
|---|---|
| Move a row on page 2 | Response is page 2, not page 1 |
| Move the top row up | Nothing happens; the arrow is a `span`, not a link |
| Delete a middle row | Remaining rows renumber to `1..N` |
| POST a move with no CSRF token | Rejected (403 for MX, 302 for plain navigation) |
| Inspect the table header | `Priority` is the first `<th>`, and each row's first `<td>` is the priority cell |

## 8. The five mistakes that cost the most time

1. `form_button()` for the arrows instead of `anchor()`.
2. `disabled` on a link — silently still fires.
3. Renumbering the whole table on every move instead of swapping two rows.
4. Ranking inside the `UPDATE` (`@rank`, window function) — read the order,
   rank in PHP, write it back.
5. `mx-swap="innerHTML"` with the wrapper inside the partial — nested
   duplicate ids, and the second click hits the stale outer element.
