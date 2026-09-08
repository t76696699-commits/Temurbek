def build_request(prompt: str) -> dict:
    """
    OpenAI-uslubidagi so'rov lug'atini (dict) shakllantirib beruvchi funksiya.
    """
    return {
        "model": "llama-3.3-70b-versatile",
        "messages": [
            {"role": "system", "content": "Sen yordamchi dasturchisan."},
            {"role": "user", "content": prompt}
        ],
        "temperature": 0.3,
        "max_tokens": 200
    }


def extract_answer(response: dict) -> str:
    """
    OpenAI-uslubidagi javob lug'atidan matnni havfsiz ajratib oluvchi funksiya.
    KeyError xatoligi chiqmasligi uchun try-except ishlatilgan.
    """
    try:
        return response["choices"][0]["message"]["content"]
    except (KeyError, IndexError, TypeError):
        return "Xatolik: Javob lug'atidan matnni ajratib bo'lmadi."


# --- TEKSHIRISH (DEMO) ---

# 1. So'rov qurish
user_prompt = "Python'da ro'yxatni teskari qanday qilaman?"
request_data = build_request(user_prompt)
print("--- Shakllantirilgan so'rov ---")
print(request_data)

# 2. Sun'iy (mock) javob lug'ati
sample_response = {
    "id": "chatcmpl-abc123",
    "choices": [
        {
            "message": {
                "role": "assistant",
                "content": "my_list[::-1] yoki my_list.reverse() ishlatishingiz mumkin."
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {"prompt_tokens": 24, "completion_tokens": 18, "total_tokens": 42}
}

# 3. Javobdan matnni ajratib olish va chiqarish
extracted_text = extract_answer(sample_response)
print("\n--- Ajratib olingan javob matni ---")
print(extracted_text)
