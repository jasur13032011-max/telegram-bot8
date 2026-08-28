Aiogram (yoki oxshash Telegram bot kutubxonalarida) filtr holatlarini filters.create(), shuningdek & (AND), | (OR) va ~ (NOT) mantiqiy operatorlaridan foydalanib yozish misoli:

Python
from aiogram import filters
from aiogram.types import Message

# Custom is_admin filtrini yarating
is_admin = filters.create(
    lambda message: message.from_user and message.from_user.id in [12345678, 87654321]
)

# Boshqa yordamchi filtrlar
is_private = filters.create(lambda message: message.chat.type == "private")
is_banned = filters.create(lambda message: message.from_user.is_bot)

# &, | va ~ operatorlarini birgalikda ishlatish:
# (Admin BO'LGAN OR Shaxsiy chatda bo'lgan) AND (Banned BO'LMAGAN)
combined_filter = (is_admin | is_private) & ~is_banned

# Bot xabar ishlovchisida (handler) qo'llanilishi:
@dp.message(combined_filter)
async def admin_or_private_handler(message: Message):
    await message.answer("Siz ushbu buyruqni bajarish huquqiga egasiz!")
Operatorlar tahlili:

filters.create(): Foydalanuvchi IDsi ro'yxatda bor-yo'qligini tekshiruvchi is_admin filtrini yaratadi.

|: Kamida bitta shart bajarilishini tekshiradi (is_admin yoki is_private).

&: Ikkala tomondagi shart ham to'g'ri bo'lishini talab qiladi.

~: Filtr natijasini teskarisiga o'zgartiradi (is_banned bo'lmagan holat).
Pyrogram kutubxonasi talablariga mos ravishda tayyorlangan to'liq kod:

Python
from pyrogram import Client, filters
from pyrogram.enums import ChatMemberStatus
from pyrogram.types import Message

app = Client("my_bot")

# 1. get_chat_member orqali is_admin filtrini filters.create yordamida yaratish
async def admin_check_func(filter, client, message: Message):
    if not message.chat or not message.from_user:
        return False
    
    # Foydalanuvchining guruhdagi maqomini tekshirish
    member = await client.get_chat_member(message.chat.id, message.from_user.id)
    return member.status in [ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.OWNER]

is_admin = filters.create(admin_check_func)

# 2. Operatorlar kombinatsiyasi (&, |, ~):
# /ban buyrug'i AND guruh/superguruh chatlari AND (admin BO'LGAN) AND (botning o'zi BO'LMAGAN)
ban_filter = filters.command("ban") & (filters.group | filters.supergroup) & is_admin & ~filters.me

@app.on_message(ban_filter)
async def ban_handler(client: Client, message: Message):
    await message.reply_text("Foydalanuvchi bloklandi!")

# 3. Media xabarlar uchun handler (filters.photo | filters.video)
media_filter = filters.photo | filters.video

@app.on_message(media_filter)
async def media_handler(client: Client, message: Message):
    await message.reply_text("Rasm yoki video qabul qilindi!")

if __name__ == "__main__":
    app.run()
Koddagi asosiy elementlar va mantiqiy operatorlar:

filters.create() va get_chat_member: admin_check_func asinxron funksiyasi orqali foydalanuvchining admin yoki guruh egasi ekanligi tekshirilib, custom is_admin filtri hosil qilindi.

& (AND): /ban buyrug'i, chat turi hamda adminlik huquqini bir vaqtda tekshirish uchun ishlatildi.

| (OR): Guruh yoki superguruh chatlarini (filters.group | filters.supergroup), shuningdek photo yoki video xabarlarni (filters.photo | filters.video) birgalikda qamrab olish uchun qo'llanildi.

~ (NOT): ~filters.me orqali xabar botning o'zidan yuborilmaganligini tekshirish uchun ishlatildi.
