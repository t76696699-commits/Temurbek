# pip install pyrogram tgcrypto python-dotenv
import os
from pyrogram import Client

from dotenv import load_dotenv
load_dotenv()

API_ID = int(os.environ["TG_API_ID"])       # my.telegram.org'dan
API_HASH = os.environ["TG_API_HASH"]        # my.telegram.org'dan
BOT_TOKEN = os.environ.get("TG_BOT_TOKEN")  # bo'lsa — bot rejimi, bo'lmasa — foydalanuvchi rejimi


def build_client() -> Client:
    """Bitta funksiya — ikkala rejimni ham qo'llab-quvvatlaydi.
    BOT_TOKEN mavjud bo'lsa bot sifatida, aks holda interaktiv
    foydalanuvchi (userbot) sifatida ulanadi."""
    if BOT_TOKEN:
        return Client(
            "bot_session",
            api_id=API_ID,
            api_hash=API_HASH,
            bot_token=BOT_TOKEN,
            workdir="./sessions",
        )
    return Client(
        "user_session",
        api_id=API_ID,
        api_hash=API_HASH,
        workdir="./sessions",
    )


app = build_client()


@app.on_message()
async def whoami(client: Client, message):
    me = await client.get_me()
    mode = "BOT" if me.is_bot else "USER"
    await message.reply_text(
        f"Men [{mode}] rejimida ishlayapman: @{me.username or me.id}"
    )


if __name__ == "__main__":
    app.run()

