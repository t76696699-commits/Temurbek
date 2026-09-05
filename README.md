"""Graceful shutdown: SIGTERM'ni tutish, joriy vazifalarni tugatish,
resurslarni tozalab yopish. aiohttp-webhook misolida health-check bilan.
"""
from __future__ import annotations

import asyncio
import logging
import signal

from aiohttp import web
from aiogram import Bot, Dispatcher

logger = logging.getLogger("shutdown")

DRAIN_TIMEOUT_SECONDS = 30
_in_flight: set[asyncio.Task] = set()
_ready = True   # readiness probe holati


def track(coro) -> asyncio.Task:
    """Har bir handler task'ini ro'yxatga oladi — shutdown paytida
    qaysilari hali tugallanmaganini bilish uchun."""
    task = asyncio.create_task(coro)
    _in_flight.add(task)
    task.add_done_callback(_in_flight.discard)
    return task


async def readiness_probe(request: web.Request) -> web.Response:
    # Orkestrator shutdown boshlanganda buni "not ready" deb o'qib,
    # yangi trafikni boshqa instansga yo'naltiradi.
    if _ready:
        return web.Response(status=200, text="ready")
    return web.Response(status=503, text="draining")


async def liveness_probe(request: web.Request) -> web.Response:
    # Joriy so'rovlar tugagunga qadar "ha" qaytadi — jarayon hali
    # osilib qolmagan, faqat yangi ish qabul qilmayapti.
    return web.Response(status=200, text="alive")


async def graceful_shutdown(bot: Bot) -> None:
    global _ready
    logger.info("SIGTERM qabul qilindi — yangi so'rovlarni to'xtatish")
    _ready = False   # 1-qadam: yangi ishni qabul qilishni to'xtatish

    if _in_flight:
        logger.info("joriy %d ta vazifa tugashi kutilmoqda (timeout=%ss)",
                     len(_in_flight), DRAIN_TIMEOUT_SECONDS)
        done, pending = await asyncio.wait(
            _in_flight, timeout=DRAIN_TIMEOUT_SECONDS
        )
        for task in pending:
            logger.warning("vazifa %s belgilangan vaqtda tugamadi — bekor qilinmoqda", task)
            task.cancel()

    await bot.session.close()   # 3-qadam: resurslarni yopish
    logger.info("bot sessiyasi yopildi, jarayon chiqadi")


def install_signal_handlers(bot: Bot) -> None:
    loop = asyncio.get_running_loop()
    stop_event = asyncio.Event()

    def _handle_signal() -> None:
        stop_event.set()

    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, _handle_signal)

    async def _waiter() -> None:
        await stop_event.wait()
        await graceful_shutdown(bot)

    loop.create_task(_waiter())


async def main() -> None:
    bot = Bot(token="BOT_TOKEN")
    dp = Dispatcher()
    install_signal_handlers(bot)

    app = web.Application()
    app.router.add_get("/health/ready", readiness_probe)
    app.router.add_get("/health/live", liveness_probe)

    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, "0.0.0.0", 8080)
    await site.start()

    await dp.start_polling(bot, handle_signals=False)  # o'z signal handler'imiz bor


if __name__ == "__main__":
    asyncio.run(main())
