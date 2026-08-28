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
