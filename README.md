"""Kanal/guruhga xavfsiz, tezlikni cheklab qo'shilish namunasi.
pip install telethon python-dotenv
"""
import asyncio
import os
import random

from dotenv import load_dotenv
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import (
    PeerFloodError,
    FloodWaitError,
    UserAlreadyParticipantError,
    InviteHashExpiredError,
)
from telethon.tl.functions.channels import JoinChannelRequest, LeaveChannelRequest
from telethon.tl.functions.messages import ImportChatInviteRequest

load_dotenv()
client = TelegramClient(
    StringSession(os.environ["TELETHON_SESSION_STRING"]),
    int(os.environ["API_ID"]),
    os.environ["API_HASH"],
)


async def join_public_channel_safely(username: str) -> bool:
    """Ochiq kanalga qo'shilish -- xatoliklarni to'g'ri ushlab."""
    try:
        await client(JoinChannelRequest(username))
        print(f"Muvaffaqiyatli qo'shildi: {username}")
        return True
    except UserAlreadyParticipantError:
        print(f"Allaqachon a'zo: {username}")
        return True
    except PeerFloodError:
        print(
            "PeerFloodError -- juda ko'p ijtimoiy amal bajarilyapti. "
            "DARHOL to'xtash va bir necha soat kutish kerak."
        )
        return False
    except FloodWaitError as e:
        print(f"FloodWait: {e.seconds} soniya kutish kerak.")
        return False


async def join_private_group_by_invite(invite_hash: str) -> bool:
    """Yopiq guruhga taklif havolasi orqali qo'shilish."""
    try:
        await client(ImportChatInviteRequest(invite_hash))
        print("Yopiq guruhga muvaffaqiyatli qo'shildi.")
        return True
    except InviteHashExpiredError:
        print("Taklif havolasi muddati o'tgan yoki bekor qilingan.")
        return False
    except PeerFloodError:
        print("PeerFloodError -- to'xtash kerak.")
        return False


async def join_many_with_responsible_pacing(usernames: list[str]) -> None:
    """MAS'ULIYATLI namuna: guruhga qo'shilishlar orasida tasodifiy,
    sezilarli tanaffus bilan. Ommaviy, tez qo'shilish HECH QACHON
    tavsiya etilmaydi -- bu shunchaki xavfni kamaytiruvchi namuna,
    ommaviy avtomatlashtirishni rag'batlantirish emas."""
    for username in usernames:
        ok = await join_public_channel_safely(username)
        if not ok:
            print("Xavfsizlik uchun jarayon to'xtatildi.")
            break
        pause = random.uniform(120, 600)  # 2-10 daqiqa
        print(f"Keyingi qo'shilishgacha {pause / 60:.1f} daqiqa kutamiz...")
        await asyncio.sleep(pause)


async def main() -> None:
    async with client:
        # Faqat DEMO uchun -- haqiqiy ro'yxat juda kichik bo'lishi kerak
        await join_many_with_responsible_pacing(["@ochiq_kanal_namunasi"])


if __name__ == "__main__":
    asyncio.run(main())
