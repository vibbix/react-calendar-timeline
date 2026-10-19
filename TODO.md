# Upstream TODO

Found while building primary-backstage on `feature/all-upstream-fixes` (verified at `4ec06bb`). Nothing here is filed or started.

| # | Item | Type | Size |
|---|---|---|---|
| 1 | Epoch `0` treated as a missing time | PR | XS |
| 2 | Item `height` scaled by `itemHeightRatio` | PR | XS |
| 3 | Headers/markers inside a fragment ignored | PR | S |
| 4 | Wrapped `SidebarHeader`/`TimelineHeaders` not recognised | PR | S |
| 5 | Props types fight wrapper components | PR | S–M |
| 6 | Pan/zoom bounds | Issue | M |
| 7 | Time zone support | Issue | M (dayjs tz) / L (Temporal) |
| 8 | Accessibility | Issue | L–XL |
| 9 | Section rows | Issue | M–L |

## Small fixes

### 1. Epoch `0` treated as a missing time

- **Problem:** `defaultTimeStart={0}` throws "You must provide either...", because the checks use truthiness. Relative or duration timelines that start at 0 can't initialise. A controlled `visibleTimeStart={0}` is ignored for the same reason.
- **Evidence:** `src/lib/Timeline.tsx:282` and `:285` (constructor), `:364` (`getDerivedStateFromProps`).
- **Change:** compare with `!= null` instead of truthiness in all three places, and add tests for `0` in both the default and the controlled path.
- **Size:** XS, about 3 lines plus tests.

### 2. Item `height` scaled by `itemHeightRatio`

- **Problem:** the `height` doc says "Fixed pixel height for this item. Overrides `lineHeight` for this item only", but the ratio still applies. With the default `0.65`, an 18px item renders at 11.7px.
- **Evidence:** `src/lib/utility/calendar.tsx:635`; doc at `src/lib/types/main.ts:42`.
- **Change:** apply `itemHeightRatio` only when falling back to `lineHeight`. This changes behaviour for anyone who compensated for the bug, so it needs a CHANGELOG note.
- **Size:** XS, 1 line plus a test.

### 3. Headers and markers inside a fragment are ignored

- **Problem:** `React.Children.map` doesn't look inside fragments. So `<>{headers}{markers}</>` makes the library render its default headers with no error, and the fragment is cloned with timeline props, which React warns about. Inside `TimelineHeaders`, a `SidebarHeader` in a fragment ends up among the calendar headers.
- **Evidence:** `src/lib/Timeline.tsx:858` (`isTimelineHeader`), `:901` (`childrenWithProps`), `:913` (`renderHeaders`); `src/lib/headers/TimelineHeaders.tsx:43` and `:53`.
- **Change:** a small helper that recursively unwraps `React.Fragment` children, used in those three places, with tests for headers, markers and sidebar headers inside fragments.
- **Size:** S, about 20 lines plus tests.

### 4. Wrapped `SidebarHeader` / `TimelineHeaders` not recognised

- **Problem:** children are matched by `type.secretKey`, so a component that renders `<SidebarHeader>` isn't recognised. It isn't dropped either: it renders among the calendar headers, and the default sidebar header is added as well. The same applies to wrapping `TimelineHeaders`.
- **Evidence:** `src/lib/headers/TimelineHeaders.tsx:43`; `src/lib/Timeline.tsx:858`; the keys are set at `src/lib/headers/SidebarHeader.tsx:68` and `src/lib/headers/TimelineHeaders.tsx:122`.
- **Workaround today (undocumented):** `MyHeader.secretKey = SidebarHeader.secretKey`. The static is reachable from the exported component, but its type may need a cast.
- **Change:** document the `secretKey` contract and type it, which is the smallest fix. The alternative is matching through a marker prop or context, which is bigger and changes behaviour.
- **Size:** S.

### 5. Props types fight wrapper components

- **Problem:** `ReactCalendarTimelineProps` marks defaulted props as required, so wrappers have to use `JSX.LibraryManagedAttributes<typeof Timeline, ...>`. Those props are `keys`, `sidebarWidth`, `rightSidebarWidth`, `lineHeight`, `itemHeightRatio`, `minZoom`, `maxZoom`, `stackItems` and `timeSteps`. It also declares `ref` on a class component's props, which breaks spreading props through a function-component wrapper (`Omit<..., 'ref'>` is needed).
- **Evidence:** `src/lib/Timeline.tsx:70-108` (props type), `:134` (`ref`), `:170-221` (`defaultProps`).
- **Change:** make the defaulted props optional in the exported props type, keep a resolved internal type for the class, and remove `ref` from the props. This is a type-only change, but it could break consumers who rely on the current shape.
- **Size:** S–M, about 30 lines plus a type test.

## Features (raise as issues first)

### 6. Pan/zoom bounds

- **Problem:** nothing limits scrolling or zooming to a time range, so consumers clamp inside `onTimeChange` themselves (primary-backstage has a `clampTimelineWindow` helper for this).
- **Evidence:** no bounds handling anywhere in `src/lib`.
- **Proposal:** `minTime`/`maxTime` (or `canvasBounds`) props respected by scrolling, zoom and `onTimeChange`.
- **Size:** M.

### 7. Time zone support

- **Problem:** everything runs in browser-local time, so header intervals and gridlines can't follow a chosen time zone. Drag snapping uses the browser's offset now (`dayjs().utcOffset()`), not the offset at the time being snapped, so it's an hour off across a DST change.
- **Evidence:** `src/lib/Timeline.tsx:972`, `src/lib/items/Item.tsx:212`; the date stepping in `iterateTimes` at `src/lib/utility/calendar.tsx:70`.
- **Proposal:** a `timeZone` prop using the dayjs `timezone` plugin, which is non-breaking and about a day of work. A larger alternative is replacing dayjs with Temporal, which breaks `Dayjs` in the public types and the format-string headers.
- **Size:** M for dayjs tz, L for Temporal.

### 8. Accessibility

- **Problem:** there are no `role`, `aria-*`, `tabIndex` or keyboard handlers anywhere in the source. Items aren't focusable or labelled, and rows have no semantics. primary-backstage works around this with a visually hidden per-lane list and its own `<a>` for linked items.
- **Evidence:** a search of `src/lib` finds none.
- **Proposal:** grid/row roles, an item `aria-label` (from the title or a key), focusable items with arrow-key navigation, and an accessible-name prop.
- **Size:** L–XL. This is the largest item and should be designed with the maintainers.

### 9. Section rows

- **Problem:** there's no built-in heading or section row that spans both sidebars and the canvas. primary-backstage fakes one with heading groups plus `horizontalLineClassNamesForGroup`, and the title only shows in the left sidebar.
- **Evidence:** no section concept in the group types (`src/lib/types/main.ts`) or in `Sidebar`/`GroupRow`.
- **Proposal:** a group flag (`isSection` or `header: true`) that renders full width across the sidebars and canvas, with its own height and class.
- **Size:** M–L.

## Suggested order

1–3 are the cheapest and help every user, and each is its own PR. 4 and 5 are small but change public API or type contracts, so an issue comment first wouldn't hurt. 6–9 need agreement on the design first. For primary-backstage, 1–5 remove about 10 lines and 6–9 about 100.
