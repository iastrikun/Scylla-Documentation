---
title: Python Documentation
---
## Table of Contents

1. [[#Overview]]
2. [[#Libraries & Setup]]
3. [[#Configuration]]
4. [[#Audio Recording]]
5. [[#Speech Recognition]]
6. [[#AI System Prompt]]
7. [[#Gemini AI Integration]]
8. [[#Text-to-Speech]]
9. [[#Audio Playback]]
10. [[#Action Triggering]]
11. [[#Main Program Loop]]
12. [[#Program Startup]]

---
## Overview

[[#Table of Contents]]

This program powers Scylla, a Filipino-speaking robot crab that:

- Listens to voice commands in Filipino
- Responds with AI-generated replies
- Performs physical movements and LED animations
- Speaks back using natural Filipino voice

**Technology Stack:**

- **Google Cloud Speech-to-Text**: Converts Filipino speech to text
- **Google Gemini AI**: Generates intelligent responses in Filipino
- **Google Cloud Text-to-Speech**: Converts text replies to Filipino audio
- **Arduino**: Controls physical servos and LEDs
- **Python**: Coordinates everything

---

## Libraries & Setup

[[#Table of Contents]]

```python
import serial
import re
import json
import time
import pygame
from google.cloud import speech
from google.cloud import texttospeech
from google.oauth2 import service_account
import google.generativeai as genai
import wave
import sounddevice as sd
import numpy as np
import io
```

### What each library does:

| Library                         | Purpose                                            | Key Commands/Methods                                  |
| ------------------------------- | -------------------------------------------------- | ----------------------------------------------------- |
| `serial`                        | Communicates with Arduino via USB                  | `Serial()`, `.write()`, `.read()`                     |
| `re`                            | Pattern matching to extract JSON from AI responses | `re.search()`, `re.DOTALL`                            |
| `json`                          | Parses JSON data (AI responses, actions)           | `json.loads()`, `.get()`                              |
| `time`                          | Adds delays and tracks timing                      | `time.sleep()`, `time.time()`                         |
| `pygame`                        | Plays audio files (robot's voice)                  | `mixer.init()`, `.play()`, `.get_busy()`              |
| `google.cloud.speech`           | Converts voice to text                             | `SpeechClient()`, `.recognize()`                      |
| `google.cloud.texttospeech`     | Converts text to voice                             | `TextToSpeechClient()`, `.synthesize_speech()`        |
| `google.oauth2.service_account` | Authenticates with Google services                 | `Credentials.from_service_account_file()`             |
| `google.generativeai`           | Connects to Gemini AI                              | `configure()`, `GenerativeModel()`, `.send_message()` |
| `wave`                          | Saves audio recordings as WAV files                | `wave.open()`, `.writeframes()`                       |
| `sounddevice`                   | Records audio from microphone                      | `sd.InputStream()`, `.read()`                         |
| `numpy`                         | Processes audio data (numbers/arrays)              | `np.linalg.norm()`, `.concatenate()`                  |
| `io`                            | Handles file operations in memory                  | `io.open()`                                           |


---

## Configuration

[[#Table of Contents]]

```python
# Configuration
GEMINI_API_KEY = "AIzaSyDZUoCSm1DFrl4dPNUed4Q9UI8r6S8OK-g"
GOOGLE_CLOUD_KEY = r"C:\extreme-lore-473311-h1-f1081b7fa77e.json"
credentials = service_account.Credentials.from_service_account_file(GOOGLE_CLOUD_KEY)
speech_client = speech.SpeechClient(credentials=credentials)
tts_client = texttospeech.TextToSpeechClient(credentials=credentials)
```

### Line-by-line explanation:

1. **GEMINI_API_KEY**: Your secret key to access Google's Gemini AI
2. **GOOGLE_CLOUD_KEY**: Path to your Google Cloud credentials file (JSON format)
    - `r""` = raw string (treats backslashes literally, important for Windows paths)
3. **credentials**: Loads authentication from the JSON file
4. **speech_client**: Creates connection to Speech-to-Text service
5. **tts_client**: Creates connection to Text-to-Speech service

### Python Syntax Explained:

**Raw Strings (`r""`)**:

```python
# Without r: backslash causes problems
path = "C:\new\file.txt"  # \n means newline!

# With r: treats \ as literal character
path = r"C:\new\file.txt"  # Correct path
```


**Object Methods (.methodName())**:

```python
object.method()  # Calls a function belonging to that object
# Example: speech_client.recognize() calls the recognize function
```

### Arduino Connection:

```python
try:
    arduino = serial.Serial('COM11', 9600, timeout=1)
    time.sleep(2)  # Wait for Arduino to initialize
    print("Arduino connected.")
except Exception as e:
    print("Failed to connect to Arduino:", e)
    arduino = None
```

**What this does:**

- Tries to connect to Arduino on port COM11
- **9600** = communication speed (baud rate)
- **timeout=1** = wait 1 second max for Arduino response
- **time.sleep(2)** = gives Arduino 2 seconds to boot up
- If connection fails, continues without Arduino (for testing)

### Python Concepts Explained:


**Serial Port Connection**:

```python
serial.Serial(port, baudrate, timeout)
```
- `port`: **COM port** (COM11, COM3, etc.) - check Device Manager on Windows
- `baudrate`: Speed of communication (must match Arduino)
- `timeout`: How long to wait for data before giving up

### Understanding `time.sleep()`

```python
time.sleep(2)  # Pause for 2 seconds
```

**Why sleep after connecting?**

- Arduino resets when serial port opens
- Needs time to boot up
- Without sleep, first commands might be missed


---

## Audio Recording

[[#Table of Contents]]

```python
def record_audio(filename="input.wav", samplerate=16000, silence_threshold=0.0002, 
                 silence_duration=1.2, max_duration=10):
```

### Parameters:

|Parameter|Default|Purpose|
|---|---|---|
|`filename`|"input.wav"|Where to save the recording|
|`samplerate`|16000|Audio quality (16,000 samples/second)|
|`silence_threshold`|0.0002|How loud before considering it "voice"|
|`silence_duration`|1.2|Seconds of silence before stopping|
|`max_duration`|10|Maximum recording length (safety)|

### How it works:

```python
print("Waiting for voice")
buffer = []
silent_chunks = 0
started = False
chunk_duration = 0.1  # seconds
chunk_samples = int(samplerate * chunk_duration)
silence_limit_chunks = int(silence_duration / chunk_duration)
start_time = time.time()
```

**Setup:**

- `buffer`: Stores all audio chunks (pieces of recording)
- `silent_chunks`: Counts how many quiet chunks in a row
- `started`: Flag to track if voice was detected
- `chunk_duration`: Record in 0.1 second pieces
- `chunk_samples`: How many audio samples in each chunk (1,600)
- `silence_limit_chunks`: How many quiet chunks = done (12 chunks = 1.2 sec)

```python
with sd.InputStream(samplerate=samplerate, channels=1, dtype='float32') as stream:
    while True:
        audio_chunk, _ = stream.read(chunk_samples)
        volume_norm = np.linalg.norm(audio_chunk) / len(audio_chunk)
```

**Recording loop:**

1. Opens microphone stream
2. Reads one chunk at a time
3. Calculates volume (loudness) of that chunk

### Python & Audio Concepts:

**With Statement**:

```python
with sd.InputStream(...) as stream:
    # Code here
# Stream automatically closes after 'with' block
```

**Why use `with`?**

- Automatically cleans up resources
- Closes stream even if error occurs
- Prevents memory leaks

**Understanding Audio Streams**:

```python
sd.InputStream(samplerate, channels, dtype)
```
- `samplerate`: Samples per second (16000 = 16kHz)
- `channels`: 1 = mono, 2 = stereo
- `dtype`: Data type ('float32' = decimal numbers)


**Reading Audio Data**:

```python
audio_chunk, _ = stream.read(chunk_samples)
# Returns: (audio_data, overflowed)
# _ means "ignore second value"
```

**Volume Calculation**:

```python
volume_norm = np.linalg.norm(audio_chunk) / len(audio_chunk)
```
- `np.linalg.norm()` = measures "loudness" mathematically
- Dividing by length = average loudness per sample


### Troubleshooting Audio Recording:

**Problem: Recording too sensitive (picks up everything)**

```python
# Increase threshold
silence_threshold = 0.001  # Higher = needs louder sound
```

**Problem: Not detecting voice**

```python
# Lower threshold
silence_threshold = 0.0001  # Lower = more sensitive
```

**Problem: Recording cuts off mid-sentence**

```python
# Increase silence duration
silence_duration = 2.0  # Wait 2 seconds of silence before stopping
```

**Problem: Records forever**

```python
# Decrease max duration
max_duration = 5  # Force stop after 5 seconds
```

```python
        print("Volume:", round(volume_norm, 4))
        if volume_norm > silence_threshold:
            buffer.append(audio_chunk.copy())
            silent_chunks = 0
            if not started:
                print("Recording")
                started = True
```

**Voice detection:**

- Prints volume so you can see if mic is working
- If louder than threshold: save chunk, reset silence counter, start recording
- First time voice detected: print "Recording"

```python
        elif started:
            buffer.append(audio_chunk.copy())
            silent_chunks += 1
            if silent_chunks > silence_limit_chunks:
                print("Stopping")
                break
```

**Silence detection:**

- If started recording and now quiet: still save chunk (might be gap between words)
- Count silent chunks
- If too many silent chunks: person stopped talking, break loop

```python
        # Timeout protection
        if time.time() - start_time > max_duration:
            print("Max duration reached, stopping.")
            break
```

**Safety:**

- If recording goes over 10 seconds, force stop (prevents infinite recording)

```python
if not buffer:
    print("No voice detected.")
    return None

audio_data = np.concatenate(buffer, axis=0)
with wave.open(filename, "wb") as wf:
    wf.setnchannels(1)
    wf.setsampwidth(2)
    wf.setframerate(samplerate)
    wf.writeframes((audio_data * 32767).astype(np.int16).tobytes())

print("Recording done.", filename)
return filename
```

**Saving:**

1. If no voice detected, return None
2. Combine all chunks into one audio file
3. Open WAV file for writing
4. Set audio properties (1 channel = mono, 2 bytes per sample, 16000 Hz)
5. Convert audio data to proper format and save
6. Return filename

---

## Speech Recognition

[[#Table of Contents]]

```python
def transcribe_audio(filename):
    with io.open(filename, "rb") as f:
        audio_data = f.read()
```

**Step 1: Load the audio file**

- Opens the WAV file in binary mode ("rb" = read binary)
- Reads all audio data into memory

```python
    audio = speech.RecognitionAudio(content=audio_data)
    config = speech.RecognitionConfig(
        encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
        sample_rate_hertz=16000,
        language_code="fil-PH"
    )
```

**Step 2: Configure recognition**

- `audio`: Wraps audio data for Google API
- `config`: Settings for recognition
    - **LINEAR16**: Audio format (16-bit PCM)
    - **16000**: Sample rate (must match recording)
    - **"fil-PH"**: Filipino (Philippines) language

```python
    response = speech_client.recognize(config=config, audio=audio)
    if response.results:
        transcript = response.results[0].alternatives[0].transcript
        print(f"You said: {transcript}")
        return transcript
    else:
        print("No speech detected.")
        return ""
```

**Step 3: Get transcription**

- Send audio to Google Speech API
- If speech found: extract text from first result
- Print what was heard
- Return the transcript (or empty string if nothing)

---

## AI System Prompt

[[#Table of Contents]]

```python
system_prompt = """
You are a robot assistant AI. Only answer in Filipino. Your response must be a valid JSON object...
```

This long text teaches the AI how to behave. Key sections:

### 1. Identity

```
You are a robot crab. Your name is Scylla, and you are based from the sea monster in Greek mythology.
```

- Gives AI personality and backstory

### 2. Valid Actions

```
**VALID ACTION LIST:**
1. **`blink`** – Controls two single-color LEDs.
2. **`servo`** – Controls servo motor movements.
3. **`composite`** – Performs multiple actions in sequence.
```

- Lists what physical actions Scylla can perform

### 3. Emotional Logic

```
**Scylla's Emotional Logic:**
- Happy/friendly → blink both LEDs twice (`blink_both`) + wave arms.
- Curious/thinking → alternate LED blinking slowly.
```

- Maps emotions to actions (teaches AI when to use each movement)

### 4. Output Format

```
{
  "reply": "Magandang araw din sa'yo! Scylla here, ready to lend a claw!",
  "action": {
    "type": "composite",
    "parameters": { ... }
  }
}
```

- Requires AI to always respond with JSON
- **reply**: What Scylla says
- **action**: What Scylla does

---

## Gemini AI Integration

[[#Table of Contents]]

```python
genai.configure(api_key=GEMINI_API_KEY)
model = genai.GenerativeModel('models/gemini-2.5-flash', system_instruction=system_prompt)
chat = model.start_chat(history=[])
```

**Setup:**

1. **configure**: Authenticates with Gemini using API key
2. **GenerativeModel**: Loads the Gemini 2.5 Flash model
    - `system_instruction`: Gives AI the Scylla personality
3. **start_chat**: Begins conversation (empty history)

```python
def call_gemini(prompt):
    print("Sending prompt to AI...")
    response = chat.send_message(prompt)
    full_response_text = response.text.strip()
    print(f"\nAI Raw JSON Response:\n{full_response_text}")
```

**Sending message:**

1. Print status
2. Send user's text to AI
3. Get response and remove extra whitespace
4. Print raw response for debugging

```python
    llm_reply = "Paumanhin, nagkaroon ng error."
    action_obj = {"type": "none"}

    match = re.search(r'\{.*\}', full_response_text, re.DOTALL)
```

**Default values & JSON extraction:**

- Set error message as default reply
- Set "no action" as default
- Search for JSON object in response (everything between `{` and `}`)
- `re.DOTALL`: Allows matching across multiple lines

```python
    if match:
        json_str = match.group(0)
        try:
            data = json.loads(json_str)
            llm_reply = data.get("reply", llm_reply)
            action_obj = data.get("action", action_obj) or {"type": "none"}
        except json.JSONDecodeError:
            print("JSON error!")
    else:
        print("Error: No JSON object found.")
    return llm_reply, action_obj
```

**Parsing:**

1. If JSON found: extract it
2. Try to parse as JSON object
3. Get "reply" and "action" fields (use defaults if missing)
4. If JSON invalid: print error, use defaults
5. Return reply text and action object

---

## Text-to-Speech

[[#Table of Contents]]

```python
def call_google_tts(text, filename="reply.mp3", voice_name="fil-PH-Wavenet-B"):
    synthesis_input = texttospeech.SynthesisInput(text=text)
```

**Step 1: Prepare text**

- Wraps the reply text for TTS API

```python
    voice = texttospeech.VoiceSelectionParams(language_code="fil-PH", name=voice_name)
    audio_config = texttospeech.AudioConfig(audio_encoding=texttospeech.AudioEncoding.MP3)
```

**Step 2: Configure voice**

- **language_code**: Filipino (Philippines)
- **voice_name**: "fil-PH-Wavenet-B" (female Filipino voice)
- **audio_config**: Save as MP3 format

```python
    response = tts_client.synthesize_speech(input=synthesis_input, voice=voice, audio_config=audio_config)

    with open(filename, "wb") as out:
        out.write(response.audio_content)
    print(f"Done.")
    return filename
```

**Step 3: Generate & save**

1. Send to Google TTS API
2. Open file for writing binary ("wb")
3. Write audio data to file
4. Return filename

---

## Audio Playback

[[#Table of Contents]]

```python
def play_audio(filename: str):
    pygame.mixer.init()
    pygame.mixer.music.load(filename)
    pygame.mixer.music.play()
```

**Step 1: Play audio**

1. Initialize pygame audio mixer
2. Load the MP3 file
3. Start playback

```python
    while pygame.mixer.music.get_busy():
        pygame.time.Clock().tick(10)
```

**Step 2: Wait for completion**

- Loop while audio is playing
- Check 10 times per second
- Prevents program from continuing before audio finishes

```python
    pygame.mixer.music.unload()
    pygame.mixer.quit()
```

**Step 3: Cleanup**

- Unload audio file from memory
- Close audio system (allows next playback)

---

## Action Triggering

[[#Table of Contents]]

```python
def trigger_action(action_obj):
    action_type = action_obj.get("type", "none")
    params = action_obj.get("parameters", {})
```

**Extract action info:**

- Get action type (servo, blink, composite, none)
- Get parameters (speed, move, pattern, etc.)

```python
    if action_type == "composite":
        for sub_action in params.get("actions", []):
            trigger_action(sub_action)
            time.sleep(0.5)
        return
```

**Composite actions:**

- If multiple actions combined
- Loop through each sub-action
- Recursively call trigger_action for each
- Wait 0.5 seconds between actions

```python
    if action_type == "servo":
        target = params.get("target", "arms")
        move = params.get("move", "rest")
        speed = params.get("speed", "normal")
        repeat = params.get("repeat", 1)
        cmd = f"SERVO:target={target},move={move},speed={speed},repeat={repeat}\n"
        print(f"Sending to Arduino: {cmd.strip()}")
        if arduino:
            arduino.write(cmd.encode())
        return
```

**Servo movements:**

1. Extract parameters (with defaults if missing)
2. Build command string: `SERVO:target=arms,move=wave,speed=normal,repeat=1`
3. Print command for debugging
4. If Arduino connected: send command
    - `.encode()`: Convert string to bytes
    - `\n`: Newline tells Arduino "command complete"

```python
    if action_type == "blink":
        pattern = params.get("pattern", "blink_both")
        delay = params.get("delay", "300")
        repeat = params.get("repeat", "1")
        cmd = f"BLINK:pattern={pattern},delay={delay},repeat={repeat}\n"
        print(f"Sending to Arduino: {cmd.strip()}")
        if arduino:
            arduino.write(cmd.encode())
        return
```

**LED blinking:**

1. Extract parameters
2. Build command: `BLINK:pattern=blink_both,delay=300,repeat=2`
3. Send to Arduino

---

## Main Program Loop

[[#Table of Contents]]

```python
def main():
    while True:
```

**Infinite loop:**

- Runs forever (until Ctrl+C or error)

```python
        filename = record_audio()
        if filename:
            transcript = transcribe_audio(filename)
        else:
            print("No speech captured.")
            continue
```

**Step 1: Listen**

1. Record audio from microphone
2. If voice detected: transcribe to text
3. If no voice: print message and restart loop

```python
        prompt_with_context = transcript

        chat.history.append({"role": "user", "parts": [prompt_with_context]})
```

**Step 2: Add to conversation history**

- Save user's message to chat history
- Helps AI remember conversation context

```python
        llm_reply, action_obj = call_gemini(prompt_with_context)

        chat.history.append({"role": "model", "parts": [llm_reply]})
```

**Step 3: Get AI response**

1. Send message to Gemini
2. Get reply text and action
3. Save AI's reply to history

```python
        trigger_action(action_obj)
```

**Step 4: Perform action**

- Execute servo movements or LED blinks
- Sends commands to Arduino

```python
        audio_file = call_google_tts(llm_reply)
        play_audio(audio_file)

        print(f"\nReply: {llm_reply}")
```

**Step 5: Respond**

1. Convert AI's reply to speech (MP3)
2. Play audio through speakers
3. Print reply text to console

**Then loop repeats!**

---

## Program Startup

[[#Table of Contents]]

```python
if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nExiting...")
    finally:
        pygame.quit()
```

**What this does:**

1. `if __name__ == "__main__"`: Only runs if this is the main program
2. `try`: Attempt to run main loop
3. `except KeyboardInterrupt`: If user presses Ctrl+C, exit gracefully
4. `finally`: Always close pygame (cleanup)

---

## Complete Flow Summary

```
1. USER SPEAKS → Microphone records
2. AUDIO SAVED → input.wav file
3. SPEECH-TO-TEXT → Filipino text
4. SEND TO AI → Gemini processes
5. GET JSON → Reply + Action
6. TRIGGER ACTION → Arduino moves servos/LEDs
7. TEXT-TO-SPEECH → Filipino voice (reply.mp3)
8. PLAY AUDIO → Speakers output
9. REPEAT → Back to step 1
```


---

## Tips for Modification

### Add new actions:

1. Add to system prompt's VALID ACTION LIST
2. Add handling in `trigger_action()`
3. Implement in Arduino code

### Change AI model:

```python
model = genai.GenerativeModel('models/gemini-1.5-pro', ...)  # More powerful
model = genai.GenerativeModel('models/gemini-2.0-flash', ...)  # Faster
```

---
