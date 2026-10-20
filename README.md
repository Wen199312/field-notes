# Field Notes

Short, verifiable write-ups of problems I hit while running real systems:
WordPress at scale, self-hosted infrastructure, macOS/Linux automation.

Each note follows the same shape: the symptom, the obvious fix, why it failed,
the fix that actually worked, and **how the result was verified**. Numbers in
these notes come from the actual systems; where I did not verify something, it
says so.

I write these mainly for my future self. They are public because the "obvious
fix" in most of them is what search results tell you to do, and a few of those
are wrong in ways that are hard to notice.

## Index

| Date | Note | Area |
|---|---|---|
| 2026-10-19 | [RankMath sitemap exclusion: a filter that passes in isolation but does nothing in production](posts/2026-10-19-rankmath-sitemap-exclusion.md) | WordPress / SEO |
| 2026-10-20 | [`is_singular()` is true on `/page/2/` when your front page is a static page](posts/2026-10-20-is-singular-static-front-page.md) | WordPress / SEO |
| 2026-10-20 | [LuckyWP TOC ignores the `id` you put on your heading](posts/2026-10-20-luckywp-toc-heading-id.md) | WordPress / plugins |

## Conventions

- **Verified vs. inferred.** Anything labelled *verified* was confirmed against a
  running system. Anything labelled *inferred* is my reading of the code and is
  marked as such.
- **Numbers are measured, not estimated.** If a count appears, it came from a
  command whose output is shown.
- **No product names are anonymised** (RankMath, Tailscale, LuckyWP, …) because
  the whole point is that you can reproduce the finding. Site-specific
  identifiers and infrastructure details are removed.

## License

Text is licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Code snippets are MIT.
