# NanoClaw Migration Guide

Generated: 2026-04-22
Base: 934f063aff5c30e7b49ce58b53b41901d3472a3e
HEAD at generation: af4eecc
Upstream: dbb859b (v2.0.1)

## Migration Plan

**Order of operations:**
1. Start with clean upstream v2 main
2. Apply channel skills via v2's `/add-telegram` and `/add-discord` (uses `channels` branch)
3. Apply Telegram customizations (topic support, file downloads — reply context already in v2)
4. Disable fork-sync CI workflow
5. Build + validate
6. Port stashed WIP separately (dashboard, db, ipc) — requires manual adaptation to v2 architecture

**Risk areas:**
- Telegram adapter is completely rewritten in v2 (uses Chat SDK bridge). Custom topic + file download features need porting to the new adapter pattern, not copy/paste.
- Credential proxy → replaced by OneCLI in v2. No migration needed — it's native now.
- Compact command → available on `upstream/skill/compact` branch in v2.
- WIP stashed changes (dashboard, db, ipc, config) target v1 architecture and likely need significant rework for v2's two-DB session split.

## Applied Skills

These are reapplied using v2's skill system:

- **Telegram** — v2 has full adapter on `upstream/channels` branch. Install via `/add-telegram` skill.
- **Discord** — v2 has full adapter on `upstream/channels` branch. Install via `/add-discord` skill.
- **Compact** — v2 has `upstream/skill/compact` branch. Merge directly.
- **Credential proxy** — replaced by OneCLI in v2. No action needed.

## Skill Interactions

No inter-skill conflicts expected — Discord and Telegram are independent channel adapters. Compact is a session command, orthogonal to channels.

## Modifications to Applied Skills

### Telegram: Topic/thread support (message_thread_id)

**Intent:** Support Telegram group topics so messages route to the correct thread.

**Files:** `src/channels/telegram.ts`

**How to apply:** After applying `/add-telegram`, check if v2's adapter already passes `message_thread_id`. The fork's v1 implementation:
- Reads `ctx.message.message_thread_id` and passes it as `thread_id` in the message
- Passes `message_thread_id` to `sendMessage()` options for replies

v2's adapter already has a `threadId` parameter in its `hostOnInbound` calls but sets `supportsThreads: false`. Update to `supportsThreads: true` and ensure `message_thread_id` is forwarded.

### Telegram: File download support

**Intent:** Download photos, videos, voice messages, and documents sent via Telegram to the group's attachments directory so the agent can process them.

**Files:** `src/channels/telegram.ts`

**How to apply:** After applying `/add-telegram`, add a `downloadFile` method that:
1. Calls `bot.api.getFile(fileId)` to get the file path
2. Downloads via `https://api.telegram.org/file/bot${token}/${filePath}`
3. Saves to `<groupDir>/attachments/<filename>`
4. Returns the container-relative path `/workspace/group/attachments/<filename>`

The fork handles these message types with file downloads:
- `photo` — takes largest size (`message.photo[-1].file_id`)
- `video` — `message.video.file_id`
- `voice` — `message.voice.file_id`
- `document` — `message.document.file_id`

Each triggers async download + delivers a text message with `[Attachment: filename]` placeholder.

## Customizations

### CI: Fork sync workflow (disabled)

**Intent:** Custom workflow to sync fork with upstream. Currently disabled because v2 merge conflicts.

**Files:** `.github/workflows/fork-sync-skills.yml`

**How to apply:** Copy the file as-is from the fork. It's already disabled (only `workflow_dispatch` trigger active). Re-enable after confirming v2 sync works.

### CI: Removed bump-version and update-tokens workflows

**Intent:** These upstream workflows aren't useful for a fork.

**Files:** `.github/workflows/bump-version.yml`, `.github/workflows/update-tokens.yml`

**How to apply:** Delete these files if they exist after upgrade.

## Stashed WIP (NOT part of this migration)

The following changes are stashed (`git stash list` to verify) and target v1 architecture:
- `src/dashboard.ts`, `src/dashboard-html.ts` — web dashboard
- `src/db.ts` — DB enhancements
- `src/ipc.ts` — IPC additions
- `src/config.ts` — config additions
- `container/skills/hive-mind/`, `container/skills/wiki/` — new skills
- `voice/` — voice support
- `scripts/start.sh` — start script

These need manual porting to v2's two-DB architecture in a separate session.
