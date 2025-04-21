import os
import time
import json
import requests
import telebot
from flask import Flask, request
from threading import Lock

bot_token = os.getenv("7857368182:AAFMjEaBQfgNcbsmN_JHzJ0UYPiOGrI3yT8")
gemini_key = os.getenv("AIzaSyDRaYQ-AaFvE1Wenj4zr8TbGFhidqhzgnE")
app_url = os.getenv("https://replit.com/@rlsarkargmniaib")
OWNER_IDS = list(map(int, os.getenv("7502366546", "7784548961","7554458891","7222400368","7857801709").split(",")))  # Comma-separated owner IDs

bot = telebot.TeleBot(7857368182:AAFMjEaBQfgNcbsmN_JHzJ0UYPiOGrI3yT8)
app = Flask(__name__)
lock = Lock()
last_used = {}
user_styles = {}
user_memory = {}
conversation_mode = {}

# Personality detection

def detect_personality(username, name, message_text, user_id):
    if user_id in user_styles:
        return user_styles[user_id]

    uname = (username or "").lower()
    name = (name or "").lower()
    msg = (message_text or "").lower()

    if any(k in uname for k in ["sad", "lonely", "broken"]) or "ভেঙে" in msg:
        return "একজন বিষণ্ণ প্রেমিক হিসেবে উত্তর দাও, যে হৃদয়ের কষ্ট বোঝে।"
    elif any(k in uname for k in ["cute", "girl", "maya", "angel"]):
        return "একজন মিষ্টি রোমান্টিক প্রেমিকা হিসেবে উত্তর দাও, যেন রূপকথার গল্পে বাস কর।"
    elif any(k in msg for k in ["funny", "joke", "হাসি"]):
        return "একজন মজার কিন্তু হৃদয়বান বন্ধুর মতো উত্তর দাও।"
    else:
        return "একজন একাকী, নীরব প্রেমিকের মতো উত্তর দাও, যার প্রতিটি শব্দে থাকে কাব্যের ছোঁয়া।"

# Gemini API call

def ask_gemini(prompt):
    url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key={AIzaSyDRaYQ-AaFvE1Wenj4zr8TbGFhidqhzgnE}"
    headers = {"Content-Type": "application/json"}
    data = {"contents": [{"parts": [{"text": prompt}]}]}
    res = requests.post(url, headers=headers, json=data)
    try:
        return res.json()['candidates'][0]['content']['parts'][0]['text']
    except:
        return "Romiyo একটু চুপ হয়ে গেছে, পরে আবার ডেকো!"

# Welcome new members
@bot.message_handler(content_types=["new_chat_members"])
def welcome_new_members(message):
    for user in message.new_chat_members:
        name = user.first_name
        welcome = f"🌟 স্বাগতম {name}!
RomiyoBot এখন তোমার পাশে, প্রশ্ন করো হৃদয়ের গভীরতা থেকে।"
        bot.send_message(message.chat.id, welcome)

# Commands
@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "🌹 স্বাগতম! আমি RomiyoBot, তোমার রোমান্টিক সহচর।\n👉 সাহায্যের জন্য /help কমান্ড ব্যবহার করো।")

@bot.message_handler(commands=['help'])
def send_help(message):
    help_text = (
        "🛠️ RomiyoBot কমান্ডসমূহ:\n"
        "/start - বট শুরু করুন\n"
        "/help - সাহায্য পেতে\n"
        "/ai [প্রশ্ন] - AI এর মাধ্যমে উত্তর পান\n"
        "/lovequote - দৈনিক প্রেমের উক্তি\n"
        "/mystory [তথ্য] - প্রেমের গল্প বানাও\n"
        "/confess [বার্তা] - গোপন প্রেমের বার্তা\n"
        "/voice [বার্তা] - ভয়েসে উত্তর চাইলে\n"
        "/setstyle [prompt] - নিজের স্টাইল নির্ধারণ (owner only)\n"
        "/chaton /chatoff - কনভারসেশন মোড চালু/বন্ধ (owner only)"
    )
    bot.reply_to(message, help_text)

