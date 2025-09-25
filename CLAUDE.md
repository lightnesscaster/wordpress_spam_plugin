# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a WordPress plugin (OpenRouter Spam Filter) that uses LLM to classify spam comments asynchronously without impacting user experience. The plugin interfaces with the OpenRouter API to analyze comments and automatically manage spam using WordPress's native moderation tools.

## Key Architecture Components

### Core Class Structure
The entire plugin is contained in `openrouter-spam-filter.php` as a single class `OpenRouter_Spam_Filter` with static methods. Key responsibilities:

- **Async Queue System**: Comments are queued via WordPress cron for background processing to avoid blocking comment submission
- **Concurrency Control**: Uses filesystem locks (with transient fallback) to limit concurrent API calls to 5 workers
- **Retry Logic**: Implements exponential backoff with jitter for both API failures and lock acquisition
- **IP Auto-ban**: Tracks spam strikes per IP and automatically adds repeat offenders to WordPress's `disallowed_keys`

### Critical Constants
- `REQUIRED_CONFIDENCE`: 0.97 - Only marks comments as spam with high confidence
- `MAX_ACTIVE_WORKERS`: 5 - Concurrent API call limit
- `MODELS`: Array of OpenRouter models with automatic fallback

## Development Commands

This is a standalone WordPress plugin with no build process. Key operations:

### Testing the Plugin
1. Activate plugin in WordPress admin
2. Submit test comments (both spam and legitimate)
3. Monitor WordPress debug log for `[OpenRouter Spam Filter]` entries
4. Check Settings → OpenRouter Spam Filter for bulk re-check functionality

### WordPress CLI Commands (if WP-CLI available)
```bash
# Clear scheduled events for a comment
wp cron event delete openrouter_spam_filter_process_comment

# View pending comments
wp comment list --status=hold

# Check option values
wp option get openrouter_spam_filter_ip_counts
wp option get disallowed_keys
```

## Configuration Requirements

The plugin requires an OpenRouter API key configured via either:
- Environment variable: `OPENROUTER_API_KEY`
- WordPress constant in wp-config.php: `define('OPENROUTER_API_KEY', 'your-key');`

## Key Implementation Details

### Notification Suppression
The plugin hooks into WordPress notification filters to prevent emails while comments are being processed, then replays appropriate notifications after classification.

### Lock Management
Primary mechanism uses filesystem locks in `wp-content/uploads/openrouter-spam-filter-locks/`. Falls back to transient-based mutex if filesystem is unavailable.

### Admin Interface
Settings page at Settings → OpenRouter Spam Filter provides:
- Pending comment count display
- Bulk re-check functionality that processes comments in 500-item batches
- Success/failure messaging via URL parameters

### Database Interactions
- Uses comment meta for tracking queue state (`_openrouter_spam_filter_*` keys)
- Stores IP strike counts in `openrouter_spam_filter_ip_counts` option
- Direct SQL queries for atomic updates to avoid race conditions