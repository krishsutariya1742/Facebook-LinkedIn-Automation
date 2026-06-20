# Facebook-LinkedIn-Automation

This repository contains automation pipelines (n8n flows) for managing and interacting
with a Facebook Page: scheduled posting, automated comment replies, and instant DM replies.

**Workflow 1 - Auto Posting to Facebook Page**

![Auto Posting to Facebook Page](/Auto%20posting%20to%20Facebook%20Page.png)

- **Source:** Content items are created and stored in Supabase for scheduled posting.
- **Schedule:** Cron job runs daily at 09:00 to poll for pending posts.
- **Action:** Fetch pending posts from Supabase and publish them to the Facebook Page via the
	Graph API with the page access token.

Key highlights:
- Fully automated scheduled publishing pipeline.
- Centralized content storage in Supabase for easy management.
- Scheduler ensures posts are published at a consistent daily time.

**Workflow 2 - Auto Reply to Facebook Comments**

![Auto Reply to Facebook Comments](/Auto%20Reply%20to%20Facebook%20Comments.png)

How it works (pipeline summary):
1. A scheduler triggers every minute to begin a polling run.
2. The flow fetches all posts from the target page via Facebook Graph API.
3. The posts are expanded and processed one-by-one to fetch their public comments (filter=stream).
4. A JavaScript node filters comments to keep only top-level, unanswered comments (comment_count = 0
	 and no existing page reply).
5. Each unanswered comment is passed to an AI agent (Claude Sonnet) which generates a short (2–3 lines),
	 context-aware reply.
6. The reply is posted back to the comment endpoint `/{comment_id}/comments` on Graph API v25.0.

Key highlights:
- Smart filtering avoids duplicate replies — only comments with zero replies are processed.
- Claude Sonnet generates concise, human-like responses; asks clarifying questions when needed.
- Two nested loops efficiently handle multiple posts and multiple comments per run.
- Runs entirely hands-free on a fixed schedule.

**Workflow 3 - Auto DM on Facebook**

![Auto DM on Facebook](/Auto%20DM%20on%20Facebook.png)

How it works (overview):
- A registered Meta webhook receives incoming Direct Messages and validates `hub_verify_token`.
- Subscription verification (hub_mode = subscribe) returns `hub_challenge` when required.
- For live messages, the message text is forwarded to the AI agent (Claude Sonnet) to generate a
	short (2–3 lines) reply.
- The reply is sent back to the sender using the Graph API `me/messages` endpoint.

Key highlights:
- Real-time responses with zero polling delay.
- Built-in webhook verification before processing any message.
- Claude produces concise, friendly, and non-AI-sounding replies and asks follow-ups when intent is unclear.

---

**LinkedIn Workflows**

**Workflow 1 - Send LinkedIn Auto Request with Custom Message**

![LinkedIn Auto Request](/Send%20auto%20request%20with%20custom%20message.png)

How it works:
- A scheduler runs daily at 08:00 (or can be triggered manually) to process pending prospects.
- The flow clears the previous Google Sheets batch, fetches up to 3 `pending` prospects from Supabase,
	and iterates each prospect one-by-one.
- `Claude Sonnet` generates a personalised connection note (under 200 characters) referencing
	title, company, and location. Notes are trimmed, saved back to Supabase, and status set to `note_ready`.
- Prepared rows (Profile URL + Message) are appended to a Google Sheet which PhantomBuster reads.
- PhantomBuster Auto Connect phantom is launched, the run is waited on, results are fetched, and
	Supabase is updated (`connect_sent`) and a batch log is written.

Key highlights:
- Notes strictly capped at 200 characters, natural tone, no emojis.
- Batches of 3 per run to remain within LinkedIn safe limits.
- Google Sheets used as a staging layer for PhantomBuster.
- Full state tracking in Supabase and batch logging after each run.

**Workflow 2 - Auto DM in LinkedIn**

![LinkedIn Auto DM](/Auto%20DM%20on%20LinkedIn.png)

How it works:
1. Runs every hour and launches a PhantomBuster inbox-scraper phantom to export the inbox.
2. The flow waits for the phantom, fetches the S3 `result.json`, and filters messages from the last hour
	 that were not sent by the account owner.
3. Each message receives a unique `message_hash` (threadUrl + lastMessageDate) and is checked against
	 `linkedin_convo` in Supabase for deduplication.
4. New messages are passed to `Claude Sonnet` which generates a short, personalised DM reply.
5. Replies are staged to Google Sheets and a PhantomBuster message-sender phantom sends them.
6. The send results are fetched and each conversation is logged to Supabase with `is_replied = true`.

Key highlights:
- Two PhantomBuster phantoms used sequentially to keep session activity realistic.
- Message hashing prevents duplicate processing.
- 1-hour recency filter keeps replies timely.
- Full logging of the conversation metadata in Supabase.

**Workflow 3 - Auto Comment in LinkedIn Posts**

![LinkedIn Auto Comment](/Auto%20Comment%20on%20LinkedIn%20Posts.png)

How it works:
- Scheduler runs every hour and fetches the 10 most recent posts from `linkedin_posts` in Supabase.
- The stored LinkedIn OAuth token is read and validated before any API calls.
- For each post URN the flow calls the LinkedIn `socialActions` API to fetch comments, extracts
	commenter name, comment text, and comment ID.
- Each comment is checked against `linkedin_comment_log`; only comments not yet replied to are processed.
- `Claude Sonnet` generates a short, human-like reply which is posted as a nested comment via LinkedIn API.
- A log row is inserted with `replied = true` to prevent future duplicates.

Key highlights:
- Token validation prevents failing runs due to expired tokens.
- Duplicate protection via Supabase comment log.
- Claude replies are capped (3 sentences), personalised, and invite further conversation.
- Full audit trail stored in Supabase for each reply.
