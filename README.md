
import telebot
from telebot import types
import sqlite3
from datetime import datetime, timedelta
from dateutil.parser import parse
from datetime import datetime, timedelta
from dateutil.parser import parse


TOKEN = '6065485834:AAHi3ytzHKc-3kiCCaIXDZILSR7uJounBfU'
bot = telebot.TeleBot(TOKEN)

# Параметры сада
income_per_hour = 3
max_trees = 100
max_water = 100
taxes = 5000000
tree_growth_income = 1
water_needed_per_hour = 1
muted_users = []
# Словарь для хранения информации о фермах
farms = {}

# Параметры фермы
gpu_price = 1000  # Цена одной видеокарты
tax_rate = 0.2  # Процент налога от прибыли

# Установка соединения с базой данных
conn = sqlite3.connect('garden.db', check_same_thread=False)
cursor = conn.cursor()

# Создание таблицы "garden"
cursor.execute('''
    CREATE TABLE IF NOT EXISTS garden (
        user_id INTEGER PRIMARY KEY,
        inventory INTEGER,
        trees INTEGER,
        water INTEGER,
        taxes_paid INTEGER,
        last_watering_time TEXT
    )
''')
# Создание таблицы "users"
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        balance INTEGER DEFAULT 0
    )
''')
cursor.execute('''
    CREATE TABLE IF NOT EXISTS bot_message_count (
        user_id INTEGER PRIMARY KEY,
        count INTEGER DEFAULT 0
    )
''')
# Создание таблицы "farms"
cursor.execute('''
    CREATE TABLE IF NOT EXISTS farms (
        user_id INTEGER PRIMARY KEY,
        gpus INTEGER,
        balance INTEGER
    )
