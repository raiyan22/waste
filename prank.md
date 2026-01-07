📋 THE ULTIMATE CHECKLIST
What you need:

Windows PC with RTX 3060.
Reference Audio: A clear reference.wav file of the 38-year-old Eurasian female voice.
Admin Rights: To install the audio driver.
STEP 1: Install the Virtual Audio Driver (VB-Cable)
We need a "virtual microphone" that your Python script talks into.

Go to VB-Audio.com.
Download and install VB-Cable.
Restart your computer. (This is mandatory for the driver to show up).
STEP 2: Install Python & Libraries
Open Command Prompt (cmd) or PowerShell.
Install the necessary libraries:
bash

pip install TTS torch sounddevice numpy
Note: torch will automatically detect your RTX 3060 if your NVIDIA drivers are up to date.

STEP 3: Setup the Audio Loop (The "Secret Sauce")
We need to configure Windows so that the AI's voice goes to both your headphones (so you can hear it) and Telegram (so your friend hears it).

A. Set the Default Output to the Cable
Right-click the Speaker Icon in your taskbar -> Sound Settings.
Click the dropdown next to "Choose where to play sound".
Select CABLE Input.
Now, your computer thinks the virtual cable is your speakers.
B. Let Yourself Hear the Audio (Monitor Mode)
If you stop here, the audio goes into the cable, and you hear nothing. We need to route it to your ears.

Right-click the Speaker Icon -> Sound Control Panel (Or click "More sound settings" at the bottom of the Settings page).
Go to the Recording tab.
Find CABLE Output. Right-click it -> Properties.
Go to the Listen tab.
Check the box that says "Listen to this device".
Below that, under "Playback through this device", select your actual Headphones (or Speakers).
Click Apply -> OK.
Result: Audio flows from Python -> Cable Input -> Cable Output -> Split to (Your Headphones + Telegram).*

STEP 4: The Prank Engine Script
Create a new folder on your desktop. Name it PrankBot. Inside it:

Put your reference.wav file there.
Create a file named engine.py.
Paste this code into engine.py:

python

import tkinter as tk
        
        with self.lock:
            if len(self.buffer) > 0:
                chunk = self.buffer[0]
                if len(chunk) > frames:
                    outdata[:] = chunk[:frames].reshape(-1, 1)
                    self.buffer[0] = chunk[frames:]
                else:
                    # Pad end of chunk with zeros if shorter than frame size
                    outdata[:len(chunk)] = chunk.reshape(-1, 1)
                    outdata[len(chunk):] = 0
                    self.buffer.pop(0)
            else:
                outdata[:] = 0 # Silence

    def generate_speech(self, text):
        """Generates audio in a background thread"""
        try:
            wav = self.tts.tts(text=text, speaker_wav=REFERENCE_AUDIO, language=LANGUAGE)
            wav_float = np.array(wav, dtype=np.float32)
            with self.lock:
                self.buffer.append(wav_float)
        except Exception as e:
            print(f"❌ TTS Error: {e}")

# --- GUI ---
class PrankApp:
    def __init__(self, root, engine):
        self.root = root
        self.engine = engine
        self.root.title("Telegram Prank Engine v1.0")
        self.root.geometry("500x600")
        self.root.configure(bg="#2c3e50")

        # Header
        lbl = tk.Label(root, text="TYPE WHAT SHE SAYS", font=("Helvetica", 16, "bold"), fg="white", bg="#2c3e50")
        lbl.pack(pady=15)

        # Text Input
        self.text_area = scrolledtext.ScrolledText(root, wrap=tk.WORD, height=12, font=("Consolas", 12), bg="#ecf0f1")
        self.text_area.pack(padx=20, pady=5, fill=tk.BOTH, expand=True)
        
        # Presets
        preset_frame = tk.Frame(root, bg="#2c3e50")
        preset_frame.pack(pady=10)
        
        presets = [
            "Hey, it's been so long.",
            "Can you hear me okay?",
            "I really missed talking to you.",
            "Who is that girl in your room?"
        ]
        
        for txt in presets:
            btn = tk.Button(preset_frame, text=txt, command=lambda t=txt: self.trigger(t), bg="#34495e", fg="white")
            btn.pack(side=tk.LEFT, padx=5)

        # Main Action Button
        self.speak_btn = tk.Button(root, text="🎙️ SPEAK TO CALL 🎙️", bg="#e74c3c", fg="white", 
                                   font=("Helvetica", 18, "bold"), command=self.on_click)
        self.speak_btn.pack(pady=20, fill=tk.X, padx=40)

    def on_click(self):
        text = self.text_area.get("1.0", tk.END).strip()
        if text:
            threading.Thread(target=self.engine.generate_speech, args=(text,), daemon=True).start()

    def trigger(self, text):
        self.text_area.delete("1.0", tk.END)
        self.text_area.insert(tk.END, text)
        self.on_click()

# --- MAIN ---
if __name__ == "__main__":
    try:
        engine = AudioEngine()
        root = tk.Tk()
        app = PrankApp(root, engine)
        
        def on_close():
            engine.stream.stop()
            engine.stream.close()
            root.destroy()
            
        root.protocol("WM_DELETE_WINDOW", on_close)
        root.mainloop()
    except Exception as e:
        print(f"CRITICAL ERROR: {e}")
        input("Press Enter to exit...")
STEP 5: Configure Telegram
Open Telegram Desktop.
Go to Settings -> Advanced -> Call Settings.
Input Device: Select CABLE Output.
Output Device: Select your Headphones (So you can hear your friend's actual voice).
Note: Do NOT set Output to Cable Input, or your friend will hear an echo of themselves.
STEP 6: EXECUTE THE PRANK
Follow this order exactly:

Run the Script: Double-click engine.py.
Wait: Wait for the terminal window to say "Model Ready! Let's prank."
Start the Call: Call your friend on Telegram.
Start Talking: Type "Hello?" in the box and click the big RED BUTTON.
What happens:

The script generates the voice using your 3060 (fast).
The audio plays through "CABLE Input".
Windows loops it back to your headphones (so you hear the woman).
Telegram captures it from "CABLE Output" and sends it to your friend.
Your friend hears: The 38-year-old Eurasian woman.
You hear: The 38-year-old Eurasian woman + Your friend's reply.

TROUBLESHOOTING
"I hear nothing, but my friend hears the voice":
Go to Sound Control Panel -> Recording -> CABLE Output -> Properties -> Check "Listen to this device". (See Step 3B).
"My friend hears nothing" (or hears static):
Check Telegram Call Settings. Ensure Input is CABLE Output.
Ensure you didn't mute the "CABLE Output" device in the Sound settings.
The script crashes:
Make sure reference.wav is in the same folder as the script.
Make sure you installed pip install TTS torch sounddevice numpy.
The voice sounds robotic or distorted:
Your reference.wav might be too quiet or have background noise. Try a cleaner, louder sample.
