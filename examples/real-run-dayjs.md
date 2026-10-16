# Real run: dayjs timezone change

On October 16, 2026, the council reviewed one real change to a popular
open-source library. This page has the unedited report and the checks run
afterwards. It is one run on one change, not an accuracy benchmark.

## The change

[dayjs](https://github.com/iamkun/dayjs) (MIT) commit
[`e27ee80`](https://github.com/iamkun/dayjs/commit/e27ee8021a512b5770ffe683e010830bb9b6fb6d),
merged from [PR #3174](https://github.com/iamkun/dayjs/pull/3174) on
August 15, 2026. It replaced the offset calculation in the timezone plugin's
`.tz()` with the plugin's `tzOffset` helper, and added tests. It shipped in
dayjs 1.11.22. Two days later,
[issue #3178](https://github.com/iamkun/dayjs/issues/3178) reported that
`.tz()` threw `RangeError` for invalid dates, which broke date pickers.
1.11.23 fixed that.

## How it was run

```bash
git clone https://github.com/iamkun/dayjs.git
cd dayjs
git checkout e27ee8021a512b5770ffe683e010830bb9b6fb6d
code-review-council --base 98364bcebc04 --format json --out report.json
```

Claude Code 2.1.295 and Codex 0.162.0-alpha.17.2, both on subscription
logins with each CLI's default model, Node.js 22.23.2, macOS. This was the only run.
The reviewers had no web access. The checkout was unchanged afterwards. The
Markdown below was rendered from `report.json` with the tool's own Markdown
formatter and is not edited.

---

## Code review council: 5 confirmed findings

Reviewers: claude, codex. Diff: 3 files, +220 -3. Time: 5m 42s. A finding needs 2 reviewers to agree.

| # | Severity | Location | Finding |
| --- | --- | --- | --- |
| 1 | medium | `src/plugin/timezone/index.js:99` | Target wall times inside the host's DST gap now return the wrong instant |
| 2 | medium | `src/plugin/timezone/index.js:99` | Preserve invalid-date propagation |
| 3 | medium | `src/plugin/timezone/index.js:99` | Fix negative timestamp rounding before reusing tzOffset |
| 4 | medium | `src/plugin/timezone/index.js:100` | tzOffset returns fractional offsets for pre-1970 timestamps with milliseconds, breaking UTC detection |
| 5 | medium | `test/plugin/timezone.test.js:486` | Freeze the current date for the ambiguous-time test |

### 1. Target wall times inside the host's DST gap now return the wrong instant

`src/plugin/timezone/index.js:99`, medium. Found by claude; confirmed by codex.

\`ins\` is still built by parsing \`target\` as a host-local wall time with \`d\(target\)\`. If that wall time falls in the host's spring-forward gap, JS Date moves it forward an hour. Example: host Europe/Paris, \`dayjs\('2050-03-27T06:30:00Z'\).tz\('America/Guadeloupe'\)\` gives a target of 02:30, but \`$d\` becomes 03:30 CEST. The old formula took its offset from \`date - new Date\(target\)\`, which absorbed that shift, so \`valueOf\(\)\` still matched the original instant \(only the offset label was wrong\). The new code uses the true zone offset \(-240\) with \`utcOffset\(offset, true\)\`. \`valueOf\(\)\` then computes \`$d - \(offset + $d.getTimezoneOffset\(\)\)\`, which gives 07:30Z, an hour off from the input, and \`format\(\)\` shows 03:30. So converting a timestamp to a zone can now silently change the instant. The new tests only use Guadeloupe midnight \(04:00Z\), which is outside the Paris gap, so they don't catch this. Build the instance from UTC parts \(e.g. \`d.utc\(target\)\` / formatToParts\) instead of a host-local parse.

### 2. Preserve invalid-date propagation

`src/plugin/timezone/index.js:99`, medium. Found by codex; confirmed by claude.

\`dayjs\(null\).tz\('America/New\_York'\)\` previously returned an invalid Day.js instance; it now throws \`RangeError: Invalid time value\` because \`tzOffset\` calls \`Intl.DateTimeFormat.formatToParts\`. This breaks callers that validate missing or malformed dates with \`.isValid\(\)\` after conversion. Guard invalid instances before calling the helper.

### 3. Fix negative timestamp rounding before reusing tzOffset

`src/plugin/timezone/index.js:99`, medium. Found by codex; confirmed by claude.

The helper removes milliseconds using \`% 1000\`, which truncates negative timestamps toward zero instead of flooring them to the second formatted by Intl. Under \`TZ=UTC\`, \`dayjs\(-1\).tz\('UTC'\).valueOf\(\)\` now returns \`59999\` instead of \`-1\`. Pre-1970 timestamps with nonzero milliseconds therefore acquire incorrect offsets and change their represented instant.

### 4. tzOffset returns fractional offsets for pre-1970 timestamps with milliseconds, breaking UTC detection

`src/plugin/timezone/index.js:100`, medium. Found by claude; confirmed by codex.

\`proto.tz\` now relies on \`tzOffset\`, which strips milliseconds with \`asTS % 1000\`. For negative timestamps that remainder is negative, so \`asTS\` is rounded toward zero while formatToParts floors the seconds. The result is off by one second, giving offsets like -1/60 minute. Example: \`dayjs\(-1500\).tz\('UTC'\)\` gets offset ≈ -0.0167. \`\!Number\(offset\)\` is false, so the UTC branch is skipped. Then \`utcOffset\(-0.0167, true\)\` treats any \|value\| &lt;= 16 as hours, giving a -1 minute offset. Non-UTC zones get non-integer minute offsets and slightly wrong \`valueOf\(\)\`/\`format\(\)\`. The old code rounded these away. Floor the milliseconds properly \(e.g. \`asTS - \(\(asTS % 1000\) + 1000\) % 1000\`\) or round the offset.

### 5. Freeze the current date for the ambiguous-time test

`test/plugin/timezone.test.js:486`, medium. Found by codex; confirmed by claude.

Day.js selects the overlap offset using New York's current offset, while Moment defaults to the earlier occurrence. With the clock set to January, this input produces \`01:30-05:00\` in Day.js versus \`01:30-04:00\` in Moment. The existing \`MockDate.set\(new Date\(\)\)\` does not fix the season, so this new test makes \`npm test\` fail during standard time. Pin the clock explicitly.

---

## Checking the findings

Each finding was checked afterwards against the published npm packages, not
the reviewers' word.

| # | Finding | Result |
| --- | --- | --- |
| 2 | Invalid dates now throw | Reproduced in 1.11.22, not in 1.11.21. This is the bug reported in [#3178](https://github.com/iamkun/dayjs/issues/3178); fixed in 1.11.23. |
| 3, 4 | Negative timestamps with milliseconds get the wrong offset | One defect, reported twice in different words. Reproduced in 1.11.22 and 1.11.23, not in 1.11.21. An open community fix exists: [#3227](https://github.com/iamkun/dayjs/pull/3227). |
| 1 | Wall times in the host's DST gap return the wrong instant | Reproduced in 1.11.22 and 1.11.23 with `TZ=Europe/Paris`, not in 1.11.21. An open community fix exists: [#3219](https://github.com/iamkun/dayjs/pull/3219). |
| 5 | A new test depends on the current season | Not checked. |

Nothing was dropped for citing lines outside the diff, and no finding was left
unconfirmed. Both open fixes were still unmerged on October 16, 2026.

The checks, using `dayjs@1.11.21`, `1.11.22` and `1.11.23` from npm with the
`utc` and `timezone` plugins:

```js
dayjs(null).tz('America/New_York').isValid()
// 1.11.21: false   1.11.22: RangeError: Invalid time value   1.11.23: false

// TZ=UTC
dayjs(-1).tz('UTC').valueOf()
// 1.11.21: -1   1.11.22: 59999   1.11.23: 59999

// TZ=Europe/Paris
dayjs('2050-03-27T06:30:00Z').tz('America/Guadeloupe').valueOf()
// 1.11.21: 2050-03-27T06:30:00Z   1.11.22 and 1.11.23: 2050-03-27T07:30:00Z
```

## Limits

- One run on one change. It shows what the council reported here, not how
  often it is right. Comparative accuracy is still unmeasured; see the
  [benchmark plan](../docs/benchmark.md).
- dayjs is public and widely used, so the models may have seen related code or
  discussion during training. They had no web access during the run.
- The change was chosen because a regression from it was already documented.
