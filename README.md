# ============================================================
# 1) So'rovga stream: true qo'shish (OpenAI-uslub, Groq/OpenAI)
# ============================================================
streaming_request_body = {
    "model": "llama-3.3-70b-versatile",
    "messages": [{"role": "user", "content": "Python haqida qisqa she'r yoz"}],
    "temperature": 0.7,
    "max_tokens": 300,
    "stream": True,  # <- yagona farq non-streaming so'rovdan
}

# ============================================================
# 2) httpx bilan SSE oqimini iste'mol qilish (konseptual, sinovdan
#    o'tkazish uchun haqiqiy provider kaliti kerak)
# ============================================================
import httpx
import json
import asyncio


async def stream_completion(prompt: str, api_key: str) -> None:
    url = "https://api.groq.com/openai/v1/chat/completions"
    body = {
        "model": "llama-3.3-70b-versatile",
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": 300,
        "stream": True,
    }
    full_text = ""
    async with httpx.AsyncClient(timeout=60.0) as client:
        async with client.stream(
            "POST", url,
            headers={"Authorization": f"Bearer {api_key}"},
            json=body,
        ) as resp:
            async for line in resp.aiter_lines():
                if not line.startswith("data: "):
                    continue
                chunk_data = line[len("data: "):]
                if chunk_data.strip() == "[DONE]":
                    break
                chunk = json.loads(chunk_data)
                delta = chunk["choices"][0]["delta"].get("content", "")
                full_text += delta
                print(delta, end="", flush=True)  # har bir bo'lakni darhol ko'rsatish
    print()  # yangi qator
    print("To'liq yig'ilgan matn:", full_text)


# ============================================================
# 3) Non-streaming bilan solishtirish — bu kurs asosan shu naqshni
#    ishlatadi, chunki natija JSON sifatida to'liq kerak
# ============================================================
non_streaming_request_body = {
    "model": "llama-3.3-70b-versatile",
    "messages": [{"role": "user", "content": "..."}],
    "response_format": {"type": "json_object"},  # streaming bilan mos kelmaydi!
    # "stream": True,  <- BUNI YOQMANG: JSON mode + streaming birga
    #                    qiyin ishlaydi, chunki JSON qisman kelganda
    #                    parslab bo'lmaydi.
}
