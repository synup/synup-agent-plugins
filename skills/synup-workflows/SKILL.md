---
name: synup-workflows
description: Use when the user wants to look something up or get something done in their Synup account through the Synup MCP server, such as finding a client or location, checking listings and directory sync, reading or replying to reviews, planning or scheduling posts, or checking local rankings. Covers how to find the right Synup tool and the rule to show drafts and ask before changing anything.
---

# Synup workflows

The `synup` MCP server acts on the user's real Synup workspace. A workspace (an agency) has **clients**, the businesses it manages. Each client has **locations**, and each location has listings on directories, reviews, posts and rankings.

## Reaching the tools

The server exposes five tools that lead to several hundred Synup actions:

- `search_os_tools`: find actions by what you want to do, for example "reply to a review".
- `list_os_tools`: browse actions by area.
- `describe_os_tool` / `describe_os_tools`: read an action's exact inputs.
- `call_os_tool`: run an action by name with its arguments.

Describe an action before you first call it. Inputs differ from action to action and can change, so never guess field names.

## Ground rules

- **Never invent ids.** Turn names into ids with `list_clients` and `list_locations`, then reuse those ids. If a name matches more than one client or location, ask which one.
- **Read freely, ask before writing.** Before anything that changes data (replying to a review, creating or editing a post, editing a location, deleting anything), show the user exactly what will be sent and where, then wait for a clear yes. Synup's approval settings don't cover Cursor, so this check is the only one.
- **Report what actually happened.** Say what the response says (saved as a draft, scheduled, published, failed), not what you expected.
- **Respect refusals.** If a call is refused, explain it plainly: the person may not have access to that client, or their plan may not include it. Don't retry with other ids to get around it.

## Common jobs

### Find a client or location

- `list_clients`: the workspace's clients, newest first. Pass `search` to find one by name.
- `list_locations`: locations, filtered by `clientId`, `search`, `status` or `tags`.
- `get_location_by_id`: one location's full profile, including address, hours and categories.

### Check listings

- `get_listings_rollup`: health across all of one client's locations, covering sync rate, connection problems, duplicates and which locations need attention.
- `get_location_listing_overview`: one location's status on each directory, how complete its Google profile is, duplicates, and what needs fixing.

Lead with what needs action. Don't list every directory.

### Read and reply to reviews

- `list_reviews`: a location's reviews (needs `clientId` and `locationId`). Filter by `sites`, `sentiment`, `status` (`responded` or `unreplied`), `keyword`, `from` and `to`, and use `sort: "lowest"` to bring low ratings first.
- `get_review_analytics`: rating, volume, response rate and themes over time.
- **Replying:** write the reply in chat and show it next to the review it answers. Only after the user says yes, call `reply_to_review` with the `reviewId` and `text`. That posts publicly on Google or Facebook under the business's name. If the user wants someone to approve it in Synup first, pass `draft: true` to save it unsent.
- Keep replies short and specific to what the reviewer said. Never include private details such as phone numbers, appointment facts or health information.

### Plan and schedule posts

- `list_posts`: existing posts for a location or a client.
- `list_publish_targets`: the connected profiles a location's post can go to.
- `create_post` **publishes immediately** unless you pass `draft: true` (saved in Synup, not published) or `scheduledFor` (a future ISO 8601 time). Use a draft, or a schedule the user has confirmed including the time zone. Publish now only when they ask for it.
- If the response says a platform has no connection, or that required fields are missing, tell the user which platform or field. Don't report it as a failure.
- `update_post`: changes a draft or scheduled post. Only the fields you send change.

### Check rankings

- `get_ranking_overview`: how a location ranks on Google for its tracked keywords, month by month.

## When something goes wrong

- **Unknown tool:** search again with `search_os_tools`.
- **Access or plan refusal:** explain it. The user fixes it in Synup.
- **Rate limit exceeded:** the error says how many seconds to wait. Wait that long instead of retrying in a loop.
