from pyrogram import Client, filters
from pyrogram.enums import ChatMemberStatus

app = Client("handlers_demo")


# Узкий (специфичный) handler — регистрируется ПЕРВЫМ
@app.on_message(filters.command("start"), group=0)
async def start_handler(client, message):
    await message.reply_text("Bu /start uchun maxsus handler.")


# Log qiluvchi handler — mustaqil guruhda, HAR BIR xabarni ko'radi
@app.on_message(group=-1)
async def logger(client, message):
    print(f"[LOG] chat={message.chat.id} matn={message.text!r}")
    message.continue_propagation()  # shu guruhdagi keyingisiga ham bersin


# Keng (umumiy) handler — ATAYLAB oxirida, aks holda hammasini "yutib qo'yadi"
@app.on_message(filters.text, group=0)
async def fallback_handler(client, message):
    await message.reply_text(f"Tushunmadim: {message.text}")


@app.on_callback_query()
async def on_button(client, callback_query):
    await callback_query.answer("Tugma bosildi!")


# Xabar tahrirlanganda ishga tushadi — jadvaldagi yana bir handler turi
@app.on_edited_message(filters.text)
async def on_edit(client, message):
    print(f"[EDIT] chat={message.chat.id} yangi matn: {message.text!r}")


# Xabar(lar) o'chirilganda ishga tushadi
@app.on_deleted_messages()
async def on_delete(client, messages):
    ids = [m.id for m in messages]
    print(f"[DELETE] o'chirilgan xabar id'lari: {ids}")


# Guruhga a'zo qo'shildi/chiqdi/ban bo'ldi — moderatsiya uchun foydali
@app.on_chat_member_updated()
async def on_member_change(client, update):
    if update.new_chat_member and update.new_chat_member.status == ChatMemberStatus.MEMBER:
        user = update.new_chat_member.user
        print(f"[JOIN] {user.first_name} guruhga qo'shildi")


if __name__ == "__main__":
    app.run()
