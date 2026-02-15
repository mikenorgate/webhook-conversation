# Testing Checklist for TTS Streaming

## Prerequisites

Before testing, ensure:
- [ ] wyoming_openai configured with `TTS_STREAMING_MODELS`
- [ ] wyoming_openai restarted after config change
- [ ] Wyoming TTS integration added in Home Assistant
- [ ] Wyoming integration reloaded in HA (Settings → Devices & Services → Wyoming → Reload)
- [ ] webhook-conversation conversation agent configured
- [ ] "Enforce Sentence Terminators" enabled
- [ ] n8n workflow ready with multiple messages

## Verify Configuration

### Check wyoming_openai

```bash
# View logs
docker logs wyoming_openai

# Should see on startup:
*** Streaming TTS Models ***
Model: speaches-ai/Kokoro-82M-v1.0-ONNX
```

If you don't see this, `TTS_STREAMING_MODELS` is not configured correctly.

### Check Wyoming in Home Assistant

**Method 1: Via UI**
- Go to: Settings → Devices & Services
- Find Wyoming integration
- Click on it to see details

**Method 2: Via Developer Tools**
- Go to: Developer Tools → Services
- Service: `wyoming.list_info`  (if available)

**Expected output**: Two TTS programs:
```yaml
tts:
  - name: "openai-streaming"
    description: "OpenAI (Streaming)"
    supports_synthesize_streaming: true
    voices: [...]
  - name: "openai"
    description: "OpenAI (Non-Streaming)"
    supports_synthesize_streaming: false
    voices: [...]
```

**⚠️ If you only see ONE program, `TTS_STREAMING_MODELS` is not configured!**

### Verify Voice Assistant Pipeline

- Go to: Settings → Voice assistants → [Your Assistant]
- Check that TTS is set to the Wyoming integration
- Preferably select the "Streaming" variant if available

## Test Cases

### Test 1: Single Short Message

**Setup**:
- n8n sends: `{"type":"item", "content":"Hello world"}`
- "Enforce Sentence Terminators" = ON

**Expected**:
- ✅ Audio plays within 1-2 seconds
- ✅ Spoken as "Hello world." (period added)

**Verify in HA logs** (Settings → System → Logs, filter: `webhook_conversation`):
```
DEBUG (MainThread) [custom_components.webhook_conversation.conversation] 
Yielding content delta: 11 chars

DEBUG (MainThread) [custom_components.webhook_conversation.conversation]
Message boundary detected

DEBUG (MainThread) [custom_components.webhook_conversation.conversation]
Adding period at message boundary for TTS: last content was 'Hello world...'
```

### Test 2: Message Already With Terminator

**Setup**:
- n8n sends: `{"type":"item", "content":"Hello world!"}`
- "Enforce Sentence Terminators" = ON

**Expected**:
- ✅ Audio plays immediately
- ✅ No period added (already has `!`)

**Verify in HA logs**:
```
DEBUG (MainThread) [custom_components.webhook_conversation.conversation]
Yielding content delta: 12 chars

DEBUG (MainThread) [custom_components.webhook_conversation.conversation]
Message boundary detected
```
(No "Adding period" message - this is correct!)

### Test 3: Multiple Messages With Delay

**Setup**:
- n8n workflow sends:
  1. `{"type":"item", "content":"One moment please."}`
  2. `{"type":"end"}`
  3. [1-2 second delay while LLM processes]
  4. `{"type":"item", "content":"Here is your answer."}`
  5. `{"type":"end"}`

**Expected**:
- ✅ T+0.5s: "One moment please" starts playing
- ✅ T+2.0s: Natural silence (LLM processing)
- ✅ T+3.0s: "Here is your answer" starts playing

**Verify in wyoming_openai logs**:
```bash
docker logs wyoming_openai | grep "Detected.*ready sentences"
```

Should see:
```
INFO: Detected 1 ready sentences for immediate synthesis: ['One moment please.']
INFO: Detected 1 ready sentences for immediate synthesis: ['Here is your answer.']
```

### Test 4: Long Multi-Sentence Response

**Setup**:
- n8n streams long text:
  - `{"type":"item", "content":"First sentence. Second sentence. Third sentence."}`

**Expected**:
- ✅ First sentence plays immediately (within 1-2s)
- ✅ Second sentence plays shortly after
- ✅ Third sentence plays shortly after
- ✅ No long pause before audio starts

**Verify in wyoming_openai logs**:
```bash
docker logs wyoming_openai | tail -20
```

Should see:
```
INFO: Detected 3 ready sentences for immediate synthesis: 
  ['First sentence.', 'Second sentence.', 'Third sentence.']
```

### Test 5: Message Without Terminator + Enforcement Disabled

**Setup**:
- "Enforce Sentence Terminators" = OFF
- n8n sends: `{"type":"item", "content":"Hello world"}` (no punctuation)
- `{"type":"end"}`

