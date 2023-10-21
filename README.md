Задание было сделать телеграмм-бота с которым можно поиграть в игру Камень, ножницы, бумага. 
Я(Михаил) делал в основном код, пытался сделать через import telegramm, но  особо не получилось реализовать( сам по себе основную работу код выполнял, мелких деталей по типу эмоджи и тд не нашел. Поговорив с тим лидом решили сделать через телебот) 





import random
import telebot
from telebot import types
import sqlite3


with sqlite3.connect("statistics.db") as con:
    cur = con.cursor()
    sqlite_query = '''
        CREATE TABLE IF NOT EXISTS games (
        id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
        player_name TEXT,
        wins INTEGER,
        losses INTEGER,
        winner TEXT
        );
    '''
    cur.execute(sqlite_query)

# Создание экземпляра бота
bot = telebot.TeleBot('6504028858:AAERNlYZ3n-2VHx1GfMcY_6ruGJWUFvVZNM')

# Глобальные переменные для счетчика побед и поражений
wins = 0
losses = 0
player_name = ''
winner = ''

# Эмодзи для камня, ножниц и бумаги
emoji_rock = '🪨'
emoji_scissors = '✂️'
emoji_paper = '📄'

# Обработчик команды /start
@bot.message_handler(commands=['start'])
def start(message):
    global wins, losses
    wins = 0
    losses = 0
    bot.send_message(message.chat.id, "Привет! Добро пожаловать в игру 'Камень, ножницы, бумага'!\nКоманды: /game - запуск игры, /name - ввод имени игрока, /exit - выход из игры")

# Обработчик команды /name
@bot.message_handler(commands=['name'])
def name(message):
    global player_name
    bot.send_message(message.chat.id, 'Введите имя')



# Обработчик команды /game
@bot.message_handler(commands=['game'])
def game(message):
    choices = ['камень', 'ножницы', 'бумага']
    computer_choice = random.choice(choices)

    markup = types.ReplyKeyboardMarkup(row_width=3)
    buttons = [types.KeyboardButton(choice) for choice in choices]
    markup.add(*buttons)

    msg = bot.send_message(message.chat.id, "Выберите свой вариант:", reply_markup=markup)
    bot.register_next_step_handler(msg, check_winner, computer_choice)


# Обработчик команды /exit
@bot.message_handler(commands=['exit'])
def exit(message):
    with sqlite3.connect("statistics.db") as con:
        cur = con.cursor()
        [print(x) for x in cur.execute("SELECT * FROM games").fetchall()]   
    bot.stop_bot()

def check_winner(message, computer_choice):
    global wins, losses, player_name
    user_choice = message.text.lower()

    choices = ['камень', 'ножницы', 'бумага']

    if user_choice not in choices:
        bot.send_message(message.chat.id, "Некорректный выбор. Попробуйте еще раз.")
        return

    if user_choice == computer_choice:
        result = "Ничья!"
        winner = "Ничья!"
    elif (user_choice == 'камень' and computer_choice == 'ножницы') or \
         (user_choice == 'ножницы' and computer_choice == 'бумага') or \
         (user_choice == 'бумага' and computer_choice == 'камень'):
        result = "Вы победили!"
        wins += 1
        winner = player_name
    else:
        result = "Компьютер победил!"
        losses += 1
        winner = 'Компьютер'

    with sqlite3.connect("statistics.db") as con:
        cur = con.cursor()
        cur.execute(
            "INSERT INTO games (player_name, wins, losses, winner) VALUES (?, ?, ?, ?);",
            (player_name, wins, losses, winner)
        )

    bot.send_message(message.chat.id, f"Вы выбрали: {get_emoji(user_choice)}\nКомпьютер выбрал: {get_emoji(computer_choice)}\n{result}")
    bot.send_message(message.chat.id, f"Победы: {wins}\nПоражения: {losses}")


@bot.message_handler(content_types='text')
def message_reply(message):
    global player_name
    if player_name == '':
        player_name = message.text
        bot.send_message(message.chat.id, f'{player_name}! Можете начинать игру.')
        bot.reply_to(message,  f'{player_name}! Можете начинать игру.')


# Обработчик неизвестных команд
@bot.message_handler(func=lambda message: True)
def unknown(message):
    bot.send_message(message.chat.id, "Извините, я не понимаю эту команду.")


# Функция для получения эмодзи по выбору пользователя
def get_emoji(choice):
    if choice == 'камень':
        return emoji_rock
    elif choice == 'ножницы':
        return emoji_scissors
    elif choice == 'бумага':
        return emoji_paper
    else:
        return ''

# Запуск бота
bot.polling()
