# TTS Streaming Delayed Until Conversation Agent Completes

## Problem Description

When using streaming conversation agents with TTS, the audio playback is delayed until the **entire conversation response completes**, even though text is available and streaming to the TTS engine immediately. This defeats the purpose of streaming and creates poor user experience for voice assistants.

### Use Case

A voice assistant that provides:
1. **Immediate acknowledgment** ("One moment while I process your request...")
2. **Long-running processing** (8+ seconds for LLM/API calls)
3. **Final response** (LLM-generated answer)

**Expected**: First message plays within 1-2 seconds, user hears feedback while waiting  
**Actual**: First message waits 8+ seconds until entire response completes

## Current Behavior

### Timeline of Events

```
T+0.0s  User stops speaking
T+0.5s  Conversation agent yields first message (69 chars, ends with ".")
T+0.5s  Text reaches Wyoming TTS immediately via SynthesizeChunk events
T+0.5s  TTS background task starts, cache created, audio generated
T+0.5s  ❌ But: Clients don't fetch yet (waiting for TTS_START event)
        [8 second delay while LLM processes]
T+8.5s  Conversation agent yields second message
T+8.5s  Generator completes, async_converse() returns
T+8.5s  Intent stage completes
T+8.5s  TTS stage begins, TTS_START event emitted
T+8.5s  ✅ Now clients start fetching audio
T+9.0s  First message finally plays (8.5 seconds late!)
```

## Technical Root Cause

### Sequential Pipeline Architecture

The assist pipeline executes stages sequentially (`pipeline.py` ~line 1780):

```python
# Intent stage
tts_input, all_targets = await self.run.recognize_intent(...)
# ↑ Blocks here until async_converse() completes (including all streaming)

if self.run.end_stage != PipelineStage.INTENT:
    # TTS stage - only reached AFTER intent completes
    if current_stage == PipelineStage.TTS:
        await self.run.text_to_speech(tts_input)
        # ↑ TTS_START event emitted here (too late!)
```

### The Streaming Paradox

Even though TTS streaming infrastructure works perfectly:

1. ✅ Conversation agent yields text immediately
2. ✅ `async_set_message_stream()` called at 60-char threshold
3. ✅ TTS background task created immediately
4. ✅ Wyoming TTS receives chunks immediately via `SynthesizeChunk` events
5. ✅ Audio is synthesized and cached immediately
6. ✅ TTSCache supports partial data serving (clients can connect anytime)

**BUT**: Clients wait for `TTS_START` event, which only fires after Intent stage completes.

### Key Code Locations

**Where streaming begins** (`pipeline.py` ~line 1200):
```python
async def chat_log_delta_listener(...):
    # When 60+ chars accumulated
    if len(tts_text) > STREAM_RESPONSE_CHARS:
        self.tts_stream.async_set_message_stream(tts_input_stream_generator())
        # ↑ Background task starts, TTS begins processing
        # ❌ But no TTS_START event emitted yet!
```

**Where TTS_START is emitted** (`pipeline.py` ~line 1455):
```python
async def text_to_speech(self, tts_input):
    # This method only called AFTER Intent stage completes
    self.process_event(
        PipelineEvent(
            PipelineEventType.TTS_START,
            {"engine": ..., "language": ..., "voice": ...}
        )
    )
```

**Why clients wait** (WebSocket/mobile/browser clients):
- Clients receive TTS URL at `RUN_START` (early)
- But are designed to wait for `TTS_START` before fetching
- This ensures proper sequencing for non-streaming cases

## Evidence / Reproduction

### Test Setup

1. **Conversation Agent**: Custom webhook-based agent (streaming enabled)
2. **Webhook Server**: n8n workflow that sends:
   - First message immediately: "One moment while I process your request."
   - 8-second delay (simulating LLM)
   - Second message: LLM response
3. **TTS**: Wyoming TTS with wyoming_openai (TTS_STREAMING_MODELS configured)
4. **Backend**: Speaches TTS

### Observed Logs