''')
conn.commit()






# Параметры фермы



@bot.message_handler(commands=['ferm'])
def start(message):
    user_id = message.from_user.id

    cursor.execute('SELECT * FROM farms WHERE user_id=?', (user_id,))
    result = cursor.fetchone()

    if result:
        gpus, balance = result[1], result[2]
    else:
        gpus, balance = 0, 0
        cursor.execute('INSERT INTO farms VALUES (?, ?, ?)', (user_id, gpus, balance))
        conn.commit()

    reply_message = f"Добро пожаловать в майнинг ферму!\n\n" \
                    f"У вас на счету: {balance} зёрен\n" \
                    f"Количество видеокарт: {gpus}\n\n" \
                    f"Используйте кнопки ниже для управления фермой."


@bot.message_handler(func=lambda message: message.text == 'Купить видеокарту')
def buy_gpu(message):
    user_id = message.from_user.id

    cursor.execute('SELECT * FROM farms WHERE user_id=?', (user_id,))
    result = cursor.fetchone()

    if result:
        gpus, balance = result[1], result[2]
    else:
        bot.reply_to(message, "Произошла ошибка при получении данных.")
        return

    if balance < gpu_price:
        bot.reply_to(message, "У вас недостаточно средств для покупки видеокарты.")
        return

    gpus += 1
    balance -= gpu_price

    cursor.execute('UPDATE farms SET gpus=?, balance=? WHERE user_id=?', (gpus, balance, user_id))
    conn.commit()

    bot.reply_to(message, "Вы успешно приобрели одну видеокарту.")


@bot.message_handler(func=lambda message: message.text == 'Добыть монеты')
def mine(message):
    user_id = message.from_user.id

    cursor.execute('SELECT * FROM farms WHERE user_id=?', (user_id,))
    result = cursor.fetchone()

    if result:
        gpus, balance = result[1], result[2]
    else:
        bot.reply_to(message, "Произошла ошибка при получении данных.")
        return

    if gpus == 0:
        bot.reply_to(message, "У вас нет видеокарт. Приобретите их с помощью кнопки 'Купить видеокарту'.")
        return

    mined_coins = gpus * 10  # Количество заработанных монет на каждой видеокарте
    balance += mined_coins

    cursor.execute('UPDATE farms SET balance=? WHERE user_id=?', (balance, user_id))
    conn.commit()

    bot.reply_to(message, f"Вы добыли {mined_coins} монет.")


@bot.message_handler(func=lambda message: message.text == 'Оплатить налоги')
def pay_taxes(message):
    user_id = message.from_user.id

    cursor.execute('SELECT * FROM farms WHERE user_id=?', (user_id,))
    result = cursor.fetchone()

    if result:
        gpus, balance = result[1], result[2]
    else:
        bot.reply_to(message, "Произошла ошибка при получении данных.")
        return

    taxes = balance * tax_rate

    if taxes == 0:
        bot.reply_to(message, "У вас нет налогов для оплаты.")
        return

    balance -= taxes

    cursor.execute('UPDATE farms SET balance=? WHERE user_id=?', (balance, user_id))
    conn.commit()

    bot.reply_to(message, f"Вы успешно оплатили налоги в размере {taxes} монет.")


@bot.message_handler(func=lambda message: message.text == 'Вывести средства')
def withdraw(message):
    user_id = message.from_user.id

    cursor.execute('SELECT * FROM farms WHERE user_id=?', (user_id,))
    result = cursor.fetchone()

    if result:
        gpus, balance = result[1], result[2]
    else:
        bot.reply_to(message, "Произошла ошибка при получении данных.")
        return

    if balance == 0:
        bot.reply_to(message, "У вас нет средств для вывода.")
        return

    cursor.execute('UPDATE farms SET balance=0 WHERE user_id=?', (user_id,))
    conn.commit()

    # Ваш код для перевода средств на игровой баланс игрока

    bot.reply_to(message, f"Вы успешно сняли {balance} монет с баланса фермы и перевели их на игровой баланс.")



# Функция для загрузки данных пользователя из базы данных
def load_data(user_id):
    cursor.execute('SELECT * FROM garden WHERE user_id=?', (user_id,))
    result = cursor.fetchone()
    if result:
        inventory = result[1]
        trees = result[2]
        water = result[3]
        taxes_paid = result[4]
        last_watering_time_str = result[5]
        last_watering_time = parse(last_watering_time_str) if last_watering_time_str else None
        return inventory, trees, water, taxes_paid, last_watering_time
    else:
        return 0, 0, 0, 0, None


# Функция для сохранения данных пользователя в базе данных
def save_data(user_id, inventory, trees, water, taxes_paid, last_watering_time):
    cursor.execute('SELECT * FROM garden WHERE user_id=?', (user_id,))
    result = cursor.fetchone()
    if result:
        cursor.execute(
            'UPDATE garden SET inventory=?, trees=?, water=?, taxes_paid=?, last_watering_time=? WHERE user_id=?',
            (inventory, trees, water, taxes_paid, last_watering_time, user_id))
    else:
        cursor.execute('INSERT INTO garden VALUES (?, ?, ?, ?, ?, ?)',
                       (user_id, inventory, trees, water, taxes_paid, last_watering_time))
    conn.commit()


# Функция для расчета общего дохода
def calculate_income(trees):
    return income_per_hour * trees


# Функция для сброса сада пользователя
def reset_garden(user_id):
    inventory, trees, water, taxes_paid, last_watering_time = 0, 0, 0, 0, None
    save_data(user_id, inventory, trees, water, taxes_paid, last_watering_time)


# Функция для создания клавиатуры состояния сада
def create_status_keyboard():
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    water_button = types.InlineKeyboardButton('Полить деревья', callback_data='water')
    plant_button = types.InlineKeyboardButton('Посадить дерево', callback_data='plant')
    paytaxes_button = types.InlineKeyboardButton('Оплатить налоги', callback_data='paytaxes')
    inventory_button = types.InlineKeyboardButton('Показать инвентарь', callback_data='inventory')
    keyboard.add(water_button, plant_button, paytaxes_button, inventory_button)
    return keyboard
    keyboard = types.ReplyKeyboardMarkup(row_width=2)
    buy_gpu_button = types.KeyboardButton('Купить видеокарту')
    mine_button = types.KeyboardButton('Добыть монеты')
    pay_taxes_button = types.KeyboardButton('Оплатить налоги')
    withdraw_button = types.KeyboardButton('Вывести средства')
    keyboard.add(buy_gpu_button, mine_button, pay_taxes_button, withdraw_button)

    bot.reply_to(message, reply_message, reply_markup=keyboard)

# Функция для создания админ-клавиатуры
def create_admin_keyboard():
    keyboard = types.InlineKeyboardMarkup(row_width=3)
    balance_button = types.InlineKeyboardButton('Выдать баланс', callback_data='admin_balance')
    keyboard.add(balance_button)
    return keyboard
    
 
# Обработчик команды /start и /баланс
@bot.message_handler(commands=['start', 'баланс'])
def start(message):
    user_id = message.from_user.id
    cursor.execute('INSERT OR IGNORE INTO users (user_id) VALUES (?)', (user_id,))
    conn.commit()

    cursor.execute('SELECT balance FROM users WHERE user_id = ?', (user_id,))
    result = cursor.fetchone()
    if result:
        balance = result[0]
        bot.reply_to(message, f'Ваш игровой баланс: {balance}')
    else:
        bot.reply_to(message, "У вас еще нет игрового баланса")

    # Добавить код для сохранения данных пользователя в таблице "garden"
    inventory, trees, water, taxes_paid, last_watering_time = load_data(user_id)
    save_data(user_id, inventory, trees, water, taxes_paid, last_watering_time)


# Обработчик команды /status
@bot.message_handler(commands=['status'])
def status(message):
    user_id = message.from_user.id
    inventory, trees, water, taxes_paid, last_watering_time = load_data(user_id)

    current_time = datetime.now()
    elapsed_time = current_time - last_watering_time if last_watering_time else timedelta(days=1)
    if elapsed_time > timedelta(days=4):
        reset_garden(user_id)
        bot.reply_to(message, 'Ваши деревья засохли из-за недостатка полива. Сад сброшен.')
        return

    income = calculate_income(trees)
    status_message = f'🥜 Доход: {income} зёрен/час\n'
    status_message += f'🌳 Деревья: {trees} шт./{max_trees} шт.\n'
    status_message += f'💦 Воды: {water}/{max_water}\n'
    status_message += f'💸 Налоги: {taxes_paid}$/5.000.000$\n'
    status_message += f'🧺 На счету: {inventory} зёрен'
    keyboard = create_status_keyboard()
    bot.reply_to(message, status_message, reply_markup=keyboard)


@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    user_id = call.from_user.id

    if call.data == 'water':
        inventory, trees, water, taxes_paid, last_watering_time = load_data(user_id)

        current_time = datetime.now()
        elapsed_time = current_time - last_watering_time if last_watering_time else timedelta(hours=1)
        if elapsed_time < timedelta(hours=1):
            bot.answer_callback_query(call.id, text='Вы уже поливали деревья в этот час. Попробуйте позже.')
            return

        if water < water_needed_per_hour:
            bot.answer_callback_query(call.id, text='Недостаточно воды для полива деревьев. Пополните запас воды.')
            return

        water -= water_needed_per_hour
        last_watering_time = current_time
        save_data(user_id, inventory, trees, water, taxes_paid, last_watering_time)
        bot.answer_callback_query(call.id,text='Деревья успешно политы!')

    elif call.data == 'plant':
        inventory, trees, water, taxes_paid, last_watering_time = load_data(user_id)

        if trees >= max_trees:
            bot.answer_callback_query(call.id, text='Вы достигли максимального количества деревьев.')
            return

        cursor.execute('SELECT balance FROM users WHERE user_id = ?', (user_id,))
        result = cursor.fetchone()
        if result:
            balance = result[0]
        else:
            balance = 0

        if balance < 1000:
            bot.answer_callback_query(call.id, text='У вас недостаточно денег для посадки дерева.')
            return

        balance -= 1000
        trees += 1
        cursor.execute('UPDATE users SET balance = ? WHERE user_id = ?', (balance, user_id))
        save_data(user_id, inventory, trees, water, taxes_paid, last_watering_time)
        bot.answer_callback_query(call.id, text='Дерево успешно посажено!')

    elif call.data == 'paytaxes':
        inventory, trees, water, taxes_paid, last_watering_time = load_data(user_id)

        if inventory < taxes:
            bot.answer_callback_query(call.id, text='У вас недостаточно зёрен для оплаты налогов.')
            return

        inventory -= taxes
        taxes_paid += taxes
        save_data(user_id, inventory, trees, water, taxes_paid, last_watering_time)
        bot.answer_callback_query(call.id, text='Налоги успешно оплачены!')

    elif call.data == 'inventory':
        inventory, _, _, _, _ = load_data(user_id)
        bot.answer_callback_query(call.id, text=f'На вашем счету {inventory} зёрен.')


@bot.message_handler(commands=['gg'])
def change_balance(message):
    if message.from_user.id != 1138581979:
        bot.reply_to(message, "У вас нет разрешения на выполнение этой команды.")
        return

    args = message.text.split(' ')
    if len(args) != 3:
        bot.reply_to(message, "Неверный формат команды. Используйте /gg <user_id> <balance>")
        return

    user_id = int(args[1])
    balance = int(args[2])

    cursor.execute('UPDATE users SET balance = ? WHERE user_id = ?', (balance, user_id))
    conn.commit()

    bot.reply_to(message, f"Игровой баланс пользователя {user_id} изменен на {balance}.")


@bot.message_handler(commands=['mute'])
def mute_user(message):
    # Проверяем, что только вы можете использовать эту команду
    if message.from_user.id != 1138581979:
        bot.reply_to(message, "У вас нет разрешения на выполнение этой команды.")
        return

    # Получаем аргументы команды
    args = message.text.split()[1:]

    if len(args) < 1:
        bot.reply_to(message, "Неверный формат команды. Используйте /mute <user_id или username>")
        return

    target_id = None
    if args[0].startswith('@'):
        # Если передан username, получаем user_id
        username = args[0][1:]
        user = bot.get_chat_member(message.chat.id, username)
        if user:
            target_id = user.user.id
    else:
        # Если передан user_id, проверяем его валидность
        try:
            target_id = int(args[0])
        except ValueError:
            bot.reply_to(message, "Неверный формат user_id.")
            return

    # Производим действия по муту пользователя с указанным user_id

    if target_id:
        # Выполните необходимые действия для мута пользователя
        # Например, используйте метод restrict_chat_member для установки ограничений

        # Отправляем ответное сообщение
        bot.reply_to(message, f"Пользователь с user_id {target_id} замучен.")
    else:
        bot.reply_to(message, "Не удалось найти пользователя с указанным user_id или username.")

@bot.message_handler(func=lambda message: message.chat.type == 'private')
def handle_private_message(message):
    # Проверяем, что пользователь имеет доступ к админ-панели (вы можете настроить это условие по своему усмотрению)
    if message.from_user.id != 1138581979:
        bot.reply_to(message, "У вас нет доступа к админ-панели.")
        return

    # Проверяем текст сообщения и обрабатываем соответствующие команды
    if message.text == '/admin':
        keyboard = create_admin_keyboard()
        bot.reply_to(message, 'Добро пожаловать в админ-панель!', reply_markup=keyboard)
    else:
        bot.reply_to(message, "Неизвестная команда.")

@bot.callback_query_handler(func=lambda call: call.data.startswith('admin_'))
def handle_admin_callback(call):
    # Проверяем, что запрос пришел из админ-панели (вы можете настроить это условие по своему усмотрению)
    if call.from_user.id != ВАШ_USER_ID:
        bot.answer_callback_query(call.id, text='У вас нет доступа к админ-панели.')
        return

    # Обрабатываем соответствующий коллбэк-запрос
    if call.data == 'admin_balance':
        bot.answer_callback_query(call.id, text='Выберите пользователя, для которого нужно выдать баланс:')
        # Получаем список пользователей из базы данных
        cursor.execute('SELECT user_id FROM users')
        results = cursor.fetchall()
        if results:
            keyboard = types.InlineKeyboardMarkup(row_width=2)
            for result in results:
                user_id = result[0]
                user_button = types.InlineKeyboardButton(f'User ID: {user_id}', callback_data=f'admin_balance_{user_id}')
                keyboard.add(user_button)
            bot.send_message(call.from_user.id, 'Выберите пользователя:', reply_markup=keyboard)
        else:
            bot.send_message(call.from_user.id, 'Нет зарегистрированных пользователей.')

    elif call.data.startswith('admin_balance_'):
        # Получаем user_id выбранного пользователя
        user_id = int(call.data.split('_')[2])

        # Получаем баланс пользователя из базы данных
        cursor.execute('SELECT balance FROM users WHERE user_id = ?', (user_id,))
        result = cursor.fetchone()
        if result:
            balance = result[0]
            bot.send_message(call.from_user.id, f'Баланс пользователя с User ID {user_id}: {balance} зерна.')
        else:
            bot.send_message(call.from_user.id, 'Пользователь не найден.')

@bot.callback_query_handler(func=lambda call: call.data.startswith('give_balance_'))
def give_balance_callback(call):
    user_id = int(call.data.split('_')[2])

    # Выполните необходимые действия для выдачи баланса пользователю с указанным user_id

    bot.answer_callback_query(call.id, text=f"Выдан баланс пользователю с user_id {user_id}")


# Запуск бота
bot.polling()

