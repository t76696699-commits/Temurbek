import os
from pyrogram import Client

API_ID = int(os.environ["TG_API_ID"])
API_HASH = os.environ["TG_API_HASH"]
BOT_TOKEN = os.environ.get("TG_BOT_TOKEN")

# 1) Fayl-asosli sessiya, alohida papkada (git'ga qo'shilmaydi)
app_persistent = Client(
    "prod_bot",
    api_id=API_ID,
    api_hash=API_HASH,
    bot_token=BOT_TOKEN,
    workdir="./sessions",  # .gitignore: sessions/*.session*
)

# 2) In-memory sessiya — diskka hech narsa yozilmaydi (masalan, testlar uchun)
app_ephemeral = Client(
    "test_bot",
    api_id=API_ID,
    api_hash=API_HASH,
    bot_token=BOT_TOKEN,
    in_memory=True,
)


async def export_portable_session():
    """Foydalanuvchi rejimidagi sessiyani boshqa muhitga ko'chirish uchun
    bitta matn qatoriga aylantiradi. Bu qatorni FAQAT maxfiy o'zgaruvchi
    sifatida saqlang (masalan CI secrets), hech qachon logga chiqarmang."""
    async with Client("user_account", api_id=API_ID, api_hash=API_HASH) as user_app:
        session_string = await user_app.export_session_string()
        # Bu yerda print() ATAYLAB yo'q — production kodida session_string
        # hech qachon konsolga yoki logga chiqarilmaydi.
        return session_string


async def restore_from_string(session_string: str) -> Client:
    """Boshqa muhitda faylsiz, session_string orqali qayta ulanish."""
    app = Client("restored", api_id=API_ID, api_hash=API_HASH, session_string=session_string)
    await app.start()
    return app
