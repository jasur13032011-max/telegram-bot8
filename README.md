Pyrogram’da xabarlarni generativ ko'rinishda (asinxron generator orqali) bittalab olish, xotirani tejash uchun ularni ro'yxatga to'liq yig'masdan darhol qayta ishlash hamda FloodWait xatoligini to'g'ri ushlab kutish kabi talablarga mos tayyorlangan namuna:

Python kodi (main.py)
Python
import asyncio
import os
from pyrogram import Client, filters
from pyrogram.errors import FloodWait
from pyrogram.types import Message

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

app = Client(
    name="my_bot_session",
    api_id=123456,
    api_hash="your_api_hash_here",
    bot_token="your_bot_token_here",
    workdir=BASE_DIR,
    in_memory=True,
)


@app.on_message(filters.command("process_history"))
async def process_history_handler(client: Client, message: Message):
    chat_id = message.chat.id
    processed_count = 0

    await message.reply_text("Xabarlar ketma-ket qayta ishlanmoqda...")

    try:
        # 1. async for ishlatilishi: xabarlar oqim ko'rinishida keladi.
        # Natija ro'yxatga (list) yig'ilmaydi, har bir xabar kelishi bilan darhol qayta ishlanadi.
        async for msg in client.get_chat_history(chat_id=chat_id, limit=50):
            try:
                # Har bir xabar uchun darhol ishlov berish (masalan, matnini konsolga chiqarish)
                if msg.text:
                    print(f"[Xabar ID: {msg.id}] -> {msg.text[:30]}")

                processed_count += 1

                # 2. Telegram FloodWait chegarasiga tushmaslik uchun kichik tanaffus
                await asyncio.sleep(0.1)

            # 3. FloodWait ushlash va kutish mantiqi
            except FloodWait as e:
                print(f"FloodWait ga tushdik! {e.value} soniya kutilmoqda...")
                # Telegram talab qilgan soniya miqdoricha kutiladi
                await asyncio.sleep(e.value)

    except FloodWait as e:
        # Umumiy oqim davomida berilgan FloodWait uchun
        print(f"Tashqi FloodWait: {e.value} soniya kutilmoqda...")
        await asyncio.sleep(e.value)

    await message.reply_text(
        f"Jarayon yakunlandi! Jami {processed_count} ta xabar qayta ishlandi."
    )


if __name__ == "__main__":
    app.run()
Talablar bo'yicha tushuntirish:
async for ishlatilishi:

Pyrogram'ning get_chat_history() funksiyasi asinxron generator qaytaradi. Shu sababli u bilan async for ishlatildi.

Natija ro'yxatga yig'ilmasligi (streaming):

Xabarlar [msg async for msg in ...] ko'rinishida hotiraga to'planmaydi (list yaratilmaydi). Har bir kelgan Message obyekti sikl ichida darhol ishlanadi va keyingi elementga o'tiladi. Bu operativ xotira (RAM) sarfini keskin kamaytiradi.

FloodWait uchun try/except va sleep:

Pyrogram Telegram API cheklovlariga duch kelganida errors.FloodWait xatoligini tashlaydi. e.value atributida Telegram qancha soniya kutish kerakligini qaytaradi.

except FloodWait as e: bloki orqali xatolik ushlanadi va await asyncio.sleep(e.value) yordamida bot belgilangan vaqt davomida to'xtatilib, so'ngra ishni xavfsiz davom ettiradi.
