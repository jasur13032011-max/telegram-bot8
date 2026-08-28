Pyrogram loyihasida sessiya fayllarini chetlab o'tish, workdir parametridan foydalanish va in_memory=True rejimida Client yaratish kabi talablarga mos tayyorlangan namuna:

1. .gitignore fayli
Sessiya fayllari (.session), ularning vaqtinchalik jurnallari (.session-journal) va boshqa maxfiy ma'lumotlar Git repozitoriyasiga tushib qolmasligi uchun .gitignore fayliga quyidagicha yoziladi:

Code snippet
# Pyrogram session fayllari
*.session
*.session-journal

# Python kesh fayllari
__pycache__/
*.pyc

# Muhit o'zgaruvchilari (API kalitlar va tokenlar)
.env
2. Python kodi (main.py)
workdir parametringiz fayllar qayerda saqlanishini aniq ko'rsatadi, in_memory=True esa seansni xotirada (diskka .session faylini yozmasdan) yuritish imkonini beradi.

Python
import os
from pyrogram import Client, filters
from pyrogram.enums import ChatMemberStatus
from pyrogram.types import Message

# Loyihaning asosiy ishchi papkasini (workdir) aniq ko'rsatish
BASE_DIR = os.path.dirname(os.path.abspath(__file__))

# in_memory=True va workdir parametrlari bilan Client yaratish
app = Client(
    name="my_memory_session",
    api_id=123456,                     # O'zingizning API ID
    api_hash="your_api_hash_here",     # O'zingizning API Hash
    bot_token="your_bot_token_here",   # O'zingizning Bot Token
    workdir=BASE_DIR,                  # Ishchi papka aniq ko'rsatilgan
    in_memory=True                     # Sessiyani diskka emas, RAM'ga saqlash
)

# 1. get_chat_member va filters.create yordamida is_admin filtrini yaratish
async def admin_check_func(filter, client, message: Message):
    if not message.chat or not message.from_user:
        return False
    
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
Asosiy jihatlar:
.gitignore: *.session va *.session-journal qoidalari orqali xavfsizlik va keraksiz fayllarni chetlab o'tish ta'minlangan.

workdir=BASE_DIR: Pyrogram dasturi qaysi papkada ishlayotganini va ishchi resurslarni qayerdan qidirishini ko'rsatadi.

in_memory=True: Sessiyani xotirada saqlaydi. Bu parametr yoqilganda Pyrogram diskda .session faylini umuman yaratmaydi, bu esa test rejimida yoki vaqtinchalik sessiyalarda juda qulay hisoblanadi.
