samaradorligini baholash uchun amaliy test va xulosa.

Benchmark testi (plugins/benchmark.py)
Ushbu kod bir xil rasmni 10 marta yuborishda faylni qayta yuklash va tayyor file_id dan foydalanish o'rtasidagi vaqt farqini time.perf_counter() yordamida o'lchaydi:

Python
import os
import time
from pyrogram import Client, filters
from pyrogram.types import Message

# Sinov uchun rasm fayli yo'li
TEST_IMAGE = "photos/photo1.jpg"
CACHED_FILE_ID = None

@Client.on_message(filters.command("benchmark") & filters.me)
async def benchmark_handler(client: Client, message: Message):
    global CACHED_FILE_ID
    chat_id = message.chat.id
    iterations = 10

    await message.reply_text("⏱ Sinov boshlanmoqda (10 marta yuborish)...")

    # -------------------------------------------------------------
    # 1-HOLAT: file_id KESHSIZ (Har safar faylni qayta diskdan yuklash)
    # -------------------------------------------------------------
    start_no_cache = time.perf_counter()
    
    for _ in range(iterations):
        sent_msg = await client.send_photo(chat_id=chat_id, photo=TEST_IMAGE)
        # Birinchi yuborilgan rasmdan file_id ni saqlab olamiz
        if not CACHED_FILE_ID:
            CACHED_FILE_ID = sent_msg.photo.file_id

    end_no_cache = time.perf_counter()
    time_no_cache = end_no_cache - start_no_cache

    # -------------------------------------------------------------
    # 2-HOLAT: file_id KESH BILAN (Tayyor file_id orqali yuborish)
    # -------------------------------------------------------------
    start_with_cache = time.perf_counter()

    for _ in range(iterations):
        await client.send_photo(chat_id=chat_id, photo=CACHED_FILE_ID)

    end_with_cache = time.perf_counter()
    time_with_cache = end_with_cache - start_with_cache

    # -------------------------------------------------------------
    # NATIJALARNI SOLISHTIRISH
    # -------------------------------------------------------------
    speedup = time_no_cache / time_with_cache if time_with_cache > 0 else 0

    result_text = (
        f"📊 **Performance Benchmark Natijalari (10 ta xabar):**\n\n"
        f"❌ **Keshsiz (Diskdan yuklash):** `{time_no_cache:.4f}` soniya\n"
        f"✅ **Kesh bilan (file_id):** `{time_with_cache:.4f}` soniya\n\n"
        f"🚀 **Tezlanish:** `{speedup:.2f}x` baravar tezroq!"
    )

    await message.reply_text(result_text)
Yozma xulosa
Vaqt va Trafik Tejalishi:

Keshsiz holatda: Har bir chaqiruvda rasm diskdan o'qiladi, tarmoq orqali Telegram serveriga qayta yuklanadi. Bu jarayon lokal internet tezligi va serverga fayl uzatish vaqtiga to'g'ridan-to'g'ri bog'liq bo'lib, sezilarli kechikish beradi.

Keshli holatda: Telegram serverida allaqachon mavjud bo'lgan faylning file_id identifikatori yuboriladi. Fayl qayta yuklanmaydi, faqatgina qisqa matnli buyruq uzatiladi.

Resurs va Cheklovlar (Rate Limits):

Tayyor file_id ishlatilganda botning CPU va tarmoq (bandwidth) sarfi bir necha o'n baravarga kamayadi.
