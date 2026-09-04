"""iter_messages orqali katta tarixni sahifalab, davom ettirib o'qish.
pip install telethon python-dotenv
"""
import asyncio
import json
import os
from pathlib import Path

from dotenv import load_dotenv
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import FloodWaitError

load_dotenv()
client = TelegramClient(
    StringSession(os.environ["TELETHON_SESSION_STRING"]),
    int(os.environ["API_ID"]),
    os.environ["API_HASH"],
)

PROGRESS_FILE = Path("scan_progress.json")


def load_progress() -> dict:
    if PROGRESS_FILE.exists():
        return json.loads(PROGRESS_FILE.read_text())
    return {}


def save_progress(chat_key: str, last_id: int) -> None:
    progress = load_progress()
    progress[chat_key] = last_id
    PROGRESS_FILE.write_text(json.dumps(progress))


async def scan_full_history(chat: str, batch_size: int = 50) -> None:
    """Chatning to'liq tarixini xronologik tartibda, oldingi to'xtagan
    joydan davom ettirib skanerlaydi."""
    progress = load_progress()
    last_id = progress.get(chat, 0)
    processed = 0
    batch: list[str] = []

    async def flush_batch() -> None:
        if not batch:
            return
        # Bu yerda haqiqiy loyihada DB'ga bulk insert bo'lardi.
        print(f"  -- {len(batch)} ta xabar saqlandi (batch)")
        batch.clear()

    try:
        async for message in client.iter_messages(
            chat, reverse=True, offset_id=last_id, wait_time=1
        ):
            if message.text:
                batch.append(message.text)
            if len(batch) >= batch_size:
                await flush_batch()
            processed += 1
            last_id = message.id

            if processed % 500 == 0:
                save_progress(chat, last_id)
                print(f"Progress saqlandi: {processed} ta xabar, oxirgi id={last_id}")
    except FloodWaitError as e:
        print(f"FloodWait: {e.seconds} soniya kutamiz va progressni saqlaymiz.")
        save_progress(chat, last_id)
        await asyncio.sleep(e.seconds)
        await scan_full_history(chat, batch_size)  # davom ettirish
        return

    await flush_batch()
    save_progress(chat, last_id)
    print(f"Skanerlash tugadi: jami {processed} ta xabar qayta ishlandi.")


async def main() -> None:
    async with client:
        await scan_full_history("@ochiq_kanal_namunasi")


if __name__ == "__main__":
    asyncio.run(main())