**Text flows immediately (with background buffering)**:
```
14:25:22.368 DEBUG: Yielding content delta: 69 chars
14:25:22.371 DEBUG: Message boundary detected
[8 second gap]
14:25:30.669 DEBUG: Yielding content delta: 5 chars
...
14:25:32.947 DEBUG: Background buffer task completed
```

**But TTS doesn't start until stream completes** (no TTS_START event in logs at T+0.5s).

### Verification with HA Core Analysis

1. **TTSCache.async_stream_data()** (`tts/__init__.py` line 189-227):
   - Supports partial data serving ✅
   - Multiple simultaneous clients ✅
   - Late-joining clients get accumulated data ✅

2. **Wyoming TTS** (`wyoming/tts.py` line 186-191):
   - Sends `SynthesizeChunk` immediately ✅
   - No buffering ✅

3. **wyoming_openai** (external):
   - pySBD detects sentence boundaries ✅
   - Synthesizes incrementally ✅

**The infrastructure is perfect. The only missing piece: Early TTS_START emission.**

## Impact

### User Experience Impact

- **Poor responsiveness**: 8+ second silence before any audio
- **No feedback**: User doesn't know if system is processing
- **Defeats streaming**: Text streams perfectly, but audio doesn't benefit

### Affected Use Cases

1. **LLM-powered assistants**: Long processing times (5-30 seconds)
2. **API-integrated workflows**: External API calls introduce delays
3. **Multi-step reasoning**: Complex queries with processing steps
4. **Acknowledgment patterns**: "Let me check..." → [process] → answer

### Who This Affects

- Custom conversation agents with streaming
- n8n workflow integrations
- LLM integrations (OpenAI, Anthropic, etc.)
- Any voice assistant with processing delays

## Proposed Solution

### Option A: Emit TTS_START When Streaming Begins (RECOMMENDED)

**Minimal, low-risk change** (~15 lines of code)

**Modify** `pipeline.py` ~line 1200 in `chat_log_delta_listener()`:

```python
if tts_input_stream and len(tts_text) > STREAM_RESPONSE_CHARS:
    self.tts_stream.async_set_message_stream(tts_input_stream_generator())
    
    # NEW: Emit TTS_START early so clients can start fetching
    if not self._tts_streaming_started:  # Prevent duplicate events
        self._tts_streaming_started = True
        self.process_event(
            PipelineEvent(
                PipelineEventType.TTS_START,
                {
                    "engine": self.tts_stream.engine,
                    "language": self.pipeline.tts_language,
                    "voice": self.pipeline.tts_voice,
                    "streaming": True,  # Flag indicating early start
                    "tts_input": tts_text[:100],  # Preview for logging
                }
            )
        )
```

**Add flag to `PipelineRun.__init__()`**:
```python
self._tts_streaming_started = False
```

