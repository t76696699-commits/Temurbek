"""i18n middleware: foydalanuvchi tilini aniqlash va handlerlarga uzatish.

Haqiqiy loyihada tarjima .mo fayllaridan gettext orqali o'qiladi;
bu yerda tushunarli bo'lishi uchun kichik in-memory lug'at ishlatilgan,
lekin _()/ngettext() interfeysi haqiqiy gettext bilan bir xil.
"""
from __future__ import annotations

import gettext
from pathlib import Path
from typing import Any, Awaitable, Callable

from aiogram import BaseMiddleware
from aiogram.types import TelegramObject, User

LOCALES_DIR = Path(__file__).parent / "locales"
DEFAULT_LOCALE = "uz"
SUPPORTED_LOCALES = ("uz", "ru")

# Har bir til uchun oldindan compile qilingan .mo asosida GNUTranslations
# obyektini keshda saqlaymiz — har update uchun diskdan qayta o'qimaslik uchun.
_translations: dict[str, gettext.NullTranslations] = {}
for lang in SUPPORTED_LOCALES:
    try:
        _translations[lang] = gettext.translation(
            "messages", localedir=LOCALES_DIR, languages=[lang]
        )
    except FileNotFoundError:
        _translations[lang] = gettext.NullTranslations()


async def get_saved_locale(user_id: int, db) -> str | None:
    """Foydalanuvchi oldin tanlagan tilni DB'dan o'qiydi (mavjud bo'lsa)."""
    row = await db.fetch_one(
        "SELECT locale FROM user_preferences WHERE user_id = :uid", {"uid": user_id}
    )
    return row["locale"] if row else None


class I18nMiddleware(BaseMiddleware):
    """Outer middleware — har bir update uchun locale'ni aniqlab, handlerga
    tarjima funksiyasini ('_' va 'ngettext') data orqali uzatadi."""

    def __init__(self, db) -> None:
        self.db = db

    async def __call__(
        self,
        handler: Callable[[TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: dict[str, Any],
    ) -> Any:
        user: User | None = data.get("event_from_user")
        locale = DEFAULT_LOCALE
        if user is not None:
            saved = await get_saved_locale(user.id, self.db)
            if saved in SUPPORTED_LOCALES:
                locale = saved
            elif user.language_code in SUPPORTED_LOCALES:
                locale = user.language_code

        translation = _translations.get(locale, _translations[DEFAULT_LOCALE])
        data["locale"] = locale
        data["_"] = translation.gettext
        data["ngettext"] = translation.ngettext
        return await handler(event, data)


# ── Handler misoli: middleware kiritgan '_' va 'ngettext'dan foydalanish ──
from aiogram import Router
from aiogram.filters import Command
from aiogram.types import Message

router = Router()


@router.message(Command("kurslar"))
async def show_courses_count(message: Message, _: Callable[[str], str],
                              ngettext: Callable[[str, str, int], str]) -> None:
    count = await get_active_courses_count()
    # ngettext o'zi son bo'yicha to'g'ri shaklni tanlaydi — dasturchi
    # if count == 1 deb yozmaydi.
    text = ngettext(
        "Sizda {n} ta faol kurs bor.",
        "Sizda {n} ta faol kurs bor.",
        count,
    ).format(n=count)
    await message.answer(_( "Ma'lumot: " ) + text)


async def get_active_courses_count() -> int:
    return 3
