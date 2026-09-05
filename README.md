# O'RNATISH: pip install pyrogram tgcrypto
#
# Bu darsda hali Client'ni to'liq sozlamaymiz (keyingi darsda batafsil) —
# lekin uchta kutubxonaning "salom dunyo" darajasidagi farqini his qilish
# uchun qisqacha taqqoslash keltirilgan. Faqat Pyrogram qismi shu kursda
# to'liq ishlaydigan holatga keladi.

# --- Pyrogram (shu kurs markazi) ---
from pyrogram import Client, filters

app = Client("my_account")  # nomi bilan session fayli yaratiladi


@app.on_message(filters.command("start") & filters.private)
async def start_handler(client: Client, message):
    await message.reply_text(
        f"Salom, {message.from_user.first_name}! Men Pyrogram orqali ishlayapman."
    )


if __name__ == "__main__":
    app.run()  # ichkarida connect() -> idle() -> disconnect() ni boshqaradi


# --- Solishtirish uchun: aiogram'da xuddi shu handler (Bot API, HTTP) ---
#
# from aiogram import Bot, Dispatcher, F
# from aiogram.filters import Command
#
# bot = Bot(token="...")
# dp = Dispatcher()
#
# @dp.message(Command("start"))
# async def start_handler(message):
#     await message.answer(f"Salom, {message.from_user.first_name}!")
#
# aiogram'da faqat bot_token bor — foydalanuvchi hisobi sifatida kira olmaysiz.


# --- Solishtirish uchun: Telethon'da xuddi shu handler (MTProto, ham dual-mode) ---
#
# from telethon import TelegramClient, events
#
# client = TelegramClient("my_account", api_id, api_hash)
#
# @client.on(events.NewMessage(pattern="/start"))
# async def start_handler(event):
#     await event.reply("Salom!")
#
# client.start()  # yoki client.start(bot_token="...") — bot rejimi uchun
# client.run_until_disconnected()
