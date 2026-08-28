Pyrogram loyihasini modulli arxitekturaga o'tkazish uchun kodingizni quyidagi strukturaga binoan ajratishingiz mumkin. Bunda bot.py faylida hech qanday handler kodi bo'lmaydi, handlerlar alohida 3 ta plugin faylida joylashadi hamda @Client.on_message dekortatoridan foydalaniladi.

Loyiha strukturasi
Plaintext
my_project/
├── bot.py
├── .gitignore
├── plugins/
│   ├── admin.py
│   ├── media.py
│   └── inline.py
└── photos/
    ├── photo1.jpg
    ├── photo2.jpg
    └── photo3.jpg
1. bot.py (Handler saqlamaydi, faqat Client va plugins yuklaydi)
Python
import os
from pyrogram import Client

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

# Client namunasi plugins papkasiga yo'naltirilgan
app = Client(
    name="my_advanced_bot",
    api_id=123456,                     # O'zingizning API ID
    api_hash="your_api_hash_here",     # O'zingizning API Hash
    bot_token="your_bot_token_here",   # O'zingizning Bot Token
    workdir=BASE_DIR,
    in_memory=True,
    plugins=dict(root="plugins")       # Handlerlar papkasi
)

if __name__ == "__main__":
    app.run()
2. plugins/admin.py (1-plugin: Admin filtr va Xabarlar oqimi)
Python
import asyncio
from pyrogram import Client, filters
from pyrogram.enums import ChatMemberStatus
from pyrogram.errors import FloodWait
from pyrogram.types import Message

# Adminlikni get_chat_member orqali tekshiruvchi funksiya
async def admin_check_func(filter, client, message: Message):
    if not message.chat or not message.from_user:
        return False
    member = await client.get_chat_member(message.chat.id, message.from_user.id)
    return member.status in [ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.OWNER]

is_admin = filters.create(admin_check_func)

# &, | va ~ operatorlari
ban_filter = filters.command("ban") & (filters.group | filters.supergroup) & is_admin & ~filters.me

# Klass dekoratori: @Client.on_message
@Client.on_message(ban_filter)
async def ban_handler(client: Client, message: Message):
    await message.reply_text("Foydalanuvchi muvaffaqiyatli ban qilindi!")

@Client.on_message(filters.command("history") & is_admin)
async def history_handler(client: Client, message: Message):
    await message.reply_text("Xabarlar oqim ko'rinishida qayta ishlanmoqda...")
    count = 0

    try:
        # async for — ro'yxatga to'liq yig'masdan, kelishi bilan qayta ishlash
        async for msg in client.get_chat_history(message.chat.id, limit=100):
            try:
                if msg.text:
                    _ = msg.text.upper()
                    count += 1
                await asyncio.sleep(0.05)
            except FloodWait as e:
                await asyncio.sleep(e.value)

    except FloodWait as e:
        await asyncio.sleep(e.value)

    await message.reply_text(f"Jarayon yakunlandi. Jami {count} ta xabar qayta ishlandi.")
3. plugins/media.py (2-plugin: Inline tugmalar va Albom keshlash)
Python
from pyrogram import Client, filters
from pyrogram.types import (
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    InputMediaPhoto,
    Message,
)

CACHED_FILE_IDS = []
LOCAL_IMAGES = ["photos/photo1.jpg", "photos/photo2.jpg", "photos/photo3.jpg"]

def get_main_keyboard():
    return InlineKeyboardMarkup([
        [
            InlineKeyboardButton("🖼 Albom yuborish", callback_data="act:album"),
            InlineKeyboardButton("ℹ️ Ma'lumot", callback_data="act:info")
        ],
        [
            InlineKeyboardButton("🌐 Veb-sayt", url="https://telegram.org")
        ]
    ])

@Client.on_message(filters.command("start"))
async def start_handler(client: Client, message: Message):
    await message.reply_text("Xush kelibsiz! Kerakli bo'limni tanlang:", reply_markup=get_main_keyboard())

@Client.on_message(filters.command("album"))
async def send_album_handler(client: Client, message: Message):
    global CACHED_FILE_IDS

    if CACHED_FILE_IDS:
        media_group = [InputMediaPhoto(media=file_id) for file_id in CACHED_FILE_IDS]
    else:
        media_group = [
            InputMediaPhoto(media=LOCAL_IMAGES[0], caption="1-rasm"),
            InputMediaPhoto(media=LOCAL_IMAGES[1], caption="2-rasm"),
            InputMediaPhoto(media=LOCAL_IMAGES[2], caption="3-rasm")
        ]

    sent_messages = await client.send_media_group(chat_id=message.chat.id, media=media_group)

    if not CACHED_FILE_IDS:
        for msg in sent_messages:
            if msg.photo:
                CACHED_FILE_IDS.append(msg.photo.file_id)
4. plugins/inline.py (3-plugin: CallbackQuery va InlineQuery)
Python
from pyrogram import Client, filters
from pyrogram.types import (
    CallbackQuery,
    InlineQuery,
    InlineQueryResultArticle,
    InputTextMessageContent,
)

@Client.on_callback_query(filters.regex(r"^act:"))
async def callback_handler(client: Client, callback_query: CallbackQuery):
    # callback_query.answer() majburiy chaqiriladi
    await callback_query.answer("So'rovingiz qabul qilindi!", show_alert=False)

    if callback_query.data == "act:info":
        await callback_query.edit_message_text("Bu bot modul arxitekturasida tuzilgan Pyrogram boti.")

@Client.on_inline_query()
async def inline_query_handler(client: Client, inline_query: InlineQuery):
    query_text = inline_query.query.strip() or "Bo'sh so'rov"

    results = [
        InlineQueryResultArticle(
            title="Izlash natijasi",
            description=f"Matn: {query_text}",
            input_message_content=InputTextMessageContent(
                message_text=f"🔍 **Inline natija:** {query_text}"
            )
        )
    ]
    await inline_query.answer(results=results, cache_time=1)
Talablar bajarilishi bo'yicha tushuntirish:
3 ta alohida plugin fayli: Handlerlar mantiqiy ravishda plugins/admin.py, plugins/media.py va plugins/inline.py fayllariga ajratildi.

bot.py ichida handler yo'qligi: bot.py faqat Client obyekti parametrlarini belgilash hamda plugins=dict(root="plugins") orqali ularni ulash vazifasini bajaradi.

@Client.on_message ishlatilishi: Barcha handler fayllarida app.on_message emas, balki Pyrogram'ning klass dekoratori bo'lgan @Client.on_message qo'llanildi. Bu pluginlar yuklanayotganda app obyektiga bog'liq bo'lib qolmaslikni ta'minlaydi.
