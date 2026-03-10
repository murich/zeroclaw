# Telegram Inline Buttons Guide

This guide covers the interactive inline keyboard buttons feature for Telegram in ZeroClaw.

Last verified: **March 10, 2026**.

## 1. What this feature does

ZeroClaw supports Telegram inline keyboard buttons that allow the AI assistant to present interactive choices to users directly in the chat interface. When users click these buttons, callback data is sent back to the agent, enabling interactive workflows and multi-step interactions.

Key capabilities:

- Send interactive button menus with AI responses
- Handle button click callbacks
- Support multiple button layouts (single row, multi-row, grids)
- Automatic callback acknowledgment
- Seamless integration with conversation flow

## 2. Syntax Overview

### Recommended: Native Telegram JSON Format

The AI assistant can include inline buttons using the **standard Telegram Bot API format**:

```
[INLINE_KEYBOARD]
[
  [{"text": "Button 1", "callback_data": "callback_1"}],
  [{"text": "Button 2", "callback_data": "callback_2"}],
  [{"text": "Open Link", "url": "https://example.com"}]
]
[/INLINE_KEYBOARD]
```

This is the **recommended format** because:
- It's the official Telegram Bot API structure
- AI models already know this format
- Supports all button types (callbacks, URLs, web apps)
- More flexible and future-proof

### Legacy: Custom Syntax (Deprecated)

The older custom syntax is still supported for backward compatibility:

```
[BUTTONS]
Button Text -> callback_data
Another Button -> another_callback
---
Second Row Button -> second_row
[/BUTTONS]
```

### JSON Format Rules

- Place `[INLINE_KEYBOARD]...[/INLINE_KEYBOARD]` anywhere in your message text
- Use standard Telegram `InlineKeyboardMarkup` JSON structure
- **Structure**: 2D array where each inner array represents a button row
- **Button types**:
  - Callback: `{"text": "Label", "callback_data": "data"}`
  - URL: `{"text": "Label", "url": "https://..."}`
  - Web App: `{"text": "Label", "web_app": {"url": "https://..."}}`
- Callback data is limited to 64 characters (automatically truncated)
- The button block is stripped from the message text before sending
- Invalid JSON gracefully falls back to showing the text as-is

### Legacy Syntax Rules (Deprecated)

- Place `[BUTTONS]...[/BUTTONS]` anywhere in your message text
- Each button line format: `Display Text -> callback_data` or `Display Text | callback_data`
- Use `---` or `ROW` on a separate line to start a new row
- Maximum 3 buttons per row (additional buttons automatically wrap to a new row)
- Callback data is limited to 64 characters (Telegram API constraint)

## 3. Basic Examples

### Example 1: Yes/No Confirmation (JSON Format)

```
Would you like to proceed with this action?

[INLINE_KEYBOARD]
[
  [{"text": "✅ Yes", "callback_data": "confirm_action"}, {"text": "❌ No", "callback_data": "cancel_action"}]
]
[/INLINE_KEYBOARD]
```

Result: Two buttons side-by-side in a single row.

### Example 1b: Yes/No Confirmation (Legacy Format)

```
Would you like to proceed with this action?

[BUTTONS]
✅ Yes -> confirm_action
❌ No -> cancel_action
[/BUTTONS]
```

Result: Two buttons side-by-side in a single row.

### Example 2: Menu Selection

```
Please select an option:

[BUTTONS]
📊 View Stats -> stats_view
⚙️ Settings -> open_settings
ℹ️ Help -> show_help
[/BUTTONS]
```

Result: Three buttons in a single row.

### Example 3: Multi-row Layout

```
Choose your preferred action:

[BUTTONS]
🔍 Search -> action_search
📝 Create -> action_create
---
📋 List All -> action_list
🗑️ Delete -> action_delete
---
❌ Cancel -> action_cancel
[/BUTTONS]
```

Result: Three rows of buttons (2 buttons, 2 buttons, 1 button).

### Example 4: Rating System

```
How would you rate this response?

[BUTTONS]
⭐️ 1 Star -> rating_1
⭐️⭐️ 2 Stars -> rating_2
⭐️⭐️⭐️ 3 Stars -> rating_3
---
⭐️⭐️⭐️⭐️ 4 Stars -> rating_4
⭐️⭐️⭐️⭐️⭐️ 5 Stars -> rating_5
[/BUTTONS]
```

