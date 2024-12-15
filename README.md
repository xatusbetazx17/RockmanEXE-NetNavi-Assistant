# RockmanEXE-NetNavi-Assistant

~~~
import os
import sys
import glob
import subprocess
import platform
import shutil
import json
import random
import time
import tkinter as tk
from tkinter import filedialog

REQUIRED_LIBRARIES = [
    "pyttsx3",
    "SpeechRecognition",
    "textblob",
]

def install_system_package(package_name):
    if shutil.which("pacman"):
        try:
            subprocess.check_call(["sudo", "pacman", "-Syu", "--noconfirm", package_name])
            print(f"[Setup] {package_name} installed successfully.")
            return True
        except subprocess.CalledProcessError:
            print(f"[Error] Failed to install {package_name} using pacman.")
            return False
    else:
        print("[Warning] Pacman not found. Cannot install system packages automatically.")
        return False

def install_system_pyaudio():
    print("[Setup] Detected Arch-based system. Installing python-pyaudio...")
    return install_system_package("python-pyaudio")

def ensure_espeak_ng():
    if shutil.which("espeak-ng") is None:
        print("[Setup] espeak-ng not found, installing...")
        install_system_package("espeak-ng")

def create_virtual_environment(venv_dir):
    if not os.path.exists(venv_dir):
        print("[Setup] Creating virtual environment...")
        subprocess.check_call([sys.executable, "-m", "venv", venv_dir])
    else:
        print("[Setup] Virtual environment already exists.")

def add_venv_site_packages_to_path(venv_dir):
    site_packages_dirs = glob.glob(os.path.join(venv_dir, 'lib', 'python*', 'site-packages'))
    if not site_packages_dirs:
        site_packages_dirs = glob.glob(os.path.join(venv_dir, 'Lib', 'site-packages'))

    if not site_packages_dirs:
        raise FileNotFoundError("[Error] Could not locate the site-packages directory in the virtual environment.")

    for sp in site_packages_dirs:
        if sp not in sys.path:
            sys.path.insert(0, sp)
    print("[Setup] Added virtual environment site-packages to sys.path.")

def install_packages(venv_dir):
    pip_exec = os.path.join(venv_dir, "bin", "pip")
    if os.name == "nt":
        pip_exec = os.path.join(venv_dir, "Scripts", "pip")

    print("[Setup] Upgrading pip, wheel, and setuptools...")
    subprocess.check_call([pip_exec, "install", "--upgrade", "pip", "wheel", "setuptools"])

    pyaudio_installed = False
    if shutil.which("pacman"):
        if install_system_pyaudio():
            try:
                import pyaudio  # noqa: F401
                pyaudio_installed = True
            except ImportError:
                pass

    all_libraries = REQUIRED_LIBRARIES
    if not pyaudio_installed:
        all_libraries.append("pyaudio")

    for library in all_libraries:
        try:
            __import__(library)
            print(f"[Setup] {library} is already installed.")
        except ImportError:
            print(f"[Setup] Installing {library}...")
            subprocess.check_call([pip_exec, "install", library])

    try:
        from textblob import download_corpora
        print("[Setup] Downloading TextBlob corpora...")
        download_corpora()
    except Exception as e:
        print(f"[Warning] Could not download TextBlob corpora: {e}")

def setup_virtual_environment():
    venv_dir = os.path.join(os.getcwd(), "env")
    create_virtual_environment(venv_dir)
    add_venv_site_packages_to_path(venv_dir)
    install_packages(venv_dir)
    ensure_espeak_ng()
    print("[Setup] Virtual environment and packages are ready.")

# --- Multi-Personality World Model System ---

WORLD_MODELS_DIR = "world_models"
DEFAULT_WORLD_MODEL = "default.json"

def load_world_model(path):
    if os.path.exists(path):
        with open(path, "r") as f:
            return json.load(f)
    else:
        # Return a default structure if not found
        model = {
            "assistant_name": "Rockman.EXE",
            "greeting": "Online! Ready to assist!",
            "facts": []
        }
        return model

def save_world_model(path, model):
    with open(path, "w") as f:
        json.dump(model, f, indent=2)

def extract_facts_from_input(user_input):
    facts = []
    lowered = user_input.lower()

    if "my name is" in lowered:
        name_part = lowered.split("my name is")[-1].strip()
        if name_part:
            name = name_part.split()[0]
            facts.append(f"User name: {name.capitalize()}")

    if "i like" in lowered:
        like_part = lowered.split("i like")[-1].strip()
        if like_part:
            facts.append(f"User likes {like_part}")
    return facts

