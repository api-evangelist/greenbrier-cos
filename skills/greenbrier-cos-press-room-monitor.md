---
name: greenbrier-press-room-monitor
description: >-
  Track new Greenbrier press releases and Perspectives & Updates articles — earnings, order-book
  news, facility and product announcements — from the public press room API, with an incremental
  polling pattern that does not re-read the archive.
api: Greenbrier Press Room API
base_url: https://www.gbrx.com/wp-json
operations:
  - listCategories
  - listPosts
  - getPost
generated: '2026-09-12'
method: generated
source: openapi/greenbrier-cos-press-room-api-openapi.yml
---

# Monitor the Greenbrier press room

245 published posts across 3 categories and 14 tags as of 2026-09-12 — quarterly earnings, railcar
order and backlog announcements, facility news, product launches and corporate commentary.

## Before you start

- **No authentication.** Anonymous `GET` only.
- **Trim hard.** `_fields=id,date_gmt,modified_gmt,link,title` turns a multi-hundred-kilobyte page
  into a few kilobytes. Fetch `content.rendered` only for the posts you actually intend to read.
- **There are no webhooks.** Greenbrier publishes no event surface of any kind, so polling is the
  only option. There is also an RSS feed at https://www.gbrx.com/feed/ if you want the same content
  without the API.

## Steps

1. **Resolve the categories once.** Call `listCategories`
   (`GET /wp/v2/categories?_fields=id,slug,name,count`). The press room and Perspectives & Updates
   live here; one of them carried 218 of the 245 posts on 2026-09-12.

2. **Do the first pull.** Call `listPosts`
   (`GET /wp/v2/posts?per_page=20&orderby=date&order=desc&_fields=id,date_gmt,modified_gmt,link,title,excerpt`).
   Record the newest `date_gmt` you saw as your watermark.

3. **Poll incrementally.** On every later run use the `after` parameter against your watermark:
   `GET /wp/v2/posts?after={watermark}&orderby=date&order=asc&per_page=100&_fields=...`.
   `after` takes an ISO 8601 timestamp and is applied to the publication date. Advance the watermark
   only after you have successfully processed the page.

4. **Catch silent edits too, if that matters.** A corrected press release keeps its `date` and
   changes its `modified_gmt`. Use `modified_after={watermark}` as a second query if you need to see
   revisions, and compare `modified_gmt` against what you stored.

5. **Read a post.** Call `getPost` (`GET /wp/v2/posts/{id}`). `content.rendered` is HTML.
   `excerpt.rendered` is the summary Greenbrier wrote — prefer quoting it over generating your own
   summary of a financial disclosure.

6. **Filter by subject.** `categories={id}`, `tags={id}` and `search={term}` all work, and
   `X-WP-Total` on the response gives you the true count for the filter.

## Handle these carefully

- **These are financial disclosures.** Earnings and backlog posts are material public statements.
  Quote the numbers as published, with the post `link`, and never restate a figure you interpolated
  or rounded.
- **Investor Relations is a different surface.** https://investors.gbrx.com/ is a separate host with
  no API; SEC filings, guidance and the earnings call material live there, not here.
