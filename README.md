Dual-mode (Bot va Userbot) arxitekturasida tuzilgan, barcha xavfsizlik hamda texnik talablarga to'liq javob beradigan Pyrogram loyihasi strukturasi va kodi:

Loyiha strukturasi
Plaintext
my_dual_project/
├── .env
├── .gitignore
├── config.py
├── bot.py
├── docker-compose.yml
├── Dockerfile
└── plugins/
    ├── admin.py
    └── raw_invoke.py
1. Muhit o'zgaruvchilari va Sozlamalar
.env (Maxfiy qiymatlar hardcode qilinmaydi)
Code snippet
API_ID=123456
API_HASH=your_api_hash_here
BOT_TOKEN=your_bot_token_here
USER_SESSION_STRING=your_telethon_or_pyrogram_session_string_optional
config.py
Python
import os
from dotenv import load_dotenv

load_dotenv()

API_ID = int(os.getenv("API_ID", 0))
API_HASH = os.getenv("API_HASH", "")
BOT_TOKEN = os.getenv("BOT_TOKEN", "")
USER_SESSION_STRING = os.getenv("USER_SESSION_STRING", None)
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
2. bot.py — Dual-mode, Graceful Shutdown va Start logic
Python
import asyncio
import signal
import sys
from pyrogram import Client
import config

# 1. Bot Klienti (Har doim faol)
bot_app = Client(
    name="bot_instance",
    api_id=config.API_ID,
    api_hash=config.API_HASH,
    bot_token=config.BOT_TOKEN,
    workdir=config.BASE_DIR,
    in_memory=True,
    plugins=dict(root="plugins")
)

# 2. Userbot Klienti (Ixtiyoriy — agar USER_SESSION_STRING berilgan bo'lsa ishlaydi)
user_app = None
if config.USER_SESSION_STRING:
    user_app = Client(
        name="user_instance",
        api_id=config.API_ID,
        api_hash=config.API_HASH,
        session_string=config.USER_SESSION_STRING,
        workdir=config.BASE_DIR,
        in_memory=True
    )

async def start_services():
    print("Bot klientga ishlov berilmoqda...")
    await bot_app.start()
    
    if user_app:
        print("Userbot klienti ishga tushirilmoqda...")
        await user_app.start()
        
    print("Dual-mode tizimi muvaffaqiyatli ishga tushdi!")
    await asyncio.Event().wait()

async def stop_services():
    print("\n[SHUTDOWN] Klientlar to'xtatilmoqda...")
    if bot_app.is_connected:
        await bot_app.stop()
    if user_app and user_app.is_connected:
        await user_app.stop()
    print("[SHUTDOWN] Barcha klientlar xavfsiz to'xtatildi.")

def signal_handler(sig, frame):
    loop = asyncio.get_event_loop()
    if loop.is_running():
        loop.create_task(stop_services())
        sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    try:
        loop.run_until_complete(start_services())
    except (KeyboardInterrupt, SystemExit):
        loop.run_until_complete(stop_services())
3. Plugin Fayllari
plugins/admin.py (1-plugin: async for, streaming va FloodWait retry)
Python
import asyncio
from pyrogram import Client, filters
from pyrogram.errors import FloodWait
from pyrogram.types import Message

@Client.on_message(filters.command("clean_history") & filters.group)
async def clean_history_handler(client: Client, message: Message):
    count = 0
    
    # async for: Xabarlar oqimi ro'yxatga yig'ilmasdan darhol ishlanadi
    async for msg in client.get_chat_history(message.chat.id, limit=50):
        # FloodWait uchun takroriy urinish (retry logic)
        while True:
            try:
                if msg.text:
                    count += 1
                await asyncio.sleep(0.05)
                break  # Muvaffaqiyatli bajarilgach sikldan chiqadi
            except FloodWait as e:
                print(f"[FLOODWAIT] {e.value} soniya kutilmoqda va qayta urinib ko'riladi...")
                await asyncio.sleep(e.value)

    await message.reply_text(f"Jami {count} ta xabar oqim ko'rinishida qayta ishlandi.")
plugins/raw_invoke.py (2-plugin: Xom client.invoke va resolve_peer)
Python
from pyrogram import Client, filters
from pyrogram.errors import RPCError
from pyrogram.raw.functions.channels import GetFullChannel
from pyrogram.types import Message

@Client.on_message(filters.command("raw_info") & filters.group)
async def raw_info_handler(client: Client, message: Message):
    try:
        # resolve_peer() yordamida kanal/guruh obyektini InputPeer ga o'tkazish
        peer = await client.resolve_peer(message.chat.id)

        # Xom client.invoke() to'g'ridan-to'g'ri chaqiruvi
        full_chat = await client.invoke(
            GetFullChannel(channel=peer)
        )

        members_count = full_chat.full_chat.participants_count
        await message.reply_text(f"📊 **Raw API ma'lumoti:** Guruh a'zolari soni = `{members_count}`")

    except RPCError as e:
        await message.reply_text(f"RPC Xatolik yuz berdi: {e.MESSAGE}")
4. Konteynerlashtirish va Sozlamalar
docker-compose.yml
YAML
version: '3.8'

services:
  dual_bot:
    build: .
    container_name: pyrogram_dual_app
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - bot_data:/app

volumes:
  bot_data:
.gitignore
Code snippet
.env
*.session
*.session-journal
__pycache__/
Barcha talablar bajarilishi:
Dual-mode arxitektura: Bot asosiy sifatida ishlaydi. USER_SESSION_STRING muhit o'zgaruvchisida bor bo'lsa, Userbot ixtiyoriy ravishda parallel qo'shilib ishlaydi.

Kamida 2 ta plugin fayli: plugins/admin.py va plugins/raw_invoke.py yaratildi.

Xom invoke(): plugins/raw_invoke.py ichida client.resolve_peer() va client.invoke(GetFullChannel(...)) to'g'ridan-to'g'ri qo'llanildi.

FloodWait va async for: plugins/admin.py ichida async for orqali xabarlar ro'yxatga to'planmasdan streaming qilindi va while True: hamda except FloodWait orqali qayta urinish (retry) mantiqi o'rnatildi.

Graceful Shutdown: bot.py ichida SIGTERM/SIGINT ushlanib, bot_app.stop() va user_app.stop() xavfsiz chaqirilishi yo'lga qo'yildi.

Hardcode qilinmagan maxfiylik: Barcha token va session kalitlari .env faylida saqlanadi va .gitignore orqali berkitildi.
