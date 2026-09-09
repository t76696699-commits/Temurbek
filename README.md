# ============================================================
# Konseptual misol — OpenAI-uslub tools sxemasi (umumiy shakl,
# har doim provider'ning JORIY hujjatini tekshiring)
# ============================================================

tools_schema = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Berilgan shahar uchun joriy ob-havoni qaytaradi",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "Shahar nomi"},
                },
                "required": ["city"],
            },
        },
    }
]

request_with_tools = {
    "model": "llama-3.3-70b-versatile",
    "messages": [{"role": "user", "content": "Toshkentda hozir ob-havo qanday?"}],
    "tools": tools_schema,
}

# ============================================================
# Model "funksiya chaqirishni so'rayotgan" javob shakli (konseptual)
# ============================================================
model_wants_tool_call = {
    "choices": [{
        "message": {
            "role": "assistant",
            "tool_calls": [{
                "id": "call_abc123",
                "function": {"name": "get_weather", "arguments": '{"city": "Toshkent"}'},
            }],
        },
    }],
}

# ============================================================
# HAQIQIY bajarish — SIZNING kodingizda, faqat ro'yxatga olingan
# funksiyalarni ishga tushiring (xavfsizlik uchun muhim!)
# ============================================================
import json


def get_weather(city: str) -> dict:
    # Bu yerda haqiqiy ob-havo API'siga so'rov bo'lardi.
    return {"city": city, "temp": 28, "condition": "quyoshli"}


_ALLOWED_TOOLS = {"get_weather": get_weather}  # oq ro'yxat — faqat shular bajariladi


def execute_tool_call(tool_call: dict) -> dict:
    name = tool_call["function"]["name"]
    if name not in _ALLOWED_TOOLS:
        raise ValueError(f"Ruxsat etilmagan funksiya: {name}")
    args = json.loads(tool_call["function"]["arguments"])
    return _ALLOWED_TOOLS[name](**args)


tool_call = model_wants_tool_call["choices"][0]["message"]["tool_calls"][0]
result = execute_tool_call(tool_call)
print("Funksiya natijasi:", result)

# Natijani modelga qaytarish uchun xabar shakli:
tool_result_message = {
    "role": "tool",
    "tool_call_id": tool_call["id"],
    "content": json.dumps(result, ensure_ascii=False),
}
