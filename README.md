"""Multi-bot (bot-farm) polling launcher.

Bitta kod bazasi, N ta mustaqil Bot obyekti, bitta umumiy Dispatcher.
Har bir tenant bot_registry jadvalidan o'qiladi; kodni o'zgartirmasdan
yangi qator qo'shish orqali yangi mijoz ulanadi.
"""
import asyncio
import logging
from dataclasses import dataclass

from aiogram import Bot, Dispatcher, Router
from aiogram.filters import CommandStart
from aiogram.types import Message
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

logger = logging.getLogger("bot_farm")


@dataclass(frozen=True)
class TenantConfig:
    bot_id: int
    token: str
    tenant_name: str


async def load_active_tenants(db) -> list[TenantConfig]:
    """bot_registry jadvalidan faol tenantlarni o'qiydi."""
    rows = await db.fetch_all(
        "SELECT bot_id, token, tenant_name FROM bot_registry WHERE is_active = true"
    )
    return [TenantConfig(r["bot_id"], r["token"], r["tenant_name"]) for r in rows]


router = Router(name="shared-handlers")


@router.message(CommandStart())
async def cmd_start(message: Message, bot: Bot) -> None:
    # bot.id — aiogram avtomatik to'ldiradi (get_me orqali), shu yerdan
    # qaysi tenant ekanini bilib olamiz, keyingi so'rovlarda bot_id
    # sifatida ishlatamiz.
    me = await bot.get_me()
    await message.answer(
        f"Salom! Siz @{me.username} boti bilan gaplashyapsiz "
        f"(bot_id={bot.id})."
    )


async def run_one_bot(dp: Dispatcher, bot: Bot, tenant: TenantConfig) -> None:
    """Bitta tenant uchun polling tsikli — xato boshqalarni to'xtatmaydi."""
    try:
        logger.info("tenant %s (bot_id=%s) polling boshlandi", tenant.tenant_name, tenant.bot_id)
        await dp.start_polling(bot, handle_signals=False)
    except Exception:
        logger.exception("tenant %s (bot_id=%s) polling'da xato", tenant.tenant_name, tenant.bot_id)
    finally:
        await bot.session.close()


async def main(db) -> None:
    dp = Dispatcher()
    dp.include_router(router)

    tenants = await load_active_tenants(db)
    if not tenants:
        logger.warning("faol tenant topilmadi")
        return

    bots = {
        t.bot_id: Bot(
            token=t.token,
            default=DefaultBotProperties(parse_mode=ParseMode.HTML),
        )
        for t in tenants
    }

    await asyncio.gather(
        *(run_one_bot(dp, bots[t.bot_id], t) for t in tenants)
    )


# ── Webhook rejimi uchun yo'naltiruvchi (aiohttp misolida) ──────────────
from aiohttp import web
from aiogram.types import Update

BOTS_REGISTRY: dict[int, Bot] = {}   # ishga tushishda load_active_tenants'dan to'ldiriladi
DISPATCHER: Dispatcher | None = None


async def webhook_handler(request: web.Request) -> web.Response:
    bot_id = int(request.match_info["bot_id"])
    bot = BOTS_REGISTRY.get(bot_id)
    if bot is None:
        # Mavjud bo'lmagan yoki o'chirilgan tenant — 404, xato jim yutilmaydi.
        return web.Response(status=404, text="unknown bot_id")

    data = await request.json()
    update = Update.model_validate(data)
    assert DISPATCHER is not None
    await DISPATCHER.feed_update(bot=bot, update=update)
    return web.Response(status=200)


def build_webhook_app() -> web.Application:
    app = web.Application()
    app.router.add_post("/webhook/{bot_id}", webhook_handler)
    return app


if __name__ == "__main__":
    # db — loyihangizning haqiqiy DB ulanish obyekti bilan almashtiriladi
    asyncio.run(main(db=None))
