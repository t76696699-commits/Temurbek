# ═══════════════════════════════════════════════════════════════════════
# Redis-based token bucket throttling + graduated ban (aiogram outer middleware)
# ═══════════════════════════════════════════════════════════════════════
import time
from typing import Any, Awaitable, Callable, Dict

from aiogram import BaseMiddleware
from aiogram.types import TelegramObject, Update
from redis.asyncio import Redis

_TOKEN_BUCKET_LUA = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call("HMGET", key, "tokens", "ts")
local tokens = tonumber(bucket[1]) or capacity
local last_ts = tonumber(bucket[2]) or now

local elapsed = now - last_ts
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens < 1 then
  return 0
end

tokens = tokens - 1
redis.call("HMSET", key, "tokens", tokens, "ts", now)
redis.call("EXPIRE", key, 3600)
return 1
"""


class RedisThrottlingMiddleware(BaseMiddleware):
    # Outer middleware — barcha workerlar uchun umumiy Redis orqali
    # token-bucket throttling + bosqichma-bosqich vaqtinchalik ban.

    def __init__(self, redis: Redis, capacity: int = 5, refill_rate: float = 1.0):
        self.redis = redis
        self.capacity = capacity
        self.refill_rate = refill_rate
        self._script = redis.register_script(_TOKEN_BUCKET_LUA)

    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: Update,
        data: Dict[str, Any],
    ) -> Any:
        user = data.get("event_from_user")
        if not user:
            return await handler(event, data)

        ban_key = f"ban:{user.id}"
        if await self.redis.exists(ban_key):
            return None  # jim rad etiladi — bloklangan foydalanuvchiga javob yo'q

        bucket_key = f"bucket:{user.id}"
        allowed = await self._script(
            keys=[bucket_key],
            args=[self.capacity, self.refill_rate, time.time()],
        )

        if not allowed:
            violations_key = f"violations:{user.id}"
            violations = await self.redis.incr(violations_key)
            await self.redis.expire(violations_key, 60)
            if violations >= 3:
                await self.redis.set(ban_key, 1, ex=300)  # 5 daqiqalik ban
            bot = data["bot"]
            chat = data.get("event_chat")
            if chat:
                await bot.send_message(chat.id, "Iltimos, biroz sekinroq yozing.")
            return None

        return await handler(event, data)


# ═══════════════════════════════════════════════════════════════════════
# Botga ulash: ikki alohida worker process bir xil Redis'ga ulanadi
# ═══════════════════════════════════════════════════════════════════════
import asyncio
import os

from aiogram import Bot, Dispatcher
from aiogram.filters import CommandStart
from aiogram.types import Message


async def create_dispatcher() -> Dispatcher:
    redis = Redis.from_url(os.environ["REDIS_URL"], decode_responses=False)
    dp = Dispatcher()
    dp.update.outer_middleware(
        RedisThrottlingMiddleware(redis, capacity=5, refill_rate=1.0)
    )

    @dp.message(CommandStart())
    async def cmd_start(message: Message) -> None:
        await message.answer("Salom! Bu botning barcha workerlari bitta Redis limitini bo'lishadi.")

    return dp


async def main() -> None:
    # WORKER_NAME faqat log/diagnostika uchun — throttling holati Redis'da,
    # shuning uchun WORKER_NAME=worker-1 yoki worker-2 bilan ishga tushirilgan
    # ikki jarayon HAM bitta umumiy limitni ko'radi.
    worker_name = os.environ.get("WORKER_NAME", "worker-1")
    bot = Bot(token=os.environ["BOT_TOKEN"])
    dp = await create_dispatcher()
    print(f"{worker_name} ishga tushdi, Redis orqali umumiy throttling faol")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())


# ═══════════════════════════════════════════════════════════════════════
# pytest: token bucket atomikligini tekshirish (fakeredis bilan)
# ═══════════════════════════════════════════════════════════════════════
import pytest


@pytest.mark.asyncio
async def test_token_bucket_blocks_after_capacity_exhausted():
    import fakeredis.aioredis
    redis = fakeredis.aioredis.FakeRedis()
    middleware = RedisThrottlingMiddleware(redis, capacity=2, refill_rate=0.0)

    calls = []

    async def fake_handler(event, data):
        calls.append(1)
        return "ok"

    class _User:
        id = 555

    class _Chat:
        id = 555

    data = {"event_from_user": _User(), "event_chat": _Chat(), "bot": None}

    # refill_rate=0.0 bo'lgani uchun to'ldirilmaydi: birinchi ikkita so'rov
    # o'tadi (capacity=2), uchinchisi token yo'qligi sababli rad etiladi
    for _ in range(3):
        await middleware(fake_handler, event=None, data=data)
    assert len(calls) == 2