def incorporate_facts_into_response(response, world_model):
    user_name = None
    user_likes = []
    for fact in world_model["facts"]:
        if fact.startswith("User name:"):
            user_name = fact.split("User name:")[-1].strip()
        if fact.startswith("User likes"):
            user_likes.append(fact.split("User likes")[-1].strip())

    if user_name and "Operator" in response:
        response = response.replace("Operator", user_name)

    if user_likes and random.random() < 0.4:
        liked_thing = random.choice(user_likes)
        response += f" I remember you like {liked_thing}."

    if "{assistant_name}" in response:
        response = response.replace("{assistant_name}", world_model.get("assistant_name", "Rockman.EXE"))

    return response

def detect_emotion(user_input):
    from textblob import TextBlob
    analysis = TextBlob(user_input).sentiment
    polarity = analysis.polarity
    # emotion mapping
    if polarity > 0.7:
        return "excited"
    elif polarity > 0.3:
        return "happy"
    elif polarity > 0:
        return "neutral"
    elif polarity > -0.3:
        return "curious"
    elif polarity > -0.7:
        return "worried"
    else:
        return "angry"

def prompt_for_personality_file():
    root = tk.Tk()
    root.withdraw()  # Hide the main window
    root.lift()  
    root.attributes('-topmost', True)

    # Prompt the user for a personality file
    file_path = filedialog.askopenfilename(
        title="Select personality JSON file (cancel to use default)",
        filetypes=[("JSON files", "*.json")]
    )
    root.destroy()

    return file_path

def run_script():
    print("[Assistant] Running Rockman.EXE Assistant...")

    # Prompt user for personality file
    chosen_file = prompt_for_personality_file()
    if not chosen_file or not os.path.exists(chosen_file):
        # If no file selected or file doesn't exist, default to default.json
        chosen_file = os.path.join(WORLD_MODELS_DIR, DEFAULT_WORLD_MODEL)
        if not os.path.exists(chosen_file):
            # Ensure default model is created
            default_model = {
                "assistant_name": "Rockman.EXE",
                "greeting": "Online! Ready to assist!",
                "facts": []
            }
            os.makedirs(WORLD_MODELS_DIR, exist_ok=True)
            save_world_model(chosen_file, default_model)

    world_model = load_world_model(chosen_file)

    import pyttsx3
    import speech_recognition as sr

    engine = pyttsx3.init()
    engine.setProperty("rate", 150)

    responses = {
        "excited": [
            "Wow, that's awesome!",
            "I'm so excited to hear that!",
            "Fantastic! Let's keep this energy going!"
        ],
        "happy": [
            "I'm glad you're feeling great!",
            "That sounds wonderful! Let's keep it up!",
            "Great to hear! How else can I help?"
        ],
        "neutral": [
            "Alright, what else can I do for you?",
            "Understood. Let’s move forward.",
            "Got it. Anything else you need?"
        ],
        "curious": [
            "Hmm, that's interesting. Can you tell me more?",
            "I'm curious about that. What did you mean exactly?",
            "That's intriguing. Would you like to expand on that?"
        ],
        "worried": [
            "I'm sensing some concern. Is there something bothering you?",
            "You seem worried. I'm here if you want to talk about it.",
            "I notice some worry in your tone. Let’s figure this out together."
        ],
        "angry": [
            "I can tell you're upset. Let’s take a moment to calm down.",
            "I understand you're angry. Let’s work through this together.",
            "I’m here to assist, even when things are tough."
        ]
    }

    def speak(text):
        assistant_name = world_model.get("assistant_name", "Rockman.EXE")
        print(f"{assistant_name}: {text}")
        engine.say(text)
        engine.runAndWait()

    def listen_to_user():
        recognizer = sr.Recognizer()
        with sr.Microphone() as source:
            recognizer.adjust_for_ambient_noise(source, duration=1)
            assistant_name = world_model.get("assistant_name", "Rockman.EXE")
            print(f"{assistant_name}: Listening...")
            try:
                audio = recognizer.listen(source)
                user_input = recognizer.recognize_google(audio)
                print(f"You: {user_input}")
                return user_input
            except sr.UnknownValueError:
                speak("I didn't catch that. Could you repeat it?")
            except sr.RequestError:
                speak("I’m having trouble accessing the recognition service.")
        return None

    def handle_command(user_input):
        lowered = user_input.lower()

        if "change personality to" in lowered:
            speak("You chose this personality at the start. Currently, dynamic changes aren't implemented. Please restart to load a new personality.")
            return False

        if "exit" in lowered or "don't need it anymore" in lowered:
            # Personalized goodbye if possible
            user_name = None
            for f in world_model["facts"]:
                if "User name:" in f:
                    user_name = f.split("User name:")[-1].strip()
                    break
            farewell = "Logging out for the day. See you tomorrow, "
            farewell += f"{user_name}!" if user_name else "Operator!"
            speak(farewell)
            return True

        if "weather" in lowered:
            resp = "Checking the weather now... It's sunny outside!"
            resp = incorporate_facts_into_response(resp, world_model)
            speak(resp)
        elif "reminder" in lowered:
            speak("What should I remind you about?")
        elif "play music" in lowered:
            speak("Playing some music to match your mood!")
        else:
            prompt = incorporate_facts_into_response("What else can I do for you, Operator?", world_model)
            speak(prompt)

        return False

    greeting = world_model.get("greeting", "Online! Ready to assist!")
    speak(greeting)

    while True:
        user_input = listen_to_user()
        if user_input:
            new_facts = extract_facts_from_input(user_input)
            if new_facts:
                for fact in new_facts:
                    if fact not in world_model["facts"]:
                        world_model["facts"].append(fact)
                # Save the model back to the chosen file
                save_world_model(chosen_file, world_model)

            emotion = detect_emotion(user_input)
            if emotion not in responses:
                emotion = "neutral"
            base_response = random.choice(responses[emotion])
            full_response = incorporate_facts_into_response(base_response, world_model)
            speak(full_response)

            should_exit = handle_command(user_input)
            if should_exit:
                break
        # If no input, continue listening

