
# Cosine similarity: avval qo'lda (faqat math moduli bilan), keyin
# numpy bilan tezroq/qisqaroq shaklda. Ikkalasi ham BIR XIL natijani
# berishi kerak — bu haqiqiy o'zaro tekshiruv (sanity check).

from __future__ import annotations
import math


def dot_product(a: list[float], b: list[float]) -> float:
    return sum(x * y for x, y in zip(a, b))


def magnitude(v: list[float]) -> float:
    return math.sqrt(sum(x * x for x in v))


def cosine_similarity(a: list[float], b: list[float]) -> float:
    """Qo'lda, faqat asosiy Python bilan hisoblangan cosine similarity —
    -1..1 oralig'ida, 1 — bir xil yo'nalish (ma'no juda yaqin)."""
    mag_a, mag_b = magnitude(a), magnitude(b)
    if mag_a == 0 or mag_b == 0:
        return 0.0  # nol vektor bilan solishtirish ma'nosiz — 0 qaytaramiz
    return dot_product(a, b) / (mag_a * mag_b)


def top_k_similar(query_vector: list[float], candidates: dict[str, list[float]], k: int = 3) -> list[tuple[str, float]]:
    """Brute-force qidiruv: HAR BIR nomzod bilan solishtiradi, eng
    yuqori cosine similarity'ga ega top-k tasini qaytaradi."""
    scored = [(name, cosine_similarity(query_vector, vec)) for name, vec in candidates.items()]
    scored.sort(key=lambda pair: pair[1], reverse=True)
    return scored[:k]


if __name__ == "__main__":
    # Qo'lda tekshirilgan misol (darsdagi hisob-kitobga mos):
    a = [1.0, 2.0]
    b = [2.0, 4.0]  # b = 2*a -> bir xil yo'nalish -> cosine ≈ 1.0
    print(f"cosine_similarity(a, b) = {cosine_similarity(a, b):.4f}  (kutilgan: ~1.0)")

    # numpy bilan bir xil natija — production kodda odatda shu ishlatiladi:
    import numpy as np
    a_np, b_np = np.array(a), np.array(b)
    cosine_np = np.dot(a_np, b_np) / (np.linalg.norm(a_np) * np.linalg.norm(b_np))
    print(f"numpy bilan:              {cosine_np:.4f}  (bir xil natija bo'lishi kerak)")

    # Brute-force qidiruv namunasi:
    candidates = {
        "Python funksiyalari haqida dars": [0.9, 0.1, 0.3],
        "CSS Flexbox haqida dars": [0.1, 0.9, 0.2],
        "Pitsa retsepti (mos emas)": [0.05, 0.02, 0.99],
    }
    query = [0.85, 0.15, 0.25]  # "funksiya qanday yoziladi" so'roviga o'xshash vektor
    print("\nTop-2 natija:")
    for name, score in top_k_similar(query, candidates, k=2):
        print(f"  {score:.4f}  {name}")
