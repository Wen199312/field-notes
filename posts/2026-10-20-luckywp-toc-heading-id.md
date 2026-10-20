---
title: "LuckyWP TOC ignores the id you put on your heading — and injects its own"
date: 2026-10-20
area: wordpress, plugins, luckywp-toc
status: verified
---

# LuckyWP TOC ignores the `id` you put on your heading

## Symptom

A long article with Chinese headings. The table of contents generates anchors by
transliterating the heading text:

```
#jie_lun_xian_xing
#wei_shen_me_di_bi_te_mo_xing_zhi_de_guan_zhu
#suan_nei_cun_zhang_bu_yao_kan_wen_jian_da_xiao
```

Functional, and useless as a URL fragment. Nobody links to `#jie_lun_xian_xing`,
and it tells a reader nothing about where they will land. The anchors should be
`#quick-vram-formula`, `#model-weight-formats`, and so on.

## The obvious fix does nothing

Add the ids directly to the headings in the post content:

```html
<h2 id="quick-vram-formula">结论先行</h2>
<h3 id="model-weight-formats">看清手上的文件是哪种格式</h3>
```

Reload. The TOC still links to `#jie_lun_xian_xing`.

The heading now has the id you asked for, and the table of contents does not use
it. Both statements are true at once, which is what makes this confusing: nothing
errors, nothing warns, and the anchor you wrote is present in the DOM — it just
is not the one the TOC points at.

## Why: the plugin never reads the node

LuckyWP's content handler walks the DOM, and for each heading calls:

```php
$label = static::makeLabel( $node );              // heading text, tags stripped
if ( $label == '' ) { continue; }

$id = static::makeHeadingId( $label, $dto );      // ← only the label goes in
```

`makeHeadingId( $label, $dto )` builds the anchor **from the text**, then
transliterates it:

```php
$id = $label;
$id = transliterator_transliterate( 'Any-Latin; Latin-ASCII;', $id );
$id = preg_replace( '/[^\d\-_A-Za-z ]+/', '', $id );
$id = str_replace( ' ', '_', $id );
$id = trim( $id, '-_' );
```

(`lwptoc_transliterator` lets you change the transliteration rule, and
`lwptoc_heading_id` — the hook that matters here — runs right after this.)

The `$node` never enters that function. The heading's existing `id` attribute is
never passed, never read, and therefore cannot influence the result. The plugin is
not "preferring its own id over yours"; it has no path by which it could see yours.

## And then it injects a second anchor

Immediately after computing the id, and **unconditionally**:

```php
$span = $dom->createElement( 'span' );
$span->setAttribute( 'id', $id );
while ( $node->childNodes->length ) {
    $span->appendChild( $node->firstChild );
}
$node->nodeValue = '';
$node->appendChild( $span );
```

There is no check for an existing id on the heading. The result for a heading you
annotated yourself is two anchors on one heading:

```html
<h2 id="quick-vram-formula">
  <span id="jie_lun_xian_xing">结论先行</span>
</h2>
```

**Measured on the live page after adding ids by hand:**

| Heading | `id` on `<h2>/<h3>` | id on injected `<span>` |
|---|---|---|
| 结论先行 | `quick-vram-formula` | `jie_lun_xian_xing` |
| 为什么低比特模型值得关注 | *(none)* | `wei_shen_me_di_bi_te_mo_xing_zhi_de_guan_zhu` |
| 看清手上的文件是哪种格式 | `model-weight-formats` | `jian_qing_shou_shang_de_wen_jian_shi_na_zhong_ge_shi` |

Both ids are valid HTML, on two different elements. Neither is a duplicate. But the
TOC links to the span, so your hand-written anchor is only reachable if something
else links to it. You have not fixed the TOC; you have added a second, unused
anchor beside the one it actually uses.

(Note on the observed pinyin: with `intl` available, `transliterator_transliterate`
turns Chinese into pinyin. Without `intl`, `remove_accents()` leaves Chinese intact
and the subsequent `[^\d\-_A-Za-z ]` strip removes it entirely, so every Chinese
heading collapses to the `lwptoc` fallback and gets a numeric suffix — *inferred
from the code path; not reproduced on this host, which has `intl`.* Pinyin in your
anchors therefore tells you `intl` is installed.)

