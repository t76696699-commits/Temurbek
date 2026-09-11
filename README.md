
# Ikkala yondashuvni ham ko'ramiz: mahalliy (sentence-transformers) va
# hosted API (Gemini). Ikkalasi ham bir xil natija turi qaytaradi:
# suzuvchi sonlar ro'yxati (vektor).

from __future__ import annotations
import math


# ---------------------------------------------------------------------------
# 1) Mahalliy model bilan (production'da haqiqiy kutubxona shunday ishlatiladi)
# ---------------------------------------------------------------------------
#
#   from sentence_transformers import SentenceTransformer
#   model = SentenceTransformer("all-MiniLM-L6-v2")   # bir marta yuklanadi
#   vector = model.encode("Python funksiyalari qanday e'lon qilinadi")
#   # vector — 384 ta floatdan iborat numpy array
#
# Bu kursda haqiqiy og'ir modelni yuklamasdan, PRINSIPNI ko'rsatish uchun
# oddiy, deterministik "soxta embedding" funksiyasidan foydalanamiz — u
# so'zlarning belgi-darajasidagi xususiyatlaridan foydalanib past o'lchamli
# vektor yasaydi. Bu HAQIQIY semantik model emas (faqat harf statistikasiga
# asoslangan), lekin vektor SHAKLI va API'si bir xil — shuning uchun
# quyidagi cosine_similarity, chunking va pgvector darslari xuddi shu
# funksiya ustida ishlaydi.

def fake_embed(text: str, dims: int = 32) -> list[float]:
    """Deterministik, kichik o'lchamli "o'quv uchun" embedding — haqiqiy
    modeldagi kabi og'ir emas, lekin xuddi shunday: matn -> son vektori."""
    text = text.lower().strip()
    vector = [0.0] * dims
    for i, ch in enumerate(text):
        idx = (ord(ch) + i) % dims
        vector[idx] += 1.0
    norm = math.sqrt(sum(v * v for v in vector)) or 1.0
    return [v / norm for v in vector]  # normallashtirilgan (uzunligi 1) vektor


# ---------------------------------------------------------------------------
# 2) Hosted API bilan (Gemini text-embedding-004) — haqiqiy HTTP so'rov shakli
# ---------------------------------------------------------------------------

import httpx
from app.config import settings


async def gemini_embed(text: str) -> list[float] | None:
    """Gemini'ning embedding endpoint'iga haqiqiy so'rov shakli.
    API kaliti bo'lmasa None qaytaradi (135-kursdagi ProviderError
    uslubiga o'xshab) — chaqiruvchi kod fallback qila oladi."""
    if not settings.GEMINI_API_KEY:
        return None
    url = (
        f"{settings.GEMINI_API_URL.rstrip('/')}/text-embedding-004:embedContent"
        f"?key={settings.GEMINI_API_KEY}"
    )
    async with httpx.AsyncClient(timeout=30.0) as client:
        resp = await client.post(url, json={"content": {"parts": [{"text": text}]}})
        if resp.status_code >= 400:
            return None
        data = resp.json()
        return data.get("embedding", {}).get("values")


if __name__ == "__main__":
    v1 = fake_embed("Python funksiyasi qanday yoziladi")
    v2 = fake_embed("def kalit so'zi bilan funksiya e'lon qilinadi")
    v3 = fake_embed("Pitsa retsepti: xamir va pomidor sousi")
    print("V1 o'lchami:", len(v1))
    print("V1[:5]:", [round(x, 3) for x in v1[:5]])
    print("V2[:5]:", [round(x, 3) for x in v2[:5]])
    print("V3[:5]:", [round(x, 3) for x in v3[:5]])
