"""Mediani ommaviy yuklab olish: progress, semaphore bilan tezlik nazorati.
pip install telethon python-dotenv
"""
import asyncio
import os
from pathlib import Path

from dotenv import load_dotenv
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import FloodWaitError
from telethon.tl.types import InputMessagesFilterPhotos

load_dotenv()
client = TelegramClient(
    StringSession(os.environ["TELETHON_SESSION_STRING"]),
    int(os.environ["API_ID"]),
    os.environ["API_HASH"],
)

DOWNLOAD_DIR = Path("downloads")
DOWNLOAD_DIR.mkdir(exist_ok=True)
MAX_CONCURRENT_DOWNLOADS = 3  # bir vaqtda ko'pi bilan shuncha yuklab olish


def make_progress_callback(label: str):
    def callback(received: int, total: int) -> None:
        pct = (received / total * 100) if total else 0
        print(f"\r{label}: {pct:5.1f}% ({received}/{total} bayt)", end="")
    return callback


async def download_one(message, semaphore: asyncio.Semaphore) -> None:
    async with semaphore:
        if not message.file:
            return
        filename = f"{message.chat_id}_{message.id}_{message.file.name or 'media'}"
        target = DOWNLOAD_DIR / filename
        if target.exists():
            print(f"O'tkazib yuborildi (mavjud): {filename}")
            return
        try:
            await client.download_media(
                message, file=str(target), progress_callback=make_progress_callback(filename)
            )
            print()  # progress qatoridan keyin yangi qator
        except FloodWaitError as e:
            print(f"\nFloodWait media yuklashda: {e.seconds}s kutamiz.")
            await asyncio.sleep(e.seconds)
            await download_one(message, semaphore)


async def bulk_download_photos(chat: str, limit: int = 100) -> None:
    semaphore = asyncio.Semaphore(MAX_CONCURRENT_DOWNLOADS)
    tasks = []
    async with client:
        async for message in client.iter_messages(
            chat, limit=limit, filter=InputMessagesFilterPhotos
        ):
            tasks.append(asyncio.create_task(download_one(message, semaphore)))
        await asyncio.gather(*tasks)
    print(f"\nJami {len(tasks)} ta media uchun yuklab olish yakunlandi.")


if __name__ == "__main__":
    asyncio.run(bulk_download_photos("@ochiq_kanal_namunasi", limit=50))
