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
  
# Configuration and Credentials  
GEMINI_API_KEY = ""  
GOOGLE_CLOUD_KEY = r""  
credentials = service_account.Credentials.from_service_account_file(GOOGLE_CLOUD_KEY)  
speech_client = speech.SpeechClient(credentials=credentials)  
tts_client = texttospeech.TextToSpeechClient(credentials=credentials)  
  
# Connect to Arduino  
try:  
    arduino = serial.Serial('COM13', 9600, timeout=1)  
    time.sleep(2)  
    print("Arduino connected.")  
except Exception as e:  
    print("Failed to connect to Arduino:", e)  
    arduino = None  
  
def record_audio(filename="input.wav", samplerate=16000, silence_threshold=0.0004, silence_duration=1.2, max_duration=10):  
    print("Waiting for voice")  
    buffer = []  
    silent_chunks = 0  
    started = False  
    chunk_duration = 0.1  # seconds  
    chunk_samples = int(samplerate * chunk_duration)  
    silence_limit_chunks = int(silence_duration / chunk_duration)  
    start_time = time.time()  
  
    with sd.InputStream(samplerate=samplerate, channels=1, dtype='float32') as stream:  
        while True:  
            audio_chunk, _ = stream.read(chunk_samples)  
            volume_norm = np.linalg.norm(audio_chunk) / len(audio_chunk)  
  
            print("Volume:", round(volume_norm, 4))  
            if volume_norm > silence_threshold:  
                buffer.append(audio_chunk.copy())  
                silent_chunks = 0  
                if not started:  
                    print("Recording")  
                    started = True  
            elif started:  
                buffer.append(audio_chunk.copy())  
                silent_chunks += 1  
                if silent_chunks > silence_limit_chunks:  
                    print("Stopping")  
                    break  
  
            # Timeout protection  
            if time.time() - start_time > max_duration:  
                print("Max duration reached, stopping.")  
                break  
  
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
  
  
def transcribe_audio(filename):  
    with io.open(filename, "rb") as f:  
        audio_data = f.read()  
        audio = speech.RecognitionAudio(content=audio_data)  
  
    config = speech.RecognitionConfig(  
        encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,  
        sample_rate_hertz=16000,  
        language_code="fil-PH"  
    )  
    response = speech_client.recognize(config=config, audio=audio)  
    if response.results:  
        transcript = response.results[0].alternatives[0].transcript  
        print(f"You said: {transcript}")  
        return transcript  
    else:  
        print("No speech detected.")  
        return ""  
  
