
# Chunking funksiyalari: fixed-size (overlap bilan) va structure-aware
# (HTML h3 teglariga asoslangan). Ikkinchisi haqiqiy lessons.text_content
# ustida ishlaydi.

from __future__ import annotations
import re


def fixed_size_chunks(text: str, chunk_size: int = 500, overlap: int = 80) -> list[str]:
    """Eng oddiy strategiya: har chunk_size belgidan keyin kesadi, har
    keyingi chunk oldingisining oxirgi `overlap` belgisini qaytadan
    o'z ichiga oladi."""
    if overlap >= chunk_size:
        raise ValueError("overlap chunk_size'dan kichik bo'lishi shart")

    chunks: list[str] = []
    start = 0
    text = text.strip()
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end].strip())
        start = end - overlap  # oldingi chunk oxiridan overlap miqdorida orqaga qaytish
    return [c for c in chunks if c]


def structure_aware_chunks(html: str) -> list[dict]:
    """lessons.text_content kabi HTML matnni <h3> sarlavhalari bo'yicha
    bo'laklarga ajratadi — har bir bo'lim (sarlavha + undan keyingi matn)
    alohida chunk bo'ladi. Production'da BeautifulSoup ishlatilardi;
    bu yerda tushunarli bo'lishi uchun oddiy regex ishlatamiz."""
    parts = re.split(r"(?=<h3>)", html)
    chunks = []
    for part in parts:
        part = part.strip()
        if not part:
            continue
        heading_match = re.search(r"<h3>(.*?)</h3>", part)
        heading = heading_match.group(1) if heading_match else "(sarlavhasiz)"
        plain_text = re.sub(r"<[^>]+>", " ", part)
        plain_text = re.sub(r"\s+", " ", plain_text).strip()
        chunks.append({"heading": heading, "text": plain_text})
    return chunks


if __name__ == "__main__":
    # Haqiqiy lessons.text_content'dan olingan qisqartirilgan namuna —
    # o'zbekcha "CSS Flexbox" darsining shakli:
    sample_lesson_html = (
        "<h3>Flexbox nima</h3><p>Flexbox — elementlarni bir qatorda yoki "
        "ustunda tekis joylashtirish uchun CSS xususiyati.</p>"
        "<h3>flex-direction xususiyati</h3><p>flex-direction: column "
        "elementlarni tepadan pastga joylashtiradi.</p>"
        "<h3>justify-content va align-items</h3><p>Bu ikkalasi elementlarni "
        "gorizontal va vertikal tekislash uchun ishlatiladi.</p>"
    )

    print("--- Fixed-size chunking (overlap=20) ---")
    for i, c in enumerate(fixed_size_chunks(sample_lesson_html, chunk_size=80, overlap=20)):
        print(f"chunk {i}: {c[:70]!r}...")

    print("\n--- Structure-aware chunking (h3 asosida) ---")
    for i, c in enumerate(structure_aware_chunks(sample_lesson_html)):
        print(f"chunk {i} [{c['heading']}]: {c['text']}")
