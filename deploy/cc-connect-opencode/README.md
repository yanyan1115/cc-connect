# cc-connect opencode Telegram deployment

This directory contains sanitized deployment templates for a Telegram bot backed
by OpenCode.

Runtime boundary:

- Source checkout: `/opt/cc-connect-opencode/src`
- Binary: `/opt/cc-connect-opencode/bin/cc-connect`
- Config: `/etc/cc-connect-opencode/config.toml`
- Environment file: `/etc/cc-connect-opencode/env`
- Workspace: `/opt/cc-connect-opencode/workspace/Assistant`
- State: `/var/lib/cc-connect-opencode`
- Logs: `/var/log/cc-connect-opencode`
- systemd service: `cc-connect-opencode.service`

Secret hygiene:

- Do not commit real Telegram bot tokens, user IDs, chat IDs, OAuth files,
  cookies, passwords, API keys, or private keys.
- Keep real Telegram values only in `/etc/cc-connect-opencode/env` on the VPS.
- Do not read or print OpenCode OAuth credential files.
- Use `/root/.opencode/bin/opencode` explicitly; non-login shells may not have
  `opencode` in `PATH`.

Expected private runtime env names:

```sh
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOW_FROM=
TELEGRAM_ALLOW_CHAT=
OPENCODE_MODEL=openai/gpt-5.5
```

`TELEGRAM_ALLOW_FROM` must be set deliberately. Use a comma-separated allowlist
for normal operation. Use `*` only for a controlled single-user test.

`TELEGRAM_ALLOW_CHAT` must also be set deliberately for Telegram groups. Use a
comma-separated allowlist of permitted chat IDs; omit unrelated groups.

Telegram group behavior:

- BotFather privacy must be disabled before the bot is expected to see ordinary
  group messages for background context.
- If privacy or group permissions were changed after the bot joined a group,
  remove and re-add the bot. If a user cannot remove it, the bot can leave the
  group via Telegram `leaveChat` and then be invited back by an admin.
- `TELEGRAM_ALLOW_FROM` gates private chats and non-group messages.
- `TELEGRAM_ALLOW_CHAT` gates group chats. Members in an allowed group may wake
  the bot; do not use the private user allowlist to block normal group members.
- Keep `group_reply_all = false` for mention-only behavior. Ordinary group
  messages should only populate background context; `@bot`, replies to the bot,
  slash commands, or configured wake words should trigger the model.
- When a user replies to someone else's message and wakes the bot, replies should
  target the replied-to message, not the trigger message.
- Busy queue acknowledgements should stay silent in groups to avoid posting
  generic `message received` templates into active chats.
- Recent group context is for understanding only. The assistant should keep its
  own persona and not imitate transient group jokes or writing style.

Group test checklist:

- In a small allowed group, send an ordinary message. The bot should not reply.
- Wake the bot with `Assistant` / `助手` or an `@bot` mention. The bot should reply.
- Ask another group member to wake the bot. It should not be blocked by
  `TELEGRAM_ALLOW_FROM`.
- Reply to another person's message while waking the bot. The bot should reply
  to that original message.
- While the bot is busy, wake it again. The message may queue, but the group
  should not receive a generic queue acknowledgement.
- Check that recent context helps the bot understand the conversation without
  copying the group's tone.

Deployment note:

- Restarting `cc-connect-opencode.service` restarts the process hosting the
  current bot session, so the tool call may be interrupted even when systemd
  completes the restart successfully.
- After a self-restart, verify with `systemctl is-active cc-connect-opencode.service`
  and `systemctl show cc-connect-opencode.service --property=ActiveState,SubState,ActiveEnterTimestamp,ExecMainStartTimestamp`.
