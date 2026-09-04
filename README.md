"""Xabar yuborish, o'qish, tahrirlash va entity resolution namunalari.
pip install telethon python-dotenv
"""
import asyncio
import os

from dotenv import load_dotenv
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import FloodWaitError

load_dotenv()
API_ID = int(os.environ["API_ID"])
API_HASH = os.environ["API_HASH"]
SESSION_STRING = os.environ["TELETHON_SESSION_STRING"]

client = TelegramClient(StringSession(SESSION_STRING), API_ID, API_HASH)


async def send_and_receive_basics() -> None:
    async with client:
        # 1) O'zingizga eslatma yuborish (Saved Messages)
        note = await client.send_message("me", "Telethon darsligi: 4-dars yakunlandi.")
        print(f"Yuborildi, id={note.id}")

        # 2) Xabarni tahrirlash
        await client.edit_message("me", note.id, "Telethon darsligi: 4-dars -- tahrirlangan.")

        # 3) Oxirgi 5 ta xabarni o'qish (eng yangisidan boshlab)
        async for msg in client.iter_messages("me", limit=5):
            sender = await msg.get_sender()
            name = getattr(sender, "first_name", "Noma'lum")
            print(f"[{msg.date}] {name}: {msg.text!r}")

        # 4) "Yozmoqda..." holatini ko'rsatish (foydalanuvchiga xos amal)
        async with client.action("me", "typing"):
            await asyncio.sleep(1)

        # 5) Xabarni o'qilgan deb belgilash
        await client.send_read_acknowledge("me")


async def resolve_entity_safely(username: str) -> None:
    """Entity topilmasa ValueError chiqishi mumkin -- buni to'g'ri ushlash."""
    async with client:
        try:
            entity = await client.get_entity(username)
            print(f"Topildi: {entity.first_name if hasattr(entity, 'first_name') else entity.title}")
        except ValueError:
            print(
                f"'{username}' entity'sini topib bo'lmadi -- hisobingiz hali "
                f"bu foydalanuvchi/kanal bilan hech qachon 'uchrashmagan' bo'lishi mumkin."
            )


async def send_with_flood_guard(entity: str, text: str) -> None:
    """Ommaviy yuborishda albatta FloodWaitError'ni ushlash kerak --
    aks holda dastur kutilmaganda to'xtab qoladi."""
    async with client:
        try:
            await client.send_message(entity, text)
        except FloodWaitError as e:
            print(f"Flood limit -- {e.seconds} soniya kutish kerak, keyin qayta urinamiz.")
            await asyncio.sleep(e.seconds)
            await client.send_message(entity, text)


if __name__ == "__main__":
    asyncio.run(send_and_receive_basics())
