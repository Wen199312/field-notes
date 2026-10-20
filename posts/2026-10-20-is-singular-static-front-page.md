---
title: "is_singular() is true on /page/2/ when your front page is a static page — and that silently kills your pagination rules"
date: 2026-10-20
area: wordpress, seo, query-conditions
status: verified
---

# `is_singular()` is true on `/page/2/` when your front page is a static page

## Symptom

A WordPress site shows 12 posts per page on the front page, so it also publishes
`/page/2/` through `/page/11/`. Those eleven URLs are exact duplicates of `/`:
same title, same 12 posts, same everything. They were being crawled and indexed.

The goal: `noindex, follow` on `/page/N/`, and nothing else touched.

## The obvious fix

Guard with `is_paged()`, and use `is_singular()` to avoid stamping `noindex` onto
genuinely paginated posts — the ones that use `<!--nextpage-->` to split a long
article into `/post-name/2/`:

```php
add_action( 'wp_head', function () {
    if ( ! is_paged() || is_singular() ) {
        return; // not pagination, or a paginated post — leave it alone
    }
    echo '<meta name="robots" content="noindex, follow" />' . "\n";
} );
```

This reads correctly. It is the shape most snippets you will find use. It does
nothing.

## Why it does nothing

The site's front page is a **static page** — `page_on_front` is set to a page ID,
not the posts index. That single configuration detail changes what the main query
*is* for `/page/2/`.

Two different questions get conflated here:

- **What does this URL look like?** An archive page: page 2 of something.
- **What query did WordPress run to serve it?** A *singular page* query for the
  static front page, with an offset applied.

`is_paged()` answers the first one only in appearance. It is true — there *is* a
pagination component in the URL. But `is_singular()` is also true, because the
queried object is a page. So the guard `is_singular()` fires, the function returns
early, and no meta tag is ever printed.

The condition matrix, **measured on real requests** (see the probe at the end):

| Condition | `/` | `/page/2/` |
|---|---|---|
| `is_front_page()` | `true` | `true` |
| `is_home()` | `false` | `false` |
| `is_paged()` | `false` | **`true`** |
| `is_singular()` | `true` | **`true`** |
| `is_singular( 'post' )` | `false` | `false` |
| `is_page()` | `true` | `true` |
| `is_archive()` | `false` | `false` |
| `get_queried_object_id()` | 591 | 591 |
| **main query `found_posts`** | **1** | **1** |

That last row is the tell. The main query returned **one row** on `/page/2/` — the
static page itself — not twelve posts. The twelve posts on screen come from a
separate query the theme runs; they are not the main loop. So `/page/2/` is not an
archive of posts at all. It is the front page, served at a paged URL, with the
same queried object as `/`.

Both `is_paged()` and `is_singular()` true at the same time is not a bug. They
describe different aspects of the same request. The bug is assuming they are
mutually exclusive.

**Verified:** with the guard as written, no `noindex` appeared on any of
`/page/2/` … `/page/11/`. Changing only the guard — nothing else — made all ten
take effect. That isolates the condition as the cause.

## The fix

Narrow the exemption from "any singular" to "a singular **post**":

```php
add_action( 'wp_head', function () {
    if ( ! is_paged() || is_singular( 'post' ) ) {
        return;
    }
    echo '<meta name="robots" content="noindex, follow" />' . "\n";
} );
```

`is_singular( 'post' )` is false on the paginated front page (the object is a
`page`, not a `post`) and true on a `<!--nextpage-->`-paginated article. That is
exactly the exemption the rule needs.

**Verified:** `/page/2/` through `/page/11/` now return
`<meta name="robots" content="noindex, follow">`. The homepage, category
archives, and product pages still return `index, follow`.

## Check the duplicate claim before suppressing the index

`noindex` on a page that turns out to be unique is a real loss, and it is easy to
convince yourself a paginated URL is duplicate without looking. So compare the
rendered output first:

```bash
for u in / /page/2/ /page/3/; do
  curl -s "https://example.com$u" | sha256sum
done
```

Identical hashes across the paged URLs and `/` means they are byte-identical
duplicates and suppressing them costs nothing. Different hashes mean you have
something to preserve — investigate before adding a rule.

