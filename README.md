import logging
from aiogram import Bot, Dispatcher, executor, types
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup

API_TOKEN = "ТОКЕН_БОТА"
ADMIN_ID = 8213203739 # твой Telegram ID

logging.basicConfig(level=logging.INFO)

bot = Bot(token='8559874170:AAG6LK4kTruqb95HXlgSKGolSEg', parse_mode="HTML")
dp = Dispatcher(bot, storage=MemoryStorage())

# ===== Хранилище каналов =====
# формат: [{"name": "Канал 1", "url": "https://t.me/example"}]
channels = []

# ===== FSM =====
class AdminState(StatesGroup):
    waiting_for_channel = State()
    waiting_for_replace = State()

# ===== Клавиатуры =====
def subscribe_keyboard():
    kb = InlineKeyboardMarkup(row_width=1)
    for ch in channels:
        kb.add(
            InlineKeyboardButton(
                text=ch["name"],
                url=ch["url"]
            )
        )
    kb.add(InlineKeyboardButton("✅ Я подписался", callback_data="continue"))
    return kb

def admin_keyboard():
    kb = InlineKeyboardMarkup(row_width=1)
    kb.add(
        InlineKeyboardButton("➕ Добавить канал", callback_data="add"),
        InlineKeyboardButton("🔁 Заменить все", callback_data="replace"),
        InlineKeyboardButton("📋 Список", callback_data="list"),
    )
    return kb

# ===== /start =====
@dp.message_handler(commands=["start"])
async def start(message: types.Message):
    if not channels:
        await message.answer("⚠️ Каналы ещё не добавлены")
        return
    await message.answer(
        "📢 Подпишись на каналы и нажми кнопку ниже:",
        reply_markup=subscribe_keyboard()
    )

# ===== Кнопка продолжить =====
@dp.callback_query_handler(text="continue")
async def continue_handler(callback: types.CallbackQuery):
    await callback.message.edit_text("🎉 Спасибо! Доступ открыт.")

# ===== Админка =====
@dp.message_handler(commands=["admin"])
async def admin(message: types.Message):
    if message.from_user.id != ADMIN_ID:
        return
    await message.answer("⚙️ Админ-панель", reply_markup=admin_keyboard())

@dp.callback_query_handler(text="add")
async def add_channel(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        return
    await callback.message.answer(
        "Отправь данные канала в формате:\n"
        "<b>Название | ссылка</b>\n\n"
        "Пример:\nКанал 1 | https://t.me/example"
    )
    await AdminState.waiting_for_channel.set()

@dp.message_handler(state=AdminState.waiting_for_channel)
async def save_channel(message: types.Message, state: FSMContext):
    try:
        name, url = map(str.strip, message.text.split("|"))
        channels.append({"name": name, "url": url})
        await message.answer("✅ Канал добавлен")
        await state.finish()
    except:
        await message.answer("❌ Ошибка формата")

@dp.callback_query_handler(text="replace")
async def replace(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        return
    await callback.message.answer(
        "Отправь все каналы, каждый с новой строки:\n\n"
        "Канал 1 | https://t.me/example1\n"
        "Канал 2 | https://t.me/example2"
    )
    await AdminState.waiting_for_replace.set()

@dp.message_handler(state=AdminState.waiting_for_replace)
async def replace_all(message: types.Message, state: FSMContext):
    global channels
    new_channels = []
    for line in message.text.splitlines():
        if "|" in line:
            name, url = map(str.strip, line.split("|"))
            new_channels.append({"name": name, "url": url})
    channels = new_channels
    await message.answer("🔁 Каналы заменены")
    await state.finish()

@dp.callback_query_handler(text="list")
async def list_channels(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        return
    if not channels:
        await callback.message.answer("Список пуст")
        return
    text = "📋 Каналы:\n"
    for i, ch in enumerate(channels, 1):
        text += f"{i}. {ch['name']} — {ch['url']}\n"
    await callback.message.answer(text)

# ===== Запуск =====
if __name__ == "__main__":
    executor.start_polling(dp, skip_updates=True)