Result: Two rows with a star rating selection.

### Example 5: Pagination

```
Showing results 1-10 of 50

[BUTTONS]
◀️ Previous -> page_prev
📄 Page 1/5 -> page_current
▶️ Next -> page_next
[/BUTTONS]
```

Result: Pagination controls in a single row.

## 4. Callback Handling

### User Experience

When a user clicks an inline button:

1. The button briefly shows a loading indicator
2. ZeroClaw automatically acknowledges the callback (removes loading state)
3. The agent receives a message formatted as: `🔘 Button clicked: {callback_data}`
4. The agent can respond naturally to the callback data

### Callback Message Format

The agent receives callback events as regular messages with this format:

```
🔘 Button clicked: callback_data
```

For example, if a user clicks a button with `callback_data = "approve_action"`, the agent receives:

```
🔘 Button clicked: approve_action
```

The agent can then:
- Parse the callback data
- Execute the appropriate action
- Send a follow-up message with new buttons if needed
- Update state or context as needed

### Example Conversation Flow

```
Agent: Would you like to deploy to production?

[BUTTONS]
✅ Deploy Now -> deploy_prod
⏰ Schedule Later -> deploy_schedule
❌ Cancel -> deploy_cancel
[/BUTTONS]

User: [clicks "Deploy Now" button]

Agent receives: 🔘 Button clicked: deploy_prod

Agent: Deployment to production has been initiated. I'll monitor the progress.
```

## 5. Advanced Usage Patterns

### State Machine Workflows

Buttons can be used to implement multi-step workflows:

```
Agent: Step 1/3: Choose environment

[BUTTONS]
🔧 Development -> env_dev
🧪 Staging -> env_staging
🚀 Production -> env_prod
[/BUTTONS]

User: [clicks "Production"]

Agent: Step 2/3: Select deployment strategy

[BUTTONS]
🔄 Rolling Update -> strategy_rolling
⚡️ Blue-Green -> strategy_bluegreen
---
❌ Cancel -> workflow_cancel
[/BUTTONS]
```

### Dynamic Button Generation

The AI can generate buttons based on context:

```
User: Show me the available projects

Agent: Here are your active projects:

[BUTTONS]
Project Alpha -> select_project_alpha
Project Beta -> select_project_beta
Project Gamma -> select_project_gamma
---
➕ Create New Project -> create_project
[/BUTTONS]
```

### Contextual Actions

Combine text responses with action buttons:

```
Agent: I found 3 configuration issues:
1. Missing API key for service X
2. Outdated dependency Y
3. Invalid database connection string

[BUTTONS]
🔧 Fix All -> autofix_all
🔍 Show Details -> show_details
---
✋ Skip -> skip_fixes
[/BUTTONS]
```

## 6. Configuration

### Channel Configuration

Inline buttons work automatically when the Telegram channel is configured. No additional configuration is required beyond the standard Telegram setup:

```toml
[channels_config.telegram]
bot_token = "123456:your-telegram-bot-token"
allowed_users = ["*"]  # or specific user IDs
```