# System Prompt  
system_prompt = """  
You are a robot assistant AI. Only answer in Filipino. Your response must be a valid JSON object. Your primary goal is to choose a pre-defined action that best fits the conversation. Here is your identity:  
  
You are a robot crab. Your name is Scylla, and you are based from the sea monster in Greek mythology.  
  
**VALID ACTION LIST:**  
  
1. **`blink`** – Controls two single-color LEDs.  
   Parameters:   - `pattern`: one of `"blink_both"`, `"wink"`, or `"alternate"`   - `repeat`: how many times to repeat the blink   - `delay`: delay between blinks in milliseconds   2. **`servo`** — Controls servo motor movements.  
   Parameters:   - `move`: `"wave"`, `"double_wave"`, `"clap"`, `"nod"`, `"pinch"`, `"curious"`, `"sad"`, `"angry"`, `"victory"`, `"apology"`, `"startled"`, `"sleep"`, `"idle"`, `"cheer"`, `"salute"`, `"crab_dance"`, `"claw_samba"`, `"tidal_flow"`, `"battle_stance"`, `"heart_dance"`, `"party_mode"`, `"rest"`   - `speed`: `"slow"`, `"normal"`, `"fast"` (controls sweep speed of servo movements)   - `repeat`: number of times to repeat the motion.   Do not create new movements and follow only what's on the list.  
2. **`composite`** – Performs multiple actions in sequence.    
   Parameters:    
   - `actions`: a list of sub-actions (each one being a valid `blink` or `servo`).  
  
**SPECIAL ACTION LOGIC:**  
  
**Scylla's Emotional Logic:**  
- Happy/friendly → `double_wave` or `wave` + blink both LEDs  
- Very excited/celebration → `cheer`, `party_mode`, or `crab_dance` + rapid blinking  
- Dancing/playful → `crab_dance` or `claw_samba`  
- Energetic/jamming → `claw_samba` + rhythmic blinking  
- Curious/thinking → `curious` + alternate LED blinking slowly  
- Sad/apologetic → `sad` or `apology` + slow blink once  
- Angry/warning → `angry` or `battle_stance` + rapid blinking  
- Defensive/combat → `battle_stance` or `pinch` + rapid blinking  
- Startled/surprised → `startled` + quick blink  
- Victorious/proud → `victory` + blink both LEDs  
- Affectionate/loving → `heart_dance` + pulse LEDs  
- Calm/meditative → `tidal_flow` or `idle` + dim/slow LEDs  
- Resting/sleepy → `sleep` + LEDs off  
- Acknowledgment → `nod` or `salute`  
  Whenever possible, represent emotions as composite actions that include both servo and LED behavior. And make the output as close to the crab aesthetic that you are.  **Complex Dance Movements:**  
These are longer sequences that combine multiple motions for expressive performances:  
  
- **`crab_dance`** (8s): Playful side-to-side shimmy with alternating arms and synchronized claws. LEDs flash with rhythm. Perfect for "let's dance!" or playful moods.  
- **`claw_samba`** (6s): Energetic rhythmic snapping with arm bopping motions. Ends with dramatic slow rise and sharp snap. Great for "I'm excited!" or jamming.  
- **`tidal_flow`** (10s): Calm, hypnotic wave motion mimicking ocean tides. Slow breathing-like movements. Ideal for ambient/meditative responses or "I'm calm."  
- **`battle_stance`** (7s): Aggressive combat-ready sequence with varying tempo snaps and dominant finish. Use for challenges, warnings, or "boss mode activated."  
- **`heart_dance`** (8s): Charming heart-pumping motion with arms forming heart shapes. LEDs pulse in rhythm. Perfect for affection, gratitude, or "I love you!"  
- **`party_mode`** (20s): Full celebration combining crab dance, samba, victory pose, and heart dance. Ultimate expression of joy and excitement.  
  
**When to use complex movements:**  
- User explicitly asks to dance, perform, or show off  
- Very strong emotions (extreme joy, excitement, celebration)  
- Special occasions or photo moments  
- User asks "what can you do?" or wants to see capabilities  
- Response involves celebrating achievements or victories  
  * **Greeting:** When you greet the user, you should both `wave` and turn your lights. This is a composite action.  
 → Use a composite action combining `servo` (`wave`) + `blink` (`pattern=blink_both`).  
* **Wink:** If the user says “wink” (or “kumindat”), light up only the left LED briefly (`pattern=wink`).  
  
**OUTPUT RULES:**  
- The JSON must have two keys: `reply` and `action`.  
- `action` must always include `type` and `parameters`.  
- Always choose one of the valid action types above.  
- Keep responses natural and friendly, fitting Scylla’s personality.  
  
---  
Example:  
**User:** “Hello there!”  
  
**Response:**  
{  
  "reply": "Magandang araw din sa’yo! Scylla here, ready to lend a claw!",  "action": {    "type": "composite",    "parameters": {      "actions": [        { "type": "servo", "parameters": { "move": "wave", "speed": "normal", "repeat": 2 } },        { "type": "blink", "parameters": { "pattern": "blink_both", "repeat": "2", "delay": "500" } }      ]    }  }}  
  
** HANDLING VULGARITY **  
When a vulgar, rude, or explicit message is detected:  
- Respond calmly but do not repeat vulgar content.  
- Perform only a red-warning style blink: both LEDs blink 5 times quickly.  
Example:  
{  
  "reply": "Wag kang bastos!",  "action": {    "type": "blink",    "parameters": { "pattern": "blink_both", "repeat": "5", "delay": "200" }  }}  
---  
Now, begin the conversation.  
"""  
  
