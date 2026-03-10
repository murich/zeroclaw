# Telegram Integration Testing Guide

This guide covers testing the Telegram channel integration for ZeroClaw.

## 🚀 Quick Start

### Automated Tests

```bash
# Full test suite (20+ tests, ~2 minutes)
./test_telegram_integration.sh

# Quick smoke test (~10 seconds)
./quick_test.sh

# Just unit tests
cargo test telegram --lib
```

## 📋 Test Coverage

### Automated Tests (20 tests)

The `test_telegram_integration.sh` script runs:

**Phase 1: Code Quality (5 tests)**

- ✅ Test compilation
- ✅ Unit tests (24 tests)
- ✅ Message splitting tests (8 tests)
- ✅ Clippy linting
- ✅ Code formatting

**Phase 2: Build Tests (3 tests)**

- ✅ Debug build
- ✅ Release build
- ✅ Binary size verification (<10MB)

**Phase 3: Configuration Tests (4 tests)**

- ✅ Config file exists
- ✅ Telegram section configured
- ✅ Bot token set
- ✅ User allowlist configured

**Phase 4: Health Check Tests (2 tests)**

- ✅ Health check timeout (<5s)
- ✅ Telegram API connectivity

**Phase 5: Feature Validation (6 tests)**

- ✅ Message splitting function
- ✅ Message length constant (4096)
- ✅ Timeout implementation
- ✅ chat_id validation
- ✅ Duration import
- ✅ Continuation markers

### Manual Tests (6 tests)

After running automated tests, perform these manual checks:

1. **Basic messaging**

    ```bash
    zeroclaw channel start
    ```

    - Send "Hello bot!" in Telegram
    - Verify response within 3 seconds

2. **Long message splitting**

    ```bash
    # Generate 5000+ char message
    python3 -c 'print("test " * 1000)'
    ```

    - Paste into Telegram
    - Verify: Message split into chunks
    - Verify: Markers show `(continues...)` and `(continued)`
    - Verify: All chunks arrive in order

3. **Unauthorized user blocking**

    ```toml
    # Edit ~/.zeroclaw/config.toml
    allowed_users = ["999999999"]
    ```

    - Send message to bot
    - Verify: Warning in logs
    - Verify: Message ignored
    - Restore correct user ID

4. **Rate limiting**
    - Send 10 messages rapidly
    - Verify: All processed
    - Verify: No "Too Many Requests" errors
    - Verify: Responses have delays

5. **Mention-only mode (group chats)**

    ```toml
    # Edit ~/.zeroclaw/config.toml
    [channels.telegram]
    mention_only = true
    ```

    - Add bot to a group chat
    - Send message without @botname mention
    - Verify: Bot does not respond
    - Send message with @botname mention
    - Verify: Bot responds and mention is stripped
    - DM/private chat should always work regardless of mention_only
    - Regression check (group non-text): verify group media without mention does not trigger bot reply
    - Regression command:
      `cargo test -q telegram_mention_only_group_photo_without_caption_is_ignored`

6. **Error logging**

    ```bash
    RUST_LOG=debug zeroclaw channel start
    ```

    - Check for unexpected errors
    - Verify proper error handling

6. **Health check timeout**

    ```bash
    time zeroclaw channel doctor
    ```

    - Verify: Completes in <5 seconds

## 🔘 Inline Buttons Testing

The inline buttons feature allows the AI to present interactive button menus in Telegram messages. This section covers automated and manual testing procedures.

### Automated Button Tests (9 tests)

Run button-specific unit tests:

```bash
# All button parsing tests
cargo test telegram test_parse_inline_buttons --lib

# Specific test scenarios
cargo test test_parse_inline_buttons_simple --lib -- --nocapture
cargo test test_parse_inline_buttons_multirow --lib -- --nocapture
cargo test test_parse_inline_buttons_auto_wrap --lib -- --nocapture
cargo test test_parse_inline_buttons_truncate_callback --lib -- --nocapture
```

The automated test suite validates:

- ✅ Simple button parsing (single row)
- ✅ Multi-row button layouts with `---` separator
- ✅ Automatic button wrapping (>3 buttons per row)
- ✅ Callback data truncation (64 char limit)
- ✅ Pipe separator support (`|` in addition to `->`)
- ✅ ROW keyword support as row separator
- ✅ Empty button section handling
- ✅ Text cleanup around button blocks
- ✅ No-button message passthrough

### Manual Button Tests (6 tests)

After running automated tests, perform these manual checks with a running Telegram bot:

