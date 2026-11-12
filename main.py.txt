main.py

import asyncio
import logging
from aiogram import Bot, Dispatcher, F, types
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup

# Tokeningiz va shaxsiy Telegram ID raqamingiz
TOKEN = "9161191790:AAE0D_04RrfYSUOQ-sZc8aRbIkjCBBLAoWo"
ADMIN_ID = 7386594868  

bot = Bot(token=TOKEN)
dp = Dispatcher()

# Holatlar (FSM)
class AlmazOrder(StatesGroup):
    waiting_for_game_id = State()
    waiting_for_amount = State()
    waiting_for_receipt = State()

# 1. /start buyrug'i
@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text=" Almaz sotib olish", callback_data="buy_almaz")]
        ]
    )
    await message.answer(
        "Assalomu alaykum! Almaz sotib olish botiga xush kelibsiz.\n"
        "Buyurtma berish uchun quyidagi tugmani bosing:",
        reply_markup=keyboard
    )

# 2. Tugma bosilganda boshlanishi
@dp.callback_query(F.data == "buy_almaz")
async def process_buy(callback: types.CallbackQuery, state: FSMContext):
    await callback.message.answer("O'yin ID raqamingizni kiriting (masalan: 12345678):")
    await state.set_state(AlmazOrder.waiting_for_game_id)
    await callback.answer()

# 3. ID raqamini qabul qilish
@dp.message(AlmazOrder.waiting_for_game_id)
async def process_game_id(message: types.Message, state: FSMContext):
    await state.update_data(game_id=message.text)
    
    # Aniq narxlar
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="100 Almaz - 11,200 so'm", callback_data="amt_100")],
            [InlineKeyboardButton(text="210 Almaz - 22,300 so'm", callback_data="amt_210")],
            [InlineKeyboardButton(text="310 Almaz - 34,000 so'm", callback_data="amt_310")]
        ]
    )
    await message.answer("Kerakli almaz miqdorini tanlang:", reply_markup=keyboard)
    await state.set_state(AlmazOrder.waiting_for_amount)

# 4. Miqdorni qabul qilish va karta raqamini chiqarish
@dp.callback_query(AlmazOrder.waiting_for_amount, F.data.startswith("amt_"))
async def process_amount(callback: types.CallbackQuery, state: FSMContext):
    amount = callback.data.split("_")[1]
    await state.update_data(amount=amount)
    
    # Karta raqamingiz
    card_number = "4916 9903 6151 8795"
    
    await callback.message.answer(
        f"Tanlangan almaz: **{amount} ta**\n\n"
        f"To'lov uchun karta: `{card_number}`\n"
        f"Iltimos, pulni o'tkazib, to'lov chekini (skrinshot) shu botga yuboring.",
        parse_mode="Markdown"
    )
    await state.set_state(AlmazOrder.waiting_for_receipt)
    await callback.answer()

# 5. Chekni qabul qilish va adminga yuborish
@dp.message(AlmazOrder.waiting_for_receipt, F.photo)
async def process_receipt(message: types.Message, state: FSMContext):
    data = await state.get_data()
    game_id = data.get("game_id")
    amount = data.get("amount")
    
    user_info = f"Foydalanuvchi: @{message.from_user.username} (ID: {message.from_user.id})"
    order_text = (
        f" **Yangi buyurtma!**\n\n"
        f"{user_info}\n"
        f"O'yin ID: `{game_id}`\n"
        f"Miqdori:  {amount} almaz"
    )
    
    # Adminga chek va ma'lumotni yuborish
    photo_id = message.photo[-1].file_id
    await bot.send_photo(chat_id=ADMIN_ID, photo=photo_id, caption=order_text, parse_mode="Markdown")
    
    await message.answer("✅ Buyurtmangiz qabul qilindi! Admin tekshirgandan so'ng almaz o'yiningizga tashlab beriladi.")
    await state.clear()

async def main():
    logging.basicConfig(level=logging.INFO)
    # Eski webhookni o'chirib tashlash uchun quyidagi qator qo'shildi:
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())