@bot.message_handler(commands=['ai'])
def ai_reply(message):
    chat_id = message.chat.id
    user_id = message.from_user.id
    username = message.from_user.username
    name = message.from_user.first_name
    text = message.text.replace("/ai", "").strip()

    if not text:
        bot.reply_to(message, "প্রশ্নটা লিখো, আমি অপেক্ষায় আছি...")
        return

    now = time.time()
    if user_id in last_used and now - last_used[user_id] < 10:
        return
    last_used[user_id] = now

    bot.send_chat_action(chat_id, "typing")
    time.sleep(2.5)

    personality = detect_personality(username, name, text, user_id)
    if user_id in user_memory:
        prompt = f"তুমি আগে বলেছিলে: {user_memory[user_id]}\n\nএখন প্রশ্ন: {text}\n{personality}"
    else:
        prompt = f"{personality}\n\n{text}"
    reply = ask_gemini(prompt)
    user_memory[user_id] = text
    bot.reply_to(message, reply)

@bot.message_handler(commands=['lovequote'])
def send_love_quote(message):
    quote = ask_gemini("একটি সুন্দর প্রেমের উক্তি দাও।")
    bot.reply_to(message, f"💖 {quote}")

@bot.message_handler(commands=['mystory'])
def make_love_story(message):
    text = message.text.replace("/mystory", "").strip()
    story_prompt = f"তুমি একজন কবি। নিম্নলিখিত তথ্য দিয়ে প্রেমের গল্প বানাও:\n{text}"
    story = ask_gemini(story_prompt)
    bot.reply_to(message, story)

@bot.message_handler(commands=['setstyle'])
def set_style(message):
    if message.from_user.id not in OWNER_IDS:
        bot.reply_to(message, "এই কমান্ডটি শুধুমাত্র Romiyo এর অনুমতিপ্রাপ্ত ব্যবহারকারীরা ব্যবহার করতে পারেন।")
        return
    prompt = message.text.replace("/setstyle", "").strip()
    user_styles[message.reply_to_message.from_user.id] = prompt
    bot.reply_to(message, "নতুন পার্সোনালিটি সেট করা হলো।")

@bot.message_handler(commands=['voice'])
def voice_reply(message):
    text = message.text.replace("/voice", "").strip()
    prompt = f"একজন রোমান্টিক প্রেমিকের মতো কিছু বলো: {text}"
    reply = ask_gemini(prompt)
    bot.reply_to(message, reply)  # Voice feature placeholder

@bot.message_handler(commands=['chaton'])
def enable_conversation(message):
    if message.from_user.id in OWNER_IDS:
        conversation_mode[message.chat.id] = True
        bot.reply_to(message, "চ্যাট মোড চালু করা হয়েছে।")

@bot.message_handler(commands=['chatoff'])
def disable_conversation(message):
    if message.from_user.id in OWNER_IDS:
        conversation_mode[message.chat.id] = False
        bot.reply_to(message, "চ্যাট মোড বন্ধ করা হয়েছে।")

@bot.message_handler(commands=['confess'])
def confess_love(message):
    confession = message.text.replace("/confess", "").strip()
    prompt = f"একটি গোপন প্রেমের বার্তা লেখো: {confession}"
    result = ask_gemini(prompt)
    bot.reply_to(message, f"গোপন বার্তা ✉️:\n{result}")

# Inline query support (basic)
@bot.inline_handler(lambda query: True)
def inline_query_handler(inline_query):
    prompt = detect_personality(inline_query.from_user.username, inline_query.from_user.first_name, inline_query.query, inline_query.from_user.id)
    response = ask_gemini(f"{prompt}\n\n{inline_query.query}")
    results = [
        telebot.types.InlineQueryResultArticle(
            id='1',
            title="RomiyoBot AI উত্তর",
            input_message_content=telebot.types.InputTextMessageContent(message_text=response)
        )
    ]
    bot.answer_inline_query(inline_query.id, results)

@app.route(f"/{7857368182:AAFMjEaBQfgNcbsmN_JHzJ0UYPiOGrI3yT8}", methods=["POST"])
def webhook():
    with lock:
        bot.process_new_updates([telebot.types.Update.de_json(request.stream.read().decode("utf-8"))])
    return "OK", 200

@app.route("/")
def index():
    return "RomiyoBot is alive!"

if __name__ == "__main__":
    bot.remove_webhook()
    bot.set_webhook(url=f"{app_url}/{bot_token}")
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
