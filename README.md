Pyrogram kutubxonasida callback_query.answer(), qisqa callback_data formati va InlineQueryResultArticle yordamida inline so'rovlarga javob berish amaliyoti:

Python kodi (main.py)
Python
import os
from pyrogram import Client, filters
from pyrogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    InlineQuery,
    InlineQueryResultArticle,
    InputTextMessageContent,
)

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

app = Client(
    name="my_bot_session",
    api_id=123456,
    api_hash="your_api_hash_here",
    bot_token="your_bot_token_here",
    workdir=BASE_DIR,
    in_memory=True,
)

# 1. Callback button va qisqa callback_data (64 bayt chegarasi ichida)
# callback_data: "act:shw" -> bor-yo'g'i 7 bayt
def get_inline_keyboard():
    return InlineKeyboardMarkup(
        [
            [
                InlineKeyboardButton(
                    "Ma'lumotni ko'rsat", callback_data="act:shw"
                )
            ]
        ]
    )


@app.on_message(filters.command("info"))
async def info_command(client: Client, message):
    await message.reply_text(
        "Tugmani bosing:", reply_markup=get_inline_keyboard()
    )


# 2. Callback handler va callback_query.answer() ishlatilishi
# HAR BIR callback handlerda answer() chaqirilishi shart!
@app.on_callback_query(filters.regex(r"^act:shw$"))
async def handle_callback(client: Client, callback_query: CallbackQuery):
    # Telegram interfeysidagi yuklanish ("loading") belgisini yo'qotish va pop-up chiqarish
    await callback_query.answer("So'rov muvaffaqiyatli bajarildi!", show_alert=True)

    # Xabar matnini yangilash
    await callback_query.edit_message_text("Tugma bosildi va ma'lumot ko'rsatildi.")


# 3. Inline Query va InlineQueryResultArticle
@app.on_inline_query()
async def inline_query_handler(client: Client, inline_query: InlineQuery):
    query_text = inline_query.query.strip() or "Bo'sh so'rov"

    results = [
        InlineQueryResultArticle(
            title="Matnni yuborish",
            description=f"Kiritilgan matn: {query_text}",
            input_message_content=InputTextMessageContent(
                message_text=f"🔍 **Inline so'rov natijasi:**\n{query_text}"
            ),
            reply_markup=InlineKeyboardMarkup(
                [[InlineKeyboardButton("Batafsil", callback_data="act:shw")]]
            ),
        )
    ]

    # Inline query natijalarini yuborish
    await inline_query.answer(results=results, cache_time=1)


if __name__ == "__main__":
    app.run()
Asosiy talablar bajarilishi:
callback_query.answer():

Har bir callback kelganda await callback_query.answer() chaqirildi. Bu Telegram tugmasida aylanib turadigan yuklanish belgisini to'xtatadi va foydalanuvchiga tasdiq (alert yoki toast bildirishnoma) qaytaradi.

Qisqa callback_data format (64 bayt):

callback_data="act:shw" ko'rinishida prefiksli va juda qisqa format ishlatildi (7 bayt). Telegram callback_data uchun maksimum 64 bayt limit belgilagan, shuning uchun prefikslardan foydalanish eng to'g'ri amaliyotdir.

InlineQueryResultArticle:

@app.on_inline_query() o'shlanganda, InlineQueryResultArticle obyektidan foydalanib matnli natija yaratildi va input_message_content orqali yuboriladigan xabar mazmuni belgilandi.
