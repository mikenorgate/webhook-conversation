# Webhook Conversation - Streaming Architecture

## Overview

This document explains how webhook-conversation integrates with Home Assistant's conversation pipeline and Wyoming TTS for incremental audio playback.

## Component Architecture

### 1. n8n Workflow (or external webhook server)

- Sends HTTP streaming response with JSON tokens
- Tokens: `{"type":"begin"}`, `{"type":"item", "content":"..."}`, `{"type":"end"}`
- Can send multiple messages with delays (for LLM processing)
- Each `{"type":"end"}` token marks a message boundary

### 2. webhook_conversation Custom Component

- Receives HTTP stream from n8n
- Transforms to HA conversation deltas
- Yields `{"role": "assistant"}` once at start
- Yields `{"content": "..."}` for each content chunk immediately
- At message boundaries (`end` token), ensures sentence terminators present
- **Critical**: NO buffering - yields content immediately

### 3. Home Assistant Conversation Pipeline

- Creates `content_stream` from deltas
- Passes to TTS via `async_stream_tts_audio()`
- **Important**: Only ONE role per conversation turn
- **Important**: TTS stream can only be set ONCE per turn

### 4. Home Assistant Wyoming TTS Integration

- Receives text stream via `message_gen` async generator
- Sends `SynthesizeStart` to Wyoming server
- Sends `SynthesizeChunk` for EACH text delta immediately
- Sends `SynthesizeStop` when stream ends
- **Myth Busted**: Does NOT buffer text before sending

**Source**: `homeassistant/components/wyoming/tts.py`

```python
async def _write_tts_message(self, message_gen, client, voice):
    await client.write_event(SynthesizeStart(voice=voice).event())
    
    async for message_chunk in message_gen:
        # Sends IMMEDIATELY - no buffering!
        await client.write_event(SynthesizeChunk(text=message_chunk).event())
    
    await client.write_event(SynthesizeStop().event())
```

### 5. wyoming_openai Proxy Server

- Receives Wyoming protocol events via TCP
- Implements two modes based on configuration:

#### When `TTS_STREAMING_MODELS` configured:
- Uses pySBD for sentence boundary detection
- Accumulates chunks until complete sentence detected (`.` `!` `?`)
- Synthesizes sentence immediately upon detection
- Parallel synthesis (max 3 concurrent requests)
- **This is what enables incremental playback**

#### When NOT configured:
- Buffers ALL text until `SynthesizeStop`
- Synthesizes everything at once
- **This causes the delay you're experiencing**

**Source**: `wyoming_openai/handler.py`

```python
async def _handle_synthesize_chunk(self, synthesize_chunk):
    # Add to accumulator
    self._text_accumulator += chunk_text
    
    # Use pySBD to detect complete sentences
    sentences = segmenter.segment(self._text_accumulator)
    
    if len(sentences) > 1:
        ready_sentences = sentences[:-1]
        # Synthesize IMMEDIATELY!
        await self._process_ready_sentences(ready_sentences)
```

### 6. Speaches TTS Backend

- OpenAI-compatible API
- Receives `/v1/audio/speech` POST requests
- Synthesizes audio
- Streams audio chunks back

## Data Flow Diagram

```
n8n workflow (HTTP streaming)
  ↓ {"type":"item", "content":"First message."}
  ↓ [1-2 second delay]
  ↓ {"type":"item", "content":"Second message."}
  ↓ {"type":"end"}
  
webhook_conversation.py
  ↓ yield {"role": "assistant"}         # Once at start
  ↓ yield {"content": "First message."}  # Immediately
  ↓ [natural delay from n8n]
  ↓ yield {"content": "Second message."} # Immediately
  ↓ yield {"content": "."}               # If terminator missing
  
HA Conversation Pipeline
  ↓ Creates async generator
  ↓ Passes to TTS
  
HA Wyoming TTS
  ↓ SynthesizeStart
  ↓ SynthesizeChunk("First message.")    # Sent immediately
  ↓ [natural delay]
  ↓ SynthesizeChunk("Second message.")   # Sent immediately
  ↓ SynthesizeStop
  
wyoming_openai (with TTS_STREAMING_MODELS)
  ↓ pySBD detects: "First message." is complete
  ↓ → Synthesize immediately!
  ↓ 🔊 Audio plays
  ↓ [natural pause - waiting for more chunks]
  ↓ pySBD detects: "Second message." is complete
  ↓ → Synthesize immediately!
  ↓ 🔊 Audio plays
```

