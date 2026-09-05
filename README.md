"""Gibrid arxitektura: Telethon userbot + aiogram bot, Redis navbat orqali.
pip install telethon aiogram redis python-dotenv
"""
import asyncio
import json
import os

from dotenv import load_dotenv
from telethon import TelegramClient, events
from telethon.sessions import StringSession
import redis.asyncio as aioredis
from aiogram import Bot, Dispatcher

load_dotenv()

QUEUE_KEY = "userbot:alerts"
redis_client = aioredis.from_url(os.environ.get("REDIS_URL", "redis://localhost:6379/0"))

# --- Userbot qismi -----------------------------------------------------
userbot = TelegramClient(
    StringSession(os.environ["TELETHON_SESSION_STRING"]),
    int(os.environ["API_ID"]),
    os.environ["API_HASH"],
)
KEYWORDS = ["chegirma", "aksiya"]
MONITORED_CHANNELS = ["@ochiq_kanal_1", "@ochiq_kanal_2"]  # ADMIN nazorat qiladi, foydalanuvchi emas!


@userbot.on(events.NewMessage(chats=MONITORED_CHANNELS))
async def on_channel_message(event: events.NewMessage.Event) -> None:
    text = (event.text or "").lower()
    if any(kw in text for kw in KEYWORDS):
        payload = {
            "chat_id": event.chat_id,
            "message_id": event.id,
            "text": event.text[:200],
        }
        await redis_client.rpush(QUEUE_KEY, json.dumps(payload))
        print(f"Navbatga qo'shildi: {payload['text'][:40]!r}")


# --- Bot qismi ----------------------------------------------------------
bot = Bot(token=os.environ["BOT_TOKEN"])
dp = Dispatcher()
SUBSCRIBERS = [111111, 222222]  # obunachi student_id'lar (haqiqiy loyihada DB'dan)


async def notify_subscribers_loop() -> None:
    """Navbatni doimiy tekshirib, topilgan moslikni obunachilarga yetkazadi."""
    while True:
        raw = await redis_client.blpop(QUEUE_KEY, timeout=5)
        if raw is None:
            continue
        _, payload_json = raw
        payload = json.loads(payload_json)
        for subscriber_id in SUBSCRIBERS:
            await bot.send_message(
                subscriber_id, f"Yangi moslik topildi:\n{payload['text']}"
            )


async def main() -> None:
    async with userbot:
        await asyncio.gather(
            userbot.run_until_disconnected(),
            notify_subscribers_loop(),
        )


if __name__ == "__main__":
    asyncio.run(main())
