---
layout: post
title: "Filtering 4,437 Links Froze the Browser"
date: 2026-11-05
categories: web performance debugging
---

My local URL archive froze when I typed `re` into its filter. The page contained 4,437 links, and the browser reported that the page was not responding.

The server returned the page quickly: one HTTP request transferred about 2.67 MB in 15 ms. That measurement did not include browser rendering or filtering. In a separate browser tab, a reproduction of the filter timed out after 20 seconds.

## The filter

The original code read each row's `innerText`, then immediately changed its visibility:

```javascript
rows.forEach(row => {
  row.style.display = row.innerText.toLowerCase().includes(query) ? '' : 'none';
});
```

[`innerText`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/innerText) represents rendered text. According to the [`textContent` documentation](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent#differences_from_innertext), reading `innerText` can trigger a reflow to account for current styles. Alternating a layout-dependent read with a style change for every row is a form of [layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing).

The replacement reads all the text first, then applies the visibility changes:

```javascript
const rows = Array.from(document.querySelectorAll('#tbl tbody tr'));
const matches = rows.map(row => row.textContent.toLowerCase().includes(query));
rows.forEach((row, i) => {
  row.style.display = matches[i] ? '' : 'none';
});
```

`textContent` is not a drop-in semantic replacement for `innerText`: it includes hidden text and text in script and style elements. In this table, the searchable row content is text in its cells. That distinction matters if the page changes.

## What I measured

| Check | Result |
|---|---:|
| Original filter, `re` | Timed out after 20 seconds |
| Updated live filter, `re` | 7.9 ms, 2,394 visible rows |
| Database count for the same query | 2,394 rows |
| Focused regression tests | 3 passed |

The browser test also checked uppercase input, an empty query, and a query with no matches. The timing measures the synchronous filter handler in one browser run; it does not establish rendering time, tail latency, or performance on other machines. I did not capture a browser performance trace, so I cannot quantify how much of the improvement came from the property change versus batching.

## A separate server issue

A redirect request timed out after the filter test. An idle browser connection could hold the single-threaded server, leaving another request waiting. I switched from Python's [`HTTPServer` to `ThreadingHTTPServer`](https://docs.python.org/3/library/http.server.html#http.server.ThreadingHTTPServer) and added a test that holds one connection open while requesting a redirect through another. The redirect and missing-link tests passed too.

The practical lesson: measure the server and the browser separately. A fast page response does not mean that interacting with the page is fast.

## Source

- [MDN: `textContent` and `innerText`](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent#differences_from_innertext)
- [web.dev: Avoid layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing)
- [Python: `ThreadingHTTPServer`](https://docs.python.org/3/library/http.server.html#http.server.ThreadingHTTPServer)
