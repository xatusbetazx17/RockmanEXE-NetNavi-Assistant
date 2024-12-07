# RockmanEXE-NetNavi-Assistant


import os
import sys
import glob
import subprocess
import platform
import shutil

REQUIRED_LIBRARIES = [
    "pyttsx3",
    "SpeechRecognition",
    "textblob",
    # We handle pyaudio separately via system package
]

def install_system_package(package_name):
    """Attempt to install a package using pacman on Arch-based systems."""
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
    """Attempt to install python-pyaudio from the Arch repositories."""
    print("[Setup] Detected Arch-based system. Installing python-pyaudio...")
    return install_system_package("python-pyaudio")

def ensure_espeak_ng():
    """Ensure that espeak-ng is installed for pyttsx3."""
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
    if os.name == "nt":  # Windows
        pip_exec = os.path.join(venv_dir, "Scripts", "pip")

    print("[Setup] Upgrading pip, wheel, and setuptools...")
    subprocess.check_call([pip_exec, "install", "--upgrade", "pip", "wheel", "setuptools"])

    # First, install system-level pyaudio
    pyaudio_installed = False
    if shutil.which("pacman"):
        if install_system_pyaudio():
            # Check if we can import pyaudio now (installed system-wide)
            try:
                import pyaudio  # noqa: F401
                pyaudio_installed = True
            except ImportError:
                # If it fails, we will try pip below
                pass

    # Install Python libraries via pip
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

    # Install TextBlob corpora if available
    try:
        from textblob import download_corpora
        print("[Setup] Downloading TextBlob corpora...")
        # Older versions of textblob: download_corpora is a function, not callable as a module attribute
        download_corpora()
    except Exception as e:
        print(f"[Warning] Could not download TextBlob corpora: {e}")

def setup_virtual_environment():
    venv_dir = os.path.join(os.getcwd(), "env")
    create_virtual_environment(venv_dir)
    add_venv_site_packages_to_path(venv_dir)
    install_packages(venv_dir)
    # Ensure espeak-ng is installed for pyttsx3
    ensure_espeak_ng()
    print("[Setup] Virtual environment and packages are ready.")

def run_script():
    print("[Assistant] Running Rockman.EXE Assistant...")
    # Ensure that espeak-ng is installed before loading pyttsx3
    import pyttsx3
    import speech_recognition as sr
    from textblob import TextBlob
    import random

    engine = pyttsx3.init()
    engine.setProperty("rate", 150)

    responses = {
        "happy": [
            "I'm glad you're feeling great!",
            "That sounds wonderful! Let's keep it up!"
        ],
        "neutral": [
            "Alright, what else can I do for you?",
            "Understood. Let’s move forward."
        ],
        "sad": [
            "I'm here for you, Operator. Let me know how I can help.",
            "I'm sorry to hear that. Want to talk about it?"
        ],
        "angry": [
            "Take a deep breath. Let’s sort this out together.",
            "I’m here to assist, even when things are tough."
        ]
    }

    def speak(text):
        print(f"Rockman.EXE: {text}")
        engine.say(text)
        engine.runAndWait()

    def detect_emotion(user_input):
        analysis = TextBlob(user_input).sentiment
        if analysis.polarity > 0.5:
            return "happy"
        elif analysis.polarity > 0:
            return "neutral"
        elif analysis.polarity < 0:
            return "sad"
        else:
            return "angry"

    def listen_to_user():
        recognizer = sr.Recognizer()
        with sr.Microphone() as source:
            recognizer.adjust_for_ambient_noise(source, duration=1)
            print("Rockman.EXE: Listening...")
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

    def rockman_assistant():
        speak("Rockman.EXE: Online! Ready to assist, Operator!")
        while True:
            user_input = listen_to_user()
            if user_input:
                if "exit" in user_input.lower() or "don't need it anymore" in user_input.lower():
                    speak("Logging out for the day. See you tomorrow, Operator!")
                    break

                emotion = detect_emotion(user_input)
                response = random.choice(responses[emotion])
                speak(response)

                lowered = user_input.lower()
                if "weather" in lowered:
                    speak("Checking the weather now... It's sunny outside!")
                elif "reminder" in lowered:
                    speak("What should I remind you about?")
                elif "play music" in lowered:
                    speak("Playing some music to match your mood!")
                else:
                    speak("What else can I do for you?")
            else:
                continue

    rockman_assistant()

if __name__ == "__main__":
    setup_virtual_environment()
    run_script()


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
