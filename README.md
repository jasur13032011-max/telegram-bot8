Pyrogram botida Restart siyosati (Docker compose orqali), Workdir saqlanishi (persistent volume) hamda SIGTERM signallarini ushlash / graceful shutdown (app.stop()) talablariga mos to'liq yechim:

1. Python kodi (bot.py) — Graceful Shutdown va SIGTERM Handler
Python
import asyncio
import os
import signal
import sys
from pyrogram import Client, filters
from pyrogram.types import Message

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

app = Client(
    name="my_graceful_bot",
    api_id=123456,                     # API ID
    api_hash="your_api_hash_here",     # API Hash
    bot_token="your_bot_token_here",   # Bot Token
    workdir=BASE_DIR,
    in_memory=True,
    plugins=dict(root="plugins")
)

# SIGTERM / SIGINT signallarini ushlab botni to'g'ri to'xtatish (Graceful Shutdown)
def signal_handler(sig, frame):
    print(f"\n[SIGNAL] {sig} signali qabul qilindi. Bot to'xtatilmoqda...")
    
    # Asinxron ravishda app.stop() chaqiruvini bajarish
    loop = asyncio.get_event_loop()
    if loop.is_running():
        loop.create_task(shutdown())
    else:
        loop.run_until_complete(shutdown())

async def shutdown():
    print("[SHUTDOWN] app.stop() chaqirilmoqda...")
    await app.stop()
    print("[SHUTDOWN] Bot muvaffaqiyatli to'xtatildi.")
    sys.exit(0)

# Signallarni ro'yxatdan o'tkazish
signal.signal(signal.SIGTERM, signal_handler)  # Docker stop signali
signal.signal(signal.SIGINT, signal_handler)   # Ctrl+C signali

if __name__ == "__main__":
    print("Bot ishga tushmoqda...")
    app.run()
2. Docker sozlamalari (docker-compose.yml) — Restart Policy & Persistent Workdir
workdir doimiy saqlanishi (persistent storage) uchun Docker Volume bog'lanadi va konteyner xatoga uchrasa yoki to'xtasa, avtomatik qayta yoqilishi uchun restart: unless-stopped (yoki on-failure) beriladi.

YAML
version: '3.8'

services:
  pyrogram_bot:
    build: .
    container_name: pyrogram_app
    # 1. Restart siyosati: unless-stopped (yoki on-failure)
    restart: unless-stopped
    
    # 2. Persistent workdir: Konteyner o'chib ketsa ham ma'lumotlar saqlanib qoladi
    volumes:
      - bot_data:/app
    
    environment:
      - PYTHONUNBUFFERED=1

volumes:
  bot_data:
3. Konteynerlashtirish uchun Dockerfile
Dockerfile
FROM python:3.10-slim

WORKDIR /app

# Talab etiladigan paketlarni o'rnatish
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Manba kodini ko'chirish
COPY . .

# SIGTERM signalini to'g'ridan-to'g'ri Python protsessiga uzatish uchun
CMD ["python", "bot.py"]
Talablar bajarilishi bo'yicha tushuntirish:
Restart siyosati (unless-stopped / on-failure):

docker-compose.yml faylida restart: unless-stopped belgilandi. Bu bot server o'chib yoqilganda yoki kutilmagan xatolik berib to'xtaganda konteynerni avtomatik qayta ishga tushiradi.

Persistent workdir:

Docker volumes: orqali konteyner ichidagi /app ishchi papkasi (workdir) bot_data nomli doimiy volume'ga bog'landi. Bu bot to'xtaganda ham ishchi fayllar va resurslar o'chib ketmasligini ta'minlaydi.

SIGTERM handler va app.stop():

signal.signal(signal.SIGTERM, signal_handler) orqali Docker konteynerni to'xtatish paytida yuboriladigan SIGTERM signali ushlab qolindi.

signal_handler ichida app.stop() funksiyasi chaqirilib, bot Telegram serverlari bilan ulanishni xavfsiz yopadi va resurslarni to'g'ri bo'shatadi (Graceful Shutdown).
