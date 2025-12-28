import os
import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackContext, CallbackQueryHandler

# BOT TOKEN (yangi token)
BOT_TOKEN = "8500456022:AAE-EWeJF3-EtR_cYaCUD7hDejaHfrUoQ_g"

# Sizning Telegram ID (admin)
ADMIN_ID = 8435791459

# Foydalanuvchi ma'lumotlari
users = {}
referrals = {}

# Hayvonlar va o'simliklar
animals = {
    "🐱 Mushuk": {"price": 70, "income": 0.7, "gift_lvl":5},
    "🐶 It": {"price": 80, "income": 0.8, "gift_lvl":10},
    "🐔 Tovuq": {"price": 90, "income": 0.9, "gift_lvl":15},
    "🐑 Quy": {"price": 100, "income": 1, "gift_lvl":20},
    "🐐 Echki": {"price": 100, "income": 1, "gift_lvl":25},
    "🐄 Mol": {"price": 150, "income": 1.5, "gift_lvl":30},
    "🐎 Ot": {"price": 200, "income": 2, "gift_lvl":35}
}

plants = {
    "🍎 Olma": {"price": 50, "income": 1},
    "🍐 Nok": {"price": 60, "income": 1.2},
    "🍇 Uzum": {"price": 70, "income": 1.4},
    "🍎 Anor": {"price": 80, "income": 1.6}  # Stekker bilan ifodalangan
}

levels = [
    (0, 100),
    (100, 500),
    (500, 1000),
    (1000, 1800),
    (1800, 3000)
]

# /start buyrug'i
async def start(update: Update, context: CallbackContext.DEFAULT_TYPE):
    user_id = update.effective_user.id
    args = context.args
    ref_id = None
    if args:
        try:
            ref_id = int(args[0])
        except:
            pass

    if user_id not in users:
        users[user_id] = {
            "balance": 100,
            "animals": {},
            "plants": {},
            "income_time": datetime.datetime.now(),
            "level": 1,
            "referral_income":0
        }
        await update.message.reply_text(f"Salom {update.effective_user.first_name}! Sizga start bonusi sifatida $100 berildi.")
        if ref_id and ref_id in users:
            users[ref_id]["balance"] += 100
            await update.message.reply_text(f"Do'stingiz sizning referral link orqali start berdi, sizga $100 bonus berildi!")
            referrals[user_id] = ref_id
    else:
        await update.message.reply_text("Siz allaqachon ro'yxatdan o'tgansiz!")

# /balance buyrug'i
async def balance(update: Update, context: CallbackContext.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user = users.get(user_id)
    if user:
        await update.message.reply_text(f"Sizning balansingiz: ${user['balance']:.2f}")
    else:
        await update.message.reply_text("Siz start qilmagansiz, /start buyrug'ini bosing.")

# /shop buyrug'i
async def shop(update: Update, context: CallbackContext.DEFAULT_TYPE):
    keyboard = []
    for name, data in animals.items():
        keyboard.append([InlineKeyboardButton(f"{name} - ${data['price']}", callback_data=f"buy_animal_{name}")])
    for name, data in plants.items():
        keyboard.append([InlineKeyboardButton(f"{name} - ${data['price']}", callback_data=f"buy_plant_{name}")])
    await update.message.reply_text("Nimani sotib olasiz?", reply_markup=InlineKeyboardMarkup(keyboard))

# Xarid callback
async def buy_callback(update: Update, context: CallbackContext.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    user = users[user_id]

    data = query.data
    if data.startswith("buy_animal_"):
        name = data.replace("buy_animal_", "")
        price = animals[name]["price"]
        if user["balance"] >= price:
            user["balance"] -= price
            user["animals"][name] = user["animals"].get(name, 0) + 1
            await query.edit_message_text(f"Siz {name} sotib oldingiz!")
        else:
            await query.edit_message_text("Balansingiz yetarli emas!")
    elif data.startswith("buy_plant_"):
        name = data.replace("buy_plant_", "")
        price = plants[name]["price"]
        if user["balance"] >= price:
            user["balance"] -= price
            user["plants"][name] = user["plants"].get(name, 0) + 1
            await query.edit_message_text(f"Siz {name} sotib oldingiz!")
        else:
            await query.edit_message_text("Balansingiz yetarli emas!")

# /collect buyrug'i
async def collect_income(update: Update, context: CallbackContext.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user = users[user_id]
    now = datetime.datetime.now()
    hours = (now - user["income_time"]).seconds // 3600
    if hours > 0:
        total_income = 0
        for name, count in user["animals"].items():
            total_income += animals[name]["income"] * count * hours
        for name, count in user["plants"].items():
            total_income += plants[name]["income"] * count * hours
        user["balance"] += total_income
        user["income_time"] = now
        await update.message.reply_text(f"Siz {hours} soat ichida ${total_income:.2f} topdingiz! Balansingiz: ${user['balance']:.2f}")

        # Levelni yangilash va sovg'a hayvon
        for i, (min_bal, max_bal) in enumerate(levels, start=1):
            if min_bal <= user["balance"] <= max_bal:
                if user["level"] < i:
                    user["level"] = i
                    for name, data in animals.items():
                        if data.get("gift_lvl") == i:
                            user["animals"][name] = user["animals"].get(name, 0) + 1
                            await update.message.reply_text(f"Tabrik! Siz {i}-lvl ga yetdingiz va sovg‘a hayvon oldingiz: {name}")
                break
    else:
        await update.message.reply_text("Daromadni olish uchun kamida 1 soat kuting.")

# Bot ishga tushirish
app = ApplicationBuilder().token(BOT_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("balance", balance))
app.add_handler(CommandHandler("shop", shop))
app.add_handler(CommandHandler("collect", collect_income))
app.add_handler(CallbackQueryHandler(buy_callback))

print(f"Bot ishga tushdi! Admin ID: {ADMIN_ID}")
app.run_polling()
