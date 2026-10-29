# Writing the review as review.html

The HTML file contains the review in the structure and order given under
"Output format" in [../SKILL.md](../SKILL.md). It must be a
single, self-contained file: no external stylesheets, scripts, fonts or
images.

## Building the file

1. Copy [../assets/review-head.html](../assets/review-head.html) verbatim to
   the start of `review.html` (in the current working directory, unless the
   user names another location). It holds the doctype, the `<head>` with the
   CSS, and the opening `<body>`. Do not change the style; the user adjusts
   it in that file. The `<title>` may be changed to name the project.
2. Append the review content.
3. End the file with `</body>` and `</html>`.

## Content markup

Use only simple tags, so the text is easy to read:

- `<h1>` for the title (once). The opening paragraph comes right after the
  `<h1>`, without a heading.
- `<h2>` for the sections: "Review notifications", "Comments", "AI
  speculations (often correct, sometimes completely nonsense)", "Possible errors
  reported by tools" and "Details".
- `<h3>` for each finding (number and the one-line heading), for the
  "Minor comments" and "Minor speculations" headings, and for subsections
  of the details. The one-line minor items under those headings form a
  `<ul>`.
- `<p>`, `<ul>`/`<ol>`/`<li>`, `<strong>`, `<em>`.
- `<code>` for identifiers and `path:line` references; `<pre>` (optionally
  with `<code>` inside) for code and constraint snippets. Inside `<pre>`,
  mark the problematic part with `<strong class="punch">` (red bold), e.g.
  `<strong class="punch">ST_START: state &lt;= ST_NEXT;</strong>`.
- `<table>`/`<tr>`/`<th>`/`<td>` only where the text format calls for a
  table (e.g. one fact that disagrees across several files).
- `<a href="...">` for links (see below). The `<base target="_blank">` in
  the head opens every link in a new tab; don't add `target` attributes.

No `class` attributes except `class="punch"`, no `style` attributes, no
`<div>`/`<span>` layout, no scripts.
Escape `<`, `>` and `&` as `&lt;`, `&gt;` and `&amp;` everywhere, including
inside `<pre>` (HDL and Tcl code is full of them).

## Linking to 01signal.com pages

The phase files tag their text with the 01signal.com pages it draws on
(`[01S6]` etc.). The tags and their pages are listed in
[01signal-pages.md](01signal-pages.md). Link only to pages from that list;
don't add pages or URLs from general knowledge.

- **Review notifications and Comments** link to the pages tagged on the
  text the finding is based on.
- **AI speculations** link to the listed pages whose tagged text deals with
  the same subject as the finding, when they are relevant: the page gives
  the reader background on the subject, even though it doesn't back the
  finding. Don't phrase the link as if the page confirms the finding.

Put the link on the words in the finding that the page explains, as on any
web page. Don't use reference numbers or a reference list, and don't show
the `01S` codes. For example:

```html
<p>Is <code>rst_n</code> released synchronously to <code>clk</code>? If not,
some flip-flops may see the first clock edge and others not, and the
<a href="https://www.01signal.com/verilog-design/reset/asynchronous-pitfalls/">one-hot
state machine can end up with two active states</a>.</p>
```
