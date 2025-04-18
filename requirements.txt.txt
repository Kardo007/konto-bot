import logging
import sqlite3
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackContext, CallbackQueryHandler

# 🔧 Настройки
TOKEN = "ТВОЙ_ТОКЕН_ОТ_BOTFATHER"

# Настрой логирование
logging.basicConfig(level=logging.INFO)

# 📂 Инициализация базы данных
def init_db():
    conn = sqlite3.connect("database.db")
    cur = conn.cursor()
    cur.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY,
        username TEXT,
        balance INTEGER DEFAULT 0,
        last_click TEXT
    )
    """)
    conn.commit()
    conn.close()

# 🎯 Команда /start
async def start(update: Update, context: CallbackContext):
    user = update.effective_user
    conn = sqlite3.connect("database.db")
    cur = conn.cursor()
    cur.execute("SELECT * FROM users WHERE id = ?", (user.id,))
    if not cur.fetchone():
        cur.execute("INSERT INTO users (id, username) VALUES (?, ?)", (user.id, user.username or "no_username"))
        conn.commit()
    conn.close()

    keyboard = [[InlineKeyboardButton("👆 Получить KONTO", callback_data="click")]]
    await update.message.reply_text("Привет! Жми кнопку раз в сутки и получай KONTO 🎉", reply_markup=InlineKeyboardMarkup(keyboard))

# ⏱ Нажатие на кнопку
async def click(update: Update, context: CallbackContext):
    query = update.callback_query
    user_id = query.from_user.id
    conn = sqlite3.connect("database.db")
    cur = conn.cursor()
    cur.execute("SELECT balance, last_click FROM users WHERE id = ?", (user_id,))
    row = cur.fetchone()

    now = datetime.utcnow()
    if row and row[1]:
        last_click = datetime.strptime(row[1], "%Y-%m-%d %H:%M:%S")
        if now - last_click < timedelta(hours=24):
            remaining = timedelta(hours=24) - (now - last_click)
            await query.answer(f"Подожди {remaining.seconds // 3600} ч {remaining.seconds % 3600 // 60} мин ⏳", show_alert=True)
            conn.close()
            return

    # Обновляем баланс
    cur.execute("UPDATE users SET balance = balance + 1, last_click = ? WHERE id = ?", (now.strftime("%Y-%m-%d %H:%M:%S"), user_id))
    conn.commit()
    conn.close()

    await query.answer("🎉 Ты получил 1 KONTO!")
    await query.edit_message_reply_markup(reply_markup=None)

# 🏆 Топ 15 пользователей
async def top(update: Update, context: CallbackContext):
    conn = sqlite3.connect("database.db")
    cur = conn.cursor()
    cur.execute("SELECT username, balance FROM users ORDER BY balance DESC LIMIT 15")
    top_users = cur.fetchall()
    conn.close()

    message = "🏆 Топ получателей KONTO:\n\n"
    for i, (username, balance) in enumerate(top_users, 1):
        message += f"{i}. @{username} — {balance} KONTO\n"

    await update.message.reply_text(message)

# 🚀 Запуск
def main():
    init_db()
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("top", top))
    app.add_handler(CallbackQueryHandler(click))
    app.run_polling()

if __name__ == "__main__":
    main()
