"""Capstone loyiha skeleti: barcha qismlarni bog'lovchi entrypoint.

bot.py — RedisStorage, Payments handlerlari, i18n/logging middleware'lari
va graceful shutdown'ni bitta joyda ulaydi. Har bir qism avvalgi
darslarda alohida chuqur o'rganilgan — bu yerda faqat ULASH ko'rsatilgan.
"""
from __future__ import annotations

import asyncio
import logging
import signal

import structlog
from aiogram import Bot, Dispatcher, Router
from aiogram.fsm.storage.redis import RedisStorage
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

# --- 4-dars: Redis FSM storage --------------------------------------------
storage = RedisStorage.from_url("redis://localhost:6379/0")

# --- 7-dars: strukturaviy logging -----------------------------------------
logger = structlog.get_logger("capstone_bot")

# --- 8-dars: rate limiting va 9-dars: middleware zanjiri -------------------
from rate_limit_middleware import RedisRateLimitMiddleware   # 8-darsdan
from logging_middleware import RequestContextMiddleware      # 7-darsdan
from i18n_middleware import I18nMiddleware                   # 11-darsdan

# --- 0-1-darslar: Mini App + initData validatsiyasi ------------------------
from mini_app_routes import mini_app_router                  # 0-1-darslardan

# --- 2-3-darslar: real Payments ---------------------------------------------
from payments_router import payments_router                  # 2-3-darslardan

# --- 12-dars: graceful shutdown ---------------------------------------------
from graceful_shutdown import install_signal_handlers


def build_dispatcher(db, redis_client) -> Dispatcher:
    dp = Dispatcher(storage=storage)

    # Outer middleware — HAR bir update uchun, tartib muhim (9-dars):
    # avval kontekst/log, keyin locale, keyin rate-limit.
    dp.update.outer_middleware(RequestContextMiddleware())
    dp.update.outer_middleware(I18nMiddleware(db=db))
    dp.update.outer_middleware(RedisRateLimitMiddleware(redis=redis_client))

    dp.include_router(mini_app_router)
    dp.include_router(payments_router)
    return dp


async def main() -> None:
    bot = Bot(
        token="BOT_TOKEN",
        default=DefaultBotProperties(parse_mode=ParseMode.HTML),
    )

    import redis.asyncio as redis
    redis_client = redis.from_url("redis://localhost:6379/1")
    db = None  # loyihaning haqiqiy DB ulanish obyekti

    dp = build_dispatcher(db, redis_client)
    install_signal_handlers(bot)   # 12-dars: SIGTERM -> drain -> yopish

    logger.info("capstone_bot_started")
    await dp.start_polling(bot, handle_signals=False)


if __name__ == "__main__":
    asyncio.run(main())


# --- 5-dars: capstone uchun minimal test misoli ----------------------------
# To'liq testlash 5-darsda o'rgangan pytest+AsyncMock naqshiga tayanadi;
# bu yerda faqat ulash to'g'riligini tekshiruvchi bitta misol keltirilgan.
import pytest
from unittest.mock import AsyncMock


@pytest.mark.asyncio
async def test_build_dispatcher_includes_both_routers() -> None:
    fake_redis = AsyncMock()
    dp = build_dispatcher(db=None, redis_client=fake_redis)

    included_routers = {r.name for r in dp.sub_routers}
    assert "mini_app" in included_routers
    assert "payments" in included_routers


@pytest.mark.asyncio
async def test_graceful_shutdown_closes_bot_session() -> None:
    fake_bot = AsyncMock()
    from graceful_shutdown import graceful_shutdown

    await graceful_shutdown(fake_bot)

    fake_bot.session.close.assert_awaited_once()
