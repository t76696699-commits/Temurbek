from __future__ import annotations
import httpx


class ProviderError(Exception):
    pass


def failure_review(error_code: str, message: str) -> dict:
    """grok_review.py dagi _failure_review bilan bir xil g'oya."""
    return {
        "grade": "F",
        "points": 0,
        "feedback": message,
        "error": error_code,
        "provider": None,
    }


async def analyze_with_soft_failure(call_chain_fn, prompt: str) -> dict:
    """'Yumshoq' xato — fon vazifasi/avtomatik oqim uchun mos."""
    try:
        text, parsed, provider, attempts = await call_chain_fn(prompt)
        parsed["provider"] = provider
        return parsed
    except ProviderError as e:
        return failure_review("all_providers_failed", f"AI baholash muvaffaqiyatsiz: {e}")


async def analyze_with_hard_failure(call_chain_fn, prompt: str) -> dict:
    """'Qattiq' xato — foydalanuvchi kutayotgan endpoint uchun mos
    (ai_review.py'dagi raise_on_error=True bilan bir xil g'oya)."""
    try:
        text, parsed, provider, attempts = await call_chain_fn(prompt)
        parsed["provider"] = provider
        return parsed
    except ProviderError as e:
        # Bu yerda real kodda FastAPI'ning HTTPException ko'tariladi;
        # bu darsda faqat g'oyani ko'rsatish uchun oddiy Exception ishlatamiz.
        raise RuntimeError(f"502 Bad Gateway: barcha AI provider ishlamadi ({e})")


# ============================================================
# call_chain ichidagi uch xil istisno turi (haqiqiy tuzilish)
# ============================================================
async def call_chain_error_handling_demo(caller, prompt: str, max_tokens: int) -> str:
    attempts: list[str] = []
    try:
        return await caller(prompt, max_tokens)
    except ProviderError as e:
        attempts.append(f"provider_error: {e}")
    except httpx.TimeoutException:
        attempts.append("timeout: so'rov belgilangan vaqtda javob bermadi")
    except httpx.HTTPError as e:
        attempts.append(f"http_error: {type(e).__name__}: {e}")
    except Exception as e:
        # Kutilmagan xato ham dasturni portlatmasin.
        attempts.append(f"unexpected: {type(e).__name__}")
    raise ProviderError("; ".join(attempts))