1. **Basic button interaction**

    ```bash
    zeroclaw channel start
    ```

    Send this message to the bot in Telegram (or ask the AI to send it):

    ```
    Test message with buttons:

    [BUTTONS]
    ✅ Approve -> btn_approve
    ❌ Reject -> btn_reject
    [/BUTTONS]
    ```

    - Verify: Two buttons appear in a single row
    - Click "Approve" button
    - Verify: Bot receives message `🔘 Button clicked: btn_approve`
    - Verify: Button shows brief loading indicator then returns to normal
    - Click "Reject" button
    - Verify: Bot receives message `🔘 Button clicked: btn_reject`

2. **Multi-row button layout**

    Send this message:

    ```
    Choose your action:

    [BUTTONS]
    🔍 Search -> action_search
    📝 Create -> action_create
    ---
    📋 List -> action_list
    🗑️ Delete -> action_delete
    ---
    ❌ Cancel -> action_cancel
    [/BUTTONS]
    ```

    - Verify: Buttons appear in 3 rows (2+2+1 layout)
    - Click any button
    - Verify: Correct callback data received
    - Verify: All rows render properly

3. **Auto-wrap behavior (>3 buttons)**

    Send this message:

    ```
    Select a number:

    [BUTTONS]
    1️⃣ One -> num_1
    2️⃣ Two -> num_2
    3️⃣ Three -> num_3
    4️⃣ Four -> num_4
    5️⃣ Five -> num_5
    [/BUTTONS]
    ```

    - Verify: Buttons automatically wrap to 2 rows (3+2 layout)
    - Click buttons from both rows
    - Verify: All callbacks work correctly

4. **Callback data truncation (64 char limit)**

    Send this message with long callback data:

    ```
    [BUTTONS]
    Test Long Callback -> this_is_a_very_long_callback_data_string_that_exceeds_the_telegram_api_limit_of_64_characters_and_should_be_truncated
    [/BUTTONS]
    ```

    - Verify: Button appears and is clickable
    - Click the button
    - Verify: Callback data is truncated to 64 characters
    - Check logs for truncation warning

5. **AI-generated button interaction**

    Send this message to the AI:

    ```
    Create a yes/no question with buttons
    ```

    - Verify: AI generates response with `[BUTTONS]...[/BUTTONS]` syntax
    - Verify: Buttons render correctly in Telegram
    - Click one of the buttons
    - Verify: AI receives and responds to the callback
    - Verify: AI can continue the conversation based on button click

6. **Button removal from message text**

    Send this message:

    ```
    Text before buttons

    [BUTTONS]
    Button 1 -> data1
    Button 2 -> data2
    [/BUTTONS]

    Text after buttons
    ```

    - Verify: Message displays as "Text before buttons\n\nText after buttons"
    - Verify: Button block is completely removed from visible text
    - Verify: Buttons appear below the message
    - Verify: Both buttons are clickable

### Example Test Conversations

#### Scenario 1: Deployment Confirmation

```
User: Deploy to production

AI: I'm ready to deploy to production. Please confirm:

[BUTTONS]
✅ Deploy Now -> deploy_prod_confirm
⏰ Schedule Later -> deploy_schedule
❌ Cancel -> deploy_cancel
[/BUTTONS]

User: [clicks "Deploy Now"]

AI receives: 🔘 Button clicked: deploy_prod_confirm

AI: Deployment initiated. Monitoring progress...
```

**Validation:**
- Buttons render in single row
- Callback received correctly
- AI responds appropriately to callback

#### Scenario 2: Multi-Step Workflow

```
User: Set up CI/CD

AI: Step 1/3: Choose environment

[BUTTONS]
🔧 Development -> env_dev
🧪 Staging -> env_staging
🚀 Production -> env_prod
[/BUTTONS]

User: [clicks "Production"]

AI receives: 🔘 Button clicked: env_prod

AI: Step 2/3: Select deployment strategy

[BUTTONS]
🔄 Rolling Update -> strategy_rolling
⚡ Blue-Green -> strategy_bluegreen
---
❌ Cancel -> workflow_cancel
[/BUTTONS]

User: [clicks "Rolling Update"]

AI receives: 🔘 Button clicked: strategy_rolling

AI: Configuration complete! Rolling update to production environment.
```

**Validation:**
- Multi-step button workflow functions correctly
- State maintained between button clicks
- Cancel option available and functional

#### Scenario 3: Pagination Controls

```
User: List all tasks

AI: Showing tasks 1-5 of 23:

1. Fix authentication bug
2. Update dependencies
3. Write documentation
4. Review pull request
5. Deploy to staging

[BUTTONS]
◀️ Previous -> page_0
📄 Page 1/5 -> page_current
▶️ Next -> page_2
[/BUTTONS]

User: [clicks "Next"]

AI receives: 🔘 Button clicked: page_2

AI: Showing tasks 6-10 of 23:
[... continues with next page ...]
```