**Benefits**:
- ✅ Clients start fetching immediately (audio plays within 1-2 seconds)
- ✅ No changes to TTS system (already supports this)
- ✅ No changes to clients (already listen for TTS_START)
- ✅ Backwards compatible (non-streaming cases unchanged)
- ✅ Low risk (doesn't alter pipeline flow)

**Edge Cases Handled**:
1. **Response < 60 chars**: TTS_START still emitted in `text_to_speech()` (existing behavior)
2. **Streaming never triggered**: Flag prevents double emission
3. **Intent fails after TTS_START**: Pipeline already handles this (aborts TTS stage)

### Option B: Parallel Intent and TTS Stages

Allow Intent and TTS stages to run concurrently when streaming is enabled.

**Pros**: More architecturally clean  
**Cons**: Larger change (~200+ lines), higher risk, affects state machine

### Option C: New Pipeline Event (TTS_READY)

Add a new event type specifically for "TTS available but Intent still processing".

**Pros**: Clearer semantics, no confusion with existing TTS_START  
**Cons**: Requires client changes (all platforms)

## Expected Behavior After Fix

### Timeline with Proposed Solution

```
T+0.0s  User stops speaking
T+0.5s  Conversation agent yields first message (69 chars)
T+0.5s  Text reaches Wyoming TTS via SynthesizeChunk
T+0.5s  TTS background task starts, audio generated
T+0.5s  ✅ TTS_START event emitted (NEW!)
T+0.5s  ✅ Clients start fetching audio immediately
T+1.0s  🔊 First message plays! (8 seconds faster!)
        [8 second delay while LLM processes]
T+8.5s  Second message arrives
T+9.0s  🔊 Second message plays
T+8.5s  Intent stage completes (TTS_START not re-emitted, flag prevents it)
```

## Alternative Workarounds

### Client-Side Solutions

**ESPHome Satellites**: Add local acknowledgment audio
- Pros: No HA changes
- Cons: Only works for custom satellites, not mobile/browser

### Conversation Agent Solutions

**Split into two requests**: Immediate response + background processing
- Pros: Works with current architecture
- Cons: Requires workflow changes, breaks single-turn pattern

### Neither is Ideal

These workarounds don't solve the fundamental problem: **HA's TTS streaming infrastructure is ready, but the pipeline prevents using it.**

## Additional Context

### Related Components

- **assist_pipeline** (`homeassistant/components/assist_pipeline/`)
- **conversation** (`homeassistant/components/conversation/`)
- **tts** (`homeassistant/components/tts/`)
- **wyoming** (`homeassistant/components/wyoming/`)

### Related Issues

- (Search for existing issues related to TTS streaming delays)
- (Link to any forum discussions)

### Testing Environment

- **Home Assistant**: 2025.8+ (Conversation entity support)
- **Wyoming Protocol**: Latest
- **TTS Backend**: wyoming_openai + Speaches
- **Custom Integration**: webhook_conversation (streaming enabled)

## Implementation Notes

### Backwards Compatibility

**Option A maintains full backwards compatibility**:

1. **Non-streaming responses**: TTS_START emitted in `text_to_speech()` as before
2. **Short responses** (<60 chars): TTS_START emitted in `text_to_speech()` as before
3. **Streaming responses**: TTS_START emitted early (new behavior)
4. **Existing clients**: Already listen for TTS_START, work unchanged
5. **Event flag**: `streaming: True` allows clients to differentiate if needed

### Security Considerations

- No security implications (event emission only)
- No new permissions required
- No data exposure changes

### Performance Impact

- Negligible (one additional event emission)
- No additional background tasks
- No additional memory usage

## Questions for Maintainers

1. **Is Option A acceptable** for merging into core?
2. **Should we add `streaming: True` flag** to TTS_START data, or create new event type?
3. **Should there be a config option** to disable early TTS_START (for compatibility)?
4. **What testing is required** for assist pipeline changes?
5. **Which HA version** should this target (2025.x)?

## Conclusion

The assist pipeline's TTS streaming infrastructure is **already perfect** for this use case. The only missing piece is emitting `TTS_START` when streaming begins, rather than waiting for Intent stage completion.

**Option A is a minimal, low-risk change** that unlocks immediate audio playback for streaming conversation agents, dramatically improving voice assistant responsiveness.

---

## References

### Code Locations (HA Core)

- `homeassistant/components/assist_pipeline/pipeline.py`
  - Line ~1200: `chat_log_delta_listener()` - where streaming begins
  - Line ~1455: `text_to_speech()` - where TTS_START currently emitted
  - Line ~1780: Sequential stage execution
  
- `homeassistant/components/tts/__init__.py`
  - Line ~775: `async_cache_message_stream_in_memory()` - background task creation
  - Line ~189: `TTSCache.async_stream_data()` - partial data serving

- `homeassistant/components/wyoming/tts.py`
  - Line ~186: `_write_tts_message()` - immediate chunk forwarding

### External Dependencies

- **wyoming_openai**: https://github.com/roryeckel/wyoming_openai
  - Provides pySBD sentence boundary detection for incremental synthesis
- **Wyoming Protocol**: https://github.com/rhasspy/wyoming
  - TTS streaming events: SynthesizeChunk, AudioChunk

### Testing Repository

- **webhook_conversation**: https://github.com/mikenorgate/webhook-conversation
  - Custom integration demonstrating the issue
  - Background buffering implementation (addresses different bottleneck)
  - Comprehensive architecture documentation
