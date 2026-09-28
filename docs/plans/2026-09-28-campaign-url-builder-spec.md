# Campaign URL Builder — minimal spec

Reference: https://ga-dev-tools.google/campaign-url-builder/
Source of truth: `googleanalytics/ga-dev-tools`, `src/components/CampaignURLBuilder/Web/*` (Apache-2.0).

## Purpose

Single page. User enters a website URL plus UTM values; the page rebuilds a tagged URL on every keystroke. No submit button, no backend.

## Scope

In: the Web tab. Out: Play and iOS tabs, the UA/GA4 version switch, bit.ly shortening (optional, section 7).

## 1. Inputs

All text fields, rendered in this order. Field state persists in `localStorage` and is restored on load.

| # | Label | Required | Query param | Helper text |
|---|-------|----------|-------------|-------------|
| 1 | website URL | yes | (base) | The full website URL (e.g. `https://www.example.com`) |
| 2 | campaign ID | no | `utm_id` | The ads campaign id. |
| 3 | campaign source | yes | `utm_source` | The referrer (e.g. `google`, `newsletter`) |
| 4 | campaign medium | yes | `utm_medium` | Marketing medium (e.g. `cpc`, `banner`, `email`) |
| 5 | campaign name | no | `utm_campaign` | Product, promo code, or slogan (e.g. `spring_sale`) One of campaign name or campaign id are required. |
| 6 | campaign term | no | `utm_term` | Identify the paid keywords |
| 7 | campaign content | no | `utm_content` | Use to differentiate ads |

Note: "One of campaign name or campaign id are required" is helper text only. The gate in section 3 does not enforce it.

## 2. Reverse parse on paste

When the website URL field changes, parse it for existing `utm_*` values and copy any found into the matching fields. Rules:

- Accept a missing protocol (`example.com?...` is treated as `https://example.com?...`).
- Read params from the query string, from the fragment (`#utm_source=x`), and from a query that follows the fragment (`#abc?utm_source=x`). A fragment value wins over a query value for the same key.
- Ignore empty values (`?utm_source` sets nothing).
- Only overwrite a field when the param is present. Fields are never cleared by this step.
- If the string does not parse as a URL, do nothing to the other fields.

## 3. Output gate

Show the generated URL only when website URL, campaign source, and campaign medium are all non-empty. Otherwise show, with an error icon:

> Fill out required fields above and a URL will be generated for you here.

Warnings replace the output entirely when any applies to the website URL:

| Condition | Message |
|-----------|---------|
| Not a parseable URL | The website URL provided is not a valid URL. |
| Hostname is the tool's own domain | You are linking to this site (`<own host>`) instead of your own. You should put your own site's URL in the Website URL field, above. |
| Hostname `play.google.com` | It appears that you are creating a Google Play Store url. You should use the Google Play URL Builder instead… |
| Hostname `itunes.apple.com` | It appears you are creating an iOS App Store URL. You should use the iOS Campaign Tracking URL Builder instead… |

## 4. URL assembly

Input: website URL string `W`, the six param values, boolean `useFragment` (default false).

1. Split `W` into `base`, `query` (from first `?`), `fragment` (from first `#`). The fragment is parsed as params only if it contains `=`; otherwise it is an opaque anchor.
2. Delete every `utm_*` key from both the query params and the fragment params.
3. For each non-empty input, in the order `utm_source, utm_medium, utm_campaign, utm_id, utm_term, utm_content`, set it into the query params (or the fragment params when `useFragment`).
4. Serialize with `URLSearchParams` semantics: space becomes `+`, reserved characters percent-encoded.
5. Result: `base` + (`?` + query if non-empty) + (`#` + fragment params if non-empty, else the original opaque anchor if there was one).

Pre-existing non-`utm_` params in query and fragment survive untouched.

## 5. Output UI

- Read-only multiline field labelled `generated URL` with a copy icon (tooltip `Copy campaign URL.`). On copy, toast `Copied to clipboard.` for 2 s.
- Checkbox below it: `Set campaign parameters in the fragment portion of the URL (not recommended)`. Toggles `useFragment` and re-renders immediately.
- Below the form, a reference table (Parameter, Required, Example, Description) for the six params. Examples: `abc.123`, `google`, `cpc`, `spring_sale`, `running+shoes`, `logolink`.

## 6. Acceptance cases

From the upstream test suite. `p` = `{utm_source: google}` unless stated.

| Website URL | useFragment | Expected |
|-------------|-------------|----------|
| `https://example.com` | no | `https://example.com?utm_source=google` |
| `https://example.com` | yes | `https://example.com#utm_source=google` |
| `https://example.com#abc` | no | `https://example.com?utm_source=google#abc` |
| `https://example.com#abc=123` | no | `https://example.com?utm_source=google#abc=123` |
| `https://example.com#utm_source=google` | no | `https://example.com?utm_source=google` |
| `https://example.com?abc=123` | no | `https://example.com?abc=123&utm_source=google` |
| `https://example.com?utm_source=google` | yes | `https://example.com#utm_source=google` |
| `https://example.com?abc=123#def=456` | no | `https://example.com?abc=123&utm_source=google#def=456` |
| `https://example.com?utm_source=google#utm_campaign=spring_sale`, p = source+campaign | no | `https://example.com?utm_source=google&utm_campaign=spring_sale` |

Reverse parse: `https://example.com?utm_source=google&utm_medium=cpc&utm_campaign=spring_sale&utm_term=running+shoes&utm_id=abcdef&utm_content=logolink` fills all six fields, with term decoded to `running shoes`. Same input with `#` instead of `?` gives the same result.

## 7. Optional: shorten link

Button `Shorten link`, enabled only when a URL is generated and no short link exists yet. Flow: bit.ly OAuth in a popup (600×700), then `POST https://api-ssl.bitly.com/v4/shorten` with `{long_url, group_guid}`. Result goes into a disabled field labelled `shortened URL` (placeholder `Click shorten link to shorten your generated URL.`) with its own copy icon. Any change to the generated URL clears the short link. Errors surface as `There was an error with shortening your URL: <message>`; refuse input that starts with `https://bit.ly`, is empty, or targets localhost / 127.0.0.1.

## Assumptions

- The live root page uses the GA4 gate (URL + source + medium). The repo source routes the root path to the legacy Universal Analytics variant, whose gate is URL + source only and which marks campaign name with an asterisk. Google's GA4 help page points to the root URL, so GA4 behavior is specified here. Assembly logic is identical in both.
- The live page could not be fetched from this environment (egress blocked). Strings above are from source at commit `14217f4`, last builder change 2022.
