# === Required: Install with `pip install python-telegram-bot==20.0b1` or latest ===

import sqlite3
import asyncio
from telegram import Update
from telegram.ext import (ApplicationBuilder, CommandHandler, MessageHandler,
                          ContextTypes, filters)

# === CONFIG ===
ADMIN_USERNAME = "earnsep"
REFERRAL_REWARD = 5
MIN_WITHDRAW = 20

# === DATABASE SETUP ===
conn = sqlite3.connect("users.db", check_same_thread=False)
c = conn.cursor()
c.execute('''CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username TEXT, balance INTEGER, referred_by INTEGER)''')
c.execute('''CREATE TABLE IF NOT EXISTS withdrawals (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, upi TEXT, amount INTEGER, status TEXT)''')
conn.commit()

# === HELPERS ===
def get_user(user_id):
    c.execute("SELECT * FROM users WHERE id=?", (user_id,))
    return c.fetchone()

def add_user(user_id, username, referred_by):
    if not get_user(user_id):
        c.execute("INSERT INTO users (id, username, balance, referred_by) VALUES (?, ?, ?, ?)", (user_id, username, 0, referred_by))
        conn.commit()
        if referred_by:
            ref_user = get_user(referred_by)
            if ref_user:
                c.execute("UPDATE users SET balance = balance + ? WHERE id = ?", (REFERRAL_REWARD, referred_by))
                conn.commit()

def get_balance(user_id):
    c.execute("SELECT balance FROM users WHERE id=?", (user_id,))
    result = c.fetchone()
    return result[0] if result else 0

def add_withdraw_request(user_id, upi, amount):
    c.execute("INSERT INTO withdrawals (user_id, upi, amount, status) VALUES (?, ?, ?, ?)", (user_id, upi, amount, "pending"))
    conn.commit()

def get_pending_withdrawals():
    c.execute("SELECT * FROM withdrawals WHERE status='pending'")
    return c.fetchall()

# === HANDLERS ===
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    args = context.args
    referred_by = int(args[0]) if args else None
    add_user(user.id, user.username, referred_by)
    balance = get_balance(user.id)
    ref_link = f"https://t.me/{context.bot.username}?start={user.id}"
    await update.message.reply_text(
        f"👋 Welcome to EarnSep!\nEarn ₹{REFERRAL_REWARD} per referral.\n\n💰 Your balance: ₹{balance}\n🔗 Your referral link: {ref_link}"
    )

async def balance(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    bal = get_balance(user_id)
    await update.message.reply_text(f"💰 Your current balance: ₹{bal}")

async def withdraw(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    bal = get_balance(user_id)
    if bal < MIN_WITHDRAW:
        await update.message.reply_text(f"❌ Minimum withdrawal is ₹{MIN_WITHDRAW}. Your balance: ₹{bal}")
        return
    await update.message.reply_text("📝 Please enter your UPI ID to request withdrawal:")
    context.user_data['awaiting_upi'] = True

async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if context.user_data.get('awaiting_upi'):
        upi = update.message.text
        user_id = update.effective_user.id
        bal = get_balance(user_id)
        add_withdraw_request(user_id, upi, bal)
        c.execute("UPDATE users SET balance = 0 WHERE id=?", (user_id,))
        conn.commit()
        context.user_data['awaiting_upi'] = False
        await update.message.reply_text("✅ Withdrawal request sent to admin. You will receive payment soon.")

        # Notify admin
        await context.bot.send_message(
            chat_id=f"@{ADMIN_USERNAME}",
            text=(
                f"💸 New withdrawal request:\n"
                f"User: @{update.effective_user.username}\n"
                f"UserID: {user_id}\n"
                f"Amount: ₹{bal}\n"
                f"UPI: {upi}"
            )
        )

# === MAIN ===
async def main():
    app = ApplicationBuilder().token("YOUR_BOT_TOKEN_HERE").build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("balance", balance))
    app.add_handler(CommandHandler("withdraw", withdraw))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))

    print("Bot running...")
    await app.run_polling()

if __name__ == "__main__":
    asyncio.run(main())
