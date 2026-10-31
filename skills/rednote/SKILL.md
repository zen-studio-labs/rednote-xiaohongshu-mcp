---
name: rednote
description: RedNote (Xiaohongshu, 小红书, XHS) data through the rednote MCP tools. Use when the user wants RedNote or Xiaohongshu notes searched by keyword or hashtag, a note's full text, images or video read, the comments and replies under a note, or a creator's profile, follower counts and recent posts.
---

# RedNote (Xiaohongshu, 小红书)

Four RedNote tools plus Apify's run and dataset tools, served by the `rednote` MCP server. Each RedNote tool starts a paid run on the user's Apify account and returns run metadata. The rows arrive later in a dataset. Every request follows the **run flow** below.

| Tool (client prefixes vary) | Use for |
|---|---|
| `zen-studio--rednote-search-scraper` | Keyword or hashtag search: notes with title, text preview, engagement, author, images, video |
| `zen-studio--rednote-note-detail-scraper` | Full notes for known links or IDs: complete text, tags, location, every image, video; optional file downloads |
| `zen-studio--rednote-comments-scraper` | Comments under notes, with nested replies |
| `zen-studio--rednote-user-profile-scraper` | Creator profile (bio, followers, likes and collects, location, agency tag) and their recent notes |
| `get-actor-run` | Wait for and check a run |
| `get-dataset-items` | Read the rows |
| `fetch-actor-details` | Full input reference and README of any of the four |
| `get-key-value-store-record` | Read a file a run saved, such as a subtitle `.srt` |
| `abort-actor-run` | Stop a run the user no longer needs |

If only Apify's generic `call-actor` is available, call it with `actor: "zen-studio/<name>"` (for example `zen-studio/rednote-search-scraper`) and the same input; the run flow is identical.

## Run flow

A tool call returns after at most 45 seconds, whether or not the run has finished. Many runs take longer, so treat the first response as a receipt, not the result.

1. **Start.** Call the RedNote tool with the recipe input and `"waitSecs": 45`. Keep the `runId` from the response. Done when you hold a `runId`.
2. **Poll.** Read `status`. While it is `READY` or `RUNNING`, call `get-actor-run` with `runId` and `waitSecs: 45`, and tell the user the run is still going between calls. Done when `status` is terminal: `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. A response that is already `SUCCEEDED` skips straight to step 3.
3. **Fetch.** Call `get-dataset-items` with `datasetId` from `storages.datasets.default.id` (the response's `nextStep` also names it). Pass `limit` (20 to 50) and a `fields` projection from **Reading results**; rows carry many fields, video notes especially. Page with `offset` until you have what the question needs. Done when every row the answer depends on is read.
4. **Answer.** Summarise for the user: engagement counts, note links (`url`), and anything filtered out. Translate Chinese text when the user writes in another language.

One run per request. A run that is still `RUNNING` keeps working and billing; starting a second run for the same question pays twice. Resume polling the same `runId` instead, including after a client timeout or a new turn.

Terminal states other than `SUCCEEDED`: `TIMED-OUT` and `ABORTED` can hold partial rows, so fetch them and say the result is partial. `FAILED`: report `statusMessage` and change the input before retrying. A `SUCCEEDED` run whose item count reads 0 can lag for a few seconds; fetch once before concluding it is empty.

## Chaining the tools

- **Search, then go deeper.** Pass a search row's `url` to Note details or Comments as it is: it carries the `xsec_token` that opens the note. The bare `id` also works.
- **Search, then the creator.** Pass a search row's `author.userid` to the profile tool. Display names are not accepted, because several accounts can share one.
- **Creator, then their posts.** Profile rows list notes with `id` but no URL; pass those IDs to Comments or Note details.

## Input recipes

Start small (20 to 50 results) unless the user asks for more; every returned note, comment, reply and profile is billed.

**Search** (`rednote-search-scraper`):

```json
{ "keywords": ["露营装备"], "maxResults": 30, "sortType": "popularity_descending",
  "noteType": "all", "timeFilter": "1w" }
```

- `keywords`: a list. Each keyword has its own `maxResults` budget, so three keywords at 30 can return up to 90 billed notes. The default `maxResults` is 100; always set it.
- Chinese, English, brand and person names, and hashtags (`#露营`) all work. For Chinese topics, use Chinese keywords: they match how people write their notes.
- `sortType`: `general` (most relevant, default), `popularity_descending` (most liked and commented), `time_descending` (newest first).
- `noteType`: `all` (default), `video`, `image`, `live`.
- `timeFilter`: `all` (default), `1d`, `1w`, `6mo`.
- `maxResults` goes up to 10,000, or `0` for everything available (a broad keyword reaches about 7,000). Ask before going above a few hundred. Leave `topUpFromOtherSorts` at its default; it only changes behaviour above 1,000 per keyword.
- `desc` in search rows is a preview of about 60 characters; `title` is complete. For the full text, pass the rows' `url` to Note details.