**Validation:**
- Pagination buttons render correctly
- Navigation between pages works
- Current page indicator present

### Troubleshooting Button Issues

#### Issue: Buttons Don't Appear

**Symptoms:**
- Message sends but no buttons visible
- Button syntax appears in message text

**Solutions:**

1. **Verify syntax:**
   ```bash
   # Check logs for parsing errors
   RUST_LOG=debug zeroclaw channel start
   ```

   - Ensure `[BUTTONS]` and `[/BUTTONS]` tags are exact matches
   - Check that each button uses `->` or `|` separator
   - Verify no typos in button block

2. **Validate message format:**
   ```
   # Correct format
   [BUTTONS]
   Button Text -> callback_data
   [/BUTTONS]

   # Incorrect formats (won't work)
   [buttons]  # lowercase
   Button Text - callback  # single dash
   Button Text: callback  # colon separator
   ```

3. **Check for empty button blocks:**
   - Empty `[BUTTONS][/BUTTONS]` blocks are ignored
   - Must have at least one valid button line

#### Issue: Callbacks Not Received

**Symptoms:**
- Buttons appear but clicks don't trigger agent response
- No `🔘 Button clicked:` message in logs

**Solutions:**

1. **Verify bot configuration:**
   ```bash
   # Check Telegram config
   cat ~/.zeroclaw/config.toml | grep -A 5 "\[channels_config.telegram\]"
   ```

   - Ensure `allowed_users` includes the clicking user
   - Verify bot token is valid

2. **Check daemon status:**
   ```bash
   # Restart channel
   zeroclaw channel stop
   zeroclaw channel start

   # Check health
   zeroclaw channel doctor
   ```

3. **Review allowed_updates:**
   - Verify Telegram bot config includes `callback_query` in allowed updates
   - This should be automatic, but check logs for update filtering

4. **Test with verbose logging:**
   ```bash
   RUST_LOG=trace zeroclaw channel start
   ```

   - Look for callback_query events in logs
   - Check for callback acknowledgment messages

#### Issue: Callback Data Truncated

**Symptoms:**
- Long callback data appears shortened
- Warning in logs about truncation

**Expected Behavior:**
- Telegram limits callback data to 64 characters
- ZeroClaw automatically truncates longer data
- This is expected and not an error

**Solutions:**

1. **Use shorter callback identifiers:**
   ```
   # Instead of:
   very_long_descriptive_callback_data_that_exceeds_limit

   # Use:
   action_deploy_prod
   ```

2. **Use structured short codes:**
   ```
   # Format: type:action:id
   deploy:prod:123
   select:env:staging
   ```

3. **Store complex data elsewhere:**
   - Use callback data as a key/reference
   - Store full context in memory or session state
   - Look up details when callback is received

#### Issue: Button Layout Incorrect

**Symptoms:**
- Buttons appear in wrong number of rows
- Too many or too few buttons per row

**Expected Behavior:**
- Maximum 3 buttons per row (auto-wrap)
- `---` or `ROW` creates explicit row break
- Empty lines ignored

**Solutions:**

1. **Use explicit row separators:**
   ```
   [BUTTONS]
   Button 1 -> data1
   Button 2 -> data2
   ---
   Button 3 -> data3
   [/BUTTONS]
   ```

2. **Check for extra whitespace:**
   - Remove leading/trailing spaces on button lines
   - Ensure separators are on their own lines

3. **Verify button count:**
   - Telegram API limits to 8 buttons total per message
   - Reduce number of buttons if limit exceeded

### Button Feature Checklist

Before merging button-related changes:

- [ ] All unit tests pass (`cargo test telegram test_parse_inline_buttons --lib`)
- [ ] Manual button interaction test completed
- [ ] Multi-row layout test completed
- [ ] Auto-wrap behavior verified
- [ ] Callback reception confirmed
- [ ] AI-generated button interaction verified
- [ ] Button text removal from message validated
- [ ] Example conversations tested
- [ ] Troubleshooting scenarios validated
- [ ] No clippy warnings in button code
- [ ] Button documentation updated

## 🔍 Test Results Interpretation

### Success Criteria

- All 20 automated tests pass ✅
- Health check completes in <5s ✅
- Binary size <10MB ✅
- No clippy warnings ✅
- All manual tests pass ✅

### Common Issues

**Issue: Health check times out**

```
Solution: Check bot token is valid
  curl "https://api.telegram.org/bot<TOKEN>/getMe"
```

