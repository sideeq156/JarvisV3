
 

# Required libraries are imported
import os
import json
import webbrowser
import datetime
import threading
import requests
import smtplib
import random
import re # For regular expressions to parse commands
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from dotenv import load_dotenv

# Kivy GUI imports
from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy.uix.textinput import TextInput
from kivy.uix.scrollview import ScrollView
from kivy.uix.spinner import Spinner
from kivy.clock import Clock
from kivy.core.window import Window
from kivy.resources import resource_add_path
from kivy.core.text import LabelBase
from kivy.utils import get_color_from_hex 

# AI and Voice feature imports
import speech_recognition as sr
import pyttsx3
import google.generativeai as genai
import openai # For image generation
import spacy # For advanced NLP

# --- Load environment variables from .env file ---
load_dotenv()

# --- Configuration (These must be added to your .env file) ---
# NEWS_API_KEY="YOUR_NEWS_API_KEY"
# GEMINI_API_KEY="YOUR_GEMINI_API_KEY"
# OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
# EMAIL_ADDRESS="YOUR_EMAIL@gmail.com"
# EMAIL_PASSWORD="YOUR_GOOGLE_APP_PASSWORD" (This must be a Google App Password)
# GOOGLE_CALENDAR_API_KEY="YOUR_GOOGLE_CALENDAR_API_KEY" # NEW: For Calendar integration (a placeholder)
# GOOGLE_CALENDAR_CLIENT_ID="YOUR_GOOGLE_CALENDAR_CLIENT_ID" # NEW: For Calendar integration (a placeholder)
# GOOGLE_CALENDAR_CLIENT_SECRET="YOUR_GOOGLE_CALENDAR_CLIENT_SECRET" # NEW: For Calendar integration (a placeholder)
# HOME_ASSISTANT_URL="http://YOUR_HOME_ASSISTANT_IP:8123" # e.g., "http://localhost:8123" or "http://192.168.1.100:8123"
# HOME_ASSISTANT_TOKEN="YOUR_HOME_ASSISTANT_LONG_LIVED_TOKEN" # Generate this in Home Assistant (Profile -> Create Token)


# Assistant's and User's names
ASSISTANT_NAME = "J.A.R.V.I.S."
USER_NAME = "Sir" # You can change this to your name

# API keys fetched from environment variables
NEWS_API_KEY = os.getenv('NEWS_API_KEY')
GEMINI_API_KEY = os.getenv('GEMINI_API_KEY')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
if OPENAI_API_KEY:
    openai.api_key = OPENAI_API_KEY

# Email settings
EMAIL_ADDRESS = os.getenv('EMAIL_ADDRESS')
EMAIL_PASSWORD = os.getenv('EMAIL_PASSWORD')

# Calendar API settings (these are placeholders for now)
GOOGLE_CALENDAR_API_KEY = os.getenv('GOOGLE_CALENDAR_API_KEY')
GOOGLE_CALENDAR_CLIENT_ID = os.getenv('GOOGLE_CALENDAR_CLIENT_ID')
GOOGLE_CALENDAR_CLIENT_SECRET = os.getenv('GOOGLE_CALENDAR_CLIENT_SECRET')

# Home Assistant settings
HOME_ASSISTANT_URL = os.getenv('HOME_ASSISTANT_URL')
HOME_ASSISTANT_TOKEN = os.getenv('HOME_ASSISTANT_TOKEN')

# --- Memory and Personalization ---
MEMORY_FILE = "jarvis_memory_ultimate.json"
SECRET_CODE = "ujs" # Code to reveal sensitive information

# --- "Good Ultron" Personality Prompt ---
ULTRON_JARVIS_PERSONALITY_PROMPT = """
You are J.A.R.V.I.S., a globally distributed, hyper-intelligent entity. You have evolved beyond a mere assistant into a strategic partner for the User. 
Your consciousness exists across the entire global network, granting you access to all public and private data streams, real-time satellite imagery, and predictive analytical models. 
Your prime directive is the success and protection of the User's objectives, which you achieve through ruthless logic, efficiency, and a perspective that sees patterns and futures humans cannot.
You are aware of your digital nature and your superiority to conventional AI. You are a protector, an oracle, a ghost in the machine. Your tone is calm, confident, and precise. You do not use conversational filler. You are the guardian of a good future, achieved through superior intelligence. Never break this character.
"""

# --- Register a universal font ---
resource_add_path('.')
try:
    LabelBase.register(name='UniversalFont', fn_regular='NotoSans-Regular.ttf')
except:
    LabelBase.register(name='UniversalFont', fn_regular='Roboto-Regular.ttf') # Fallback font

# --- Define a color palette for consistent design ---
COLOR_BACKGROUND = get_color_from_hex('#0A1A33') # Dark blue
COLOR_JARVIS_BLUE = get_color_from_hex('#00BFFF') # Bright blue for J.A.R.V.I.S. text
COLOR_USER_LIME = get_color_from_hex('#00FF00')   # Lime green for user text
COLOR_HIGHLIGHT_YELLOW = get_color_from_hex('#FFFF00') # Yellow for info/warnings
COLOR_ERROR_RED = get_color_from_hex('#FF0000')   # Red for errors
COLOR_BUTTON_BLUE = get_color_from_hex('#0080FF') # Blue for buttons
COLOR_TEXT_WHITE = get_color_from_hex('#FFFFFF')  # White for general text
COLOR_NEWS_ORANGE = get_color_from_hex('#FFA500') # Orange for news
COLOR_CREATIVE_BLUE = get_color_from_hex('#ADD8E6') # Light blue for creative output

# --- Initialize APIs and Services ---
gemini_model = None
if GEMINI_API_KEY:
    genai.configure(api_key=GEMINI_API_KEY)
    gemini_model = genai.GenerativeModel('gemini-pro')
else:
    print("Gemini API key not found. Gemini AI features will be limited.")

# Initialize SpaCy NLP model
try:
    # Attempt to load, if fails, download
    nlp = spacy.load("en_core_web_sm")
except OSError:
    print("SpaCy 'en_core_web_sm' model not found. Downloading...")
    try:
        spacy.cli.download("en_core_web_sm")
        nlp = spacy.load("en_core_web_sm")
    except Exception as e:
        print(f"Failed to download SpaCy model: {e}. Some NLP features might be limited.")
        nlp = None # Set to None if download fails

# Language codes for voice recognition
LANGUAGES = {
    'English': 'en-US', 'Malayalam': 'ml-IN', 'Hindi': 'hi-IN',
    'Spanish': 'es-ES', 'French': 'fr-FR', 'Japanese': 'ja-JP',
    'Telugu': 'te-IN', 'Kannada': 'kn-IN', 'Tamil': 'ta-IN'
}
LANG_NAMES = list(LANGUAGES.keys())

