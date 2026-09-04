4-Xabar yuborish va o'qish: haqiqiy hisob nomidan ishlash
Урок 4 из 13
· 3 раздела
✓ Пройден
📝
Matn
Matn
#1
send_message va get_messages -- tanish, lekin boshqacha
Metodlar nomi aiogram'dagiga o'xshab tuyulishi mumkin, lekin ma'no boshqa: client.send_message(entity, text) chaqirilganda, xabar bot nomidan emas, balki sizning haqiqiy hisobingiz nomidan yuboriladi — qabul qiluvchi buni oddiy foydalanuvchi xabari sifatida ko'radi, "Bot" belgisi yo'q. client.get_messages(entity, limit=N) esa — standart holatda eng yangi xabarlardan boshlab, teskari xronologik tartibda — oxirgi N ta xabarni qaytaradi (bu 6-darsda ko'radigan iter_messages'dan farqli, u katta hajmdagi tarixni sahifalab o'qish uchun).

"entity" nima -- va nega bu tushuncha muhim
Telethon'da deyarli har bir metod "entity" kutadi — bu foydalanuvchi, guruh yoki kanalni bildiruvchi mavhum tushuncha. Entity sifatida quyidagilarni berish mumkin:

Username qatori: "@someuser" yoki "someuser" (ikkalasi ham ishlaydi).
Butun son ID: 777000 (Telegram xizmat hisobi kabi).
Maxsus qator "me" — bu sizning o'z hisobingiz (Saved Messages'ga yuborish uchun juda qulay: client.send_message("me", "eslatma")).
Oldindan olingan User/Chat/Channel obyekti (masalan, get_dialogs()dan qaytgan).
Ostida esa Telegram serveri har doim PeerUser/PeerChat/PeerChannel (ID) VA access_hash (o'sha entity uchun bir martalik ruxsat kaliti) juftligini talab qiladi. Telethon buni sizdan yashiradi — u ichki entity cache'da username/ID'larni access_hash'larga moslashtirib boradi. Shu sababli ba'zan ValueError: Could not find the input entity xatosi chiqadi: agar hisobingiz hali hech qachon o'sha foydalanuvchi/kanal bilan "uchrashmagan" bo'lsa (masalan, umumiy guruhda bo'lmagan notanish foydalanuvchi ID'si), Telethon uning access_hash'ini qayerdan olishni bilmaydi.

username / 'me' / ID

Topildi

Topilmadi

User/Chat/Channel obyekti

client.send_message(entity, text)

entity turi?

Entity cache'dan qidiriladi

access_hash bilan PeerUser/PeerChannel tuziladi

ValueError: Could not find the input entity

Xom MTProto so'rovi (messages.sendMessage)

Diagramma shuni ko'rsatadi: entity resolution — bu Bot API'da umuman yo'q qo'shimcha qatlam, chunki Bot API sizning nomingizdan har doim to'g'ridan-to'g'ri ID orqali ishlaydi va access_hash muammosi Telegram tomonida hal qilinadi.

O'qish, tahrirlash, o'chirish -- foydalanuvchiga xos amallar
Haqiqiy hisob sifatida sizda bot hech qachon qila olmaydigan amallar bor: client.send_read_acknowledge(entity) — xabarlarni "o'qilgan" deb belgilash (bot buni hech qachon boshqara olmaydi, chunki botning "o'qish holati" tushunchasi yo'q); client.action(entity, "typing") — "yozmoqda..." holatini ko'rsatish; client.edit_message(entity, message, new_text) — avval yuborilgan xabarni tahrirlash; client.delete_messages(entity, message_ids) — xabarlarni o'chirish. Bu metodlarning barchasi Bot API'da ham mavjud, lekin faqat botning O'Z xabarlari uchun ishlaydi; userbot esa (agar guruh sozlamalari ruxsat bersa) boshqa a'zolarning xabarlarini ham o'chira oladi, agar admin bo'lsa.

Tezlik cheklovlari -- bu yerda ham amal qiladi
Xabar yuborish tezligi cheksiz emas — ayniqsa yangi yoki ko'p kontaktga ega bo'lmagan hisoblar uchun Telegram FloodWaitError qaytarishi mumkin. Bir nechta xabarni ketma-ket, halqada (loop) yuborishdan saqlaning — bu 7-darsda batafsil ko'rib chiqiladigan "ommaviy avtomatlashtirish xavfi"ning eng oddiy shakli.

Qo'shimcha yuborish parametrlari
send_message yana bir nechta foydali parametrni qabul qiladi, ular Bot API'da ham mavjud, lekin userbot kontekstida boshqacha ma'no kasb etadi: silent=True — qabul qiluvchiga ovozli bildirishnomasiz yetkazish; schedule=datetime(...) — xabarni kelajakdagi aniq vaqtga rejalashtirish (Telegram serverining o'zi saqlaydi, sizning dasturingiz ishlab turishi shart emas); link_preview=False — havola oldindan ko'rinishini o'chirish; reply_to=message_id — muayyan xabarga javob sifatida yuborish. Bulardan tashqari, foydalanuvchi hisobiga xos yana bir imkoniyat — qoralamalar (drafts): client.get_drafts() orqali barcha chatlardagi hali yuborilmagan qoralama matnlarni olish mumkin, bu bot hisobida umuman mavjud bo'lmagan tushuncha, chunki qoralama — klient ilovasining shaxsiy holati, bot esa bunday "ilova holati"ga ega emas.

💻
Kod
Kod
#2
python
 Nusxalash
"""Xabar yuborish, o'qish, tahrirlash va entity resolution namunalari.
pip install telethon python-dotenv
"""
import asyncio
import os

from dotenv import load_dotenv
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import FloodWaitError

load_dotenv()
API_ID = int(os.environ["API_ID"])
API_HASH = os.environ["API_HASH"]
SESSION_STRING = os.environ["TELETHON_SESSION_STRING"]

client = TelegramClient(StringSession(SESSION_STRING), API_ID, API_HASH)


async def send_and_receive_basics() -> None:
    async with client:
        # 1) O'zingizga eslatma yuborish (Saved Messages)
        note = await client.send_message("me", "Telethon darsligi: 4-dars yakunlandi.")
        print(f"Yuborildi, id={note.id}")

        # 2) Xabarni tahrirlash
        await client.edit_message("me", note.id, "Telethon darsligi: 4-dars -- tahrirlangan.")

        # 3) Oxirgi 5 ta xabarni o'qish (eng yangisidan boshlab)
        async for msg in client.iter_messages("me", limit=5):
            sender = await msg.get_sender()
            name = getattr(sender, "first_name", "Noma'lum")
            print(f"[{msg.date}] {name}: {msg.text!r}")

        # 4) "Yozmoqda..." holatini ko'rsatish (foydalanuvchiga xos amal)
        async with client.action("me", "typing"):
            await asyncio.sleep(1)

        # 5) Xabarni o'qilgan deb belgilash
        await client.send_read_acknowledge("me")


async def resolve_entity_safely(username: str) -> None:
    """Entity topilmasa ValueError chiqishi mumkin -- buni to'g'ri ushlash."""
    async with client:
        try:
            entity = await client.get_entity(username)
            print(f"Topildi: {entity.first_name if hasattr(entity, 'first_name') else entity.title}")
        except ValueError:
            print(
                f"'{username}' entity'sini topib bo'lmadi -- hisobingiz hali "
                f"bu foydalanuvchi/kanal bilan hech qachon 'uchrashmagan' bo'lishi mumkin."
            )


async def send_with_flood_guard(entity: str, text: str) -> None:
    """Ommaviy yuborishda albatta FloodWaitError'ni ushlash kerak --
    aks holda dastur kutilmaganda to'xtab qoladi."""
    async with client:
        try:
            await client.send_message(entity, text)
        except FloodWaitError as e:
            print(f"Flood limit -- {e.seconds} soniya kutish kerak, keyin qayta urinamiz.")
            await asyncio.sleep(e.seconds)
            await client.send_message(entity, text)


if __name__ == "__main__":
    asyncio.run(send_and_receive_basics())