## Timing Analysis

### Without TTS_STREAMING_MODELS (The Problem)

```
T+0.0s  User stops speaking
T+0.1s  webhook_conversation yields "First message."
T+0.1s  HA sends SynthesizeChunk("First message.")
T+0.1s  wyoming_openai BUFFERS (no streaming config)
T+1.5s  n8n sends second message "LLM response."
T+1.5s  HA sends SynthesizeChunk("LLM response.")
T+1.5s  wyoming_openai BUFFERS (accumulating)
T+2.0s  Stream ends, HA sends SynthesizeStop
T+2.0s  wyoming_openai NOW synthesizes all: "First message. LLM response."
T+2.5s  🔊 ALL audio starts playing (user waited 2.5s)
```

### With TTS_STREAMING_MODELS (The Solution)

```
T+0.0s  User stops speaking
T+0.1s  webhook_conversation yields "First message."
T+0.1s  HA sends SynthesizeChunk("First message.")
T+0.1s  wyoming_openai pySBD detects complete sentence
T+0.2s  wyoming_openai sends to Speaches
T+0.5s  🔊 First message starts playing (user waited 0.5s!)
T+2.0s  🔊 First message finishes
        [Natural 1.5s pause - LLM processing]
T+2.5s  n8n sends "LLM response."
T+2.5s  HA sends SynthesizeChunk("LLM response.")
T+2.5s  wyoming_openai detects sentence, synthesizes
T+2.8s  🔊 Second message starts playing
```

**Improvement**: First audio plays in 0.5s instead of 2.5s (5x faster!)

## Common Misconceptions

### ❌ Myth: "HA buffers all TTS text before sending"

**Reality**: HA sends `SynthesizeChunk` events immediately for each content delta. You can verify this in the source code at `homeassistant/components/wyoming/tts.py:186-191`.

### ❌ Myth: "We need to yield multiple roles to force TTS flushing"

**Reality**: This breaks HA's state machine causing `InvalidStateError`. The TTS stream can only be set once per conversation turn (see `homeassistant/components/assist_pipeline/pipeline.py:1471`).

### ❌ Myth: "We need to add separators between messages for TTS"

**Reality**: Natural pauses come from:
1. LLM processing delays (1-2 seconds between messages in n8n)
2. Sentence terminators creating natural pause points in speech

### ❌ Myth: "Wyoming protocol doesn't support streaming"

**Reality**: Wyoming has `SynthesizeStart`/`Chunk`/`Stop` events specifically for streaming. Home Assistant uses them by default.

### ❌ Myth: "pySBD buffering happens in Home Assistant"

**Reality**: Home Assistant doesn't use pySBD at all. The pySBD sentence detection happens in wyoming_openai when `TTS_STREAMING_MODELS` is configured.

### ✅ Truth: "wyoming_openai needs TTS_STREAMING_MODELS configured"

**This is the only issue!** One configuration line solves everything:

```yaml
TTS_STREAMING_MODELS: "speaches-ai/Kokoro-82M-v1.0-ONNX"
```

## Why Sentence Terminators Matter

### pySBD Sentence Boundary Detection

pySBD (Pragmatic Sentence Boundary Disambiguator) requires punctuation to detect complete sentences:

- **With terminator**: `"Hello world."` → Complete sentence → Synthesize immediately
- **Without terminator**: `"Hello world"` → Incomplete → Wait for more text

### The Role of `enforce_sentence_terminators`

When enabled, webhook-conversation automatically adds a period at message boundaries:

```python
# n8n sends:
{"type": "item", "content": "One moment"}
{"type": "end"}

# webhook-conversation yields:
{"content": "One moment"}
{"content": "."}  # ← Added automatically
```

This ensures wyoming_openai's pySBD can detect the sentence boundary and synthesize immediately.

## Configuration Requirements

### Minimal Working Setup