On this site the check was conclusive: same `<title>`, same 12 post links, and
zero `page-numbers` elements in the pagination block. `/page/2/` was the homepage
with a different URL.

## Both directions, always

A rule that excludes everything also reports zero errors. Assert all three:

1. the targets are `noindex, follow` — `/page/2/` … `/page/11/`
2. the homepage is still `index, follow` — `/`
3. unaffected archives are still `index, follow` — a category and a product page

Only after all three are observed is the rule doing what was asked rather than
something adjacent to it.

## Getting the condition values on your own site

Do not try to get these from `wp eval`. It looks like it should work:

```bash
# WRONG — every condition comes back false
wp eval 'var_dump(is_paged(), is_singular());' --url=https://example.com/page/2/
```

`wp eval` boots WordPress but **never runs the main query**, so every `is_*()`
conditional evaluates against an empty `$wp_query` and returns `false`. The output
looks like a clean answer and is completely meaningless. (I wrote this snippet
first, ran it, got all-false for `/`, `/page/2/` and `/page/3/`, and only then
noticed the tell: `get_queried_object_id()` was `0` on every URL.)

What works is a temporary mu-plugin that records the conditions during a **real
front-end request**:

```php
// wp-content/mu-plugins/cond-probe.php — delete after use
add_action( 'wp', function () {
    if ( empty( $_GET['condprobe'] ) ) { return; }
    if ( '127.0.0.1' !== ( $_SERVER['REMOTE_ADDR'] ?? '' ) ) { return; }

    $q = $GLOBALS['wp_query'];
    $row = sprintf(
        "uri=%s | front_page=%s home=%s paged=%s singular=%s singular_post=%s page=%s archive=%s | qid=%d | found_posts=%d\n",
        $_SERVER['REQUEST_URI'],
        var_export( is_front_page(), true ),
        var_export( is_home(), true ),
        var_export( is_paged(), true ),
        var_export( is_singular(), true ),
        var_export( is_singular( 'post' ), true ),
        var_export( is_page(), true ),
        var_export( is_archive(), true ),
        get_queried_object_id(),
        $q instanceof WP_Query ? (int) $q->found_posts : -1
    );
    @file_put_contents( '/tmp/cond.log', $row, FILE_APPEND | LOCK_EX );
}, 99 );
```

Then hit the URL you care about with the flag and read the log:

```bash
curl -s "https://example.com/page/2/?condprobe=1" -o /dev/null
cat /tmp/cond.log
```

Three details that make the probe trustworthy:

- **Hook `wp`, not `init`.** `init` runs before the main query is parsed; the
  conditions are not populated yet.
- **Gate on `REMOTE_ADDR`.** Otherwise anyone can trigger log writes by adding the
  query parameter.
- **Log `found_posts` from the main query.** It is the difference between "this URL
  is an archive of posts" and "this URL is a single page", and it is not visible
  from the condition booleans alone — as the table above shows, four conditions
  agree between `/` and `/page/2/`.

Delete the mu-plugin afterwards. `php -l` it before copying it in; a broken
mu-plugin takes the whole site down.

## The general lesson

`is_*()` conditionals answer **"what query is this?"**, not **"what does this URL
look like?"**. The two diverge whenever a URL that reads like one thing is served
by a query of another kind. A static front page is the most common case in
WordPress, but not the only one: a page assigned as the posts page, a static page
serving a search-like route, a CPT archive with a rewrites rule that shadows a
page.

The pattern to watch for is a guard whose condition is **broader than intended**.
`is_singular()` was chosen to mean "a paginated article"; it actually means "any
single object", which on this site included the paginated homepage. Guards fail
open — they let things through or block things that should have been handled —
and the failure is invisible because the code returns early without an error.

When a conditional rule "doesn't fire", dump every relevant `is_*()` value for the
exact URL before touching the logic. The logic is usually right. The question is
whether it was asked about the right thing.

---

*Verified on WordPress 7.1.x with a static page as the front page and twelve posts
per page. Condition values and entry behaviour above are from that site.*