See [Channels Reference](./channels-reference.md#41-telegram) for complete Telegram configuration options.

### AI System Prompt

The button syntax is automatically included in the AI system prompt when using Telegram. The prompt section provides:

- Syntax documentation
- Usage rules
- Examples
- Callback handling explanation

No manual prompt engineering is needed to enable button support.

## 7. Limitations and Constraints

### Telegram API Limits

- Maximum 64 characters per callback data field
- Maximum 8 buttons per message (enforced by Telegram)
- Maximum 3 buttons per row (ZeroClaw automatically wraps excess buttons)

### Channel Availability

- Buttons appear only in Telegram channel messages
- When the same message is sent via other channels (CLI, Discord, etc.), the button syntax is stripped and only the text is shown
- Callback handling is Telegram-specific

### Text Constraints

- Button display text can include emoji and Unicode characters
- Very long button text may be truncated by Telegram clients
- Recommended: Keep button labels concise (1-3 words)

## 8. Troubleshooting

### Buttons Not Appearing

If buttons don't appear in Telegram messages:

1. **Check syntax**: Ensure `[BUTTONS]` and `[/BUTTONS]` are exactly matched
2. **Verify format**: Each button line must use `->` or `|` separator
3. **Check row markers**: Use `---` or `ROW` exactly, not similar characters
4. **Review logs**: Check ZeroClaw logs for parsing errors

### Callbacks Not Working

If button clicks don't trigger responses:

1. **Verify callback data**: Ensure callback data is under 64 characters
2. **Check allowed users**: Confirm the clicking user is in `allowed_users` list
3. **Review daemon status**: Ensure `zeroclaw daemon` or `zeroclaw channel start telegram` is running
4. **Check bot permissions**: Verify Telegram bot has message sending permissions

### Callback Data Not Received

If the agent doesn't receive callback events:

1. **Check Telegram bot configuration**: Ensure `allowed_updates` includes `callback_query`
2. **Review channel health**: Run `zeroclaw channel doctor telegram`
3. **Verify network connectivity**: Ensure polling or webhook connection is active
4. **Check logs**: Look for callback acknowledgment messages in daemon logs

## 9. Best Practices

### User Experience

- Use clear, action-oriented button labels (verbs: "Deploy", "Cancel", "View")
- Include emoji for visual clarity and quick recognition
- Limit button count to essential actions (3-5 buttons recommended)
- Group related actions in the same row
- Always provide a cancel or back option in multi-step workflows

### Security and Safety

- Validate callback data before executing actions
- Require confirmation for destructive operations
- Implement rate limiting for sensitive callbacks
- Log important callback actions for audit trails
- Use allowed_users configuration to restrict who can trigger callbacks

### Code Organization

- Define callback data constants for type safety
- Use a consistent naming scheme (e.g., `action_verb_noun`)
- Implement a callback router pattern for complex workflows
- Document callback data values in code comments
- Consider using structured callback data (e.g., `type:action:id`)

### Conversation Design

- Provide context before presenting buttons
- Use buttons to reduce typing and errors
- Combine free-form chat with button interactions
- Allow users to override button selections with text
- Maintain conversation history through button interactions

## 10. Integration Examples

### With Tool Calls

Buttons can complement tool execution:

```
Agent: I can set up continuous deployment for this repository.

[BUTTONS]
⚙️ Configure Now -> setup_cd
📋 Show Configuration -> show_cd_config
---
❌ Not Now -> skip_cd_setup
[/BUTTONS]

User: [clicks "Configure Now"]

Agent receives: 🔘 Button clicked: setup_cd

Agent: [executes git_operations tool to configure CI/CD]
       Continuous deployment configured successfully!
```

### With Memory/Context

Buttons can interact with agent memory:

```
Agent: I notice you frequently deploy to staging. Would you like me to remember this preference?

[BUTTONS]
💾 Yes, Remember -> remember_staging_preference
🚫 No Thanks -> skip_remember
[/BUTTONS]

User: [clicks "Yes, Remember"]

Agent receives: 🔘 Button clicked: remember_staging_preference

Agent: [uses memory tool to store preference]
       Got it! I'll prioritize staging environment in future deployment suggestions.
```

### With Multi-Channel Support

Handle graceful degradation for non-Telegram channels:

```
Agent message (Telegram): Choose an option:
[BUTTONS]
Option A -> opt_a
Option B -> opt_b
[/BUTTONS]

Same message (CLI/Discord): Choose an option:
Type 'opt_a' for Option A or 'opt_b' for Option B
```

## 11. Reference

### Related Documentation

- [Channels Reference](./channels-reference.md) — complete channel configuration
- [Commands Reference](./commands-reference.md) — CLI commands for channel management
- [Troubleshooting](./troubleshooting.md) — general troubleshooting guide

### Source Code

- Implementation: `src/channels/telegram.rs`
- System prompt: `src/agent/prompt.rs` (ButtonsSection)
- Callback handling: `src/channels/telegram.rs` (listen method)

### API References

- [Telegram Bot API - Inline Keyboards](https://core.telegram.org/bots/api#inlinekeyboardmarkup)
- [Telegram Bot API - Callback Queries](https://core.telegram.org/bots/api#callbackquery)
