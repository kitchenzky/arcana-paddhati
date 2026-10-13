# Contents fix preflight: stopped on a contradicted cause

Base: `55ffc49`. Branch: `fix/toc-reading-place-pr2`.

The supplied `zayavka-codex-pr2-fixes.md`, section 4, requires stopping and
reporting if measurements do not confirm a displacement after `revealOpened`
sets the list position. Section 5 also requires reporting when programmatic
scrolling emits a scroll event, contradicting the proposed missed-event race.
These conditions occurred before any application changes.

## Method

Baseline static export, the existing QA server and canonical Playwright
`iphone14` (WebKit) and `pixel7` (Chromium) profiles, blocked service workers,
and real synthetic touches through `qa/lib/toc.py`'s Contents fixture.
The observer recorded native scroll events, scrollTop setter calls, geometry
after DOM mutations, the existing 120 ms callback, its next animation frame,
and the settled state. It did not change application state or event behavior.
Measurements used `.toc-scroll`, the actual list scroller, rather than its
non-scrolling parent `nav`.

## Chapter visibility: different cause confirmed

Start at chapter 4 subsection 4.11, read to 4.14, open Contents, scroll chapter
5 into view, and touch its collapsed row.

| Measurement | iPhone 14 | Pixel 7 |
| --- | ---: | ---: |
| Visible list top | 180.25 px | 180.25 px |
| Chapter 5 row before touch | 742.25–786.25 px | 807.25–851.25 px |
| Chapter 5 row after settling | 82.25–126.25 px | 153.25–197.25 px |
| List scrollTop before / after | 684 / 596 px | 619 / 525 px |

A focused iPhone repeat found chapter 5 already at 82.25–126.25 px in the
first DOM-mutation observation. It stayed at those exact coordinates before
and after the 120 ms callback, in the following animation frame, and at one
second. There were **no list scrollTop setter calls** during that opening.
Thus this was not a successful reveal followed by a later layout displacement.

`components/Sidebar.tsx:449` returns as soon as the branch bottom fits below
the list bottom. It does not check the chapter row's top. Once the DOM change
and native scroll anchoring leave the row above the list, both reveal calls
return without correcting it. The visible list starts below the fixed cover
row; this measurement does not demonstrate a header overlapping the actual
`.toc-scroll` rectangle.

Opening chapter 6 after chapter 5 in this sequence left its row visible:
308.22–352.22 px on iPhone and 379.25–423.25 px on Pixel. This confirms the
reported chapter-5 screenshot case; it does not establish that every chapter-6
sequence passes.

## Stale subsection mark reproduced

Deep-link to `#11-gandha`, touch the book, scroll to `14-dipa`, settle, then
open Contents. On both devices the footer read **Глава 4.14**, but Contents
marked **sub:arcana-procedure#11-gandha**. Reading scrollTop was 94531 px on
the focused iPhone repeat and 90907 px on Pixel. The hash retained 4.11.
This is consistent with the hash-induced lock described in section 3.

## Back: restored indicators correct, marked row invisible

Fresh deep-link to 4.11, open Contents, touch chapter 2, and use native Back.
After restoration and an additional 1.1 seconds, both devices displayed
**Глава 4.11** and marked **sub:arcana-procedure#11-gandha**. The reported
iPhone chapter-2 footer/mark mismatch did not reproduce in this sequence.

| Measurement | iPhone 14 | Pixel 7 |
| --- | ---: | ---: |
| Restored reading scrollTop | 93000 px | 89504 px |
| Marked row top / bottom | 1103.25 / 1147.25 px | 1116.25 / 1160.25 px |
| Visible list bottom | 787 px | 858 px |

The final restore setter ran about 602 ms after popstate on each phone. Native
main-scroll events were recorded: on iPhone about 18 ms and 55 ms after
popstate; on Pixel about 5.5 ms after popstate. Later setters repeated the
already-restored offset. Timer ordering alone therefore does not establish
the proposed stale-indicator cause. The off-screen marked row did reproduce
on both phones: restoring `apNav` returns before `revealLit` is considered.

## Validation and remaining work

`npm run build` passed. No application, rules, suite, assertion, threshold or
SKIP was changed. The close/backdrop jump measurements, fixes, rule 9.13,
new s37 suite, full gate, commit, push and staging PR remain unperformed
because the document explicitly requires this stop. The existing untracked
`Claude outputs/` directory was preserved.
