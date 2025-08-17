from flask import Flask, render_template, request, jsonify
import os
import requests
from datetime import datetime
import os
import openai
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

app = Flask(__name__)

# Set your OpenAI API key from environment variables
api_key = os.environ.get('OPENAI_API_KEY')

# Check OpenAI version and set up client accordingly
try:
    # For newer OpenAI versions (>=1.0.0)
    from openai import OpenAI
    client = OpenAI(api_key=api_key)
    USING_NEW_CLIENT = True
except ImportError:
    # For older OpenAI versions
    openai.api_key = api_key
    USING_NEW_CLIENT = False

class WebAssistant:
    def __init__(self):
        self.reminders = []
        self.qa_sets = []
        self.skills = {
            'help': self.help,
            'time': self.tell_time,
            'weather': self.weather,
            'search': self.web_search,
            'calc': self.calculate,
            'reminder': self.add_reminder,
            'joke': self.tell_joke,
            'qa': self.get_qa_set,
        }

    def help(self, *args):
        cmds = ', '.join(self.skills.keys())
        return f"Available commands: {cmds}"

    def tell_time(self, *args):
        now = datetime.now().strftime('%H:%M:%S')
        return f'The current time is {now}.'

    def weather(self, city=None):
        if not city:
            return 'Please provide a city name.'
        return f'Weather in {city}: 22°C, partly cloudy (simulated data)'

    def web_search(self, query=None):
        if not query:
            return 'Please provide a search query.'
        return f'Search results for: {query}'

    def calculate(self, expr=None):
        if not expr:
            return 'Please provide an expression to calculate.'
        try:
            result = eval(expr, {'__builtins__': {}})
            return f'Result: {result}'
        except Exception:
            return 'Invalid expression.'

    def add_reminder(self, text=None):
        if not text:
            return 'Please provide reminder text.'
        self.reminders.append(text)
        return f'Reminder added: {text}'

    def tell_joke(self, *args):
        jokes = [
            "Why don't scientists trust atoms? Because they make up everything!",
            "Why did the scarecrow win an award? Because he was outstanding in his field!",
            "Why don't eggs tell jokes? They'd crack each other up!",
            "What do you call a fake noodle? An impasta!",
            "Why did the math book look so sad? Because it had too many problems!"
        ]
        import random
        return random.choice(jokes)
        
    def get_qa_set(self, topic=None):
        if not topic:
            return 'Please provide a topic for the Q&A set.'
            
        try:
            # Generate a set of questions and answers on the given topic using GPT
            prompt = f"Generate a set of 5 questions and answers about {topic}. Format each as 'Q: [question]\nA: [answer]' with a blank line between each Q&A pair."
            
            system_content = "You are an advanced GPT-4 AI assistant that provides comprehensive, accurate, and educational content in a Q&A format. Your answers should be detailed yet concise, and reflect the latest knowledge available to GPT-4."
            
            if USING_NEW_CLIENT:
                # For newer OpenAI client
                response = client.chat.completions.create(
                    model="gpt-4",  # Using GPT-4 (or specify "gpt-4-turbo" if available)
                    messages=[
                        {"role": "system", "content": system_content},
                        {"role": "user", "content": prompt}
                    ],
                    max_tokens=1000,
                    temperature=0.7
                )
                qa_content = response.choices[0].message.content
            else:
                # For older OpenAI client
                response = openai.ChatCompletion.create(
                    model="gpt-4",  # Using GPT-4 (or specify "gpt-4-turbo" if available)
                    messages=[
                        {"role": "system", "content": system_content},
                        {"role": "user", "content": prompt}
                    ],
                    max_tokens=1000,
                    temperature=0.7
                )
                qa_content = response.choices[0].message['content']
            
            # Store the Q&A set for future reference
            self.qa_sets.append({
                'topic': topic,
                'content': qa_content,
                'timestamp': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            })
            
            return qa_content
        except Exception as e:
            return f"Error generating Q&A set: {str(e)}"

    def ask_gpt(self, prompt):
        try:
            system_content = "You are an advanced GPT-4 AI assistant with enhanced capabilities for understanding context, generating creative content, and providing accurate information based on the latest knowledge available to GPT-4. Your responses should be helpful, informative, and tailored to the user's needs."
            
            if USING_NEW_CLIENT:
                # For newer OpenAI client
                response = client.chat.completions.create(
                    model="gpt-4",  # Using GPT-4 (or specify "gpt-4-turbo" if available)
                    messages=[
                        {"role": "system", "content": system_content},
                        {"role": "user", "content": prompt}
                    ],
                    max_tokens=800,  # Increased token limit for more detailed responses
                    temperature=0.7
                )
                return response.choices[0].message.content
            else:
                # For older OpenAI client
                response = openai.ChatCompletion.create(
                    model="gpt-4",  # Using GPT-4 (or specify "gpt-4-turbo" if available)
                    messages=[
                        {"role": "system", "content": system_content},
                        {"role": "user", "content": prompt}
                    ],
                    max_tokens=800,
                    temperature=0.7
                )
                return response.choices[0].message['content']
        except Exception as e:
            return f"Error connecting to GPT: {str(e)}"
    
    def process_command(self, user_input):
        original_input = user_input
        user_input = user_input.strip().lower()
        
        # Check for specific commands
        if user_input.startswith('weather'):
            city = user_input.replace('weather', '').strip()
            return self.weather(city)
        elif user_input.startswith('search'):
            query = user_input.replace('search', '').strip()
            return self.web_search(query)
        elif user_input.startswith('calc'):
            expr = user_input.replace('calc', '').strip()
            return self.calculate(expr)
        elif user_input.startswith('reminder'):
            text = user_input.replace('reminder', '').strip()
            return self.add_reminder(text)
        elif user_input.startswith('qa'):
            topic = user_input.replace('qa', '').strip()
            return self.get_qa_set(topic)
        
        # Check for direct skill matches
        for skill in self.skills:
            if user_input == skill:
                return self.skills[skill]()
        
        # If no specific command matches, use ChatGPT
        return self.ask_gpt(original_input)

# Initialize assistant
assistant = WebAssistant()

@app.route('/')
def home():
    return render_template('index.html')

@app.route('/api/chat', methods=['POST'])
def chat():
    data = request.get_json()
    user_message = data.get('message', '')
    voice_reply = data.get('voice_reply', False)
    
    if not user_message:
        return jsonify({'response': 'Please provide a message.', 'source': 'local'})
    
    # Check if this is a direct command or needs GPT
    user_input = user_message.strip().lower()
    is_gpt_response = True
    
    # Check for specific commands
    if any(user_input.startswith(cmd) for cmd in ['weather', 'search', 'calc', 'reminder', 'qa']) or user_input in assistant.skills:
        is_gpt_response = False
    
    response = assistant.process_command(user_message)
    
    return jsonify({
        'response': response,
        'voice_reply': voice_reply,
        'source': 'gpt' if is_gpt_response else 'local'
    })

@app.route('/api/reminders')
def get_reminders():
    return jsonify({'reminders': assistant.reminders})

@app.route('/api/qa_sets')
def get_qa_sets():
    return jsonify({'qa_sets': assistant.qa_sets})

if __name__ == '__main__':
    # Ensure templates directory exists
    os.makedirs('templates', exist_ok=True)
    app.run(debug=True, host='0.0.0.0', port=5000)