**1. wyoming_openai configuration:**
```yaml
services:
  wyoming_openai:
    environment:
      TTS_OPENAI_URL: http://speaches:8000/v1
      TTS_MODELS: "speaches-ai/Kokoro-82M-v1.0-ONNX"
      TTS_STREAMING_MODELS: "speaches-ai/Kokoro-82M-v1.0-ONNX"  # ← Required!
```

**2. webhook-conversation configuration:**
- ✅ Enable Response Streaming: ON
- ✅ Enforce Sentence Terminators: ON (recommended)

**3. n8n workflow:**
- Messages should end with `.` `!` or `?`
- Or rely on `enforce_sentence_terminators` to add them

### Verification

After configuration:

1. **Check wyoming_openai logs:**
   ```bash
   docker logs wyoming_openai | grep "Detected.*ready sentences"
   ```
   Should see: `Detected 1 ready sentences for immediate synthesis: ['...']`

2. **Check HA Wyoming integration:**
   - Should show TWO TTS programs
   - "OpenAI (Streaming)" - with `supports_synthesize_streaming: true`
   - "OpenAI (Non-Streaming)" - with `supports_synthesize_streaming: false`

3. **Test timing:**
   - First message should play within 1-2 seconds
   - Subsequent messages should play as they arrive

## Future Improvements

### Potential Enhancements

1. **Direct Wyoming Integration**: Skip HA pipeline, send directly to Wyoming TTS
2. **Custom Sentence Chunking**: Add chunking logic in webhook_conversation itself
3. **Streaming Health Checks**: Auto-detect wyoming_openai configuration issues
4. **Performance Metrics**: Report actual time-to-first-audio in logs

### Why Not Implemented Now

- HA's architecture already works perfectly when wyoming_openai is configured correctly
- Adding complexity is premature optimization
- Current approach follows HA's conventions and best practices
- Focus on documentation and clarity over code complexity

## Debugging Guide

### Enable Debug Logging

**webhook-conversation:**
```yaml
# configuration.yaml
logger:
  logs:
    custom_components.webhook_conversation: debug
```

**wyoming_openai:**
```yaml
# docker-compose.yml
environment:
  WYOMING_LOG_LEVEL: DEBUG
```

### Key Log Messages

**webhook-conversation (DEBUG):**
```
Yielding content delta: 15 chars
Message boundary detected
Adding period at message boundary for TTS: last content was 'Hello world...'
```

**wyoming_openai (INFO):**
```
Detected 1 ready sentences for immediate synthesis: ['Hello world.']
```

**Home Assistant Wyoming TTS (DEBUG):**
```
Writing SynthesizeChunk: "Hello world."
```

### Common Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| All audio plays at once after delay | `TTS_STREAMING_MODELS` not set | Add to wyoming_openai config |
| No "ready sentences" logs | Config not loaded or wrong model name | Restart wyoming_openai, check model name |
| Only one TTS program in HA | `TTS_STREAMING_MODELS` not recognized | Check YAML syntax, reload HA integration |
| First message doesn't play | Missing sentence terminator | Enable `enforce_sentence_terminators` |
| InvalidStateError in HA logs | Code trying to set TTS stream twice | Update webhook_conversation to latest |

## References

### Source Code

- **Home Assistant Core**:
  - Wyoming TTS: `homeassistant/components/wyoming/tts.py`
  - Assist Pipeline: `homeassistant/components/assist_pipeline/pipeline.py`
  - Conversation Chat Log: `homeassistant/components/conversation/chat_log.py`

- **wyoming_openai**:
  - Handler: `src/wyoming_openai/handler.py`
  - Compatibility: `src/wyoming_openai/compatibility.py`

- **webhook_conversation**:
  - Conversation: `custom_components/webhook_conversation/conversation.py`
  - Entity: `custom_components/webhook_conversation/entity.py`

### External Links

- [wyoming_openai GitHub](https://github.com/roryeckel/wyoming_openai)
- [Speaches GitHub](https://github.com/speaches-ai/speaches)
- [pySBD Documentation](https://github.com/nipunsadvilkar/pySBD)
- [Wyoming Protocol](https://github.com/rhasspy/wyoming)
