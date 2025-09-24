import logging
import re
import json
import threading
import os
import sqlite3
from datetime import datetime

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler,
    ContextTypes, MessageHandler, filters
)

# Логирование
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)

# Конфигурация
BOT_TOKEN = os.environ.get("BOT_TOKEN", "8350574234:AAEvzdWpWS0WFYzm2Rs-YKsZvtetrIciFm8")
ADMIN_IDS = [5224395619]  # замените на свои ID админов

# --- БД ---
class CreativeDatabaseManager:
    def __init__(self, db_path="creative_payments.db"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self.lock = threading.Lock()
        self.create_tables()

    def create_tables(self):
        with self.lock:
            cur = self.conn.cursor()
            cur.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    username TEXT,
                    start_date TEXT,
                    total_paid REAL DEFAULT 0,
                    paid_requests INTEGER DEFAULT 0
                )
            """)
            cur.execute("""
                CREATE TABLE IF NOT EXISTS requests (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER,
                    username TEXT,
                    deal_tag TEXT,
                    screenshots TEXT,
                    nft_link TEXT,
                    wallet_address TEXT,
                    status TEXT DEFAULT 'pending',
                    created_date TEXT
                )
            """)
            self.conn.commit()

    def get_profile(self, user_id):
        with self.lock:
            cur = self.conn.cursor()
            cur.execute("SELECT username, start_date, total_paid, paid_requests FROM users WHERE user_id=?", (user_id,))
            row = cur.fetchone()
            if row:
                username, start_date, total_paid, paid_requests = row
                try:
                    start_date_fmt = datetime.fromisoformat(start_date).strftime("%d.%m.%Y %H:%M")
                except:
                    start_date_fmt = datetime.now().strftime("%d.%m.%Y %H:%M")
            else:
                now = datetime.now().isoformat()
                username = f"user_{user_id}"
                cur.execute("INSERT INTO users (user_id, username, start_date) VALUES (?, ?, ?)", (user_id, username, now))
                self.conn.commit()
                start_date_fmt, total_paid, paid_requests = datetime.now().strftime("%d.%m.%Y %H:%M"), 0.0, 0
            return {"username": username, "start_date": start_date_fmt, "total_paid": total_paid, "paid_requests": paid_requests}

    def save_request(self, user_id, username, deal_tag, screenshots, nft_link, wallet):
        with self.lock:
            cur = self.conn.cursor()
            cur.execute("""
                INSERT INTO requests (user_id, username, deal_tag, screenshots, nft_link, wallet_address, created_date)
                VALUES (?, ?, ?, ?, ?, ?, ?)
            """, (user_id, username, deal_tag, json.dumps(screenshots), nft_link, wallet, datetime.now().isoformat()))
            self.conn.commit()
            return cur.lastrowid

    def update_request_status(self, request_id, status):
        with self.lock:
            cur = self.conn.cursor()
            cur.execute("UPDATE requests SET status=? WHERE id=?", (status, request_id))
            self.conn.commit()

    def get_request(self, request_id):
        with self.lock:
            cur = self.conn.cursor()
            cur.execute("SELECT id, user_id, username, deal_tag, screenshots, nft_link, wallet_address, status FROM requests WHERE id=?", (request_id,))
            row = cur.fetchone()
            if not row:
                return None
            return {
                "id": row[0],
                "user_id": row[1],
                "username": row[2],
                "deal_tag": row[3],
                "screenshots": json.loads(row[4]) if row[4] else [],
                "nft_link": row[5],
                "wallet": row[6],
                "status": row[7]
            }

    def add_payment_to_user(self, user_id, amount):
        with self.lock:
            cur = self.conn.cursor()
            cur.execute("UPDATE users SET total_paid = total_paid + ?, paid_requests = paid_requests + 1 WHERE user_id=?", (amount, user_id))
            self.conn.commit()

# --- Бот ---
class CreativePaymentBot:
    def __init__(self, token):
        self.token = token
        self.app = None
        self.db = CreativeDatabaseManager()
        self.sessions = {}

    def session(self, uid):
        if uid not in self.sessions:
            self.sessions[uid] = {"state": None, "deal_tag": None, "screenshots": [],
                                  "nft_link": None, "wallet": None}
        return self.sessions[uid]

    # --- Команды ---
    async def start(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        user = update.effective_user
        text = (
            f"🎉 Привет, @{user.username or user.id}!\n\n"
            "Я помогу быстро создавать заявки на выплаты 🚀\n\n"
            "Используй /menu для доступа к панели."
        )
        await update.message.reply_text(text)

    async def menu(self, update: Update, context: ContextTypes.DEFAULT_TYPE, query=None):
        user = update.effective_user
        self.session(user.id)["state"] = None
        profile = self.db.get_profile(user.id)
        text = (
            f"👤 ПРОФИЛЬ\n\n"
            f"🆔 ID: {user.id}\n"
            f"🚀 Стартовал: {profile['start_date']}\n"
            f"💰 Выплачено: {profile['total_paid']:.2f} TON\n"
            f"📊 Оплачено заявок: {profile['paid_requests']}\n\n"
            "🎯 Выберите действие:"
        )
        kb = [
            [InlineKeyboardButton("📥 Новая заявка", callback_data="new_request")],
            [InlineKeyboardButton("📊 Статус заявок", callback_data="status")],
            [InlineKeyboardButton("🔄 Обновить", callback_data="refresh_menu")]
        ]
        if user.id in ADMIN_IDS:
            kb.append([InlineKeyboardButton("👑 Админ", callback_data="admin_panel")])
        markup = InlineKeyboardMarkup(kb)
        if query:
            await query.edit_message_text(text, reply_markup=markup)
        else:
            await update.message.reply_text(text, reply_markup=markup)

    # --- Обработчики кнопок ---
    async def buttons(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        query = update.callback_query
        user = query.from_user
        s = self.session(user.id)
        await query.answer()
        data = query.data

        # Пользовательский поток
        if data == "new_request":
            s.update({"state": "deal", "deal_tag": None, "screenshots": [], "nft_link": None, "wallet": None})
            await query.edit_message_text("✍️ Введите тег сделки (например, #123456):", reply_markup=self.cancel_kb())
        elif data == "screenshots_done":
            if not s["screenshots"]:
                await query.answer("Прикрепите хотя бы 1 скрин!", show_alert=True)
            else:
                s["state"] = "nft"
                await query.edit_message_text("🔗 Пришлите ссылку на NFT:", reply_markup=self.cancel_kb())
        elif data == "cancel_request":
            self.sessions[user.id] = {"state": None, "deal_tag": None, "screenshots": [], "nft_link": None, "wallet": None}
            await query.edit_message_text("❌ Заявка отменена.", reply_markup=self.back_kb())
        elif data == "confirm_request":
            await self.confirm(query, context, s)
        elif data == "back_to_menu":
            await self.menu(update, context, query)
        elif data == "refresh_menu":
            await self.menu(update, context, query)
        elif data == "status":
            await query.edit_message_text("📊 Статус в разработке...")
        elif data == "admin_panel":
            await query.edit_message_text("👑 Админ-панель в разработке...")

        # Админские кнопки
        elif data.startswith("approve_") or data.startswith("reject_"):
            rid = int(data.split("_")[1])
            await self.admin_action(query, context, rid, data.startswith("approve_"))

    # --- Подтверждение заявки пользователем ---
    async def confirm(self, query, context, s):
        user = query.from_user
        if not all([s["deal_tag"], s["nft_link"], s["wallet"]]):
            await query.answer("Заполните все поля!", show_alert=True)
            return
        rid = self.db.save_request(user.id, user.username or f"user_{user.id}",
                                   s["deal_tag"], s["screenshots"], s["nft_link"], s["wallet"])
        text = (
            f"✅ Заявка #{rid} создана!\n\n"
            f"• Тег: {s['deal_tag']}\n"
            f"• Скрины: {len(s['screenshots'])}\n"
            f"• NFT: {s['nft_link']}\n"
            f"• Кошелек: {s['wallet'][:10]}...{s['wallet'][-6:]}\n\n"
            "Ожидает проверки."
        )
        # Уведомление админам
        kb = [
            [InlineKeyboardButton("✅ Одобрить", callback_data=f"approve_{rid}")],
            [InlineKeyboardButton("❌ Отклонить", callback_data=f"reject_{rid}")]
        ]
        for admin_id in ADMIN_IDS:
            try:
                await context.bot.send_message(
                    admin_id,
                    f"📢 Новая заявка #{rid}\n\n"
                    f"👤 @{user.username or user.id} (ID: {user.id})\n"
                    f"• Тег: {s['deal_tag']}\n"
                    f"• NFT: {s['nft_link']}\n"
                    f"• Кошелек: {s['wallet']}\n"
                    f"📸 Скриншотов: {len(s['screenshots'])}",
                    reply_markup=InlineKeyboardMarkup(kb)
                )
                for file_id in s["screenshots"]:
                    await context.bot.send_photo(admin_id, file_id)
            except Exception as e:
                logging.error(f"Ошибка отправки админу {admin_id}: {e}")
        self.sessions[user.id] = {"state": None, "deal_tag": None, "screenshots": [], "nft_link": None, "wallet": None}
        await query.edit_message_text(text, reply_markup=self.back_kb())

    # --- Действие админа ---
    async def admin_action(self, query, context, rid, approved):
        req = self.db.get_request(rid)
        if not req:
            await query.edit_message_text("❌ Заявка не найдена.")
            return
        new_status = "approved" if approved else "rejected"
        self.db.update_request_status(rid, new_status)
        if approved:
            self.db.add_payment_to_user(req["user_id"], 10.0)  # 💰 тут можно подставить реальную сумму
        # Сообщение админу
        await query.edit_message_text(
            f"📋 Заявка #{rid} ({req['deal_tag']}) { '✅ ОДОБРЕНА' if approved else '❌ ОТКЛОНЕНА' }"
        )
        # Уведомление пользователя
        try:
            await context.bot.send_message(
                req["user_id"],
                f"📢 Ваша заявка #{rid} { '✅ одобрена и скоро будет оплачена' if approved else '❌ отклонена администратором' }."
            )
        except Exception as e:
            logging.error(f"Ошибка отправки пользователю {req['user_id']}: {e}")

    # --- Сообщения ---
    async def handle_text(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        user = update.effective_user
        s = self.session(user.id)
        text = update.message.text.strip()
        if s["state"] == "deal":
            if not re.match(r"^#[A-Za-z0-9_-]+$", text):
                await update.message.reply_text("❌ Некорректный тег. Формат: #123456")
                return
            s["deal_tag"], s["state"] = text, "screens"
            await update.message.reply_text("📸 Пришлите скриншоты сделки:", reply_markup=self.done_kb())
        elif s["state"] == "nft":
            if not text.startswith("http"):
                await update.message.reply_text("❌ Пришлите корректную ссылку NFT")
                return
            s["nft_link"], s["state"] = text, "wallet"
            await update.message.reply_text("💳 Введите TON-кошелек:", reply_markup=self.cancel_kb())
        elif s["state"] == "wallet":
            if not re.match(r"^UQ[A-Za-z0-9_-]{46}$", text):
                await update.message.reply_text("❌ Некорректный адрес")
                return
            s["wallet"], s["state"] = text, "confirm"
            await self.preview(update, s)

    async def handle_photos(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        user = update.effective_user
        s = self.session(user.id)
        if s["state"] != "screens":
            return
        file_id = update.message.photo[-1].file_id
        s["screenshots"].append(file_id)
        await update.message.reply_text(f"📸 Загружено: {len(s['screenshots'])}", reply_markup=self.done_kb())

    # --- Превью заявки ---
    async def preview(self, update: Update, s):
        text = (
            "📋 Проверьте заявку:\n\n"
            f"• Тег: {s['deal_tag']}\n"
            f"• Скриншотов: {len(s['screenshots'])}\n"
            f"• NFT: {s['nft_link']}\n"
            f"• Кошелек: {s['wallet'][:10]}...{s['wallet'][-6:]}\n\n"
            "Подтвердите или отмените."
        )
        kb = [
            [InlineKeyboardButton("✅ Подтвердить", callback_data="confirm_request")],
            [InlineKeyboardButton("❌ Отменить", callback_data="cancel_request")]
        ]
        await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(kb))

    # --- Клавиатуры (широкие кнопки) ---
    def cancel_kb(self):
        return InlineKeyboardMarkup([[InlineKeyboardButton("❌ Отмена", callback_data="cancel_request")]])
    def done_kb(self):
        return InlineKeyboardMarkup([
            [InlineKeyboardButton("✅ Готово", callback_data="screenshots_done")],
            [InlineKeyboardButton("❌ Отмена", callback_data="cancel_request")]
        ])
    def back_kb(self):
        return InlineKeyboardMarkup([[InlineKeyboardButton("🏠 Главное меню", callback_data="back_to_menu")]])

    # --- Запуск ---
    def run(self):
        app = Application.builder().token(self.token).build()
        self.app = app
        app.add_handler(CommandHandler("start", self.start))
        app.add_handler(CommandHandler("menu", self.menu))
        app.add_handler(CallbackQueryHandler(self.buttons))
        app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, self.handle_text))
        app.add_handler(MessageHandler(filters.PHOTO, self.handle_photos))
        logging.info("🚀 Бот запущен")
        app.run_polling()

if __name__ == "__main__":
    if BOT_TOKEN == "PUT_YOUR_TOKEN_HERE":
        logging.error("Установите BOT_TOKEN")
    else:
        CreativePaymentBot(BOT_TOKEN).run()
