import asyncio
from datetime import datetime, timedelta, timezone

from aiogram import Bot, Dispatcher, F
from aiogram.exceptions import TelegramForbiddenError
from aiogram.filters import CommandStart
from aiogram.types import (
    Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton
)
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from sqlalchemy import Column, Integer, String, DateTime, Boolean, select
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

# ===== الإعدادات =====
BOT_TOKEN = "8521419777:AAFnaQxSG79sZy-WX8yPdzV2oWrc3f5z6SA"
SCHEDULE_DAYS = [0, 3, 7, 14, 28, 56, 84, 112]  # الأيام من تاريخ الإضافة
REMIND_AGAIN_HOURS = 12      # إذا ما ضغط المستخدم، يعاد التذكير بعد كم ساعة
CHECK_EVERY_SECONDS = 60     # كل كم ثانية يفحص المُجدوِل

# ===== قاعدة البيانات =====
class Base(DeclarativeBase):
    pass

class Memory(Base):
    __tablename__ = "memories"
    id = Column(Integer, primary_key=True)
    chat_id = Column(Integer, index=True, nullable=False)
    content = Column(String, nullable=False)
    start_at = Column(DateTime, nullable=False)          # UTC بدون منطقة (أبسط مع SQLite)
    step = Column(Integer, default=0)                    # رقم الخطوة الحالية في SCHEDULE_DAYS
    next_review_at = Column(DateTime, nullable=False)    # UTC
    active = Column(Boolean, default=True)

engine = create_async_engine("sqlite+aiosqlite:///memories.db")
Session = async_sessionmaker(engine, expire_on_commit=False)

bot = Bot(BOT_TOKEN)
dp = Dispatcher()
scheduler = AsyncIOScheduler()
waiting_for_content: set[int] = set()

def now() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)

def review_keyboard(memory_id: int, step: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="تذكّرته ✅", callback_data=f"ok:{memory_id}:{step}"),
        InlineKeyboardButton(text="ألغِ هذي ❌", callback_data=f"stop:{memory_id}"),
    ]])

# ===== الأوامر =====
@dp.message(CommandStart())
async def start(message: Message):
    await message.answer(
        "أهلاً! اكتب /add ثم المعلومة اللي تبي تسترجعها.\n"
        "مثال: /add عاصمة أستراليا canberra"
    )

@dp.message(F.text.startswith("/add"))
async def add_memory(message: Message):
    content = message.text.removeprefix("/add").strip()
    if not content:
        waiting_for_content.add(message.chat.id)
        await message.answer("اكتب المعلومة التي تريد حفظها او تذكير:")
        return
    await save_memory(message.chat.id, content)

@dp.message(F.text & ~F.text.startswith("/"))
async def free_text(message: Message):
    if message.chat.id in waiting_for_content:
        waiting_for_content.discard(message.chat.id)
        await save_memory(message.chat.id, message.text.strip())

async def save_memory(chat_id: int, content: str):
    async with Session() as s:
        s.add(Memory(
            chat_id=chat_id,
            content=content,
            start_at=now(),
            step=0,
            next_review_at=now(),  # أول تذكير اليوم (يُرسل خلال دقيقة)
        ))
        await s.commit()
    await bot.send_message(chat_id, "تم الحفظ ✅\nأول تذكير بيجيك خلال دقيقة، وبعده حسب الجدول.")

# ===== الأزرار =====
@dp.callback_query(F.data.startswith("ok:"))
async def reviewed(cb: CallbackQuery):
    _, mem_id, step_str = cb.data.split(":")
    mem_id, step = int(mem_id), int(step_str)

    async with Session() as s:
        mem = await s.get(Memory, mem_id)
        if not mem or mem.chat_id != cb.message.chat.id:  # عزل المستخدمين
            await cb.answer("غير موجود")
            return
        if not mem.active:
            await cb.answer("هذي المعلومة متوقفة")
            return
        if mem.step != step:  # زر من تذكير قديم
            await cb.answer("هذا التذكير قديم")
            await cb.message.edit_reply_markup(reply_markup=None)
            return

        if mem.step + 1 < len(SCHEDULE_DAYS):
            mem.step += 1
            mem.next_review_at = mem.start_at + timedelta(days=SCHEDULE_DAYS[mem.step])
        else:
            mem.active = False  # خلصت الخطوات
        await s.commit()

    await cb.answer("تم تسجيل الحفظ ✅")
    await cb.message.edit_reply_markup(reply_markup=None)

@dp.callback_query(F.data.startswith("stop:"))
async def stop(cb: CallbackQuery):
    mem_id = int(cb.data.split(":")[1])
    async with Session() as s:
        mem = await s.get(Memory, mem_id)
        if mem and mem.chat_id == cb.message.chat.id:
            mem.active = False
            await s.commit()
    await cb.answer("تم الإلغاء")
    await cb.message.edit_reply_markup(reply_markup=None)

# ===== المُجدوِل =====
async def send_due_reviews():
    async with Session() as s:
        rows = (await s.execute(
            select(Memory).where(Memory.active == True, Memory.next_review_at <= now())
        )).scalars().all()

        for mem in rows:
            try:
                await bot.send_message(
                    mem.chat_id,
                    f"🔔 تذكير:\n{mem.content}",
                    reply_markup=review_keyboard(mem.id, mem.step),
                )
                mem.next_review_at = now() + timedelta(hours=REMIND_AGAIN_HOURS)
            except TelegramForbiddenError:
                mem.active = False  # المستخدم حظر البوت، نوقف تذكيراته
            except Exception as e:
                print(f"فشل الإرسال لـ {mem.chat_id}: {e}")
                mem.next_review_at = now() + timedelta(minutes=10)  # نعيد المحاولة بعد 10 دقائق
            await s.commit()  # نحفظ بعد كل تذكير

# ===== التشغيل =====
async def main():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    scheduler.add_job(send_due_reviews, "interval", seconds=CHECK_EVERY_SECONDS)
    scheduler.start()
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
