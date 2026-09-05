from pyrogram import Client, filters

app = Client("filters_demo")


async def _is_admin(_, client, message):
    member = await client.get_chat_member(message.chat.id, message.from_user.id)
    return member.status in ("administrator", "creator")


is_admin = filters.create(_is_admin)


@app.on_message(filters.command("ban") & filters.group & is_admin)
async def ban_handler(client, message):
    await message.reply_text("Foydalanuvchi ban qilindi (demo).")


@app.on_message(filters.command("ban") & filters.group & ~is_admin)
async def ban_denied(client, message):
    await message.reply_text("Sizda /ban buyrug'i uchun huquq yo'q.")


@app.on_message(filters.photo | filters.video)
async def media_handler(client, message):
    kind = "rasm" if message.photo else "video"
    await message.reply_text(f"{kind.capitalize()} qabul qilindi.")


@app.on_message(filters.private & filters.text & ~filters.command("start"))
async def private_text(client, message):
    await message.reply_text(f"Shaxsiy xabar: {message.text}")


if __name__ == "__main__":
    app.run()