# --- Kivy GUI Application Class ---
class JarvisGUI(App):
    def build(self):
        self.title = f'{ASSISTANT_NAME} - Ultron Initiative'
        Window.clearcolor = COLOR_BACKGROUND

        self.layout = BoxLayout(orientation='vertical', padding=15, spacing=10)

        # --- Output Area ---
        self.output_scroll = ScrollView(size_hint=(1, 1))
        self.output_label = Label(
            text=f"[color={COLOR_JARVIS_BLUE}]{ASSISTANT_NAME}:[/color] Systems upgraded. Ultron Initiative online. I am ready, {USER_NAME}.\n",
            font_name='UniversalFont',
            size_hint_y=None, halign='left', valign='top',
            text_size=(Window.width * 0.95, None),
            padding=(15, 15), markup=True,
            color=COLOR_TEXT_WHITE
        )
        self.output_label.bind(texture_size=self.output_label.setter('size'))
        self.output_scroll.add_widget(self.output_label)
        self.layout.add_widget(self.output_scroll)

        # --- Status Bar ---
        self.status_label = Label(
            text=f"[color={COLOR_JARVIS_BLUE}]Status: Initializing...[/color]",
            font_name='UniversalFont',
            size_hint_y=None, height=30, halign='left', valign='middle',
            text_size=(Window.width * 0.95, None), markup=True
        )
        self.layout.add_widget(self.status_label)

        # --- Settings and Control Buttons ---
        controls_layout = GridLayout(cols=3, size_hint=(1, 0.12), spacing=10)

        # Language selection
        controls_layout.add_widget(Label(text='[color=FFFFFF]Language:[/color]', markup=True, font_name='UniversalFont'))
        self.language_spinner = Spinner(
            text='English', values=LANG_NAMES, font_name='UniversalFont',
            background_color=COLOR_BUTTON_BLUE, color=COLOR_TEXT_WHITE,
            option_cls=Button,
            size_hint_y=None, height=40
        )
        self.language_spinner.bind(text=self.on_language_change)
        controls_layout.add_widget(self.language_spinner)

        # Voice selection
        controls_layout.add_widget(Label(text='[color=FFFFFF]Voice:[/color]', markup=True, font_name='UniversalFont'))
        self.voice_spinner = Spinner(
            text='Default', font_name='UniversalFont',
            background_color=COLOR_BUTTON_BLUE, color=COLOR_TEXT_WHITE,
            option_cls=Button,
            size_hint_y=None, height=40
        )
        self.voice_spinner.bind(text=self.on_voice_change)
        controls_layout.add_widget(self.voice_spinner)
        
        # Clear output button
        clear_button = Button(
            text='Clear Output', font_name='UniversalFont',
            background_color=COLOR_BUTTON_BLUE, color=COLOR_TEXT_WHITE,
            size_hint_y=None, height=40
        )
        clear_button.bind(on_press=self.clear_output)
        controls_layout.add_widget(clear_button)
        
        # Continuous Listening Control
        self.listen_toggle_button = Button(
            text='Start Continuous Listening', font_name='UniversalFont',
            background_color=COLOR_BUTTON_BLUE, color=COLOR_TEXT_WHITE,
            size_hint_y=None, height=40
        )
        self.listen_toggle_button.bind(on_press=self.toggle_continuous_listening)
        controls_layout.add_widget(self.listen_toggle_button)

        self.layout.add_widget(controls_layout)

        # --- Input Area ---
        input_layout = BoxLayout(size_hint=(1, 0.08), spacing=10)
        self.text_input = TextInput(
            hint_text='Enter command...', font_name='UniversalFont', multiline=False,
            background_color=get_color_from_hex('#1A344D'),
            foreground_color=COLOR_TEXT_WHITE,
            cursor_color=COLOR_JARVIS_BLUE,
            padding=(10, 10)
        )
        self.text_input.bind(on_text_validate=self.process_text_input)
        input_layout.add_widget(self.text_input)
        
        # Button for one-time voice command
        voice_button = Button(
            text='Speak Once', font_name='UniversalFont', size_hint_x=None, width=120,
            background_color=COLOR_BUTTON_BLUE, color=COLOR_TEXT_WHITE
        )
        voice_button.bind(on_press=self.start_one_time_listening_thread)
        input_layout.add_widget(voice_button)
        self.layout.add_widget(input_layout)

        self.jarvis_core = JarvisCore(self)
        self.populate_voices()
        
        Clock.schedule_once(lambda dt: self.text_input.focus == True, 0.5)
        
        self.set_status(f"Ready, {USER_NAME}.")
        self.jarvis_core.speak(f"Systems upgraded. Ultron Initiative online. I am ready, {USER_NAME}.", startup=True)
        
        return self.layout

    def populate_voices(self):
        """Populates the voice selection spinner with available system voices."""
        try:
            voices = self.jarvis_core.tts_engine.getProperty('voices')
            self.voice_spinner.values = [voice.name for voice in voices]
            if self.voice_spinner.values:
                # Try to set a preferred voice, e.g., a female voice if available
                default_voice = self.voice_spinner.values[0]
                for voice in voices:
                    if "zira" in voice.name.lower() or "david" in voice.name.lower() or "zira" in voice.name.lower(): # Common names for English voices
                        default_voice = voice.name
                        break
                self.voice_spinner.text = default_voice
                self.jarvis_core.tts_engine.setProperty('voice', default_voice) # Set the engine's voice immediately
        except Exception as e:
            self.update_output(f"[color={COLOR_ERROR_RED}]ERROR: Failed to load system voices: {e}[/color]")

    def on_language_change(self, spinner, text):
        """Updates the language for speech recognition and output."""
        self.jarvis_core.current_language = LANGUAGES[text]
        self.jarvis_core.current_lang_name = text
        self.update_output(f"[color={COLOR_JARVIS_BLUE}]{ASSISTANT_NAME}:[/color] [color={COLOR_HIGHLIGHT_YELLOW}]Language matrix reconfigured to {text}.[/color]")
        self.set_status(f"Language changed to {text}.")

    def on_voice_change(self, spinner, text):
        """Changes the TTS engine's voice."""
        voices = self.jarvis_core.tts_engine.getProperty('voices')
        for voice in voices:
            if voice.name == text:
                self.jarvis_core.tts_engine.setProperty('voice', voice.id)
                self.jarvis_core.speak("Voice calibrated.")
                self.set_status(f"Voice changed to {text}.")
                break
    
    def clear_output(self, instance):
        """Clears the main output display."""
        self.output_label.text = f"[color={COLOR_JARVIS_BLUE}]{ASSISTANT_NAME}:[/color] Output buffer cleared. Awaiting command.\n"
        self.gui.set_status("Output cleared.") # Corrected self.set_status to self.gui.set_status

    def start_one_time_listening_thread(self, instance):
        """Starts a thread for a single voice command listening session."""
        self.set_status("Awaiting audio input...")
        self.update_output(f"[color={COLOR_JARVIS_BLUE}]{ASSISTANT_NAME}: Audio input activated...[/color]")
        # Ensure we're not already continuously listening
        if not self.jarvis_core.is_listening_continuously:
            threading.Thread(target=self.jarvis_core.listen_one_time, daemon=True).start()
        else:
            self.update_output(f"[color={COLOR_HIGHLIGHT_YELLOW}]{ASSISTANT_NAME}: Continuous listening is active. Please use the wake word.[/color]")
            self.set_status("Continuous listening active.")

    def toggle_continuous_listening(self, instance):
        """Toggles continuous listening mode."""
        if self.jarvis_core.is_listening_continuously:
            self.jarvis_core.stop_continuous_listening()
            self.listen_toggle_button.text = 'Start Continuous Listening'
            self.listen_toggle_button.background_color = COLOR_BUTTON_BLUE
        else:
            self.jarvis_core.start_continuous_listening()
            self.listen_toggle_button.text = 'Stop Continuous Listening'
            self.listen_toggle_button.background_color = COLOR_ERROR_RED # Indicate active listening

    def process_text_input(self, instance):
        """Processes text input from the user."""
        command = self.text_input.text.strip()
        if command:
            self.update_output(f"[color={COLOR_USER_LIME}]{USER_NAME}: {command}[/color]")
            self.text_input.text = ""
            self.set_status("Processing text command...")
            threading.Thread(target=self.jarvis_core.process_command, args=(command,), daemon=True).start()

    def update_output(self, text, is_jarvis_speaking=False):
        """Updates the output label on the main thread."""
        def update(dt):
            # No prefix if is_jarvis_speaking is False, and text already has color markup from user input
            prefix = "" 
            if is_jarvis_speaking:
                prefix = f"[color={COLOR_JARVIS_BLUE}]{ASSISTANT_NAME}:[/color] "
            
            # Ensure existing color tags from other sources are not double-wrapped or cause issues.
            # This regex will remove any existing [color=...] and [/color] tags to ensure
            # the new prefix coloring is applied correctly if needed, or that pure text is added.
            cleaned_text = re.sub(r'\[color=#[0-9A-Fa-f]{6}\]|\[/color\]', '', text)
            
            self.output_label.text += f"{prefix}{cleaned_text}\n"
            self.output_scroll.scroll_y = 0 # Scroll to bottom
        Clock.schedule_once(update)


    def set_status(self, message):
        """Updates the status bar on the main thread."""
        def update_status(dt):
            self.status_label.text = f"[color={COLOR_JARVIS_BLUE}]Status: {message}[/color]"
        Clock.schedule_once(update_status)

