import speech_recognition as sr
import pyttsx3
import datetime
import webbrowser
import requests
import os
import re
from urllib.parse import quote

class VoiceAssistant:
    def __init__(self, name="Assistant"):
        self.engine = pyttsx3.init()
        self.name = name
        self.recognizer = sr.Recognizer()
        
        # Configure speech recognition parameters
        self.recognizer.energy_threshold = 4000
        self.recognizer.dynamic_energy_threshold = True
        
        # Configure voice properties
        voices = self.engine.getProperty('voices')
        self.engine.setProperty('voice', voices[1].id)
        self.engine.setProperty('rate', 150)
        self.engine.setProperty('volume', 0.9)

    def speak(self, text):
        """Convert text to speech"""
        print(f"{self.name}: {text}")
        self.engine.say(text)
        self.engine.runAndWait()
        
    def listen(self):
        """Listen for user input through microphone with better configuration"""
        with sr.Microphone() as source:
            # Adjust for ambient noise with longer calibration
            self.recognizer.adjust_for_ambient_noise(source, duration=2)
            print("Listening... (speak now)")
            
            # Capture audio with explicit timeouts
            audio = self.recognizer.listen(
                source,
                timeout=5,
                phrase_time_limit=8
            )
            
            # Perform speech recognition
            text = self.recognizer.recognize_google(
                audio,
                language="en-US",
                show_all=False
            ).lower()
            
            print(f"You: {text}")
            return text

    def get_time(self):
        """Get current time"""
        current_time = datetime.datetime.now().strftime("%I:%M %p")
        self.speak(f"The current time is {current_time}")

    def get_weather(self, city):
        """Get weather information for a city"""
        API_KEY = os.getenv("WEATHER_API_KEY")
        base_url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={API_KEY}&units=metric"
        
        response = requests.get(base_url, timeout=5)
        response.raise_for_status()
        data = response.json()
        
        if data["cod"] == 200:
            temp = data["main"]["temp"]
            desc = data["weather"][0]["description"]
            self.speak(f"The temperature in {city} is {temp}°C with {desc}")
        else:
            self.speak("Couldn't retrieve weather information for that city.")

    def search_web(self, query):
        """Search the web using default browser"""
        url = f"https://www.google.com/search?q={quote(query)}"
        webbrowser.open(url)
        self.speak(f"Searching for {query}")

    def process_command(self, command):
        """Process voice commands with improved pattern matching"""
        # Normalize command text
        command = re.sub(r'[^\w\s]', '', command).lower()
        
        if "time" in command:
            self.get_time()
            
        elif "weather" in command:
            # Improved city extraction using regex
            match = re.search(r'weather (?:in|for) (.+)', command)
            if match:
                city = match.group(1).strip()
                self.get_weather(city)
            else:
                self.speak("Please specify a city for weather information.")
                
        elif "search for" in command:
            query = re.sub(r'search for', '', command).strip()
            if query:
                self.search_web(query)
            else:
                self.speak("What would you like me to search for?")
            
        elif "goodbye" in command:
            self.speak("Goodbye! Have a great day!")
            return False
            
        elif any(word in command for word in ["hello", "hi", "hey"]):
            self.speak(f"Hello! How can I help you today?")
            
        else:
            self.speak("Could you please rephrase that command?")
            
        return True

    def run(self):
        """Main loop with better user feedback"""
        self.speak(f"Hello! I'm {self.name}, ready to assist you.")
        
        running = True
        while running:
            command = self.listen()
            running = self.process_command(command)

def main():
    assistant = VoiceAssistant("Alice")
    assistant.run()

if __name__ == "__main__":
    main()
