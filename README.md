# OpenRouter Spam Filter

A WordPress plugin that routes new and pending comments through an OpenRouter-hosted LLM to catch high-confidence spam without slowing down commenters. The plugin defers classification to async jobs, reuses WordPress' native moderation tools, and automatically bans repeat offender IPs.

## Features
- Queues new comments for asynchronous LLM review so visitors see the usual submission response instantly.
- Calls `openai/gpt-oss-20b:free` first (with automatic fallback) using a deterministic system prompt that expects `{ "is_spam": bool, "confidence": float }`.
- Marks comments as spam only when confidence ≥ 0.97 and leaves pending comments untouched if the classifier is unsure or unavailable.
- Suppresses moderator notifications until a verdict is known, then replays the appropriate WordPress emails.
- Tracks spam strikes by IP: two flagged comments add the IP to `disallowed_keys`, allowing core to block future posts automatically.
- Admin settings page with a “Re-check Pending Comments” button that re-queues every pending comment in paginated batches (500 at a time).
- Concurrency guard rails (filesystem + transient mutex) and retry logic to avoid overwhelming PHP workers.

## Requirements
- WordPress 6.0 or newer.
- PHP 7.4 or newer.
- An OpenRouter API key with access to `openai/gpt-oss-20b` (free tier recommended as default).

## Installation
1. Copy `openrouter-spam-filter.php` into its own folder under `wp-content/plugins/openrouter-spam-filter/`.
2. Ensure the web server exposes `OPENROUTER_API_KEY` (environment variable or define `OPENROUTER_API_KEY` in `wp-config.php`).
3. In wp-admin, activate **OpenRouter Spam Filter**.

## Configuration
- **API key**: Either export `OPENROUTER_API_KEY` for the PHP-FPM/Apache user or add `define( 'OPENROUTER_API_KEY', 'your-key' );` to `wp-config.php`.
- **LLM prompt**: Hard-coded with deterministic instructions. Modify `SYSTEM_PROMPT` in the plugin if needed.
- **Spam threshold**: The default (0.97) lives in `REQUIRED_CONFIDENCE`.

## Admin Tools
- Navigate to **Settings → OpenRouter Spam Filter** to see the current pending count and trigger a bulk re-check. The plugin paginates through pending comments, reusing the async queue for every comment it touches and displaying a success/empty notice when finished.

## Notifications
- Moderator and author emails are paused while comments await classification. After the final verdict, the plugin replays WordPress’ native notifications (pending confirmations or author pings) and avoids duplicate emails.

## IP Auto-ban
- Each spammed comment increments a strike counter stored in `openrouter_spam_filter_ip_counts`.
- On the second strike, the IP is added to `disallowed_keys` and removed from the strike log.
- WordPress then blocks new comments from that IP before the LLM runs.

## Localization
- Textdomain `openrouter-spam-filter` is loaded on `plugins_loaded`. Place translations under `languages/` following WordPress conventions.

## Rate Limits & Concurrency
- The plugin throttles to five concurrent workers via filesystem locks (with fallbacks) and backs off with jitter if slots are busy or storage is unavailable.
- LLM calls are retried per model, and comments fall back to their original status if all attempts fail.

## Testing & Verification
- After activation, submit test comments (spammy and legitimate) to confirm spam catches and that legit comments keep their original state.
- Use the settings page button to validate the bulk re-check flow and notice messaging.
- Monitor the WordPress debug log for any `[OpenRouter Spam Filter]` warnings about lock failures.

## License
This plugin ships without a bundled license file. Add one if you plan to distribute it.

## Maintainer
- lightnesscaster (johnstondaniel4@gmail.com)
