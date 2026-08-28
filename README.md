Shu paytgacha ko'rib chiqilgan barcha 4 ta qismning talablarini to'liq birlashtirgan, ishga tushirishga tayyor Pyrogram boti kodi:

Python
import asyncio
import os
from pyrogram import Client, filters
from pyrogram.enums import ChatMemberStatus
from pyrogram.errors import FloodWait
from pyrogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    InlineQuery,
    InlineQueryResultArticle,
    InputMediaPhoto,
    InputTextMessageContent,
    Message,
)

# ------------------------------------------------------------------
# CONFIG & CLIENT (Qism 2 talablari: workdir, in_memory=True)
# ------------------------------------------------------------------
BASE_DIR = os.path.dirname(os.path.abspath(__file__))

app = Client(
    name="my_advanced_bot",
    api_id=123456,                     # O'zingizning API ID
    api_hash="your_api_hash_here",     # O'zingizning API Hash
    bot_token="your_bot_token_here",   # O'zingizning Bot Token
    workdir=BASE_DIR,                  # Aniq ishchi papka
    in_memory=True                     # Session RAMda saqlanadi
)

# Kesh uchun lug'at (Qism 3)
CACHED_FILE_IDS = []
LOCAL_IMAGES = ["photos/photo1.jpg", "photos/photo2.jpg", "photos/photo3.jpg"]


# ------------------------------------------------------------------
# QISM 1: Custom Filter (is_admin, filters.create, &, |, ~ operatorlari)
# ------------------------------------------------------------------
async def admin_check_func(filter, client, message: Message):
    if not message.chat or not message.from_user:
        return False
    member = await client.get_chat_member(message.chat.id, message.from_user.id)
    return member.status in [ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.OWNER]

# 1. filters.create orqali is_admin yaratish
is_admin = filters.create(admin_check_func)

# &, | va ~ operatorlarini ishlatish
ban_filter = filters.command("ban") & (filters.group | filters.supergroup) & is_admin & ~filters.me

@app.on_message(ban_filter)
async def ban_handler(client: Client, message: Message):
    await message.reply_text("Foydalanuvchi muvaffaqiyatli ban qilindi!")


# ------------------------------------------------------------------
# QISM 3: InlineKeyboardMarkup (>=3 tugma), send_media_group & file_id Caching
# ------------------------------------------------------------------
def get_main_keyboard():
    # Kamida 3 ta inline tugma
    return InlineKeyboardMarkup([
        [
            InlineKeyboardButton("🖼 Albom yuborish", callback_data="act:album"),
            InlineKeyboardButton("ℹ️ Ma'lumot", callback_data="act:info")
        ],
        [
            InlineKeyboardButton("🌐 Veb-sayt", url="https://telegram.org")
        ]
    ])

@app.on_message(filters.command("start"))
async def start_handler(client: Client, message: Message):
    await message.reply_text("Xush kelibsiz! Kerakli bo'limni tanlang:", reply_markup=get_main_keyboard())

@app.on_message(filters.command("album"))
async def send_album_handler(client: Client, message: Message):
    global CACHED_FILE_IDS

    # file_id keshlash mantiqi (ikkinchi marta qayta yuklanmaydi)
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


# ------------------------------------------------------------------
# QISM 4 (a): Callback Query handler & Inline Query (InlineQueryResultArticle)
# ------------------------------------------------------------------
@app.on_callback_query(filters.regex(r"^act:"))
async def callback_handler(client: Client, callback_query: CallbackQuery):
    # HAR BIR callback handlerda callback_query.answer() bo'lishi shart
    await callback_query.answer("So'rovingiz qabul qilindi!", show_alert=False)

    if callback_query.data == "act:info":
        await callback_query.edit_message_text("Bu bot Pyrogram misolida barcha topshiriqlarni bajarish uchun yaratildi.")
    elif callback_query.data == "act:album":
        await send_album_handler(client, callback_query.message)

@app.on_inline_query()
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


# ------------------------------------------------------------------
# QISM 4 (b): async for (streaming) & FloodWait handling
# ------------------------------------------------------------------
@app.on_message(filters.command("history") & is_admin)
async def history_handler(client: Client, message: Message):
    await message.reply_text("Xabarlar oqim ko'rinishida qayta ishlanmoqda...")
    count = 0

    try:
        # async for orqali xabarlarni kelishi bilan darhol qayta ishlash (ro'yxatga yig'masdan)
        async for msg in client.get_chat_history(message.chat.id, limit=100):
            try:
                if msg.text:
                    # Darhol ishlov berish
                    _ = msg.text.upper()
                    count += 1
                await asyncio.sleep(0.05)

            except FloodWait as e:
                # FloodWait bo'lsa ko'rsatilgan soniya davomida kutadi
                await asyncio.sleep(e.value)

    except FloodWait as e:
        await asyncio.sleep(e.value)

    await message.reply_text(f"Jarayon yakunlandi. Jami {count} ta xabar qayta ishlandi.")


# ------------------------------------------------------------------
# BOTNI ISHGA TUSHRISH
# ------------------------------------------------------------------
if __name__ == "__main__":
    app.run()
.gitignore fayli (Proyekt ildiz papkasiga tashlanadi):
Code snippet
*.session
*.session-journal
__pycache__/
.env
Barcha talablarning bajarilishi:
is_admin & filters.create(): admin_check_func orqali get_chat_member yordamida yaratildi. &, | va ~ operatorlari birlashtirildi.

Client va .gitignore: workdir=BASE_DIR va in_memory=True ko'rsatildi, .session fayllari .gitignore ga qo'shildi.

Media va Kesh: InlineKeyboardMarkupda 3 ta tugma bor, send_media_group 3 ta elementdan iborat va CACHED_FILE_IDS orqali qayta yuklanmasdan ishlaydi.

Callback & Inline Query: callback_query.answer() har safar chaqiriladi, callback_data qisqa formatda (act:info), InlineQueryResultArticle ishlatildi.

async for va FloodWait: Chat tarixi xotiraga ro'yxat qilib yig'ilmasdan (async for orqali) darhol qayta ishlanadi hamda try/except FloodWait va asyncio.sleep(e.value) kodi kiritildi.