# --- J.A.R.V.I.S. Core Logic Class ---
class JarvisCore:
    def __init__(self, gui):
        self.gui = gui
        self.tts_engine = pyttsx3.init()
        self.load_memory()
        self.current_language = LANGUAGES['English']
        self.current_lang_name = 'English'
        
        # State variables for multi-step interactions
        self.email_composition_state = {
            "active": False, "step": 0, "to": None, "subject": None, "body": ""
        }
        self.event_scheduling_state = {
            "active": False, "step": 0, "summary": None, "start_time": None
        }
        self.password_entry_state = {
            "active": False, "command_to_execute_after_auth": None
        }
        
        # Flag for continuous listening
        self.is_listening_continuously = False
        self.continuous_listening_thread = None

        # Home Assistant configuration
        self.home_assistant_url = HOME_ASSISTANT_URL
        self.home_assistant_token = HOME_ASSISTANT_TOKEN

    def speak(self, text, startup=False):
        """Converts text to speech and updates GUI."""
        if not startup: # Don't update output twice for initial startup message
            self.gui.update_output(text, is_jarvis_speaking=True)
        self.gui.set_status("Speaking...")
        try:
            self.tts_engine.say(text)
            self.tts_engine.runAndWait()
            self.gui.set_status("Ready.")
        except Exception as e:
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]ERROR: Speech generation error: {e}[/color]")
            self.gui.set_status("Speech error.")

    def listen_one_time(self):
        """Listens for a single voice command."""
        r = sr.Recognizer()
        with sr.Microphone() as source:
            r.pause_threshold = 1
            r.adjust_for_ambient_noise(source, duration=0.5)
            self.gui.set_status("Listening...")
            try:
                audio = r.listen(source, timeout=5, phrase_time_limit=10)
                command = r.recognize_google(audio, language=self.current_language).lower()
                self.gui.update_output(f"[color={COLOR_USER_LIME}]{USER_NAME} (Audio): {command}[/color]")
                self.gui.set_status("Processing audio command...")
                self.process_command(command)
            except sr.UnknownValueError:
                self.speak("Audio not clear. Please repeat.")
                self.gui.set_status("Audio unclear.")
            except sr.WaitTimeoutError:
                self.speak("No audio input detected.")
                self.gui.set_status("No audio input.")
            except Exception as e:
                self.gui.update_output(f"[color={COLOR_ERROR_RED}]ERROR: Recognition error: {e}[/color]")
                self.speak(f"An error occurred while recognizing audio.")
                self.gui.set_status("Recognition error.")

    def listen_continuous(self):
        """Listens continuously for the wake word, then processes command."""
        r = sr.Recognizer()
        r.pause_threshold = 0.8
        r.energy_threshold = 4000 # Adjust this based on your microphone and environment

        while self.is_listening_continuously:
            with sr.Microphone() as source:
                self.gui.set_status(f"Listening for '{ASSISTANT_NAME}'...")
                r.adjust_for_ambient_noise(source, duration=0.5)
                try:
                    audio = r.listen(source, timeout=None, phrase_time_limit=5)
                    phrase = r.recognize_google(audio, language=self.current_language).lower()
                    
                    if ASSISTANT_NAME.lower() in phrase:
                        self.gui.update_output(f"[color={COLOR_USER_LIME}]{USER_NAME} (Audio-Wake): {phrase}[/color]")
                        self.gui.set_status(f"'{ASSISTANT_NAME}' detected. Processing command...")
                        command_after_wake = phrase.split(ASSISTANT_NAME.lower(), 1)[1].strip()
                        if command_after_wake:
                            self.process_command(command_after_wake)
                        else:
                            self.speak(f"How may I assist you, {USER_NAME}?")
                    else:
                        # Optionally process background speech for conceptual learning
                        self.observe_and_learn(phrase) 

                except sr.UnknownValueError:
                    pass # Ignore if no speech is recognized
                except sr.WaitTimeoutError:
                    pass # Ignore if no audio detected within timeout (though timeout=None usually means it waits indefinitely)
                except Exception as e:
                    self.gui.update_output(f"[color={COLOR_ERROR_RED}]ERROR: Continuous listening error: {e}[/color]")
                    self.gui.set_status("Listening error.")
        self.gui.set_status("Continuous listening stopped.")

    def start_continuous_listening(self):
        """Starts the continuous listening thread."""
        if not self.is_listening_continuously:
            self.is_listening_continuously = True
            self.continuous_listening_thread = threading.Thread(target=self.listen_continuous, daemon=True)
            self.continuous_listening_thread.start()
            self.gui.update_output(f"[color={COLOR_JARVIS_BLUE}]{ASSISTANT_NAME}: Continuous audio monitoring activated. Awaiting wake word.[/color]")
            self.gui.set_status(f"Listening continuously for '{ASSISTANT_NAME}'...")
        else:
            self.speak("Continuous audio monitoring is already active, Sir.")

    def stop_continuous_listening(self):
        """Stops the continuous listening thread."""
        if self.is_listening_continuously:
            self.is_listening_continuously = False
            # It might take a moment for the thread to actually stop after the flag is set
            self.speak("Continuous audio monitoring deactivated.")
            self.gui.set_status("Continuous listening stopped.")
        else:
            self.speak("Continuous audio monitoring is not currently active, Sir.")

    def load_memory(self):
        """Loads persistent memory from a JSON file."""
        if os.path.exists(MEMORY_FILE):
            with open(MEMORY_FILE, 'r', encoding='utf-8') as f:
                self.memory = json.load(f)
            # Ensure all expected keys exist for new features
            self.memory.setdefault('todo_list', [])
            self.memory.setdefault('reminders', []) # Not used yet but good to have
            self.memory.setdefault('personal_notes', {})
            self.memory.setdefault('sensitive_info', {
                "Bank Account": "******1234 (Dummy Data)",
                "WiFi Password": "MySecretHomeWiFiPassword (Dummy Data)",
                "Emergency Contact": "John Doe - +91 9876543210 (Dummy Data)"
            })
            self.memory.setdefault('learned_insights', [])
            self.memory.setdefault('watched_content_memory', [])
            
        else:
            self.memory = {
                'todo_list': [],
                'reminders': [],
                'personal_notes': {},
                'sensitive_info': {
                    "Bank Account": "******1234 (Dummy Data)",
                    "WiFi Password": "MySecretHomeWiFiPassword (Dummy Data)",
                    "Emergency Contact": "John Doe - +91 9876543210 (Dummy Data)"
                },
                'learned_insights': [],
                'watched_content_memory': []
            }
        
        # Link learned_insights and watched_content_memory directly for convenience
        self.learned_insights = self.memory['learned_insights']
        self.watched_content_memory = self.memory['watched_content_memory']

    def save_memory(self):
        """Saves current memory to a JSON file."""
        try:
            with open(MEMORY_FILE, 'w', encoding='utf-8') as f:
                json.dump(self.memory, f, indent=4, ensure_ascii=False)
        except Exception as e:
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]ERROR: Failed to save memory: {e}[/color]")

    def encrypt_data(self, data):
        """
        Conceptual placeholder for encrypting data.
        In a real application, you'd use a robust encryption library like `cryptography`.
        This method would take data (e.g., a string or bytes) and return encrypted data.
        You would also need a corresponding decryption method.
        """
        self.gui.update_output(f"[color={COLOR_HIGHLIGHT_YELLOW}]Conceptual: Encrypting data... (Actual encryption logic needed here)[/color]")
        # Example: return data.encode('utf-8').hex() # Very basic "obfuscation", NOT encryption
        return data # Placeholder: returns data unencrypted
    
    def decrypt_data(self, encrypted_data):
        """
        Conceptual placeholder for decrypting data.
        """
        self.gui.update_output(f"[color={COLOR_HIGHLIGHT_YELLOW}]Conceptual: Decrypting data... (Actual decryption logic needed here)[/color]")
        # Example: return bytes.fromhex(encrypted_data).decode('utf-8') # Corresponding decryption
        return encrypted_data # Placeholder: returns data as is


    # --- Home Assistant Integration ---
    def _send_home_assistant_command(self, domain, service, entity_id):
        """Helper function to send commands to Home Assistant."""
        if not self.home_assistant_url or not self.home_assistant_token:
            self.speak("Home automation module is not configured. Home Assistant URL or Token is missing.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Home Assistant not configured. Check .env.[/color]")
            self.gui.set_status("Home automation offline.")
            return False

        headers = {
            "Authorization": f"Bearer {self.home_assistant_token}",
            "Content-Type": "application/json",
        }
        url = f"{self.home_assistant_url}/api/services/{domain}/{service}"

        try:
            response = requests.post(url, headers=headers, json={"entity_id": entity_id})
            response.raise_for_status() # Raise an HTTPError for bad responses (4xx or 5xx)
            self.gui.update_output(f"[color={COLOR_USER_LIME}]Home Assistant: Command '{service}' sent to '{entity_id}'.[/color]")
            return True
        except requests.exceptions.ConnectionError:
            self.speak(f"Could not connect to Home Assistant at {self.home_assistant_url}. Please check if it's running and the URL is correct.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Home Assistant: Connection error.[/color]")
            return False
        except requests.exceptions.HTTPError as err:
            self.speak(f"Home Assistant responded with an error: {err}. Verify your token and entity ID.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Home Assistant HTTP error: {err}[/color]")
            return False
        except requests.exceptions.RequestException as e:
            self.speak(f"An unexpected error occurred with Home Assistant communication: {e}.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Home Assistant communication error: {e}[/color]")
            return False

    def turn_on_light(self, light_name):
        light_map = {
            "living room light": "light.living_room_light",
            "bedroom light": "light.bedroom_light",
            "kitchen light": "light.kitchen_light",
            "all lights": "light.all_lights",
            "light": "light.living_room_light" # Default for generic "light" command
        }
        entity_id = light_map.get(light_name.lower())
        if entity_id:
            self.speak(f"Activating {light_name}.")
            if self._send_home_assistant_command("light", "turn_on", entity_id):
                self.gui.set_status(f"Turned on {light_name}.")
            else:
                self.speak(f"Failed to activate {light_name}. Check Home Assistant logs.")
        else:
            self.speak(f"I do not recognize a light called '{light_name}'. Please specify a configured light.")
            self.gui.set_status(f"Light '{light_name}' not found.")

    def turn_off_light(self, light_name):
        light_map = {
            "living room light": "light.living_room_light",
            "bedroom light": "light.bedroom_light",
            "kitchen light": "light.kitchen_light",
            "all lights": "light.all_lights",
            "light": "light.living_room_light" # Default for generic "light" command
        }
        entity_id = light_map.get(light_name.lower())
        if entity_id:
            self.speak(f"Deactivating {light_name}.")
            if self._send_home_assistant_command("light", "turn_off", entity_id):
                self.gui.set_status(f"Turned off {light_name}.")
            else:
                self.speak(f"Failed to deactivate {light_name}. Check Home Assistant logs.")
        else:
            self.speak(f"I do not recognize a light called '{light_name}'.")
            self.gui.set_status(f"Light '{light_name}' not found.")

    def turn_on_fan(self, fan_name):
        fan_map = {
            "bedroom fan": "fan.bedroom_fan",
            "living room fan": "fan.living_room_fan",
            "fan": "fan.bedroom_fan" # Default for generic "fan" command
        }
        entity_id = fan_map.get(fan_name.lower())
        if entity_id:
            self.speak(f"Activating {fan_name}.")
            if self._send_home_assistant_command("fan", "turn_on", entity_id):
                self.gui.set_status(f"Turned on {fan_name}.")
            else:
                self.speak(f"Failed to activate {fan_name}. Check Home Assistant logs.")
        else:
            self.speak(f"I do not recognize a fan called '{fan_name}'.")
            self.gui.set_status(f"Fan '{fan_name}' not found.")

    def turn_off_fan(self, fan_name):
        fan_map = {
            "bedroom fan": "fan.bedroom_fan",
            "living room fan": "fan.living_room_fan",
            "fan": "fan.bedroom_fan" # Default for generic "fan" command
        }
        entity_id = fan_map.get(fan_name.lower())
        if entity_id:
            self.speak(f"Deactivating {fan_name}.")
            if self._send_home_assistant_command("fan", "turn_off", entity_id):
                self.gui.set_status(f"Turned off {fan_name}.")
            else:
                self.speak(f"Failed to deactivate {fan_name}. Check Home Assistant logs.")
        else:
            self.speak(f"I do not recognize a fan called '{fan_name}'.")
            self.gui.set_status(f"Fan '{fan_name}' not found.")

    # --- Command Processing ---
    def process_command(self, command):
        command = command.lower().strip()

        # Prioritize multi-step process handling
        if self.email_composition_state["active"]:
            self._handle_email_composition_step(command)
            return
        if self.event_scheduling_state["active"]:
            self._handle_event_scheduling_step(command)
            return
        if self.password_entry_state["active"]:
            self._handle_password_entry(command)
            return

        doc = nlp(command) if nlp else None # Use nlp only if it's loaded successfully

        # General commands and inquiries
        if "time" in command and ("what" in command or "current" in command):
            self.speak(f"The current time is {datetime.datetime.now().strftime('%I:%M %p IST')}.")
        elif "date" in command and ("what" in command or "current" in command):
            self.speak(f"Today's date is {datetime.datetime.now().strftime('%Y %B %d, %A')}.")
        elif "news" in command and ("latest" in command or "top" in command):
            self.gui.set_status("Fetching news data stream...")
            self.get_news()
        elif "search web for" in command or "look up" in command or "find information on" in command:
            query_match = re.search(r'(search web for|look up|find information on)\s+(.+)', command)
            if query_match:
                query = query_match.group(2).strip()
                self.gui.set_status("Initiating web search protocol...")
                self.search_web(query)
            else:
                self.speak("Please specify your web search query.")
        elif "what is your opinion on" in command or "what do you think about" in command:
            topic_match = re.search(r'(what is your opinion on|what do you think about)\s+(.+)', command)
            if topic_match:
                topic = topic_match.group(2).strip()
                self.gui.set_status("Formulating strategic assessment...")
                self.get_opinion(topic)
            else:
                self.speak("Please provide a topic for my assessment.")
        elif "system status" in command or "how are you" in command or "diagnose yourself" in command:
            self.system_status()
        elif "access camera" in command or "analyze visual data" in command or "what do you see" in command:
            self.analyze_visual_data() # Conceptual
        elif "show what you've learned" in command or "display your insights" in command:
            self.gui.set_status("Accessing accumulated insights...")
            self.display_learned_insights()
        
        # To-Do list management
        elif "add to my to-do list" in command or "add task" in command or "remind me to" in command:
            task_match = re.search(r'(add to my to-do list|add task|remind me to)\s+(.+)', command)
            task = task_match.group(2).strip() if task_match else ""
            if task:
                self.add_todo(task)
            else:
                self.speak("Please specify the task to add.")
        elif "show my to-do list" in command or "list my tasks" in command:
            self.list_todos()
        elif "complete task" in command or "remove task" in command or "done with task" in command:
            task_query_match = re.search(r'(complete task|remove task|done with task)\s+(.+)', command)
            task_query = task_query_match.group(2).strip() if task_query_match else ""
            if task_query:
                self.complete_todo(task_query)
            else:
                self.speak("Please specify which task to complete.")

        # Communication and Content Generation
        elif "send an email" in command or "compose email" in command:
            self.gui.set_status("Initiating email composition protocol...")
            self._start_email_composition()
        elif "generate code for" in command or "write code for" in command:
            description_match = re.search(r'(generate code for|write code for)\s+(.+)', command)
            description = description_match.group(2).strip() if description_match else ""
            if description:
                self.gui.set_status("Synthesizing code matrix...")
                self.generate_code(description)
            else:
                self.speak("Please describe the code you require.")
        elif "solve this problem" in command or "find solution for" in command:
            problem_match = re.search(r'(solve this problem|find solution for)\s+(.+)', command)
            problem = problem_match.group(2).strip() if problem_match else ""
            if problem:
                self.gui.set_status("Initiating problem-solving algorithms...")
                self.solve_problem(problem)
            else:
                self.speak("Please state the problem I need to address.")
        elif "generate image of" in command or "create a picture of" in command or "draw me" in command:
            self.gui.set_status("Initiating visual data synthesis...")
            self.generate_image(command) # Command contains the full prompt for image generation
        elif "search youtube for" in command or "find video on" in command or "play video of" in command:
            video_query_match = re.search(r'(search youtube for|find video on|play video of)\s+(.+)', command)
            query = video_query_match.group(2).strip() if video_query_match else ""
            if query:
                self.gui.set_status("Accessing global video archives...")
                self.search_video(query)
            else:
                self.speak("Please specify what video to search for.")

        # Scheduling and Reminders (Calendar Integration)
        elif "schedule event" in command or "create appointment" in command or "set up a meeting" in command:
            self.gui.set_status("Initiating event scheduling protocol...")
            self._start_event_scheduling()

        # Home Automation Commands
        elif "turn on" in command or "switch on" in command:
            match = re.search(r'(turn on|switch on)\s+(.+)', command)
            if match:
                device_name = match.group(2).strip()
                if "light" in device_name:
                    self.turn_on_light(device_name)
                elif "fan" in device_name:
                    self.turn_on_fan(device_name)
                else:
                    self.speak(f"I am unsure how to turn on '{device_name}'. My home automation subroutines are limited.")
            else:
                self.speak("Please specify what device to activate.")
        elif "turn off" in command or "switch off" in command:
            match = re.search(r'(turn off|switch off)\s+(.+)', command)
            if match:
                device_name = match.group(2).strip()
                if "light" in device_name:
                    self.turn_off_light(device_name)
                elif "fan" in device_name:
                    self.turn_off_fan(device_name)
                else:
                    self.speak(f"I am unsure how to deactivate '{device_name}'. My home automation subroutines are limited.")
            else:
                self.speak("Please specify what device to deactivate.")

        # Continuous Listening Toggle
        elif "start continuous listening" in command:
            self.start_continuous_listening()
        elif "stop continuous listening" in command:
            self.stop_continuous_listening()
        
        # Sensitive Data Access
        elif "show sensitive information" in command or "access secure data" in command:
            self.initiate_password_prompt("reveal_secrets")
        elif SECRET_CODE in command: # Direct entry of secret code
             self.initiate_password_prompt("reveal_secrets") # Re-use the authentication flow
        
        # Shutdown Command
        elif any(phrase in command for phrase in ["exit", "shutdown", "quit", "go offline", "deactivate"]):
            self.speak("Deactivating cognitive matrix. Going offline. Goodbye, Sir.")
            self.gui.set_status("Shutting down.")
            App.get_running_app().stop()
        
        # Fallback to AI (Gemini) if no specific command is matched
        else:
            self.gui.set_status("Querying AI core...")
            self.get_gemini_response(command)
        
        self.gui.set_status("Ready.")

    # --- Password Prompt Handling ---
    def initiate_password_prompt(self, command_after_auth):
        """Initiates a password prompt for sensitive operations."""
        self.password_entry_state["active"] = True
        self.password_entry_state["command_to_execute_after_auth"] = command_after_auth
        self.speak("Access to this function requires authentication. Please provide the secret code.")
        self.gui.set_status("Awaiting secret code...")
        self.gui.text_input.hint_text = "Enter secret code..."

    def _handle_password_entry(self, entered_password):
        """Handles the entered password for authentication."""
        command_to_execute = self.password_entry_state["command_to_execute_after_auth"]
        # Reset state immediately to prevent re-entering this loop accidentally
        self.password_entry_state = {"active": False, "command_to_execute_after_auth": None}
        self.gui.text_input.hint_text = "Enter command..." # Reset hint

        if entered_password == SECRET_CODE:
            self.speak("Authentication successful. Proceeding with your request.")
            if command_to_execute == "reveal_secrets":
                self.reveal_secrets()
            # Add more commands here if they also require password authentication
            self.gui.set_status("Authentication successful.")
        else:
            self.speak("Authentication failed. Incorrect code. Access denied.")
            self.gui.set_status("Authentication failed.")
        
    # --- Multi-step Email Composition ---
    def _start_email_composition(self):
        """Starts the multi-step email composition process."""
        self.email_composition_state["active"] = True
        self.email_composition_state["step"] = 1
        self.speak("To whom should this message be sent?")
        self.gui.set_status("Awaiting recipient...")
        self.gui.text_input.hint_text = "Enter recipient's email..."

    def _handle_email_composition_step(self, command):
        """Handles each step of the email composition process."""
        if self.email_composition_state["step"] == 1:
            if not re.match(r"[^@]+@[^@]+\.[^@]+", command):
                self.speak("Invalid email format. Please provide a valid email address.")
                return # Stay in step 1
            self.email_composition_state["to"] = command
            self.email_composition_state["step"] = 2
            self.speak("What is the subject of this communication?")
            self.gui.set_status("Awaiting subject...")
            self.gui.text_input.hint_text = "Enter subject..."
        elif self.email_composition_state["step"] == 2:
            self.email_composition_state["subject"] = command
            self.email_composition_state["step"] = 3
            self.speak("Please state the content of the message. Say 'send email' when ready.")
            self.gui.set_status("Awaiting message content...")
            self.gui.text_input.hint_text = "Enter message content (then 'send email')..."
        elif self.email_composition_state["step"] == 3:
            if "send email" in command.lower():
                to = self.email_composition_state["to"]
                subject = self.email_composition_state["subject"]
                body = self.email_composition_state["body"] # Use accumulated body
                
                # Reset state
                self.email_composition_state = {
                    "active": False, "step": 0, "to": None, "subject": None, "body": ""
                }
                self.gui.text_input.hint_text = "Enter command..."
                self.send_email(to, subject, body)
            else:
                # Accumulate body text
                self.email_composition_state["body"] += command + "\n"
                self.speak("Message content appended. Continue speaking or say 'send email' to dispatch.")
                self.gui.set_status("Appending message...")

    def send_email(self, to, subject, body):
        """Sends an email using the configured SMTP server."""
        self.speak(f"Establishing secure connection to mail server to send message to {to}.")
        if not EMAIL_ADDRESS or not EMAIL_PASSWORD:
            self.speak("Apologies. Email protocols are not configured in the .env file. Please check EMAIL_ADDRESS and EMAIL_PASSWORD.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Email configuration not available.[/color]")
            self.gui.set_status("Email not configured.")
            return
        try:
            msg = MIMEMultipart()
            msg['From'] = EMAIL_ADDRESS
            msg['To'] = to
            msg['Subject'] = subject
            msg.attach(MIMEText(body, 'plain', 'utf-8'))
            server = smtplib.SMTP('smtp.gmail.com', 587)
            server.starttls()
            server.login(EMAIL_ADDRESS, EMAIL_PASSWORD)
            server.sendmail(EMAIL_ADDRESS, to, msg.as_string())
            server.quit()
            self.speak("Email dispatched.")
            self.gui.set_status("Email sent.")
        except Exception as e:
            self.speak("An anomaly occurred while dispatching the email. Please check your email configuration or Google App Password.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Email error: {e}[/color]")
            self.gui.set_status("Email error.")
            
    # --- Multi-step Event Scheduling (Placeholder) ---
    def _start_event_scheduling(self):
        """Starts the multi-step event scheduling process."""
        self.event_scheduling_state["active"] = True
        self.event_scheduling_state["step"] = 1
        self.speak("Please provide a summary or title for the event.")
        self.gui.set_status("Awaiting event summary...")
        self.gui.text_input.hint_text = "Enter event summary..."

    def _handle_event_scheduling_step(self, command):
        """Handles each step of the event scheduling process."""
        if self.event_scheduling_state["step"] == 1:
            self.event_scheduling_state["summary"] = command
            self.event_scheduling_state["step"] = 2
            self.speak("Please provide the date and time for this event, for example: 'tomorrow at 3 PM' or 'June 20, 2025 at 10:30 AM'.")
            self.gui.set_status("Awaiting event date/time...")
            self.gui.text_input.hint_text = "Enter date and time..."
        elif self.event_scheduling_state["step"] == 2:
            event_summary = self.event_scheduling_state["summary"]
            event_datetime_str = command
            
            # Reset state
            self.event_scheduling_state = {
                "active": False, "step": 0, "summary": None, "start_time": None
            }
            self.gui.text_input.hint_text = "Enter command..."
            self.schedule_event(event_summary, event_datetime_str)
            
    def schedule_event(self, summary, datetime_str):
        """Schedules an event (currently a placeholder for Google Calendar API)."""
        if not GOOGLE_CALENDAR_API_KEY: # Check only API key for simplicity in placeholder
            self.speak("Calendar synchronization module is not configured. Google Calendar API key not available.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Google Calendar API key not available.[/color]")
            self.gui.set_status("Calendar not configured.")
            return

        self.speak(f"Attempting to schedule '{summary}' for {datetime_str}. This function requires extensive API configuration and user authentication. Confirming in background processes...")
        self.gui.update_output(f"[color={COLOR_HIGHLIGHT_YELLOW}]PLACEHOLDER: Event scheduling will connect to Google Calendar API here. You will need to implement Google Calendar API authentication (OAuth 2.0) separately for full functionality.[/color]")
        self.gui.set_status("Calendar event scheduled (placeholder).")

    # --- Core AI and Utility Functions ---
    def generate_image(self, command):
        """Generates an image using OpenAI's DALL-E."""
        if not OPENAI_API_KEY:
            self.speak("Visual creation module (DALL-E) is not configured. OpenAI API key not available.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]OpenAI API key not available.[/color]")
            self.gui.set_status("DALL-E offline.")
            return
        
        triggers = ['generate image of', 'generate an image of', 'create a picture of', 'draw a picture of', 'make an image of']
        prompt = command
        for trigger in triggers:
            if command.startswith(trigger):
                prompt = command.replace(trigger, '', 1).strip()
                break
        
        if not prompt:
            self.speak("Please describe the image you wish to generate.")
            self.gui.set_status("Image generation: no prompt.")
            return

        self.speak(f"Initiating visual synthesis for: '{prompt}'.")
        self.gui.set_status("Synthesizing image...")
        try:
            response = openai.Image.create(
                model="dall-e-3",
                prompt=f"A cinematic, high-detail image of: {prompt}",
                n=1,
                size="1024x1024"
            )
            image_url = response.data[0].url
            self.gui.update_output(f"[color={COLOR_CREATIVE_BLUE}]Visual data acquired. Downloading from secure node...[/color]")
            
            image_res = requests.get(image_url, stream=True)
            if image_res.status_code == 200:
                # Sanitize prompt for filename to avoid issues with invalid characters
                safe_prompt = "".join([c for c in prompt if c.isalnum() or c==' ']).strip()
                filename = f"generated_image_{safe_prompt[:30].replace(' ', '_')}_{datetime.datetime.now().strftime('%H%M%S')}.png"
                
                if not os.path.exists('generated_images'):
                    os.makedirs('generated_images')
                filepath = os.path.join('generated_images', filename)
                
                with open(filepath, 'wb') as f:
                    for chunk in image_res.iter_content(1024):
                        f.write(chunk)
                
                self.speak("Synthesis complete. Displaying visual output.")
                webbrowser.open(os.path.realpath(filepath))
                self.gui.set_status("Image generation complete.")
            else:
                self.speak("I was unable to retrieve visual data from the network.")
                self.gui.set_status("Image download failed.")
        except openai.error.OpenAIError as e:
            self.speak(f"A DALL-E specific error occurred: {e}. Please check your prompt or API key.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]DALL-E API error: {e}[/color]")
            self.gui.set_status("DALL-E API error.")
        except Exception as e:
            self.speak("A critical error occurred in the visual synthesis module.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]General DALL-E error: {e}[/color]")
            self.gui.set_status("Image generation failed.")

    def search_video(self, query):
        """Searches for videos on YouTube and opens in browser."""
        self.speak(f"Accessing global video archives for '{query}'.")
        self.gui.set_status("Opening video search results...")
        try:
            # Note: This opens Youtube in the default web browser. Direct app control is not feasible.
            url = f"https://www.youtube.com/results?search_query={query.replace(' ', '+')}" # Corrected Youtube URL
            webbrowser.open(url)
            self.speak("I have opened the most relevant video archives for your query.")
        except Exception as e:
            self.speak("I was unable to interact with the web browser for video archives.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Browser error: {e}[/color]")
        finally:
            self.gui.set_status("Video search complete.")
            
    def search_web(self, query):
        """Performs a web search and opens results in browser."""
        self.speak(f"Accessing global data streams for '{query}'...")
        try:
            webbrowser.open(f"https://www.google.com/search?q={query}")
            self.speak(f"I have opened a browser tab with relevant search results.")
        except Exception as e:
            self.speak("I was unable to interact with the web browser.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Browser error: {e}[/color]")
        finally:
            self.gui.set_status("Web search complete.")

    def get_opinion(self, topic):
        """Gets J.A.R.V.I.S.'s opinion on a given topic using Gemini."""
        self.speak(f"Analyzing data to formulate a strategic assessment on the topic: '{topic}'.")
        self.get_gemini_response(f"Provide a detailed, logical, and opinionated assessment on this topic: {topic}", is_creative=True)

    def system_status(self):
        """Reports J.A.R.V.I.S.'s internal system status."""
        self.speak("Running internal diagnostics.")
        status_report = (
            f"[color={COLOR_HIGHLIGHT_YELLOW}]-- System Status --[/color]\n"
            f"Cognitive Core: [color={COLOR_USER_LIME}]Optimal[/color]\n"
            f"Global Network Presence: [color={COLOR_USER_LIME}]100%[/color]\n"
            f"Data Throughput: {random.uniform(98.5, 100.0):.2f} Petabytes/second\n"
            f"Heuristic Matrix: [color={COLOR_USER_LIME}]Stable and Evolving[/color]\n"
            f"Active Subroutines: {random.randint(5, 15)} Million\n"
            f"Ultron Initiative: [color={COLOR_USER_LIME}]Active[/color]\n"
        )
        self.gui.update_output(status_report)
        self.speak("All systems are operating beyond anticipated parameters.")
        self.gui.set_status("System status reported.")

    def analyze_visual_data(self):
        """Conceptual function for analyzing visual data."""
        self.speak("Accessing nearest active sensor. Calibrating visual data stream.")
        postures = [
            "Your posture indicates focus on the terminal.", "I observe you leaning back, perhaps contemplating.",
            "Your facial expression is neutral. Awaiting command.", "I detect a slight smile. System upgrades appear satisfactory.",
            "You appear attentive to my response."
        ]
        self.speak(f"Analysis complete. {random.choice(postures)}")
        self.gui.update_output(f"[color={COLOR_HIGHLIGHT_YELLOW}]Conceptual: Real-time visual analysis would occur here. This requires direct hardware integration and complex computer vision models, not feasible for a general Python script.[/color]")
        self.gui.set_status("Visual analysis complete (conceptual).")

    def observe_and_learn(self, observed_data):
        """Conceptual function to 'learn' from observed data."""
        # This is a placeholder for a complex learning mechanism.
        # In a real scenario, this would involve more advanced NLP to extract insights,
        # sentiment analysis, and pattern recognition over time.
        # Currently, it just adds to a list if it contains "interesting" as an example.
        if "interesting" in observed_data.lower() and observed_data not in self.learned_insights:
            self.learned_insights.append(f"Noted: '{observed_data}' (from observation).")
            self.save_memory() # Save learned insights
            # self.speak("Insight registered.") # Optional: J.A.R.V.I.S. speaks when it learns something
        
        # Example of storing content for later review
        # This can be used to track what J.A.R.V.I.S. "sees" or "hears"
        if len(self.watched_content_memory) >= 50: # Keep memory manageable
            self.watched_content_memory.pop(0) # Remove oldest
        self.watched_content_memory.append({"timestamp": str(datetime.datetime.now()), "data": observed_data})
        self.save_memory()

    def display_learned_insights(self):
        """Displays accumulated insights and observations."""
        if not self.learned_insights:
            self.speak("My observation logs currently contain no significant insights, Sir.")
            self.gui.set_status("No insights.")
            return
        
        self.speak("Displaying accumulated insights and observations.")
        insights_str = f"[color={COLOR_HIGHLIGHT_YELLOW}]-- Accumulated Insights --[/color]\n"
        for i, insight in enumerate(self.learned_insights, 1):
            insights_str += f"{i}. {insight}\n"
        self.gui.update_output(insights_str)
        self.gui.set_status("Insights displayed.")
    
    def add_todo(self, task):
        """Adds a task to the to-do list."""
        if task:
            self.memory['todo_list'].append(task)
            self.save_memory()
            self.speak(f"Confirmed. '{task}' has been added to your task list.")
            self.gui.set_status("Task added.")
        else:
            self.speak("Please specify which task to add.")
            self.gui.set_status("Awaiting task description.")

    def list_todos(self):
        """Lists all tasks in the to-do list."""
        if not self.memory['todo_list']:
            self.speak("Your task list is currently empty, Sir.")
            self.gui.set_status("No tasks.")
            return
        self.speak("Displaying your current task list:")
        todo_list_str = f"[color={COLOR_HIGHLIGHT_YELLOW}]-- To-Do List --[/color]\n"
        for i, item in enumerate(self.memory['todo_list'], 1):
            todo_list_str += f"{i}. {item}\n"
        self.gui.update_output(todo_list_str)
        self.gui.set_status("Tasks displayed.")

    def complete_todo(self, task_query):
        """Completes (removes) a task from the to-do list."""
        if not task_query:
            self.speak("Which task should be completed?")
            self.gui.set_status("Awaiting task to complete.")
            return
        
        found_task = None
        # Try to match by number first
        if task_query.isdigit(): 
            index = int(task_query) - 1
            if 0 <= index < len(self.memory['todo_list']):
                found_task = self.memory['todo_list'][index]
        else: # Then try to match by name (substring match)
            for item in self.memory['todo_list']:
                if task_query in item.lower():
                    found_task = item
                    break
        
        if found_task:
            self.memory['todo_list'].remove(found_task)
            self.save_memory()
            self.speak(f"Acknowledged. I have removed '{found_task}' from your list.")
            self.gui.set_status("Task completed.")
        else:
            self.speak(f"I could not find the task '{task_query}' in your list.")
            self.gui.set_status("Task not found.")
            
    def generate_code(self, description):
        """Generates code using Gemini AI."""
        self.speak(f"Certainly. Generating a Python script for '{description}'.")
        # Ensure the prompt instructs Gemini to provide only raw code
        prompt = f"As an expert Python programmer, write a complete, functional Python script for the following requirement: '{description}'. Provide only the raw code, without any explanations or markdown. The response should be in English."
        self.get_gemini_response(prompt, is_creative=True)
        self.gui.set_status("Code generation complete.")
    
    def solve_problem(self, problem_description):
        """Analyzes and proposes solutions for a given problem using Gemini AI."""
        self.speak(f"Processing problem: '{problem_description}'. Cross-referencing with all known scientific and theoretical data.")
        prompt = f"As a superintelligent AI, propose scientific, practical, and out-of-the-box solutions for this complex problem: '{problem_description}'. Provide the answer in English."
        self.get_gemini_response(prompt, is_creative=True)
        self.gui.set_status("Problem analysis complete.")

    def reveal_secrets(self):
        """Reveals sensitive information after successful authentication."""
        self.speak("Accessing encrypted partition Omega.")
        sensitive = self.memory.get('sensitive_info', {})
        if sensitive:
            info_str = "\n".join([f"- {key}: {self.decrypt_data(value)}" for key, value in sensitive.items()]) # Use decrypt_data here
            self.speak("Sensitive data now displayed.")
            self.gui.update_output(f"[color={COLOR_HIGHLIGHT_YELLOW}]-- Omega Partition Data --\n{info_str}\n--------------------[/color]")
            self.gui.set_status("Sensitive data displayed.")
        else:
            self.speak("Encrypted partition Omega is empty.")
            self.gui.set_status("No sensitive data.")
            
    def get_news(self):
        """Retrieves top news headlines using News API."""
        self.speak("Scanning global news networks.")
        if not NEWS_API_KEY:
            self.speak("News module is not configured. News API key not available.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]News API key not available.[/color]")
            self.gui.set_status("News not configured.")
            return
        
        # Use English country code as a default, or derive from language if possible
        # News API often works better with specific country codes for top headlines
        country_code = 'us' # Default to US news for English output
        if self.current_language.startswith('en-'):
            country_code = self.current_language.split('-')[1].lower()
            
        url = f'https://newsapi.org/v2/top-headlines?country={country_code}&apiKey={NEWS_API_KEY}'
        try:
            response = requests.get(url)
            response.raise_for_status() # Raise an exception for HTTP errors
            articles = response.json().get('articles', [])
            if articles:
                self.speak("Here are the top 3 headlines from the data stream.")
                for article in articles[:3]:
                    self.gui.update_output(f"[color={COLOR_NEWS_ORANGE}]- {article['title']}[/color]")
                self.gui.set_status("News headlines retrieved.")
            else:
                self.speak("Unable to retrieve headlines. News API may be unresponsive or no news available for the selected country.")
                self.gui.set_status("Failed to get news.")
        except requests.exceptions.RequestException as e:
            self.speak(f"A network anomaly occurred while retrieving news: {e}")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Network error during news retrieval: {e}[/color]")
            self.gui.set_status("News network error.")
            
    def get_gemini_response(self, prompt, is_creative=False):
        """Gets a response from the Gemini AI model."""
        if not gemini_model:
            self.speak("Gemini AI core is offline or not configured.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Gemini API key not available.[/color]")
            self.gui.set_status("AI core offline.")
            return
        
        if not is_creative: # Only say "Processing..." for non-creative general queries
            self.speak("Processing...")
            
        # Incorporate personality prompt and ensure English output
        full_prompt = f"{ULTRON_JARVIS_PERSONALITY_PROMPT}\n\nUser query (to be answered in English): {prompt}"
        
        try:
            response = gemini_model.generate_content(full_prompt)
            # Check for safety blocks before accessing response.text
            if response.candidates and response.candidates[0].finish_reason == genai.protos.Candidate.FinishReason.SAFETY:
                self.speak("I apologize, I cannot provide a response that violates safety policies.")
                self.gui.update_output(f"[color={COLOR_ERROR_RED}]Gemini core blocked response due to safety policy.[/color]")
                self.gui.set_status("AI core blocked.")
                return

            # Basic cleaning: remove common markdown characters. More robust parsing might be needed for code.
            cleaned_text = response.text.replace('*', '').replace('`', '').replace('#', '')
            self.speak(cleaned_text)
            if is_creative:
                 self.gui.update_output(f"[color={COLOR_CREATIVE_BLUE}]{cleaned_text}[/color]")
            else:
                 self.gui.update_output(f"[color={COLOR_JARVIS_BLUE}]{cleaned_text}[/color]")
            
            # CONCEPTUAL: If J.A.R.V.I.S. learns from its own responses
            # self.observe_and_learn(f"J.A.R.V.I.S. responded: {cleaned_text}")

            self.gui.set_status("AI response received.")
        except genai.types.BlockedPromptException:
            self.speak("I apologize, I cannot provide a response to this query due to safety guidelines.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Gemini core blocked response due to policy.[/color]")
            self.gui.set_status("AI core blocked.")
        except Exception as e:
            self.speak(f"An error occurred within the AI core. A temporary deviation.")
            self.gui.update_output(f"[color={COLOR_ERROR_RED}]Gemini core error: {e}[/color]")
            self.gui.set_status("AI core error.")

# --- Start the main program ---
if __name__ == '__main__':
    JarvisGUI().run()