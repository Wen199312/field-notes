---
title: "RankMath sitemap exclusion: a filter that passes in isolation but does nothing in production"
date: 2026-10-19
area: wordpress, seo, rankmath
status: verified
---

# RankMath sitemap exclusion: a filter that passes in isolation but does nothing in production

## Symptom

A WordPress site publishes a daily market-dashboard post. After a year there were
45 of them, all near-duplicates of each other, all listed in `post-sitemap.xml`
next to 9 evergreen pages that actually deserve the crawl budget.

The goal was narrow, and worth stating precisely because it rules out the usual
advice:

- keep the daily posts **readable** and **internally linked** — they still rank
  for long-tail date queries
- remove them from the **sitemap**
- do **not** delete them
- use `noindex, follow`, not `nofollow`, so link equity keeps flowing

## The obvious fix

RankMath documents a `rank_math/sitemap/entry` filter. Return an empty string and
the entry is skipped:

```php
add_filter( 'rank_math/sitemap/entry', function ( $url, $type, $post ) {
    if ( 'post' === $type && my_is_daily_slice( $post ) ) {
        return '';
    }
    return $url;
}, 10, 3 );
```

Called directly, it behaves exactly as advertised:

```php
$url = apply_filters( 'rank_math/sitemap/entry', 'https://example.com/daily-1/', 'post', $post );
var_dump( '' === $url );    // bool(true)
var_dump( empty( $url ) );  // bool(true)
```

Both assertions pass. It looks done.

## The sitemap does not change

Rebuild the sitemap: still **116 entries**, all 45 daily posts present, unchanged.

Eliminate the caching layer as a suspect — the XML cache files live under
`wp-content/uploads/rank-math/` and are sometimes not purged, because in a
containerised WordPress `Helper::get_filesystem()` can resolve to
`WP_Filesystem_ftpsockets` instead of Direct, at which point `delete()` silently
does nothing. So: delete those files directly, regenerate, re-count.

**Verified:** 116 entries before, 116 after. Caching is not the cause.

## Why it does nothing

`rank_math/sitemap/entry` fires late. It is a **formatting** hook — it runs inside
the loop that builds each `<url>` node, after the set of posts has already been
decided by an earlier query plus an indexability gate:

```php
foreach ( $posts as $post ) {
    if ( ! Sitemap::is_object_indexable( $post->ID ) ) {
        continue;
    }
    $url = apply_filters( 'rank_math/sitemap/entry', $url, 'post', $post );
    // ... render <url> node
}
```

So the filter can only reformat rows that already survived. It cannot remove a row
from the set.

**Verified:** a debug probe inside the callback never fired during a real sitemap
generation for any of the 45 target posts, while the identical callback returned
`''` when invoked through `apply_filters()` by hand. The callback was registered
correctly; the generation path simply never reached it for these posts.

That is the part worth internalising: *the function works, it just never runs.*
Debugging its logic would have been wasted effort.

## The fix that works

Set membership has to be changed earlier — before any entry is rendered:

```php
add_filter( 'rank_math/sitemap/posts_to_exclude', function ( $ids ) {
    $mine = get_posts( [
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'fields'         => 'ids',
        'posts_per_page' => -1,
        'no_found_rows'  => true,
    ] );
    $mine = array_filter( $mine, 'my_is_daily_slice' );
    return array_merge( (array) $ids, $mine );
} );
```

This hook participates in building the post set itself, so the excluded posts are
never candidates in the first place.

**Verified:** 116 → **71 entries**. Daily posts present: **0**. The 9 evergreen
pages all still present.

Keep `rank_math/sitemap/entry` as a second layer if you like — it costs nothing —
but do not rely on it as the only mechanism.

## Verify the artifact, not the intent

The verification that matters is a count of the file the crawler actually reads:

```bash
curl -s https://example.com/post-sitemap.xml | grep -c '<loc>'
# 71
```

These prove nothing:

- the filter returns the right value
- the callback is registered
- the code reads correctly
- `apply_filters()` in a scratch file gives the expected output

All four can be true while the sitemap is unchanged. That was the entire failure
mode here.

## Two guards worth adding

**1. A whitelist that outranks every rule.** Slug patterns are greedy and pages
get renamed. A hard list protects the pages you cannot afford to lose:

```php
$whitelist = [ 'evergreen-one', 'evergreen-two' ];
if ( in_array( $post->post_name, $whitelist, true ) ) {
    return false; // never a slice, whatever the regex says
}
```

**2. Assert both directions.** Check that the excluded count is 0 *and* that each
whitelisted URL is still in the sitemap. Excluding everything also gives you 0.

## The general lesson

A WordPress filter has two lives:

1. as a function you can call
2. as a hook the framework actually reaches

`apply_filters()` in isolation proves only the first. When a filter "doesn't
work", check **whether it runs at all** — a probe that writes to the error log is
five lines — before rewriting its logic. Late-stage filters can be structurally
incapable of the change you are asking for, and no amount of correct logic fixes
that.

The same pattern shows up elsewhere in RankMath: `is_object_indexable()` gates
before `sitemap/entry` the way `is_singular()` gates before pagination checks in
core WordPress. When a hook seems inert, look for the gate that runs first.

---

*Verified on WordPress 7.1.x with RankMath on a site publishing 45 machine-generated
daily posts. Entry counts and hook behaviour above are from that site.*
