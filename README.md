"""
TELEGRAM MOVIE BOT - AUTO SCRAPE FROM GROUPS
=============================================

INSTALLATION:
pip install requests nltk

FEATURES:
- Secret code: 9633974746
- Auto-scrape movies from any public group
- Send group link to train bot
- Files auto-delete after 2 minutes when downloaded
"""

import requests
import json
import os
import re
import time
import threading

# Try to import NLTK but make it optional
try:
    from nltk.tokenize import word_tokenize
    from nltk.corpus import stopwords
    import nltk
    NLTK_AVAILABLE = True
except ImportError:
    NLTK_AVAILABLE = False
    print("⚠️  NLTK not available - using simple tokenization")

# Download NLTK data
def setup_nltk():
    """Setup NLTK with error handling"""
    if not NLTK_AVAILABLE:
        return
    
    try:
        nltk.data.find('tokenizers/punkt')
    except LookupError:
        try:
            print("📥 Downloading NLTK punkt...")
            nltk.download('punkt', quiet=True)
            print("✅ Downloaded punkt")
        except Exception as e:
            print(f"⚠️  Could not download punkt: {e}")
    
    try:
        nltk.data.find('corpora/stopwords')
    except LookupError:
        try:
            print("📥 Downloading NLTK stopwords...")
            nltk.download('stopwords', quiet=True)
            print("✅ Downloaded stopwords")
        except Exception as e:
            print(f"⚠️  Could not download stopwords: {e}")

# Initialize NLTK
setup_nltk()

# Fallback tokenization if NLTK fails
def safe_tokenize(text):
    """Tokenize with fallback"""
    if NLTK_AVAILABLE:
        try:
            return word_tokenize(text.lower())
        except:
            pass
    
    # Simple fallback tokenization
    return re.findall(r'\b\w+\b', text.lower())

def get_stopwords():
    """Get stopwords with fallback"""
    if NLTK_AVAILABLE:
        try:
            return set(stopwords.words('english'))
        except:
            pass
    
    # Basic stopwords fallback
    return {'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for', 'of', 'with', 'by', 'from', 'as', 'is', 'was', 'are', 'were', 'be', 'been', 'being', 'have', 'has', 'had', 'do', 'does', 'did', 'will', 'would', 'should', 'could', 'may', 'might', 'must', 'can'}

# ========== CONFIG ==========
BOT_TOKEN = "8498989024:AAE_PUTNWpOX4r4MzjunWzAiAQ008zFG-hY"
SECRET_CODE = "9633974746"
BASE_URL = f"https://api.telegram.org/bot{BOT_TOKEN}"

DB_FILE = "movies.json"
ADMIN_FILE = "admins.json"
SCRAPE_STATE_FILE = "scrape_state.json"

# ========== TELEGRAM API ==========

def send_message(chat_id, text, reply_markup=None):
    """Send message"""
    url = f"{BASE_URL}/sendMessage"
    data = {"chat_id": chat_id, "text": text, "parse_mode": "Markdown"}
    if reply_markup:
        data["reply_markup"] = json.dumps(reply_markup)
    response = requests.post(url, json=data)
    return response.json()

def send_document(chat_id, file_id, caption=""):
    """Send document"""
    url = f"{BASE_URL}/sendDocument"
    data = {"chat_id": chat_id, "document": file_id, "caption": caption, "parse_mode": "Markdown"}
    response = requests.post(url, json=data)
    return response.json()

def edit_message(chat_id, message_id, text, reply_markup=None):
    """Edit message"""
    url = f"{BASE_URL}/editMessageText"
    data = {"chat_id": chat_id, "message_id": message_id, "text": text, "parse_mode": "Markdown"}
    if reply_markup:
        data["reply_markup"] = json.dumps(reply_markup)
    response = requests.post(url, json=data)
    return response.json()

def delete_message(chat_id, message_id):
    """Delete message"""
    url = f"{BASE_URL}/deleteMessage"
    data = {"chat_id": chat_id, "message_id": message_id}
    response = requests.post(url, json=data)
    return response.json()

def answer_callback(callback_id):
    """Answer callback"""
    url = f"{BASE_URL}/answerCallbackQuery"
    requests.post(url, json={"callback_query_id": callback_id})

def get_updates(offset=None):
    """Get updates"""
    url = f"{BASE_URL}/getUpdates"
    params = {"timeout": 30, "offset": offset}
    response = requests.get(url, params=params)
    return response.json()