**Note details** (`rednote-note-detail-scraper`):

```json
{ "noteUrls": ["http://xhslink.com/o/6oKW7wkJf09", "69db721a00000000220290a8"] }
```

- Share links (`xhslink.com`), full note URLs, the whole share text copied from the app, or bare 24-character note IDs. One billed result per note.
- A profile link without a note after it is a person, not a note; use the profile tool.
- `downloadVideos` (with `videoQuality`: `highest` up to 4K, or `compatible` H.264), `downloadImages` and `downloadCovers` save the original files. They need a paid Apify plan and are billed per saved file; enable them only when the user asks for the files. `downloadSubtitles` saves `.srt` transcripts of video notes that have subtitles, at no extra charge.
- Saved files go to the run's key-value store and are linked from `videoFile`, `coverFile`, `imageFiles`, `livePhotoFiles` and `subtitleFiles` (`kvUrl`). Read a saved text file such as a subtitle with `get-key-value-store-record` (`keyValueStoreId` from `kvStoreId`, `recordKey` from `kvStoreKey`). Opening a `kvUrl` in a browser needs the user's Apify token, or the user opens the run in Apify Console.

**Comments** (`rednote-comments-scraper`):

```json
{ "noteUrls": ["69db721a00000000220290a8"], "sortBy": "most_liked",
  "maxCommentsPerNote": 50, "includeReplies": true, "maxRepliesPerComment": 3 }
```

- Always set `maxCommentsPerNote`: the default `0` means all, and a viral note can hold thousands.
- Every top-level comment and every reply is billed. `maxRepliesPerComment` (default 5, `0` means all) is the main cost lever; `includeReplies: false` returns top-level comments only.
- `sortBy`: `latest` (default), `recommended`, `most_liked`.
- One row per top-level comment, replies nested in `replies[]`. `ip_location` is the region RedNote shows for the commenter.

**Creator profile** (`rednote-user-profile-scraper`):

```json
{ "profileUrls": ["https://www.xiaohongshu.com/user/profile/65c381340000000011014d37"],
  "includeNotes": true, "maxNotesPerProfile": 20 }
```

- Profile URL, 24-character user ID, share link, RedNote ID (小红书号, shown on the profile) or a note link, which resolves to the note's author.
- Each profile is billed once and each collected note separately. `includeNotes: false` is the cheapest way to look up profiles only. The default `maxNotesPerProfile` is 100; always set it.
- One row per profile: `profile` plus a `notes[]` list, newest first with pinned posts near the top.

## Reading results

Project fields with `get-dataset-items` `fields` (dot paths allowed). Good starting sets:

- Search: `id,type,title,desc,url,timestamp,engagement,author.nickname,author.userid`
- Note details: `id,type,title,desc,url,timestamp,ip_location,engagement,tags,author.nickname,video.duration_seconds`
- Comments: `note_id,content,time,ip_location,like_count,sub_comment_count,user.nickname,replies`
- Creator profile: `profile.nickname,profile.red_id,profile.desc,profile.ip_location,profile.fans,profile.liked,profile.collected,profile.tags,notes`. Notes arrive in the same row, so keep `maxNotesPerProfile` small when the user only needs the profile.

`engagement` holds `liked_count`, `comments_count`, `collected_count` and `shared_count`. Times are Unix timestamps in seconds or, with 13 digits, milliseconds; convert them before quoting dates. Image and video links expire within hours to a few days; the note `url` is the lasting reference. `fetch-actor-details` returns the full field reference when the user asks for something outside these sets.

## Cost and limits

- Pricing is per result on each tool's Apify Store page (`https://apify.com/zen-studio/<name>/pricing`) and is billed to the user's Apify account. Quote the page rather than a remembered number.
- On Apify's free plan each tool allows a few preview runs with small caps: search 3 runs in total (about 20 notes from the first keyword), the other tools 15 runs each (comments: about 20 top-level comments without replies; creator profile: one profile and about 20 notes; note details: up to 20 notes). File downloads need a paid plan. Search runs are scarce on the free plan, so do not spend them on tests. A capped run says so in its `statusMessage` ("Free tier preview: ..."); tell the user the result is partial and that any paid Apify plan removes the caps.
- `maxResults`, `maxCommentsPerNote`, `maxRepliesPerComment` and `maxNotesPerProfile` are the cost controls. State the caps you chose in the answer.
