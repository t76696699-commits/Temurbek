import asyncio
import random


class RetryableError(Exception):
    """429, 500, 502, 503, timeout kabi VAQTINCHALIK xatolar uchun."""


class FatalError(Exception):
    """401, 400, 404 kabi qayta urinish befoyda bo'lgan xatolar uchun."""


async def call_with_backoff(
    fn, *args,
    max_retries: int = 3,
    base_delay: float = 1.0,
    **kwargs,
):
    """Umumiy exponential backoff + jitter naqshi — ushbu platformada
    HALI amalga oshirilmagan, lekin bitta provider bilan ishlaganda
    foydali bo'ladigan umumiy texnika."""
    last_error: Exception | None = None
    for attempt in range(max_retries):
        try:
            return await fn(*args, **kwargs)
        except RetryableError as e:
            last_error = e
            if attempt == max_retries - 1:
                break
            delay = base_delay * (2 ** attempt) + random.uniform(0, 0.5)  # jitter
            print(f"Urinish {attempt + 1} muvaffaqiyatsiz ({e}), {delay:.1f}s kutamiz...")
            await asyncio.sleep(delay)
        except FatalError:
            raise  # qayta urinish befoyda — darhol tashqariga chiqarish

    raise RetryableError(f"{max_retries} urinishdan keyin ham muvaffaqiyatsiz: {last_error}")


# ============================================================
# Ushbu platformaning HAQIQIY yondashuvi — solishtirish uchun
# (_ask_ai ichidagi _call_grok'dan, soddalashtirilgan):
# ============================================================
#
#   if response.status_code == 429:
#       return None   # <- HECH QANDAY kutish, darhol keyingi providerga
#   if response.status_code == 200:
#       return response.json()["choices"][0]["message"]["content"]
#   return None
#
# Ya'ni: 429 shunchaki "bu provider hozir band" signali sifatida
# ishlatiladi, chaqiruvchi kod (_ask_ai) navbatdagi providerni sinaydi.


async def flaky_call(fail_times: list[bool]) -> str:
    """Sinov uchun: dastlab RetryableError beradi, keyin muvaffaqiyatli bo'ladi."""
    if fail_times:
        fail_times.pop(0)
        raise RetryableError("429 Too Many Requests")
    return "muvaffaqiyatli javob"


async def main():
    fail_plan = [True, True]  # dastlabki 2 urinish muvaffaqiyatsiz
    result = await call_with_backoff(flaky_call, fail_plan, max_retries=4)
    print("Yakuniy natija:", result)


if __name__ == "__main__":
    asyncio.run(main())
