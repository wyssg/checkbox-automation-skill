---
name: checkbox-automation
description: "Select, check, or uncheck exact inclusive ranges across paginated web lists and result tables by global item number. Use when a user asks for ranges such as questions 501–750, wants different ranges assigned to multiple browser tabs, needs previous selections cleared outside each range, or wants the tabs verified and left open."
---

# Pagination Selection Loop

Use this skill to apply exact global ranges to checkbox lists that are split across pages. It is intended for question banks, search results, bulk-selection tables, and similar authenticated web pages. The operation changes selection state only; do not export, submit, delete, or otherwise create a side effect unless the user explicitly asks for it.

## Request contract

Before changing anything, identify:

- The browser and the exact existing tabs to use.
- The result-set total, if visible.
- The page size, or the page size to set.
- An inclusive 1-based range for each tab, such as `501–750`.
- The requested final action: leave open, export, or another explicit action.

Calculate each expected count as `end - start + 1`. If a range is outside the result set, reversed, or ambiguous, stop and clarify. If the user assigns multiple ranges to multiple tabs, preserve that mapping exactly.

## Browser and tab handling

- Use the appropriate browser-control skill. For existing Chrome tabs, use `chrome:control-chrome`.
- Inspect the current page before interacting. Confirm the URL, result-set status, page size, visible rows, and current selection status.
- When taking over user tabs, use the exact tab objects returned by `openTabs()`; never guess numeric tab IDs.
- Do not inspect cookies, local storage, passwords, session stores, or authentication tokens.
- Keep user tabs open when requested. At the end, finalize the browser session with the assigned tabs marked as deliverable. Do not perform further browser actions after finalization.

## Selection workflow

1. Confirm the result set and wait for loading to finish. A visible-page checkbox count is not the total result count.
2. Set the requested page size through the page UI, if needed. A common question-bank setting is 50 per page, but inspect the page rather than assuming it.
3. Navigate to page 1 before calculating global indexes. If the page size is `P`, the checkbox at zero-based row position `i` on zero-based page number `p` represents:

   `globalIndex = p * P + i + 1`

4. On every page, set each row checkbox to:

   `checked === (start <= globalIndex && globalIndex <= end)`

   This both selects the requested range and clears stale selections outside it.
5. After checkbox changes, wait for the page’s selection state to settle. After pagination clicks, wait until the first row changes or an equivalent page-state signal changes. Do not click through pages while the table is still loading.
6. Continue until the next-page control is disabled or absent. Handle a short final page; do not assume every page has `P` rows.
7. Verify using the site’s authoritative selection status, such as `250 questions selected for export`, and compare it with `end - start + 1`. Also verify that the result-set total is unchanged.

## Preferred page-loop implementation

Use semantic DOM selectors or supported browser locators where practical. For long, repetitive tables, the page’s supported CDP capability may be used after reading its capability documentation. A page-context loop should follow this shape:

```js
for (let page = 0; ; page++) {
  await waitUntilRowsExist();
  const rows = getTableRowCheckboxes();

  for (let i = 0; i < rows.length; i++) {
    const globalIndex = page * pageSize + i + 1;
    const wanted = start <= globalIndex && globalIndex <= end;
    if (rows[i].checked !== wanted) {
      rows[i].click();
      await shortSettleDelay();
    }
  }

  await selectionSettleDelay();
  const next = getNextPageControl();
  if (!next || isDisabled(next)) break;
  const oldFirstRow = getFirstRowIdentity();
  next.click();
  await waitUntilFirstRowChanges(oldFirstRow);
}
```

For sites with the College Board-style controls used in testing, useful heuristics are:

- Row checkboxes: `input[type="checkbox"]` whose nearest `tr` is a result row; exclude header/select-all controls.
- Next page: a link whose text is `Go to next page`, or the site’s documented next-page control.
- Page-size control: use the visible button or select option labelled `50` only when the page confirms that it is the desired setting.
- Page 1: use the visible pagination link or its accessible label rather than guessing a URL.

Run independent tab loops concurrently only after each tab has been separately identified and claimed. Never run two loops against the same tab at once.

## Verification and handoff

For each tab, report or internally verify:

- Assigned range.
- Expected count.
- Site-reported selected count.
- Result-set total.
- Whether the final requested tab disposition was applied.

Default to leaving the tabs open and do not click Export unless the user explicitly requests Export. If Export is requested, preserve any original or official PDF files and save downloads as separate copies. If the site reports a page-expiration, processing, or generic export error, keep the verified selections intact, report the site-side failure, and do not retry destructive or duplicate exports blindly.

## Recovery rules

- If the page reloads or expires, stop the loop, inspect the fresh page state, and restart only after confirming the result set and page size.
- If browser attachment is slow or interrupted, reacquire the exact current tab objects from `openTabs()`; do not guess IDs or switch browsers silently.
- If the status says `Loading`, wait and recheck rather than treating visible checkboxes as final.
- If a CAPTCHA, login prompt, permission prompt, or download prompt appears, pause and ask the user when confirmation is required.

## Reusable prompts

Single tab:

```text
Use $checkbox-automation on the existing results tab. Set 50 per page, select global questions 501–750, clear selections outside that range, verify the site reports 250 selected, and leave the tab open. Do not click Export.
```

Multiple tabs:

```text
Use $checkbox-automation on the existing results tabs. The result set has 1297 items and should use 50 per page. Tab 1: select 501–750. Tab 2: select 751–1000. Clear selections outside each assigned range, verify 250 selected in each tab, and leave both tabs open. Do not click Export.
```