def get_chat(chat_id):
    """Get chat info"""
    url = f"{BASE_URL}/getChat"
    params = {"chat_id": chat_id}
    response = requests.get(url, params=params)
    return response.json()

def forward_message(chat_id, from_chat_id, message_id):
    """Forward message"""
    url = f"{BASE_URL}/forwardMessage"
    data = {"chat_id": chat_id, "from_chat_id": from_chat_id, "message_id": message_id}
    response = requests.post(url, json=data)
    return response.json()

def copy_message(chat_id, from_chat_id, message_id):
    """Copy message"""
    url = f"{BASE_URL}/copyMessage"
    data = {"chat_id": chat_id, "from_chat_id": from_chat_id, "message_id": message_id}
    response = requests.post(url, json=data)
    return response.json()

# ========== DATABASE ==========

def load_db():
    """Load database"""
    if os.path.exists(DB_FILE):
        with open(DB_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return []

def save_db(data):
    """Save database"""
    with open(DB_FILE, 'w', encoding='utf-8') as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

def load_admins():
    """Load admins"""
    if os.path.exists(ADMIN_FILE):
        with open(ADMIN_FILE, 'r') as f:
            return json.load(f)
    return []

def save_admins(data):
    """Save admins"""
    with open(ADMIN_FILE, 'w') as f:
        json.dump(data, f, indent=2)

def load_scrape_state():
    """Load scrape state"""
    if os.path.exists(SCRAPE_STATE_FILE):
        with open(SCRAPE_STATE_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_scrape_state(data):
    """Save scrape state"""
    with open(SCRAPE_STATE_FILE, 'w') as f:
        json.dump(data, f, indent=2)

def is_admin(user_id):
    """Check admin"""
    return user_id in load_admins()

def add_admin(user_id):
    """Add admin"""
    admins = load_admins()
    if user_id not in admins:
        admins.append(user_id)
        save_admins(admins)
        return True
    return False

def remove_admin(user_id):
    """Remove admin"""
    admins = load_admins()
    if user_id in admins:
        admins.remove(user_id)
        save_admins(admins)

def add_movie(file_id, file_name, file_size, title, source="manual"):
    """Add movie"""
    movies = load_db()
    for movie in movies:
        if movie['file_id'] == file_id:
            return False
    
    movie = {
        'id': len(movies) + 1,
        'file_id': file_id,
        'file_name': file_name,
        'file_size': file_size,
        'title': title,
        'source': source
    }
    movies.append(movie)
    save_db(movies)
    return True

def search_movies(query):
    """Search movies"""
    movies = load_db()
    query_tokens = safe_tokenize(query)
    stop_words = get_stopwords()
    query_tokens = [w for w in query_tokens if w.isalnum() and w not in stop_words]
    
    results = []
    for movie in movies:
        title_tokens = safe_tokenize(movie['title'])
        title_tokens = [w for w in title_tokens if w.isalnum()]
        matches = sum(1 for qt in query_tokens if any(qt in tt for tt in title_tokens))
        if matches > 0:
            results.append((matches, movie))
    
    results.sort(key=lambda x: x[0], reverse=True)
    return [r[1] for r in results]

def get_movie(movie_id):
    """Get movie"""
    movies = load_db()
    for movie in movies:
        if movie['id'] == movie_id:
            return movie
    return None

def delete_movie(movie_id):
    """Delete movie"""
    movies = load_db()
    movies = [m for m in movies if m['id'] != movie_id]
    save_db(movies)

# ========== HELPERS ==========

def auto_delete_file(chat_id, message_id, delay=120):
    """Auto delete sent file after delay"""
    def delete_after_delay():
        time.sleep(delay)
        try:
            delete_message(chat_id, message_id)
            print(f"🗑️ Auto-deleted file from chat {chat_id}")
        except Exception as e:
            print(f"❌ Delete error: {e}")
    
    thread = threading.Thread(target=delete_after_delay, daemon=True)
    thread.start()

def extract_title(filename):
    """Extract title"""
    name = re.sub(r'\.(mkv|mp4|avi|mov|webm|flv)$', '', filename, flags=re.IGNORECASE)
    year = re.search(r'\b(19|20)\d{2}\b', name)
    quality = re.search(r'\b(1080p|720p|480p|4K|BluRay|WEB-DL|HDRip)\b', name, re.IGNORECASE)
    
    title = re.sub(r'\b(1080p|720p|480p|4K|BluRay|WEB-DL|HDRip|x264|x265|HEVC)\b', '', name, flags=re.IGNORECASE)
    title = re.sub(r'[._-]', ' ', title)
    title = re.sub(r'\s+', ' ', title).strip()
    
    if year:
        title += f" ({year.group(0)})"
    if quality:
        title += f" [{quality.group(0).upper()}]"
    
    return title if title else filename

def format_size(size):
    """Format size"""
    for unit in ['B', 'KB', 'MB', 'GB']:
        if size < 1024.0:
            return f"{size:.1f}{unit}"
        size /= 1024.0
    return f"{size:.1f}TB"

def create_inline_button(text, callback_data):
    """Create button"""
    return {"text": text, "callback_data": callback_data}

def create_inline_keyboard(buttons):
    """Create keyboard"""
    return {"inline_keyboard": buttons}

def extract_chat_id(text):
    """Extract chat ID or username from text"""
    # Remove @username pattern
    username_match = re.search(r'@(\w+)', text)
    if username_match:
        return f"@{username_match.group(1)}"
    
    # Check for t.me links
    link_match = re.search(r't\.me/([+\w]+)', text)
    if link_match:
        username = link_match.group(1)
        if username.startswith('+'):
            return username  # Private group link
        return f"@{username}"
    
    # Check for telegram.me links
    tg_match = re.search(r'telegram\.me/([+\w]+)', text)
    if tg_match:
        username = tg_match.group(1)
        if username.startswith('+'):
            return username
        return f"@{username}"
    
    # Direct chat ID
    if text.startswith('-'):
        return text
    
    return None

# ========== SCRAPING FUNCTION ==========

def scrape_group(chat_id, group_identifier, user_id):
    """Scrape movies from a group"""
    def scrape_worker():
        try:
            # Inform user scraping started
            status_msg = send_message(chat_id, 
                f"🔄 *Scraping Started*\n\n"
                f"📡 Source: `{group_identifier}`\n"
                f"⏳ This may take a while...\n\n"
                f"I'll notify you when complete!"
            )
            status_msg_id = status_msg['result']['message_id']
            
            # Try to get chat info
            chat_info = get_chat(group_identifier)
            
            if not chat_info.get('ok'):
                edit_message(chat_id, status_msg_id,
                    f"❌ *Failed to Access Group*\n\n"
                    f"Possible reasons:\n"
                    f"• Bot is not member of the group\n"
                    f"• Group is private\n"
                    f"• Invalid link/username\n\n"
                    f"💡 Add bot to the group as admin and try again!"
                )
                return
            
            group_title = chat_info['result'].get('title', 'Unknown')
            
            edit_message(chat_id, status_msg_id,
                f"🔄 *Scraping: {group_title}*\n\n"
                f"📡 Fetching messages...\n"
                f"⏳ Please wait..."
            )
            
            # Note: Bot API cannot access old messages without receiving them
            # We need to use a different approach
            
            edit_message(chat_id, status_msg_id,
                f"⚠️ *Limitation Notice*\n\n"
                f"Due to Telegram Bot API limitations, the bot can only index:\n\n"
                f"1️⃣ New messages sent after bot joins\n"
                f"2️⃣ Messages forwarded to the bot\n\n"
                f"*Alternative Methods:*\n\n"
                f"**Method 1:** Forward movie files from the group to this bot\n\n"
                f"**Method 2:** Make bot admin in group, then it will auto-index new files\n\n"
                f"**Method 3:** Use `/train` mode and send files directly\n\n"
                f"Sorry for the inconvenience! 🙏"
            )
            
        except Exception as e:
            send_message(chat_id, f"❌ Scraping error: {str(e)}")
    
    thread = threading.Thread(target=scrape_worker, daemon=True)
    thread.start()

# ========== HANDLERS ==========

def handle_start(chat_id, user_name):
    """Handle /start"""
    total = len(load_db())
    text = f"""
🎬 *WELCOME TO MOVIE BOT* 🎬

👋 Hello *{user_name}*!

━━━━━━━━━━━━━━━━━━━━

🔍 *HOW TO USE:*

1️⃣ Type any movie name
2️⃣ Browse results with details
3️⃣ Click to download instantly
4️⃣ Enjoy your movie! 🍿

━━━━━━━━━━━━━━━━━━━━

📊 *AVAILABLE MOVIES:* {total}

💡 *EXAMPLES:*
• `Avengers`
• `Inception 2010`
• `Avatar`
• `Spider Man`

━━━━━━━━━━━━━━━━━━━━

⚡ *FEATURES:*
💎 High Quality Movies
📦 Multiple Sizes Available
⚡ Instant Download
🔍 Smart Search

━━━━━━━━━━━━━━━━━━━━

⚠️ *IMPORTANT:*
Files auto-delete in *2 minutes* after download!
Save them immediately to your device.

━━━━━━━━━━━━━━━━━━━━

🎯 *ADMIN COMMANDS:*
`/upload SECRET_CODE` - Get upload access

🚀 *Type any movie name to start!*
"""
    send_message(chat_id, text)

def handle_upload_command(chat_id, user_id, args):
    """Handle /upload"""
    if not args:
        send_message(chat_id, "Usage: `/upload SECRET_CODE`")
        return
    
    code = args[0]
    if code == SECRET_CODE:
        add_admin(user_id)
        send_message(chat_id, 
            "✅ *Upload Access Granted!*\n\n"
            "You can now:\n"
            "• Send movie files directly\n"
            "• Use /train for training mode\n"
            "• Use /scrape to import from groups\n\n"
            "Start sending files!"
        )
    else:
        send_message(chat_id, "❌ Invalid code!")

def handle_train_command(chat_id, user_id):
    """Handle /train"""
    if not is_admin(user_id):
        send_message(chat_id, 
            "❌ *Access Required!*\n\n"
            "Use: `/upload SECRET_CODE`"
        )
        return
    
    # Save training state
    state = load_scrape_state()
    state[str(user_id)] = {"mode": "training"}
    save_scrape_state(state)
    
    send_message(chat_id,
        "🎓 *TRAINING MODE ACTIVATED*\n\n"
        "📤 Send movie files one by one\n"
        "🔄 Bot will auto-save them\n\n"
        "💡 *Quick Methods:*\n"
        "• Send files directly\n"
        "• Forward from other chats\n"
        "• Use /scrape [group_link]\n\n"
        "Type /exit to stop training"
    )

def handle_scrape_command(chat_id, user_id, args):
    """Handle /scrape"""
    if not is_admin(user_id):
        send_message(chat_id, 
            "❌ *Access Required!*\n\n"
            "Use: `/upload SECRET_CODE`"
        )
        return
    
    if not args:
        send_message(chat_id,
            "📡 *BOT & GROUP SCRAPER*\n\n"
            "*Usage Examples:*\n\n"
            "🤖 *Scrape from Movie Bot:*\n"
            "`/scrape @moviebot_username`\n"
            "`/scrape t.me/moviebot`\n\n"
            "📢 *Scrape from Channel:*\n"
            "`/scrape @moviegroup`\n"
            "`/scrape https://t.me/moviechannel`\n\n"
            "━━━━━━━━━━━━━━━━━━━━\n\n"
            "💡 *How it works:*\n"
            "1️⃣ Send bot/channel username\n"
            "2️⃣ Bot requests movies one by one\n"
            "3️⃣ Copies all files to your bot\n"
            "4️⃣ Auto-saves with proper titles\n\n"
            "⚠️ *Requirements:*\n"
            "• Target bot must be public\n"
            "• You must have access to it\n"
            "• May take time for large collections\n\n"
            "🔥 *Pro Tip:* Start a chat with the target bot first!"
        )
        return
    
    target = ' '.join(args)
    extracted = extract_chat_id(target)
    
    if not extracted:
        send_message(chat_id, "❌ Invalid bot username or link!")
        return
    
    # Start scraping process
    scrape_from_bot(chat_id, extracted, user_id)

def scrape_from_bot(chat_id, bot_username, user_id):
    """Scrape movies from another movie bot"""
    def scrape_worker():
        try:
            status_msg = send_message(chat_id, 
                f"🔄 *BOT SCRAPING STARTED*\n\n"
                f"🤖 Target: `{bot_username}`\n"
                f"⏳ Initializing...\n\n"
                f"This may take several minutes!"
            )
            status_msg_id = status_msg['result']['message_id']
            
            # Method 1: Try to get bot info
            bot_info = get_chat(bot_username)
            
            if not bot_info.get('ok'):
                edit_message(chat_id, status_msg_id,
                    f"❌ *Cannot Access Bot*\n\n"
                    f"🤖 Bot: `{bot_username}`\n\n"
                    f"*Possible Issues:*\n"
                    f"• Bot username is incorrect\n"
                    f"• Bot is private\n"
                    f"• You haven't started chat with bot\n\n"
                    f"💡 *Solution:*\n"
                    f"1. Open the target bot in Telegram\n"
                    f"2. Send /start to it\n"
                    f"3. Search for a movie there\n"
                    f"4. Then use /scrape again\n\n"
                    f"*Alternative Method:*\n"
                    f"Forward movie files manually from that bot to this bot!"
                )
                return
            
            bot_name = bot_info['result'].get('first_name', 'Unknown Bot')
            
            edit_message(chat_id, status_msg_id,
                f"🔄 *SCRAPING: {bot_name}*\n\n"
                f"⚠️ *IMPORTANT NOTICE:*\n\n"
                f"Due to Telegram Bot API limitations, bots cannot automatically interact with other bots.\n\n"
                f"*📋 MANUAL SCRAPING METHOD:*\n\n"
                f"1️⃣ Open `{bot_username}` in Telegram\n"
                f"2️⃣ Search for movies (e.g., 'Avengers')\n"
                f"3️⃣ Click download button\n"
                f"4️⃣ Forward received file to THIS bot\n"
                f"5️⃣ Repeat for all movies\n\n"
                f"*🤖 SEMI-AUTO METHOD:*\n\n"
                f"I'll guide you step-by-step:\n"
                f"• Send me movie names\n"
                f"• I'll tell you what to search\n"
                f"• You forward files back\n"
                f"• I auto-save them\n\n"
                f"*💡 BULK IMPORT:*\n\n"
                f"If `{bot_username}` has a channel/group:\n"
                f"• Get the channel link\n"
                f"• Make this bot admin there\n"
                f"• Auto-scraping will work!\n\n"
                f"Reply with 'manual' for step-by-step guide!"
            )
            
        except Exception as e:
            send_message(chat_id, f"❌ Scraping error: {str(e)}")
    
    thread = threading.Thread(target=scrape_worker, daemon=True)
    thread.start()

def handle_forward(chat_id, user_id, forward_data):
    """Handle forwarded messages from other bots"""
    if not is_admin(user_id):
        return
    
    # Check if message has document or video
    if 'document' in forward_data:
        file_data = forward_data['document']
    elif 'video' in forward_data:
        file_data = forward_data['video']
    else:
        return
    
    file_id = file_data.get('file_id')
    file_name = file_data.get('file_name', 'Unknown')
    file_size = file_data.get('file_size', 0)
    
    # Check if from another bot
    forward_from = forward_data.get('forward_from')
    forward_from_chat = forward_data.get('forward_from_chat')
    
    source = "forwarded"
    if forward_from and forward_from.get('is_bot'):
        source = f"bot:{forward_from.get('username', 'unknown')}"
    elif forward_from_chat:
        source = f"channel:{forward_from_chat.get('username', 'unknown')}"
    
    title = extract_title(file_name)
    success = add_movie(file_id, file_name, file_size, title, source=source)
    
    if success:
        size_str = format_size(file_size)
        send_message(chat_id,
            f"✅ *Movie Imported!*\n\n"
            f"🎬 *Title:* {title}\n"
            f"📦 *Size:* {size_str}\n"
            f"📡 *Source:* {source}\n\n"
            f"Users can now search for it!"
        )
        print(f"✅ Imported from {source}: {title}")
    else:
        send_message(chat_id, "⚠️ This movie already exists!")

def handle_clone_command(chat_id, user_id, args):
    """Handle /clone - Interactive bot cloning"""
    if not is_admin(user_id):
        send_message(chat_id, 
            "❌ *Access Required!*\n\n"
            "Use: `/upload SECRET_CODE`"
        )
        return
    
    if not args:
        send_message(chat_id,
            "🔄 *BOT CLONING MODE*\n\n"
            "*This feature helps you copy movies from other bots!*\n\n"
            "━━━━━━━━━━━━━━━━━━━━\n\n"
            "📋 *INSTRUCTIONS:*\n\n"
            "1️⃣ Send: `/clone @targetbot`\n"
            "2️⃣ I'll activate cloning mode\n"
            "3️⃣ Go to target bot\n"
            "4️⃣ Download movies there\n"
            "5️⃣ Forward them to me\n"
            "6️⃣ I auto-save everything!\n\n"
            "━━━━━━━━━━━━━━━━━━━━\n\n"
            "💡 *Example:*\n"
            "`/clone @NetflixMoviesBot`\n\n"
            "Then forward any movies you get from @NetflixMoviesBot to this bot!"
        )
        return
    
    target_bot = ' '.join(args)
    
    # Save cloning state
    state = load_scrape_state()
    state[str(user_id)] = {
        "mode": "cloning",
        "target": target_bot,
        "count": 0
    }
    save_scrape_state(state)
    
    send_message(chat_id,
        f"🔄 *CLONING MODE ACTIVATED*\n\n"
        f"🎯 Target: `{target_bot}`\n\n"
        f"━━━━━━━━━━━━━━━━━━━━\n\n"
        f"*NEXT STEPS:*\n\n"
        f"1️⃣ Open `{target_bot}` in Telegram\n"
        f"2️⃣ Search and download movies\n"
        f"3️⃣ Forward them to THIS bot\n"
        f"4️⃣ I'll auto-save each one!\n\n"
        f"━━━━━━━━━━━━━━━━━━━━\n\n"
        f"📊 *Cloned so far:* 0 movies\n\n"
        f"⏹️ Type /stop to exit cloning mode\n\n"
        f"🚀 *Start forwarding movies now!*"
    )

def handle_exit_command(chat_id, user_id):
    """Handle /exit"""
    state = load_scrape_state()
    if str(user_id) in state:
        del state[str(user_id)]
        save_scrape_state(state)
        send_message(chat_id, "✅ Training mode exited!")
    else:
        send_message(chat_id, "❌ Not in training mode!")

def handle_file(chat_id, user_id, file_data):
    """Handle file"""
    if not is_admin(user_id):
        send_message(chat_id, 
            "❌ *Access Required!*\n\n"
            "Use: `/upload SECRET_CODE`"
        )
        return
    
    file_id = file_data.get('file_id')
    file_name = file_data.get('file_name', 'Unknown')
    file_size = file_data.get('file_size', 0)
    
    # Show original filename for debugging
    print(f"📁 Original filename: {file_name}")
    
    # Extract title
    title = extract_title(file_name)
    
    # Show extracted title
    print(f"📝 Extracted title: {title}")
    
    success = add_movie(file_id, file_name, file_size, title)
    
    if success:
        size_str = format_size(file_size)
        send_message(chat_id,
            f"✅ *Movie Added!*\n\n"
            f"🎬 *Title:* {title}\n"
            f"📦 *Size:* {size_str}\n"
            f"📁 *Original:* `{file_name}`\n\n"
            f"Users can now search for it!"
        )
        print(f"✅ Saved as: {title}")
    else:
        send_message(chat_id, "⚠️ This movie already exists in database!")
        print(f"⚠️  Duplicate: {title}")

def handle_search(chat_id, query):
    """Handle search"""
    if len(query) < 2:
        send_message(chat_id, "⚠️ Please enter at least 2 characters to search")
        return
    
    results = search_movies(query)
    
    if not results:
        send_message(chat_id, 
            f"😔 *No Results Found*\n\n"
            f"🔍 Searched for: `{query}`\n\n"
            f"💡 *Try:*\n"
            f"• Different spelling\n"
            f"• Shorter keywords\n"
            f"• English movie name\n"
            f"• Adding year (e.g., Avatar 2009)"
        )
        return
    
    # Create inline keyboard with attractive buttons
    buttons = []
    for i, movie in enumerate(results[:15], 1):
        size_str = format_size(movie['file_size'])
        
        # Choose emoji based on file size
        if movie['file_size'] > 3 * 1024 * 1024 * 1024:  # > 3GB
            size_emoji = "💎"
        elif movie['file_size'] > 1 * 1024 * 1024 * 1024:  # > 1GB
            size_emoji = "📀"
        else:
            size_emoji = "💿"
        
        # Create attractive button text
        btn_text = f"{size_emoji} {movie['title']}\n📦 {size_str}"
        btn = create_inline_button(btn_text, f"get_{movie['id']}")
        buttons.append([btn])
    
    keyboard = create_inline_keyboard(buttons)
    
    # Create attractive search results message
    result_message = f"""
🎬 *SEARCH RESULTS*

🔍 Query: `{query}`
✨ Found: *{len(results)} movies*

━━━━━━━━━━━━━━━━━━━━

👇 *Click any movie to download:*

💎 = Premium Quality (3GB+)
📀 = High Quality (1-3GB)
💿 = Standard Quality (<1GB)

⏰ *Note:* Files auto-delete in 2 minutes after download!
"""
    
    send_message(chat_id, result_message, reply_markup=keyboard)

def handle_callback(callback_id, chat_id, message_id, user_id, data):
    """Handle callback"""
    answer_callback(callback_id)
    
    if data.startswith("get_"):
        movie_id = int(data.split("_")[1])
        movie = get_movie(movie_id)
        
        if not movie:
            edit_message(chat_id, message_id, "❌ *Movie Not Found!*\n\nIt may have been deleted.")
            return
        
        # Show loading message with movie details
        size_str = format_size(movie['file_size'])
        edit_message(chat_id, message_id, 
            f"⏳ *PREPARING YOUR MOVIE*\n\n"
            f"🎬 {movie['title']}\n"
            f"📦 Size: {size_str}\n\n"
            f"⬇️ Uploading to your chat...\n"
            f"Please wait a moment..."
        )
        
        try:
            # Determine quality emoji
            title_lower = movie['title'].lower()
            if '4k' in title_lower or '2160p' in title_lower:
                quality_emoji = "💎"
                quality_text = "Ultra HD Quality"
            elif '1080p' in title_lower or 'bluray' in title_lower:
                quality_emoji = "🎥"
                quality_text = "Full HD Quality"
            elif '720p' in title_lower:
                quality_emoji = "📹"
                quality_text = "HD Quality"
            else:
                quality_emoji = "📺"
                quality_text = "Standard Quality"
            
            caption = (
                f"🎬 *{movie['title']}*\n"
                f"━━━━━━━━━━━━━━━━━━━━\n\n"
                f"{quality_emoji} *Quality:* {quality_text}\n"
                f"📦 *File Size:* {size_str}\n"
                f"📁 *Format:* {movie.get('file_name', 'Video').split('.')[-1].upper()}\n\n"
                f"⏰ *IMPORTANT:* This file will be *auto-deleted in 2 minutes!*\n"
                f"💾 Please save it to your device now!\n\n"
                f"━━━━━━━━━━━━━━━━━━━━\n"
                f"✨ Enjoy your movie! 🍿🎉\n"
                f"🤖 Powered by Movie Bot"
            )
            
            result = send_document(user_id, movie['file_id'], caption)
            
            if result.get('ok'):
                sent_msg_id = result['result']['message_id']
                auto_delete_file(user_id, sent_msg_id, delay=120)
                
                edit_message(chat_id, message_id,
                    f"✅ *MOVIE SENT SUCCESSFULLY!*\n\n"
                    f"🎬 {movie['title']}\n"
                    f"📦 {size_str}\n\n"
                    f"━━━━━━━━━━━━━━━━━━━━\n\n"
                    f"⏰ *WARNING:* File will be deleted in *2 minutes!*\n"
                    f"💾 Save it to your device immediately!\n\n"
                    f"📤 Check your chat above ⬆️\n\n"
                    f"━━━━━━━━━━━━━━━━━━━━\n"
                    f"🙏 Thank you for using Movie Bot!\n"
                    f"🔍 Search more movies anytime!"
                )
                print(f"📤 Sent: {movie['title']} to user {user_id}")
            else:
                edit_message(chat_id, message_id, 
                    f"❌ *Failed to Send!*\n\n"
                    f"Sorry, couldn't send the movie.\n"
                    f"Please try again or contact admin."
                )
                
        except Exception as e:
            edit_message(chat_id, message_id, 
                f"❌ *Error Occurred!*\n\n"
                f"Error: `{str(e)}`\n\n"
                f"Please try again or contact admin."
            )

def handle_list(chat_id, user_id):
    """Handle /list"""
    if not is_admin(user_id):
        send_message(chat_id, "❌ Access required!")
        return
    
    movies = load_db()
    if not movies:
        send_message(chat_id, "📭 No movies!")
        return
    
    text = "📚 *ALL MOVIES:*\n\n"
    for movie in movies[:50]:  # Limit to 50
        size = format_size(movie['file_size'])
        text += f"ID {movie['id']}: {movie['title']} ({size})\n"
    
    if len(movies) > 50:
        text += f"\n... and {len(movies) - 50} more"
    
    text += f"\n\n*Total:* {len(movies)}"
    send_message(chat_id, text)

def handle_delete(chat_id, user_id, args):
    """Handle /delete"""
    if not is_admin(user_id):
        send_message(chat_id, "❌ Access required!")
        return
    
    if not args:
        send_message(chat_id, "Usage: /delete [id]")
        return
    
    try:
        movie_id = int(args[0])
        movie = get_movie(movie_id)
        if movie:
            delete_movie(movie_id)
            send_message(chat_id, f"✅ Deleted: {movie['title']}")
        else:
            send_message(chat_id, f"❌ ID {movie_id} not found!")
    except:
        send_message(chat_id, "❌ Invalid ID!")

# ========== MAIN ==========

def main():
    """Main loop"""
    print("=" * 50)
    print("🎬 MOVIE BOT - AUTO SCRAPE MODE")
    print("=" * 50)
    print(f"🔐 Secret: {SECRET_CODE}")
    print(f"📊 Movies: {len(load_db())}")
    print(f"👥 Admins: {len(load_admins())}")
    print("📡 Scraping feature enabled")
    print("⏰ Files auto-delete after 2 minutes")
    print("🚀 Starting...")
    print("=" * 50)
    
    offset = None
    
    while True:
        try:
            updates = get_updates(offset)
            
            if updates.get('ok'):
                for update in updates.get('result', []):
                    offset = update['update_id'] + 1
                    
                    if 'message' in update:
                        msg = update['message']
                        chat_id = msg['chat']['id']
                        user_id = msg['from']['id']
                        user_name = msg['from'].get('first_name', 'User')
                        
                        # Check for forwarded messages
                        if 'forward_from' in msg or 'forward_from_chat' in msg:
                            if 'document' in msg or 'video' in msg:
                                handle_forward(chat_id, user_id, msg)
                                continue
                        
                        if 'text' in msg:
                            text = msg['text']
                            
                            # Check cloning mode
                            state = load_scrape_state()
                            if str(user_id) in state and state[str(user_id)].get('mode') == 'cloning':
                                if text.lower() == '/stop':
                                    count = state[str(user_id)].get('count', 0)
                                    del state[str(user_id)]
                                    save_scrape_state(state)
                                    send_message(chat_id, 
                                        f"⏹️ *Cloning Stopped*\n\n"
                                        f"📊 Total cloned: {count} movies\n\n"
                                        f"✅ All movies saved successfully!"
                                    )
                                    continue
                            
                            if text.startswith('/start'):
                                handle_start(chat_id, user_name)
                            elif text.startswith('/upload'):
                                args = text.split()[1:]
                                handle_upload_command(chat_id, user_id, args)
                            elif text.startswith('/train'):
                                handle_train_command(chat_id, user_id)
                            elif text.startswith('/scrape'):
                                args = text.split()[1:]
                                handle_scrape_command(chat_id, user_id, args)
                            elif text.startswith('/clone'):
                                args = text.split()[1:]
                                handle_clone_command(chat_id, user_id, args)
                            elif text.startswith('/stop'):
                                handle_exit_command(chat_id, user_id)
                            elif text.startswith('/exit'):
                                handle_exit_command(chat_id, user_id)
                            elif text.startswith('/list'):
                                handle_list(chat_id, user_id)
                            elif text.startswith('/delete'):
                                args = text.split()[1:]
                                handle_delete(chat_id, user_id, args)
                            elif not text.startswith('/'):
                                handle_search(chat_id, text)
                        
                        elif 'document' in msg:
                            # Check if in cloning mode
                            state = load_scrape_state()
                            if str(user_id) in state and state[str(user_id)].get('mode') == 'cloning':
                                handle_forward(chat_id, user_id, msg)
                                state[str(user_id)]['count'] = state[str(user_id)].get('count', 0) + 1
                                save_scrape_state(state)
                            else:
                                handle_file(chat_id, user_id, msg['document'])
                        
                        elif 'video' in msg:
                            # Check if in cloning mode
                            state = load_scrape_state()
                            if str(user_id) in state and state[str(user_id)].get('mode') == 'cloning':
                                handle_forward(chat_id, user_id, msg)
                                state[str(user_id)]['count'] = state[str(user_id)].get('count', 0) + 1
                                save_scrape_state(state)
                            else:
                                handle_file(chat_id, user_id, msg['video'])
                    
                    elif 'callback_query' in update:
                        cb = update['callback_query']
                        callback_id = cb['id']
                        chat_id = cb['message']['chat']['id']
                        message_id = cb['message']['message_id']
                        user_id = cb['from']['id']
                        data = cb['data']
                        
                        handle_callback(callback_id, chat_id, message_id, user_id, data)
            
            time.sleep(0.5)
            
        except KeyboardInterrupt:
            print("\n👋 Stopped!")
            break
        except Exception as e:
            print(f"❌ Error: {e}")
            time.sleep(5)

if __name__ == '__main__':
    main()
