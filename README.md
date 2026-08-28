Pyrogram-da Telegram API raw metodlarini to'g'ri qo'llash, ya'ni client.invoke(), resolve_peer() va pyrogram.errors xatoliklarini bitta plugin faylida ishlatish usuli:

plugins/raw_methods.py
Python
from pyrogram import Client, filters
from pyrogram.errors import PeerIdInvalid, UserNotParticipant, RPCError
from pyrogram.raw.functions.channels import GetParticipant
from pyrogram.types import Message

# Klass dekoratori qo'llaniladi (@Client.on_message)
@Client.on_message(filters.command("check_member") & filters.group)
async def check_member_handler(client: Client, message: Message):
    if len(message.command) < 2:
        await message.reply_text("Iltimos, foydalanuvchi ID yoki username'ini kiriting.\nMisol: `/check_member @username`")
        return

    user_input = message.command[1]

    try:
        # 1. resolve_peer() yordamida obyektni Peer ko'rinishiga o'tkazish
        peer_channel = await client.resolve_peer(message.chat.id)
        peer_user = await client.resolve_peer(user_input)

        # 2. client.invoke() ni to'g'ridan-to'g mezonlar bilan chaqirish (raw TL-method)
        participant_info = await client.invoke(
            GetParticipant(
                channel=peer_channel,
                participant=peer_user
            )
        )

        await message.reply_text(f"Foydalanuvchi guruh a'zosi! Roli: {type(participant_info.participant).__name__}")

    # 3. Xatolarni pyrogram.errors orqali ushlash
    except PeerIdInvalid:
        await message.reply_text("Kiritilgan foydalanuvchi yoki chat topilmadi!")
    except UserNotParticipant:
        await message.reply_text("Ushbu foydalanuvchi guruhda mavjud emas.")
    except RPCError as e:
        await message.reply_text(f"Telegram API xatoligi yuz berdi: {e.MESSAGE}")
Asosiy jihatlar:
client.invoke(): Wrapper funksiyalar (masalan, get_chat_member) ishlatilmasdan, to'g'ridan-to'g'ri Pyrogram raw funksiyasi (GetParticipant) invoke() ga uzatildi.

client.resolve_peer(): Chat va foydalanuvchi identifikatorlarini Raw TL obyekti tushunadigan InputPeer formatiga o'tkazish uchun ishlatildi.

pyrogram.errors: API chaqiruvidan chiqadigan xatoliklar (PeerIdInvalid, UserNotParticipant, RPCError) aniq ushlab olindi.