**Expected**:
- ⚠️ Audio may be delayed (pySBD waiting for terminator)
- ⚠️ Or may not play until stream ends

**Purpose**: This demonstrates why enforcement should be ON by default

## Debugging

### Check HA Logs

**Enable debug logging:**
```yaml
# configuration.yaml
logger:
  logs:
    custom_components.webhook_conversation: debug
```

Then restart HA and check: Settings → System → Logs

**Filter**: `webhook_conversation`

**Look for**:
```
DEBUG (MainThread) [custom_components.webhook_conversation.conversation] 
Yielding content delta: X chars
```

If you **don't** see these logs, streaming is not active.

### Check wyoming_openai Logs

```bash
# Follow logs in real-time
docker logs -f wyoming_openai

# Or search for specific messages
docker logs wyoming_openai | grep "ready sentences"
```

**Look for**:
```
INFO: Detected 1 ready sentences for immediate synthesis: ['...']
```

If you **see** this, streaming is working! ✅

**If you see**:
```
DEBUG: No complete sentences ready yet, accumulator has: '...'
```

This means text doesn't end with terminator, or streaming models not configured. ❌

### Common Issues

#### Issue: Logs show content deltas, but no wyoming_openai "ready sentences"

**Cause**: `TTS_STREAMING_MODELS` not configured  
**Solution**: 
1. Add to `docker-compose.yml` or environment:
   ```yaml
   TTS_STREAMING_MODELS: "speaches-ai/Kokoro-82M-v1.0-ONNX"
   ```
2. Restart container: `docker compose restart wyoming_openai`
3. Reload HA integration: Settings → Devices & Services → Wyoming → Reload

#### Issue: No content delta logs in HA

**Cause**: Streaming not enabled  
**Solution**: 
- Check webhook-conversation config
- "Enable Response Streaming" should be ON
- Restart HA after changing config

#### Issue: Audio plays but all at once after delay

**Cause**: Wrong TTS program selected in pipeline  
**Solution**: 
- Ensure pipeline uses "OpenAI (Streaming)" not "OpenAI (Non-Streaming)"
- Check: Settings → Voice assistants → [Your Assistant] → Text-to-speech

#### Issue: InvalidStateError in HA logs

**Cause**: Old version of webhook_conversation trying to yield multiple roles  
**Solution**:
- Pull latest code
- Restart HA
- Check logs for the error - should be gone

#### Issue: First message doesn't have terminator added

**Cause**: "Enforce Sentence Terminators" is OFF  
**Solution**:
- Enable in webhook-conversation configuration
- Or manually add periods in your n8n workflow

## Performance Verification

### Timing Test

Use a stopwatch or check log timestamps:

1. **Start timer** when you stop speaking
2. **Mark time** when first audio starts playing
3. **Expected**: 0.5-2.0 seconds

**If longer than 3 seconds**, streaming is not working correctly.

### Log Timestamps

With DEBUG logging enabled, check timestamps:

```
2024-02-15 10:00:00.100  Yielding content delta: 20 chars
2024-02-15 10:00:00.500  [wyoming_openai] Detected 1 ready sentences
```

**Time difference should be ~100-500ms**

## Success Criteria

All tests pass when:
- ✅ First message plays within 1-2 seconds
- ✅ Multiple messages play incrementally with natural pauses
- ✅ Sentence terminators automatically added when missing
- ✅ wyoming_openai logs show "ready sentences" detected
- ✅ No `InvalidStateError` in HA logs
- ✅ Clean DEBUG logs showing content deltas yielded immediately
- ✅ Wyoming integration shows TWO TTS programs (streaming + non-streaming)

## Cleanup

After testing, you can disable DEBUG logging:

```yaml
# configuration.yaml
logger:
  logs:
    custom_components.webhook_conversation: info  # or remove line
```

Restart Home Assistant to apply.

## Advanced Testing

### Test with n8n LLM streaming

If your n8n workflow streams LLM responses character-by-character:

**Expected behavior**:
- Text accumulates until pySBD detects sentence boundary
- First sentence plays immediately when `.` `!` or `?` is received
- Subsequent sentences play as they complete

**Monitor wyoming_openai logs** to see sentence detection in real-time:
```bash
docker logs -f wyoming_openai | grep "Detected"
```

### Test with different TTS voices

Try different voices configured in wyoming_openai:
- Each should have the same streaming behavior
- Check that voice selection works in HA

### Test with different languages

If you have multilingual setup:
- pySBD supports multiple languages
- Check that sentence detection works correctly for your language
- See ARCHITECTURE.md for pySBD language support details

## Reporting Issues

If tests fail, please provide:

1. **wyoming_openai configuration** (remove sensitive data)
2. **wyoming_openai logs** (last 50 lines)
3. **HA logs** (webhook_conversation, filtered)
4. **HA version** and **webhook_conversation version**
5. **Which test case failed** and **expected vs actual behavior**

Create an issue at: https://github.com/mikenorgate/webhook-conversation/issues