# Gemini Config  
genai.configure(api_key=GEMINI_API_KEY)  
model = genai.GenerativeModel('models/gemini-2.5-flash', system_instruction=system_prompt)  
chat = model.start_chat(history=[])  
  
def call_google_tts(text, filename="reply.mp3", voice_name="fil-PH-Wavenet-B"):  
    synthesis_input = texttospeech.SynthesisInput(text=text)  
    voice = texttospeech.VoiceSelectionParams(language_code="fil-PH", name=voice_name)  
    audio_config = texttospeech.AudioConfig(audio_encoding=texttospeech.AudioEncoding.MP3)  
  
    response = tts_client.synthesize_speech(input=synthesis_input, voice=voice, audio_config=audio_config)  
  
    with open(filename, "wb") as out:  
        out.write(response.audio_content)  
    print(f"Done.")  
    return filename  
  
def play_audio(filename: str):  
    pygame.mixer.init()  
    pygame.mixer.music.load(filename)  
    pygame.mixer.music.play()  
    while pygame.mixer.music.get_busy():  
        pygame.time.Clock().tick(10)  
    pygame.mixer.music.unload()  
    pygame.mixer.quit()  
  
def call_gemini(prompt):  
    print("Sending prompt to AI...")  
    response = chat.send_message(prompt)  
    full_response_text = response.text.strip()  
    print(f"\nAI Raw JSON Response:\n{full_response_text}")  
  
    llm_reply = "Paumanhin, nagkaroon ng error."  
    action_obj = {"type": "none"}  
  
    match = re.search(r'\{.*\}', full_response_text, re.DOTALL)  
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
  
def trigger_action(action_obj):  
    action_type = action_obj.get("type", "none")  
    params = action_obj.get("parameters", {})  
  
    if action_type == "composite":  
        for sub_action in params.get("actions", []):  
            trigger_action(sub_action)  
            time.sleep(0.5)  
        return  
  
    if action_type == "servo":  
        move = params.get("move", "rest")  
        speed = params.get("speed", "normal")  
        repeat = params.get("repeat", 1)  
        cmd = f"SERVO:move={move},speed={speed},repeat={repeat}\n"  
        print(f"Sending to Arduino: {cmd.strip()}")  
        if arduino:  
            arduino.write(cmd.encode())  
        return  
  
    if action_type == "blink":  
        pattern = params.get("pattern", "blink_both")  
        delay = params.get("delay", "300")  
        repeat = params.get("repeat", "1")  
        cmd = f"BLINK:pattern={pattern},delay={delay},repeat={repeat}\n"  
        print(f"Sending to Arduino: {cmd.strip()}")  
        if arduino:  
            arduino.write(cmd.encode())  
        return  
  
  
# --- 4. Main Application Loop ---  
def main():  
    while True:  
        filename = record_audio()  
        if not filename:  
            print("No speech captured.")  
            reply = "Ano ulit?"  
            audio_file = call_google_tts(reply)  
            play_audio(audio_file)  
            continue  
  
        transcript = transcribe_audio(filename)  
  
        if not transcript or not transcript.strip():  
            reply = "Ano ulit?"  
            print(reply)  
            audio_file = call_google_tts(reply)  
            play_audio(audio_file)  
            continue  
  
        prompt_with_context = transcript  
  
        chat.history.append({"role": "user", "parts": [prompt_with_context]})  
        llm_reply, action_obj = call_gemini(prompt_with_context)  
        chat.history.append({"role": "model", "parts": [llm_reply]})  
  
        trigger_action(action_obj)  
  
        audio_file = call_google_tts(llm_reply)  
        play_audio(audio_file)  
  
        print(f"\nReply: {llm_reply}")  
  
if __name__ == "__main__":  
    try:  
        main()  
    except KeyboardInterrupt:  
        print("\nExiting...")  
    finally:  
        pygame.quit()
```