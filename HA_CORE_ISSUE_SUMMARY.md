# TTS Streaming Issue - Executive Summary

## The Problem in One Sentence

Audio playback is delayed until the entire conversation response completes, even though text is streaming to TTS immediately and audio is ready.

## Current vs Expected Behavior

| Timeline | Current Behavior | Expected Behavior |
|----------|-----------------|-------------------|
| T+0.5s | Text yields, TTS processes | Text yields, TTS processes |
| T+0.5s | ❌ Clients wait (no TTS_START) | ✅ TTS_START emitted, clients fetch |
| T+1.0s | ❌ Silence (waiting) | ✅ Audio plays! |
| T+8.5s | ✅ TTS_START emitted | ✅ Continue streaming |
| T+9.0s | ✅ Audio finally plays | ✅ Audio continues |

**Impact**: 8+ second delay for first message, even though audio is ready at T+1.0s

## Root Cause

**Sequential pipeline**: TTS_START event only emitted **after Intent stage completes**

```python
# Current flow (pipeline.py ~line 1780)
await recognize_intent()  # ← Blocks until generator finishes (8+ seconds)
await text_to_speech()    # ← TTS_START emitted here (too late!)
```

**The paradox**: TTS infrastructure works perfectly (streaming, caching, background tasks), but clients don't fetch because they're waiting for TTS_START.

## Proposed Solution

**Emit TTS_START when streaming begins** (15 lines of code)

```python
# In chat_log_delta_listener() when threshold hit (~line 1200)
if len(tts_text) > STREAM_RESPONSE_CHARS:
    self.tts_stream.async_set_message_stream(...)
    
    # NEW: Emit TTS_START immediately
    if not self._tts_streaming_started:
        self._tts_streaming_started = True
        self.process_event(PipelineEvent(
            PipelineEventType.TTS_START,
            {"streaming": True, ...}
        ))
```

## Why This Works

1. ✅ TTSCache already supports partial data serving
2. ✅ Clients already have TTS URL (from RUN_START)
3. ✅ Clients already listen for TTS_START
4. ✅ TTS background task already runs immediately
5. ✅ No client changes needed
6. ✅ Backwards compatible (flag prevents double emission)

## Benefits

- 🚀 **7-10 seconds faster** first response
- 🎯 **Immediate feedback** for users
- 📦 **Minimal change** (15 lines, low risk)
- ✅ **No breaking changes** (existing behavior preserved)
- 🔧 **Unlocks existing infrastructure** (streaming already works!)

## Use Cases That Benefit

- LLM-powered assistants (long processing times)
- API-integrated workflows (external delays)
- Multi-message responses with acknowledgments
- Any voice assistant with processing delays

## Files to Modify

- `homeassistant/components/assist_pipeline/pipeline.py` (~line 1200, ~line 580)

## Risk Level

**LOW** - Emits event earlier, doesn't change pipeline logic or state machine

---

**Full details**: See `HA_CORE_ISSUE_TTS_STREAMING.md`