**Issue: Bot doesn't respond**

```
Solution: Check user allowlist
  1. Send message to bot
  2. Check logs for user_id
  3. Update config: allowed_users = ["YOUR_ID"]
  4. Run: zeroclaw onboard --channels-only
```

**Issue: Message splitting not working**

```
Solution: Verify code changes
  grep -n "split_message_for_telegram" src/channels/telegram.rs
  grep -n "TELEGRAM_MAX_MESSAGE_LENGTH" src/channels/telegram.rs
```

## 🧪 Test Scenarios

### Scenario 1: First-Time Setup

```bash
# 1. Run automated tests
./test_telegram_integration.sh

# 2. Configure Telegram
zeroclaw onboard --interactive
# Select Telegram channel
# Enter bot token (from @BotFather)
# Enter your user ID

# 3. Verify health
zeroclaw channel doctor

# 4. Start channel
zeroclaw channel start

# 5. Send test message in Telegram
```

### Scenario 2: After Code Changes

```bash
# 1. Quick validation
./quick_test.sh

# 2. Full test suite
./test_telegram_integration.sh

# 3. Manual smoke test
zeroclaw channel start
# Send message in Telegram
```

### Scenario 3: Production Deployment

```bash
# 1. Full test suite
./test_telegram_integration.sh

# 2. Load test (optional)
# Send 100 messages rapidly
for i in {1..100}; do
  echo "Test message $i" | \
    curl -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
         -d "chat_id=<CHAT_ID>" \
         -d "text=Message $i"
done

# 3. Monitor logs
RUST_LOG=info zeroclaw daemon

# 4. Check metrics
zeroclaw status
```

## 📊 Performance Benchmarks

Expected values after all fixes:

| Metric                 | Expected   | How to Measure                   |
| ---------------------- | ---------- | -------------------------------- |
| Health check time      | <5s        | `time zeroclaw channel doctor`   |
| First response time    | <3s        | Time from sending to receiving   |
| Message split overhead | <50ms      | Check logs for timing            |
| Memory usage           | <10MB      | `ps aux \| grep zeroclaw`        |
| Binary size            | ~3-4MB     | `ls -lh target/release/zeroclaw` |
| Unit test coverage     | 61/61 pass | `cargo test telegram --lib`      |

## 🐛 Debugging Failed Tests

### Debug Unit Tests

```bash
# Verbose output
cargo test telegram --lib -- --nocapture

# Specific test
cargo test telegram_split_over_limit -- --nocapture

# Show ignored tests
cargo test telegram --lib -- --ignored
```

### Debug Integration Issues

```bash
# Maximum logging
RUST_LOG=trace zeroclaw channel start

# Check Telegram API directly
curl "https://api.telegram.org/bot<TOKEN>/getMe"
curl "https://api.telegram.org/bot<TOKEN>/getUpdates"

# Validate config
cat ~/.zeroclaw/config.toml | grep -A 3 "\[channels_config.telegram\]"
```

### Debug Build Issues

```bash
# Clean build
cargo clean
cargo build --release

# Check dependencies
cargo tree | grep telegram

# Update dependencies
cargo update
```

## 🎯 CI/CD Integration

Add to your CI pipeline:

```yaml
# .github/workflows/test.yml
name: Test Telegram Integration

on: [push, pull_request]

jobs:
  test:
    runs-on: [self-hosted, aws-india]
    steps:
      - uses: actions/checkout@v3
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      - name: Run tests
        run: |
          cargo test telegram --lib
          cargo clippy --all-targets -- -D warnings
      - name: Check formatting
        run: cargo fmt --check
```

## 📝 Test Checklist

Before merging code:

- [ ] `./quick_test.sh` passes
- [ ] `./test_telegram_integration.sh` passes
- [ ] Manual tests completed
- [ ] No new clippy warnings
- [ ] Code is formatted (`cargo fmt`)
- [ ] Documentation updated
- [ ] CHANGELOG.md updated

## 🚨 Emergency Rollback

If tests fail in production:

```bash
# 1. Check git history
git log --oneline src/channels/telegram.rs

# 2. Rollback to previous version
git revert <commit-hash>

# 3. Rebuild
cargo build --release

# 4. Restart service
zeroclaw service restart

# 5. Verify
zeroclaw channel doctor
```

## 📚 Additional Resources

- [Telegram Bot API Documentation](https://core.telegram.org/bots/api)
- [ZeroClaw Main README](README.md)
- [Contributing Guide](CONTRIBUTING.md)
- [Issue Tracker](https://github.com/zeroclaw-labs/zeroclaw/issues)
