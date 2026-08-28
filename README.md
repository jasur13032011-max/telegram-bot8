Pyrogram kutubxonasida InlineKeyboardMarkup (3 ta tugma bilan), send_media_group (kamida 3 ta element) va file_id keshlash mantiqi (ikkinchi marta fayl yuklanmasdan, saqlab qolingan file_id orqali tezkor yuborilishi) ko'rsatilgan to'liq va amaliy misol:

Python kodi (main.py)
Python
import os
from pyrogram import Client, filters
from pyrogram.types import (
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    InputMediaPhoto,
    Message,
)

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

app = Client(
    name="my_bot_session",
    api_id=123456,  # O'zingizning API ID
    api_hash="your_api_hash_here",  # O'zingizning API Hash
    bot_token="your_bot_token_here",  # O'zingizning Bot Token
    workdir=BASE_DIR,
    in_memory=True,
)

# File ID larni xotirada keshlash uchun lug'at (Cache)
CACHED_FILE_IDS = []

# Yuboriladigan mahalliy (lokal) rasmlar yo'li
LOCAL_IMAGES = ["photos/photo1.jpg", "photos/photo2.jpg", "photos/photo3.jpg"]


# 1. InlineKeyboardMarkup: Kamida 3 ta tugma bilan
def get_main_keyboard():
    return InlineKeyboardMarkup(
        [
            [
                InlineKeyboardButton(
                    "🖼 Albom yuborish", callback_data="send_album"
                ),
                InlineKeyboardButton("ℹ️ Ma'lumot", callback_data="info"),
            ],
            [
                InlineKeyboardButton(
                    "🌐 Veb-saytga o'tish", url="https://telegram.org"
                )
            ],
        ]
    )


@app.on_message(filters.command("start"))
async def start_handler(client: Client, message: Message):
    await message.reply_text(
        text="Xush kelibsiz! Quyidagi tugmalardan birini tanlang:",
        reply_markup=get_main_keyboard(),
    )


# 2. send_media_group + file_id keshlash mantiqi
@app.on_message(filters.command("album"))
async def send_album_handler(client: Client, message: Message):
    global CACHED_FILE_IDS

    # KESH TEKSHIRUVI:
    # Agar keshda file_id lar bor bo'lsa (2-chaqiruv va undan son'g), lokal fayllar qayta YUKLANMAYDI!
    if CACHED_FILE_IDS:
        print("[KESH] Rasm file_id lari keshdan olindi va qayta yuklanmasdan yuborilmoqda...")
        media_group = [InputMediaPhoto(media=file_id) for file_id in CACHED_FILE_IDS]
    
    # Birinchi marta chaqirilganda: fayllar diskdan o'qiladi va Telegram serveriga yuklanadi
    else:
        print("[SERVER] Rasmlar birinchi marta diskdan Telegram'ga yuklanmoqda...")
        media_group = [
            InputMediaPhoto(media=LOCAL_IMAGES[0], caption="1-rasm (Albom)"),
            InputMediaPhoto(media=LOCAL_IMAGES[1], caption="2-rasm (Albom)"),
            InputMediaPhoto(media=LOCAL_IMAGES[2], caption="3-rasm (Albom)"),
        ]

    # send_media_group orqali kamida 3 ta elementdan iborat media guruh yuborish
    sent_messages = await client.send_media_group(
        chat_id=message.chat.id,
        media=media_group
    )

    # Birinchi yuklashdan so'ng Telegram qaytargan file_id larni keshga saqlab qo'yamiz
    if not CACHED_FILE_IDS:
        for msg in sent_messages:
            if msg.photo:
                CACHED_FILE_IDS.append(msg.photo.file_id)
        print(f"[KESH] 3 ta rasmning file_id lari xotiraga saqlandi: {CACHED_FILE_IDS}")


if __name__ == "__main__":
    app.run()
Koddagi talablar bajarilishi bo'yicha tushuntirish:
InlineKeyboardMarkup (3 ta tugma):

get_main_keyboard() funksiyasida InlineKeyboardMarkup orqali 3 ta tugma yaratildi: "🖼 Albom yuborish", "ℹ️ Ma'lumot" va "🌐 Veb-saytga o'tish".

send_media_group (3 ta element):

/album buyrug'i berilganda client.send_media_group() funksiyasi orqali kamida 3 ta rasmdan iborat albom yuboriladi.

file_id keshlash mantiqi:

Birinchi chaqiruvda: CACHED_FILE_IDS bo'sh bo'ladi. Rasmlar diskdagi fayllardan (photos/photo1.jpg, ...) Telegram serveriga yuklanadi va Telegram tomonidan berilgan har bir rasmning file_id si CACHED_FILE_IDS ro'yxatiga saqlanadi.

Ikkinchi (va keyingi) chaqiruvlarda: Dastur if CACHED_FILE_IDS: shartiga kiradi va fayllarni internet orqali qayta diskdan yuklab o'tirmay, Telegram serveridagi tayyor file_id lardan foydalanadi. Bu botning ishlash tezligini keskin oshiradi va trafikni tejaydi.
