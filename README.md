"""Turli xil events.* handlerlari namunasi.
pip install telethon python-dotenv
"""
import asyncio
import os
import re

from dotenv import load_dotenv
from telethon import TelegramClient, events
from telethon.sessions import StringSession
from telethon.tl.types import PeerChannel

load_dotenv()
client = TelegramClient(
    StringSession(os.environ["TELETHON_SESSION_STRING"]),
    int(os.environ["API_ID"]),
    os.environ["API_HASH"],
)


# 1) Oddiy buyruq -- faqat kiruvchi shaxsiy xabarlarda
@client.on(events.NewMessage(pattern=r"(?i)^/ping$", incoming=True))
async def on_ping(event: events.NewMessage.Event) -> None:
    if event.is_private:
        await event.reply("pong")


# 2) Regex bilan naqsh -- masalan, "eslatma: <matn>" ko'rinishidagi xabarlar
@client.on(events.NewMessage(pattern=re.compile(r"^eslatma:\s*(.+)$", re.IGNORECASE)))
async def on_reminder(event: events.NewMessage.Event) -> None:
    reminder_text = event.pattern_match.group(1)
    await event.respond(f"Eslatma saqlandi: {reminder_text!r}")
    raise events.StopPropagation  # boshqa handlerlar bu xabar uchun ishga tushmasin


# 3) Faqat belgilangan kanal/guruhlardagi xabarlarni kuzatish
MONITORED_CHATS = [-1001234567890]  # kanal/guruh ID'lari


@client.on(events.NewMessage(chats=MONITORED_CHATS))
async def on_monitored_message(event: events.NewMessage.Event) -> None:
    print(f"[Kuzatilayotgan chat] {event.chat_id}: {event.text[:80]!r}")


# 4) Guruhga a'zo qo'shilganda/chiqqanda
@client.on(events.ChatAction())
async def on_chat_action(event: events.ChatAction.Event) -> None:
    if event.user_joined or event.user_added:
        user = await event.get_user()
        print(f"Yangi a'zo: {user.first_name}")
    elif event.user_left or event.user_kicked:
        print("Bir a'zo chiqib ketdi/chiqarildi.")


# 5) Xabar tahrirlanganda kuzatish
@client.on(events.MessageEdited())
async def on_message_edited(event: events.MessageEdited.Event) -> None:
    print(f"Xabar tahrirlandi (id={event.id}): {event.text[:80]!r}")


# 6) Dinamik ravishda handler qo'shish (dekoratorsiz)
async def dynamic_handler(event: events.NewMessage.Event) -> None:
    await event.reply("Dinamik ro'yxatga olingan handler ishladi.")


def register_dynamic_handler() -> None:
    client.add_event_handler(dynamic_handler, events.NewMessage(pattern="/dinamik"))


async def main() -> None:
    register_dynamic_handler()
    async with client:
        print("Handlerlar tayyor, tinglanmoqda...")
        await client.run_until_disconnected()


if __name__ == "__main__":
    asyncio.run(main())
