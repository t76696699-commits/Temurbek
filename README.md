
# Namuna: "RAG'siz" LLM chaqiruvi platforma ma'lumotlari haqida
# nima uchun ishonchsiz javob berishini ko'rsatadi (135-kursdagi
# _ask_ai() ORQALI — biz uni qaytadan yozmaymiz, faqat import qilamiz).

import asyncio
from app.services.grok_ai_client import _ask_ai


async def ask_without_rag(question: str) -> str | None:
    """RAG'siz to'g'ridan-to'g'ri savol — modelning javobi platformaning
    HAQIQIY ma'lumotlariga emas, o'zining umumiy "bilimi"ga asoslanadi."""
    prompt = (
        f"Savol: {question}\n\n"
        "Iltimos aniq va qisqa javob ber."
    )
    return await _ask_ai(prompt)


async def main() -> None:
    question = "Ushbu talaba platformasida nechta kurs bor va ular qaysi kategoriyalarga bo'lingan?"
    answer = await ask_without_rag(question)
    print("Savol:", question)
    print("Model javobi (RAG'siz):", answer)
    print(
        "\nDIQQAT: bu javob ishonchli ko'rinishi mumkin, lekin model "
        "hech qachon ushbu platformaning courses jadvalini ko'rmagan — "
        "u statistik ehtimollik asosida matn generatsiya qilmoqda, "
        "haqiqiy ma'lumotni qaytarmoqda emas. Bu — gallyutsinatsiya."
    )


if __name__ == "__main__":
    asyncio.run(main())


# ---------------------------------------------------------------------------
# Taqqoslash uchun: RAG variantda promptga HAQIQIY ma'lumot qo'shiladi.
# (Keyingi darslarda buni to'liq quramiz — bu yerda faqat farqni ko'ramiz.)
# ---------------------------------------------------------------------------

async def ask_with_manual_context(question: str, real_facts: str) -> str | None:
    """RAG'ning eng oddiy shakli: hali qidiruv yo'q, lekin haqiqiy
    faktlarni promptga qo'lda qo'shib qo'yish orqali javob sifati keskin
    o'zgarishini ko'rish mumkin."""
    prompt = (
        "Quyidagi HAQIQIY ma'lumotdan foydalanib savolga javob ber. "
        "Agar ma'lumotda javob bo'lmasa, 'ma'lumotda bu haqida yo'q' deb ayt "
        "— o'ylab topma.\n\n"
        f"MA'LUMOT:\n{real_facts}\n\n"
        f"SAVOL: {question}"
    )
    return await _ask_ai(prompt)