if __name__ == "__main__":
    if not os.path.exists("world_models"):
        os.makedirs("world_models")
    default_path = os.path.join("world_models", DEFAULT_WORLD_MODEL)
    if not os.path.exists(default_path):
        default_model = {
            "assistant_name": "Rockman.EXE",
            "greeting": "Online! Ready to assist!",
            "facts": []
        }
        save_world_model(default_path, default_model)

    setup_virtual_environment()
    run_script()


~~~

RockmanEXE-NetNavi-Assistant is a Python-based, voice-activated AI assistant inspired by the NetNavis from the Megaman.EXE universe. It uses speech recognition, sentiment analysis, and text-to-speech to engage in natural, voice-driven interactions. The assistant provides context-aware responses, can set reminders, discuss the weather, and remain active until you say "exit" or a similar command. It’s an early-stage prototype aiming to emulate a NetNavi-like companion experience.

## Key Features
- **Voice Interaction:** Uses `SpeechRecognition` and a microphone to listen and respond in real-time.
- **Sentiment Analysis:** Employs `TextBlob` to gauge user emotion (happy, neutral, sad, angry) and tailor responses.
- **Text-to-Speech:** Utilizes `pyttsx3` to deliver spoken replies, making the interaction feel natural and conversational.
- **Contextual Responses:** Capable of responding to specific keywords (e.g., "weather", "reminder", "play music") and offers a dynamic, mood-based response system.
- **Virtual Environment Setup:** Automatically creates and configures a Python virtual environment, installing all required dependencies.
- **Automatic System Dependencies (Arch-based):** Attempts to install required system packages (like `python-pyaudio` and `espeak-ng`) on Arch-based systems for seamless setup.
  
## Getting Started

### Prerequisites
- Python 3.11 or later recommended
- A microphone and speakers/headphones for voice interaction
- On Arch-based Linux systems (e.g., Steam Deck), `sudo` privileges to install system packages if needed.

### Installation
1. **Clone the Repository:**
   ```bash
   git clone https://github.com/YourUsername/RockmanEXE-NetNavi-Assistant.git
   cd RockmanEXE-NetNavi-Assistant

2. Run the Script:

bash
```
python rockman-exe-Net-Navi.py
```

## The script will:

Create a virtual environment in env/ if not present.
Add site-packages to sys.path.
Upgrade pip, wheel, and setuptools.
Install required Python libraries (pyttsx3, SpeechRecognition, textblob, pyaudio).
If on Arch, attempt to install python-pyaudio and espeak-ng from system packages if needed.
Once set up, the assistant will start listening.

## Usage
Speak commands into your microphone.
The assistant analyzes your sentiment and responds accordingly.
Say keywords like "weather" to get a weather-related response, "reminder" to set a reminder, or "play music" to simulate playing music.
To stop the assistant, say "exit" or "don't need it anymore."
Troubleshooting
Missing system dependencies: The script tries to install them automatically. If issues persist, manually install the packages as prompted.
Microphone or TTS issues: Ensure your microphone is recognized by SpeechRecognition and that espeak-ng is installed for pyttsx3.
Roadmap
Integrate real weather APIs for accurate reports.
Implement a reminder system that stores and recalls reminders over time.
Add more natural language understanding with advanced NLP frameworks.
Allow customization of voice, speech rate, and responses.
Contributing
Contributions are welcome! Feel free to open an issue or submit a pull request. Before contributing, consider discussing major changes via issues to ensure alignment with the project’s direction.

## License
This project is licensed under the MIT License. See the LICENSE file for details.

## Acknowledgments
Inspired by the NetNavis concept from the Megaman.EXE series.
Uses open-source Python libraries: SpeechRecognition, pyttsx3, textblob, and more.
Thanks to the open-source community for tools and documentation that make projects like this possible.
