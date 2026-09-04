import asyncio
from telethon import TelegramClient

api_id = 12345
api_hash = "sizning_api_hash"

client = TelegramClient("session_nomi", api_id, api_hash)


async def main():
    await client.start()

    # 3 ta xabar yuborish
    msg1 = await client.send_message("me", "1-eslatma: darsni tugatish")
    msg2 = await client.send_message("me", "2-eslatma: mashqni topshirish")
    msg3 = await client.send_message("me", "3-eslatma: kod review")
    print(f"3 ta xabar yuborildi: {msg1.id}, {msg2.id}, {msg3.id}")

    # Kamida bittasini tahrirlash
    await client.edit_message("me", msg1, "1-eslatma (YANGILANDI): allaqachon tugatdim!")
    print("Birinchi xabar tahrirlandi")

    # entity ValueError'ni to'g'ri ushlash — hisob hali "uchrashmagan" entity
    try:
        await client.send_message("mavjud_bolmagan_foydalanuvchi_12345xyz", "Salom")
    except ValueError as e:
        print(f"Kutilgan xato ushlandi: {e}")
        print("Sabab: hisob hali bu entity bilan uchrashmagan — access_hash topilmadi")

    # typing holati va o'qilgan deb belgilash — foydalanuvchiga xos amallar
    async with client.action("me", "typing"):
        await asyncio.sleep(1)
    await client.send_read_acknowledge("me")

    await client.disconnect()


if __name__ == "__main__":
    asyncio.run(main())
