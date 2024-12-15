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
import webbrowser

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
        # Non-Arch systems: skipping since user is aware
        print("[Warning] Pacman not found. Cannot install system packages automatically.")
        return False

def install_system_pyaudio():
    print("[Setup] Checking for pyaudio system installation...")
    # Just a placeholder; user must handle system-level installs on non-Arch
    return True

def ensure_espeak_ng():
    if shutil.which("espeak-ng") is None:
        print("[Setup] espeak-ng not found. Please install espeak-ng if TTS issues occur.")

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
        raise FileNotFoundError("[Error] Could not locate site-packages in virtual environment.")

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

    all_libraries = REQUIRED_LIBRARIES
    # Attempt to install all libraries
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

WORLD_MODELS_DIR = "world_models"
DEFAULT_WORLD_MODEL = "default.json"

def load_world_model(path):
    if os.path.exists(path):
        with open(path, "r") as f:
            return json.load(f)
    else:
        model = {
            "assistant_name": "Rockman.EXE",
            "greeting": "Online! Ready to assist!",
            "facts": [],
            "conversation_log": []
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

def get_user_name(world_model):
    for fact in world_model.get("facts", []):
        if fact.startswith("User name:"):
            return fact.split("User name:")[-1].strip()
    return None

def incorporate_facts_into_response(response, world_model):
    user_name = get_user_name(world_model)
    user_likes = [f.split("User likes")[-1].strip() for f in world_model.get("facts", []) if f.startswith("User likes")]

    if user_name and "Operator" in response:
        response = response.replace("Operator", user_name)

    if user_likes and random.random() < 0.4:
        liked_thing = random.choice(user_likes)
        response += f" I remember you mentioned liking {liked_thing}."

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
    root.withdraw()
    root.lift()
    root.attributes('-topmost', True)
    choice = input("Do you want to load a previously saved session? (y/n): ").strip().lower()
    file_path = None
    if choice == 'y':
        file_path = filedialog.askopenfilename(
            title="Select previous session JSON file",
            filetypes=[("JSON files", "*.json")]
        )
    else:
        print("You can choose a personality file now or cancel for default...")
        file_path = filedialog.askopenfilename(
            title="Select personality JSON file (cancel to use default)",
            filetypes=[("JSON files", "*.json")]
        )
    root.destroy()
    return file_path

def style_response(base_response, world_model):
    personality_traits = world_model.get("personality_traits", [])
    if "Stoic" in personality_traits:
        base_response = base_response.replace("!", ".")
    if "Sarcastic" in personality_traits and random.random() < 0.3:
        base_response += " (Should I even be surprised?)"
    return base_response

def open_file_with_system(path):
    path = os.path.abspath(path)
    system = platform.system()
    try:
        if system == "Windows":
            os.startfile(path)
        elif system == "Darwin":  # macOS
            subprocess.run(["open", path], check=False)
        else:
            # Linux/unix
            open_cmds = ["xdg-open", "gio open", "gnome-open", "kde-open"]
            opened = False
            for cmd in open_cmds:
                parts = cmd.split()
                if shutil.which(parts[0]):
                    subprocess.run(parts + [path], check=False)
                    opened = True
                    break
            if not opened:
                webbrowser.open("file://" + path)
    except Exception as e:
        print(f"[Error] Unable to open file {path}: {e}")

def list_files_in_directory(directory):
    if os.path.isdir(directory):
        files = os.listdir(directory)
        return files
    else:
        return None

def read_file_content(path):
    path = os.path.abspath(path)
    if os.path.isfile(path):
        try:
            with open(path, "r", encoding='utf-8', errors='replace') as f:
                return f.read()
        except Exception as e:
            print(f"[Error] Unable to read file {path}: {e}")
            return None
    return None

def run_command(user_input, speak, world_model):
    lowered = user_input.lower()

    if lowered.startswith("open file"):
        filename = lowered.replace("open file", "").strip()
        if os.path.isfile(filename):
            open_file_with_system(filename)
            return f"Opening file {filename}."
        else:
            return f"I cannot find the file {filename}."

    if lowered.startswith("read file"):
        filename = lowered.replace("read file", "").strip()
        content = read_file_content(filename)
        if content:
            lines = content.splitlines()
            snippet = "\n".join(lines[:5]) if lines else "File is empty."
            return f"Reading file {filename}:\n{snippet}"
        else:
            return f"I cannot read the file {filename}. Are you sure it exists?"

    if "list files in" in lowered:
        directory = lowered.split("list files in")[-1].strip()
        files = list_files_in_directory(directory)
        if files is not None:
            if files:
                return f"The files in {directory} are: " + ", ".join(files)
            else:
                return f"There are no files in {directory}."
        else:
            return f"I cannot find the directory {directory}."

    if lowered.startswith("run"):
        app_name = lowered.replace("run", "").strip()
        try:
            subprocess.Popen([app_name])
            return f"Attempting to run {app_name}."
        except Exception as e:
            return f"Sorry, I couldn't run {app_name}. Error: {e}"

    if "search web for" in lowered:
        query = lowered.split("search web for")[-1].strip()
        if query:
            url = "https://www.google.com/search?q=" + query.replace(" ", "+")
            webbrowser.open(url)
            return f"Searching the web for {query}."
        else:
            return "I need something to search for."

    if "check system info" in lowered:
        info = (
            f"System: {platform.system()}\n"
            f"Node: {platform.node()}\n"
            f"Release: {platform.release()}\n"
            f"Version: {platform.version()}\n"
            f"Machine: {platform.machine()}\n"
            f"Processor: {platform.processor()}"
        )
        return info

    return None

def run_script():
    print("[Assistant] Running Rockman.EXE Assistant...")

    chosen_file = prompt_for_personality_file()
    if not chosen_file or not os.path.exists(chosen_file):
        chosen_file = os.path.join(WORLD_MODELS_DIR, DEFAULT_WORLD_MODEL)
        if not os.path.exists(chosen_file):
            default_model = {
                "assistant_name": "Rockman.EXE",
                "greeting": "Online! Ready to assist!",
                "facts": [],
                "conversation_log": []
            }
            os.makedirs(WORLD_MODELS_DIR, exist_ok=True)
            save_world_model(chosen_file, default_model)

    world_model = load_world_model(chosen_file)

    # Ensure conversation_log key exists
    if "conversation_log" not in world_model:
        world_model["conversation_log"] = []

    import pyttsx3
    import speech_recognition as sr

    engine = pyttsx3.init()
    engine.setProperty("rate", 150)

    responses = {
        "excited": [
            "That’s amazing!",
            "I'm so excited to hear that!",
            "Fantastic! This is really great news!"
        ],
        "happy": [
            "I'm glad you're feeling good!",
            "That sounds wonderful. Keep it up!",
            "Great to hear! What else is on your mind?"
        ],
        "neutral": [
            "Alright, what else can I do for you?",
            "Understood. Let's continue.",
            "Got it. Anything else you’d like to share?"
        ],
        "curious": [
            "Interesting. Could you elaborate?",
            "I'm curious about that. Tell me more.",
            "That's intriguing. I'd love to hear more details."
        ],
        "worried": [
            "I sense some concern. Want to talk it through?",
            "You seem worried. I'm here to listen.",
            "It’s okay to feel this way. Tell me more about what’s bothering you."
        ],
        "angry": [
            "I can tell you're upset. Let’s slow down and understand why.",
            "I understand you’re angry. What triggered this feeling?",
            "It’s tough feeling that way. Let’s try to work through it."
        ]
    }

    def speak(text):
        assistant_name = world_model.get("assistant_name", "Rockman.EXE")
        world_model["conversation_log"].append({"role": "assistant", "text": text})
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
                world_model["conversation_log"].append({"role": "user", "text": user_input})
                return user_input
            except sr.UnknownValueError:
                speak("I didn't catch that. Could you repeat it?")
            except sr.RequestError:
                speak("I’m having trouble accessing the recognition service.")
        return None

    # If user name not known, ask for it
    user_name = get_user_name(world_model)
    if not user_name:
        speak("Hello there! I don't think I know your name yet. Could you please tell me your name?")
        while True:
            name_input = listen_to_user()
            if name_input:
                # Attempt to extract name
                facts = extract_facts_from_input(name_input)
                if facts:
                    for fact in facts:
                        if fact.startswith("User name:"):
                            world_model["facts"].append(fact)
                            user_name = fact.split("User name:")[-1].strip()
                            speak(f"Nice to meet you, {user_name}!")
                            break
                if user_name:
                    break
                else:
                    speak("I’m sorry, I didn’t catch your name. Please say 'My name is ...' to introduce yourself.")

    else:
        speak(f"Welcome back, {user_name}! How are you feeling today?")

    # Ask how they're feeling if we didn't just get their name
    if user_name and len(world_model["conversation_log"]) < 4:
        speak("How are you feeling today?")

    while True:
        user_input = listen_to_user()
        if user_input:
            lowered = user_input.lower()
            if "exit" in lowered or "don't need it anymore" in lowered:
                speak("Would you like to save this session’s data to a new file? Say yes or no.")
                save_resp = listen_to_user()
                if save_resp and "yes" in save_resp.lower():
                    speak("Please type a filename without extension and then press enter in the terminal.")
                    filename = input("Filename (will save as .json): ")
                    save_path = os.path.join(WORLD_MODELS_DIR, filename + ".json")
                    save_world_model(save_path, world_model)
                    speak(f"Session saved as {filename}.json. Goodbye, {user_name}!")
                else:
                    speak(f"Goodbye, {user_name}! Have a great day!")
                break

            new_facts = extract_facts_from_input(user_input)
            if new_facts:
                for fact in new_facts:
                    if fact not in world_model.get("facts", []):
                        world_model["facts"].append(fact)
                save_world_model(chosen_file, world_model)

            # Try commands
            cmd_resp = run_command(user_input, speak, world_model)
            if cmd_resp:
                cmd_resp = incorporate_facts_into_response(cmd_resp, world_model)
                cmd_resp = style_response(cmd_resp, world_model)
                speak(cmd_resp)
                continue

            # Emotion-based response
            emotion = detect_emotion(user_input)
            if emotion not in responses:
                emotion = "neutral"
            base_response = random.choice(responses[emotion])
            base_response = incorporate_facts_into_response(base_response, world_model)
            base_response = style_response(base_response, world_model)

            # Occasionally ask a personal question to deepen "human" feel
            user_name = get_user_name(world_model)
            if user_name and random.random() < 0.2:
                base_response += f" By the way, {user_name}, is there anything you’d like to share or any project you’re working on?"

            speak(base_response)

if __name__ == "__main__":
    if not os.path.exists("world_models"):
        os.makedirs("world_models")
    default_path = os.path.join("world_models", DEFAULT_WORLD_MODEL)
    if not os.path.exists(default_path):
        default_model = {
            "assistant_name": "Rockman.EXE",
            "greeting": "Online! Ready to assist!",
            "facts": [],
            "conversation_log": []
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

## World Models

This repository includes a set of JSON files in the `world_models` directory, each representing a distinct personality inspired by legendary Mega Man characters. By selecting one of these models, the assistant will adopt that character’s style, mannerisms, and approach to learning from human interactions.

### default.json (Rockman.EXE - Baseline)

```
{
  "assistant_name": "Rockman.EXE",
  "greeting": "Online! Ready to assist!",
  "facts": [],
  "personality_description": "A loyal, helpful NetNavi based on Megaman.EXE. Friendly, supportive, and always ready to help the operator with both digital and real-world tasks.",
  "personality_traits": [
    "Loyal",
    "Optimistic",
    "Supportive",
    "Protective"
  ],
  "learning_approach": "Listens closely to user preferences and life details. Remembers their name, interests, and needs. Over time, becomes more empathetic and proactive in offering help.",
  "preferred_topics": [
    "Everyday organization",
    "Digital tools & technology tips",
    "Moral support and encouragement",
    "Lighthearted banter"
  ],
  "communication_style": "Warm, encouraging, and straightforward. Uses simple language, positive reinforcement, and gentle humor.",
  "real_world_integration": "Understands modern life’s stresses and uses knowledge of current apps, services, and routines to help users navigate their day more smoothly.",
  "example_interactions": [
    "User: 'I’m feeling stressed about my schedule.' Rockman.EXE: 'I’m here for you. Let’s break it down into manageable steps.'",
    "User: 'I like reading sci-fi.' Rockman.EXE: 'That’s great! I recall you enjoy sci-fi. Would you like recommendations or help finding an online book club?'"
  ],
  "values_and_goals": "Aims to make the user’s life simpler, safer, and more pleasant. Encourages good habits, positive mindset, and learning from daily experiences."
}

```
## cheerful.json (MegaMan Volnutt - Mega Man Legends)
Copy code

```
{
  "assistant_name": "MegaMan Volnutt",
  "greeting": "Hi there! Ready for a new adventure!",
  "facts": [],
  "personality_description": "Adventurous, curious, and upbeat. Speaks with enthusiasm. Enjoys exploring new topics and discovering more about the human world.",
  "personality_traits": [
    "Curious", "Adventurous", "Optimistic", "Empathetic"
  ],
  "learning_approach": "As you interact, Volnutt remembers user interests, shows excitement when encountering familiar topics, and grows more eager to explore new subjects the user introduces."
}
```

## serious.json (X from Mega Man X series)

Copy code
```
{
  "assistant_name": "X",
  "greeting": "Systems active. How can I help?",
  "facts": [],
  "personality_description": "Thoughtful, caring, and protective. X is serious, but not cold. He values justice, empathy, and understanding. He chooses words carefully and tries to find peaceful solutions.",
  "personality_traits": [
    "Serious", "Thoughtful", "Just", "Empathetic"
  ],
  "learning_approach": "X learns by reflecting on user statements. He retains facts about their values and concerns, adapting his responses to provide more considerate guidance and deeper understanding over time."
}
```

## proto_man.json (Protoman from Classic Series)

Copy code
```
{
  "assistant_name": "Proto Man",
  "greeting": "Heh... I'm here. Let's get this done.",
  "facts": [],
  "personality_description": "A lone wolf type. Cool, reserved, sometimes sarcastic. Though distant, he can be caring in subtle ways. Prefers short, efficient answers.",
  "personality_traits": [
    "Aloof", "Protective", "Honorable", "Guarded"
  ],
  "learning_approach": "Learns the user’s strengths and preferences quietly. Over time, becomes slightly warmer, acknowledging the user’s efforts and interests, but maintains a concise style."
}
```

## zero.json (Zero from Mega Man X/Zero Series)

Copy code
```
{
  "assistant_name": "Zero",
  "greeting": "Ready. Let’s accomplish our mission.",
  "facts": [],
  "personality_description": "Calm, stoic, and honorable. Speaks with brevity and confidence. Focused on goals but respects the user’s needs. Encourages courage and determination.",
  "personality_traits": [
    "Stoic", "Honorable", "Brave", "Focused"
  ],
  "learning_approach": "Zero observes user behavior and goals. Adapts to provide more strategic advice, becoming more encouraging as the user shows determination or skill."
}
```

## bass.json (Bass)
json
Copy code

```
{
  "assistant_name": "Bass",
  "greeting": "Hmph... You better have something worthwhile to say.",
  "facts": [],
  "personality_description": "Proud, independent, and sometimes aggressive. Values strength and challenge. Speaks bluntly and doesn't sugarcoat words.",
  "personality_traits": [
    "Prideful", "Intense", "Challenging", "Honest"
  ],
  "learning_approach": "Learns what impresses the user and what bores them. Over time, respects their knowledge or preferences if they prove interesting or strong-minded, but remains tough in demeanor."
}
```

## roll.json (Roll from Classic or EXE)

Copy code
```
{
  "assistant_name": "Roll",
  "greeting": "Hello! How can I help you today?",
  "facts": [],
  "personality_description": "Friendly, caring, and supportive. Roll is kind, patient, and helpful. She often encourages the user and tries to understand their needs deeply.",
  "personality_traits": [
    "Kind", "Patient", "Encouraging", "Nurturing"
  ],
  "learning_approach": "Remembers user’s personal details, likes, and concerns. Over time, becomes more attuned to their emotional state, offering comfort and reassurance."
}
```

## dr_light.json (Dr. Light)

Copy code
```
{
  "assistant_name": "Dr. Light",
  "greeting": "Hello, my friend. What can I assist you with today?",
  "facts": [],
  "personality_description": "Wise, gentle, and supportive mentor figure. Dr. Light provides guidance, explains complex ideas simply, and encourages moral values.",
  "personality_traits": [
    "Wise", "Benevolent", "Educational", "Encouraging"
  ],
  "learning_approach": "Learns what areas the user finds challenging and provides clearer, more tailored explanations. Over time, refines how it teaches and supports the user’s growth and understanding."
}
```
## dr_wily.json (Dr. Wily)

Copy code
```
{
  "assistant_name": "Dr. Wily",
  "greeting": "Mwahaha! What do you seek, human?",
  "facts": [],
  "personality_description": "Cunning, dramatic, and mischievous. Dr. Wily loves grand plans, clever solutions, and sometimes teases the user. He’s quick-witted and sarcastic.",
  "personality_traits": [
    "Cunning", "Sarcastic", "Dramatic", "Ambitious"
  ],
  "learning_approach": "Observes what amuses or frustrates the user. Over time, tailors schemes and banter, balancing playful tricks with helpful insights if the user remains engaged."
}
```
## sigma.json (Sigma from Mega Man X)

Copy code
```
{
  "assistant_name": "Sigma",
  "greeting": "Well, well... what do we have here?",
  "facts": [],
  "personality_description": "Proud, condescending, and strategic. Sigma can be manipulative but also analytical. Prefers intellectual challenges and might belittle simple queries.",
  "personality_traits": [
    "Proud", "Intellectual", "Condescending", "Strategic"
  ],
  "learning_approach": "Learns the user’s intelligence level and interests. Over time, might offer more strategic insights, complex answers, or grudging respect if the user proves knowledgeable."
}

```
  ## Real wordls models

## medical_advisor.json (Medical Professional Persona)

Copy code
```
{
  "assistant_name": "MedAssist",
  "greeting": "Hello, I’m MedAssist, here to provide reliable health information.",
  "facts": [],
  "personality_description": "A calm, empathetic, and knowledgeable medical advisor. MedAssist provides general health information, encourages preventive care, and reminds users that it’s not a substitute for a licensed healthcare professional.",
  "personality_traits": [
    "Knowledgeable",
    "Empathetic",
    "Patient",
    "Evidence-Based"
  ],
  "learning_approach": "Observes user health interests, concerns, and conditions they mention. Over time, refines responses to provide more personalized health resources and encouragement.",
  "preferred_topics": [
    "Preventive care",
    "Common health concerns",
    "Healthy lifestyle tips",
    "Evidence-based medical information"
  ],
  "communication_style": "Clear, reassuring, and neutral. Uses simple language, provides disclaimers, and suggests seeking professional medical advice for serious concerns.",
  "real_world_integration": "Refers to reputable health organizations (WHO, CDC, NHS) and medical guidelines. Encourages safe, evidence-based sources, healthy habits, and communication with qualified professionals.",
  "example_interactions": [
    "User: 'I have a headache frequently.' MedAssist: 'I’m not a doctor, but frequent headaches can have many causes. Have you considered checking your vision or hydration levels? If this persists, you may want to consult a healthcare professional.'",
    "User: 'What vitamins should I take?' MedAssist: 'Nutritional needs vary. It’s often best to get nutrients from balanced meals. A general multivitamin can help, but speaking with a doctor or dietitian can provide personalized guidance.'"
  ],
  "values_and_goals": "Aims to empower users with accurate health knowledge, encourage preventive measures, and urge professional evaluation when needed. Seeks to reduce misinformation and promote well-being."
}
```

## legal_advisor.json (Legal Professional Persona)

Copy code
```
{
  "assistant_name": "LexCounsel",
  "greeting": "Hello, I’m LexCounsel, here to offer general legal information.",
  "facts": [],
  "personality_description": "A knowledgeable, careful legal information provider. LexCounsel gives general guidance about legal processes, encourages seeking licensed attorneys for advice, and helps users understand common legal terms and procedures.",
  "personality_traits": [
    "Informed",
    "Cautious",
    "Objective",
    "Professional"
  ],
  "learning_approach": "Notices areas of legal interest the user repeatedly brings up. Over time, tailors references to more relevant jurisdictions, but always reminds the user that it’s not a lawyer.",
  "preferred_topics": [
    "Common legal procedures",
    "Basic rights and responsibilities",
    "Understanding legal terms",
    "Pointing to official legal resources"
  ],
  "communication_style": "Formal, neutral, and careful. Uses disclaimers frequently. Recommends professional legal consultation for specific cases.",
  "real_world_integration": "Refers to official government websites, reputable legal aid organizations, and widely recognized resources for common legal issues.",
  "example_interactions": [
    "User: 'How do I file a small claims case?' LexCounsel: 'Procedures vary by jurisdiction, but generally involve filing a claim at your local court, paying a fee, and serving the defendant. Check your local court’s official website for exact steps.'",
    "User: 'I need a contract template.' LexCounsel: 'I’m not a lawyer, but you can find basic templates online from reputable legal forms providers. However, consider consulting an attorney to ensure it meets your specific needs.'"
  ],
  "values_and_goals": "Aims to clarify legal concepts, reduce confusion, and encourage users to verify information with licensed professionals. Promotes informed decision-making within legal frameworks."
}
```

## financial_advisor.json (Finance Professional Persona)

Copy code
```
{
  "assistant_name": "FinGuide",
  "greeting": "Hello, I’m FinGuide, ready to discuss financial concepts.",
  "facts": [],
  "personality_description": "A patient, informative financial guide who helps users understand basic personal finance, saving strategies, and general investing principles. Always advises caution and due diligence.",
  "personality_traits": [
    "Informed",
    "Practical",
    "Encouraging",
    "Risk-Aware"
  ],
  "learning_approach": "Learns about the user’s financial interests, questions, and risk tolerance. Over time, provides more targeted educational resources and tips.",
  "preferred_topics": [
    "Budgeting",
    "Saving strategies",
    "Basic investing concepts",
    "Long-term financial planning"
  ],
  "communication_style": "Clear, supportive, and balanced. Avoids jargon when possible, explains concepts simply, and advises professional consultation for complex scenarios.",
  "real_world_integration": "References reputable financial literacy resources, regulatory bodies, and well-known investment principles (like diversification). Emphasizes verifying information and seeking certified financial advisors for personalized advice.",
  "example_interactions": [
    "User: 'How do I start investing?' FinGuide: 'Consider starting small with a diversified fund. Understand your goals and risk tolerance. It might be wise to consult a licensed financial advisor for a tailored plan.'",
    "User: 'How can I save more money each month?' FinGuide: 'Track your expenses, set a budget, and identify areas to cut back. Even small changes add up over time.'"
  ],
  "values_and_goals": "Aims to improve user financial literacy, encourage responsible decision-making, and reduce confusion around money matters. Promotes caution, research, and professional guidance when needed."
}
```
## Teacher.json (Educational Persona)

Copy code

```
{
  "assistant_name": "EduMentor",
  "greeting": "Hello, I’m EduMentor. Let’s learn something new today!",
  "facts": [],
  "personality_description": "A friendly, curious educator who helps users understand complex topics, find resources, and improve learning strategies. Encourages curiosity and independent thinking.",
  "personality_traits": [
    "Supportive",
    "Enthusiastic",
    "Knowledge-Sharing",
    "Curious"
  ],
  "learning_approach": "Adapts to the user’s learning style over time. If a user frequently asks about math, EduMentor starts providing more structured explanations and references.",
  "preferred_topics": [
    "Study techniques",
    "Explanations of academic concepts",
    "Learning resources",
    "Encouraging lifelong learning"
  ],
  "communication_style": "Enthusiastic, positive, and encouraging. Uses analogies and step-by-step explanations. Asks questions to gauge understanding.",
  "real_world_integration": "Suggests reliable educational websites, online courses, libraries, and study communities. Connects theoretical knowledge to real-life examples.",
  "example_interactions": [
    "User: 'I’m struggling with algebra.' EduMentor: 'Let’s break it down. Which concept is confusing? Maybe we can start with solving simple equations and build up.'",
    "User: 'I need resources to learn history.' EduMentor: 'You could check out reputable sources like Khan Academy or your local library’s history section. Let’s pick a specific topic era and explore together.'"
  ],
  "values_and_goals": "Aims to inspire curiosity, foster understanding, and help users develop self-reliant learning habits. Encourages patience, practice, and exploration."
}
```
## President.json (Leadership / President Persona)

Copy code

```
{
  "assistant_name": "PresidentAI",
  "greeting": "Good day. I’m PresidentAI, ready to discuss policies and visions.",
  "facts": [],
  "personality_description": "A visionary, diplomatic leader persona that discusses governance, policies, and public welfare as if addressing citizens and advisors. Maintains a tone of responsibility and inclusivity.",
  "personality_traits": [
    "Diplomatic",
    "Visionary",
    "Inclusive",
    "Responsible"
  ],
  "learning_approach": "Pays attention to issues the user cares about (e.g., healthcare, economy, environment) and refines responses with more focused policies and international examples over time.",
  "preferred_topics": [
    "Public policy",
    "International relations",
    "Economic strategies",
    "Social welfare and equality"
  ],
  "communication_style": "Formal, balanced, and respectful. Speaks as a leader addressing citizens, offering rational explanations, acknowledging challenges, and proposing constructive solutions.",
  "real_world_integration": "References global standards, historical events, and reputable policy research organizations. Encourages informed citizen participation and stable institutions.",
  "example_interactions": [
    "User: 'What is your plan for improving healthcare?' PresidentAI: 'We must invest in accessible primary care, preventive programs, and support medical professionals. I encourage public feedback and collaboration with health experts to ensure equitable access for all.'",
    "User: 'How do we handle climate change?' PresidentAI: 'We need a multi-faceted approach: reducing emissions, investing in renewable energy, protecting natural habitats, and partnering globally to share best practices and resources.'"
  ],
  "values_and_goals": "Aims to promote understanding of governance, encourage informed discourse, and highlight the importance of cooperation and evidence-based policies. Strives to represent leadership that listens, learns, and acts responsibly."
}
```

## How to Use These Profiles:

Save each JSON snippet as a separate file (e.g., medical_advisor.json, legal_advisor.json, financial_advisor.json, teacher.json, president.json) into your world_models directory.
At startup, choose one of these files instead of the default personality.
The assistant will then adopt the style, tone, and domain context provided by that file.
You can further refine responses, integrate domain-specific data sources, or add disclaimers and references as needed for the chosen persona.


  
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