## The fix: filter the derived id

`lwptoc_heading_id` receives the computed id and the label it came from:

```php
add_filter( 'lwptoc_heading_id', function ( $id, $label ) {
    $map = [
        '结论先行'                 => 'quick-vram-formula',
        '看清手上的文件是哪种格式' => 'model-weight-formats',
        '算内存账，不要看文件大小' => 'vram-calculation-rules',
        '这套判断方法'             => 'hardware-checklist',
    ];
    return $map[ $label ] ?? $id;
}, 10, 2 );
```

Two things to know about the arguments:

- **`$label` is the stripped text**, after `wp_strip_all_tags()` and
  `html_entity_decode()`. If a heading contains `<code>`, an emoji, or inline
  markup, the key you must match is the text with all of that removed. Match on the
  visible text, exactly, including punctuation and spacing.
- **The filter runs before de-duplication.** Right after it, the plugin checks
  whether the id is already taken and appends a counter (`$baseId . $c`) if so. Two
  headings whose labels map to the same value will silently become `x` and `x1`.

A word of warning about the mapping approach: it is keyed on heading text, so
renaming a heading silently drops it back to the computed id. If the headings are
generated from a template, prefer deriving the id from a stable attribute rather
than the label — or accept that the map needs updating alongside the content.

## Verify by walking the rendered DOM

The check that proves this works is not "does the heading have the id I wrote" —
it obviously does, and that was never the problem. The check is **whether every
TOC link target exists**, and whether the targets are the ones you want:

```python
import re, urllib.request
html = urllib.request.urlopen( URL ).read().decode()

toc     = set( re.findall( r'href="#([^"]+)"', html ) )
present = set( re.findall( r'id="([^"]+)"', html ) )

print( "TOC anchors:", len( toc ) )
print( "broken:", sorted( toc - present ) )
print( "pinyin leftovers:", [a for a in toc if re.search( r'_[a-z]', a )] )
```

**Before the filter** (mu-plugin disabled, page reloaded): 13 TOC anchors,
**0 broken**, and **12** of them pinyin — the thirteenth is a `#content` link. So
the TOC was *working*. It was pointing at anchors that existed, because it had
injected them. "The anchors are ugly" and "the anchors are broken" are different
problems, and only the second one shows up as a broken link.

**After the filter:** the same 13 anchors, 0 broken, pinyin down to **8**. The four
mapped headings now resolve to `#quick-vram-formula`, `#model-weight-formats`,
`#vram-calculation-rules`, `#hardware-checklist`.

Assert all three lines, not just the broken count. A filter that returns a constant
gives you zero broken links too.

## Two related traps

**`modify = false` makes the anchors disappear.** The span injection is gated on
the `modify` setting. Turn it off (some setups do, to keep the post content
byte-identical) and there is no injected `<span>` at all — at which point the TOC
links to nothing, unless your headings already carry matching ids. If you disable
`modify`, you own the ids entirely.

**`hashFormat` changes the shape before your filter runs.** `counter` gives
`lwptoc1`, `lwptoc2`; `asheadingwotransliterate` skips transliteration but replaces
entities and some punctuation. Your filter receives whichever form the setting
produced, so a hardcoded `$id` comparison is fragile — match on `$label`.

## The general lesson

When a plugin derives something from your content — an anchor, a slug, a cache key
— adding an input that *looks* like what it derives does not feed back into the
derivation. `makeHeadingId( $label )` cannot see the node, so no attribute on the
node can affect it. The plugin then writes its derived value back into the DOM as a
second anchor, and the original is orphaned.

The reliable move is to intercept the derivation at the point where the plugin
exposes it (`lwptoc_heading_id` here) rather than to annotate the input and hope it
propagates. The tell that you are in this situation: your change is visible in the
HTML but the feature you were trying to change is unaffected.

---

*Verified on luckywp-table-of-contents with `intl` available; source paths and
filter signatures read from the installed plugin. The `remove_accents()` fallback
behaviour is inferred from the code and marked as such above.*
