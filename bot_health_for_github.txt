from telegram import Update, ReplyKeyboardMarkup, ReplyKeyboardRemove
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, ConversationHandler
import sqlite3
import logging
import random

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

GENDER, WEIGHT, HEIGHT, AGE, ACTIVITY, ALLERGIES = range(6)

class Database:
    def __init__(self, db_path: str = 'health_bot.db'):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self.conn.execute("PRAGMA foreign_keys = ON")
        self.create_tables()

    def create_tables(self):
        cursor = self.conn.cursor()

        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                gender TEXT,
                weight REAL,
                height REAL,
                age INTEGER,
                activity_level TEXT,
                max_calories INTEGER,
                allergies TEXT
            )
        ''')

        cursor.execute('''
            CREATE TABLE IF NOT EXISTS weekly_menu (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                day TEXT,
                meal_type TEXT,
                dish_name TEXT,
                calories INTEGER,
                FOREIGN KEY(user_id) REFERENCES users(user_id) ON DELETE CASCADE
            )
        ''')

        self.conn.commit()

    def save_user_data(self, user_id, data: dict):
        cursor = self.conn.cursor()
        
        gender = data.get('gender', '')
        weight = data.get('weight', None)
        height = data.get('height', None)
        age = data.get('age', None)
        activity_level = data.get('activity_level', '')
        max_calories = data.get('max_calories', None)
        allergies = data.get('allergies', '')

        cursor.execute('''
            INSERT INTO users (user_id, gender, weight, height, age, activity_level, max_calories, allergies)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT(user_id) DO UPDATE SET
                gender=excluded.gender,
                weight=excluded.weight,
                height=excluded.height,
                age=excluded.age,
                activity_level=excluded.activity_level,
                max_calories=excluded.max_calories,
                allergies=excluded.allergies
        ''', (user_id, gender, weight, height, age, activity_level, max_calories, allergies))
        
        self.conn.commit()

    def get_user_data(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
        result = cursor.fetchone()
        return result

db = Database()

MEALS = {
    'завтрак': [
        {'name': 'Овсяная каша с фруктами', 'calories': 300, 'allergens': ['глютен']},
        {'name': 'Творог с ягодами', 'calories': 250, 'allergens': ['лактоза']},
        {'name': 'Яичница с овощами', 'calories': 350, 'allergens': ['яйца']},
        {'name': 'Гречневая каша', 'calories': 280, 'allergens': []},
        {'name': 'Смузи bowl', 'calories': 320, 'allergens': ['лактоза']},
        {'name': 'Тосты с авокадо', 'calories': 280, 'allergens': ['глютен']},
        {'name': 'Рисовая каша', 'calories': 270, 'allergens': []},
        {'name': 'Блины', 'calories': 310, 'allergens': ['глютен', 'яйца']},
    ],
    'обед': [
        {'name': 'Куриная грудка с гречкой', 'calories': 450, 'allergens': []},
        {'name': 'Рыба на пару с овощами', 'calories': 400, 'allergens': ['рыба']},
        {'name': 'Салат с тунцом', 'calories': 380, 'allergens': ['рыба']},
        {'name': 'Овощной суп', 'calories': 350, 'allergens': []},
        {'name': 'Индейка с бурым рисом', 'calories': 420, 'allergens': []},
        {'name': 'Чечевичный суп', 'calories': 370, 'allergens': []},
        {'name': 'Овощное рагу', 'calories': 320, 'allergens': []},
        {'name': 'Макаронник с курицей', 'calories': 480, 'allergens': ['глютен']},
        {'name': 'Борщ со сметаной', 'calories': 360, 'allergens': ['лактоза']},
    ],
    'ужин': [
        {'name': 'Салат с курицей', 'calories': 350, 'allergens': []},
        {'name': 'Тушеные овощи', 'calories': 300, 'allergens': []},
        {'name': 'Творожная запеканка', 'calories': 320, 'allergens': ['лактоза', 'яйца']},
        {'name': 'Рыба на гриле', 'calories': 380, 'allergens': ['рыба']},
        {'name': 'Омлет с зеленью', 'calories': 280, 'allergens': ['яйца']},
        {'name': 'Овощи на гриле', 'calories': 250, 'allergens': []},
        {'name': 'Куриные котлеты на пару', 'calories': 330, 'allergens': []},
        {'name': 'Запеченая картошка с рыбой', 'calories': 400, 'allergens': ['рыба']},
    ],
    'перекус': [
        {'name': 'Яблоко', 'calories': 50, 'allergens': []},
        {'name': 'Греческий йогурт', 'calories': 120, 'allergens': ['лактоза']},
        {'name': 'Орехи', 'calories': 180, 'allergens': ['орехи']},
        {'name': 'Протеиновый батончик', 'calories': 150, 'allergens': []},
        {'name': 'Банан', 'calories': 90, 'allergens': []},
        {'name': 'Морковь', 'calories': 40, 'allergens': []},
        {'name': 'Груша', 'calories': 60, 'allergens': []},
        {'name': 'Творог', 'calories': 110, 'allergens': ['лактоза']},
    ]
}

def calc_calories(w, h, a, gender, activity):
    """Считаем калории по формуле Миффлина"""
    activity = (activity or '').lower()

    if gender == 'мужской':
        base = 10 * w + 6.25 * h - 5 * a + 5
    else:
        base = 10 * w + 6.25 * h - 5 * a - 161

    mults = {
        'сидячий': 1.2,
        'сидячий образ жизни': 1.2,
        'легкая': 1.375,
        'легкая активность': 1.375,
        'умеренная': 1.55,
        'умеренная активность': 1.55,
        'высокая': 1.725,
        'высокая активность': 1.725,
        'экстремальная': 1.9,
        'экстремальная активность': 1.9
    }

    mult = mults.get(activity, 1.2)
    return int(base * mult)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    await update.message.reply_text(
        f"Привет, {user.first_name}! 🏃‍♂️\n"
        "Я твой помощник для расчета калорий и меню.\n\n"
        "Что я умею:\n"
        "• 📊 Рассчитать твою дневную норму\n"
        "• 🍽 Составить меню на неделю\n"
        "• 🚫 Учитывать аллергии\n\n"
        "Команды:\n"
        "/calculate - рассчитать калории\n"
        "/menu - создать меню\n"
        "/my_data - твои данные\n"
        "/help - справка"
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = (
        "📋 Команды:\n\n"
        "/start - начало\n"
        "/calculate - расчет калорий\n"
        "/menu - меню на неделю\n"
        "/my_data - показать мои данные\n"
        "/help - эта справка\n\n"
        "Учитываю:\n"
        "• Вес, рост, возраст\n"
        "• Активность\n"
        "• Аллергии\n"
        "• Баланс питания"
    )
    await update.message.reply_text(help_text)

async def calculate_calories_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [['Мужской', 'Женский']]
    reply_markup = ReplyKeyboardMarkup(keyboard, one_time_keyboard=True, resize_keyboard=True)
    await update.message.reply_text(
        "Рассчитаем твою норму калорий! 🔥\n"
        "Выбери пол:",
        reply_markup=reply_markup
    )
    return GENDER

async def get_gender(update: Update, context: ContextTypes.DEFAULT_TYPE):
    gender_input = update.message.text.lower()

    if gender_input in ['мужской', 'муж', 'м']:
        context.user_data['gender'] = 'мужской'
    elif gender_input in ['женский', 'жен', 'ж']:
        context.user_data['gender'] = 'женский'
    else:
        context.user_data['gender'] = gender_input

    await update.message.reply_text(
        "Введи вес в кг (например: 70):",
        reply_markup=ReplyKeyboardRemove()
    )
    return WEIGHT

async def get_weight(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        w = float(update.message.text.replace(',', '.'))
        if w <= 0 or w > 300:
            await update.message.reply_text("Введи нормальный вес (1-300 кг):")
            return WEIGHT
        context.user_data['weight'] = w

        await update.message.reply_text("Теперь рост в см (например: 175):")
        return HEIGHT
    except ValueError:
        await update.message.reply_text("Должно быть число:")
        return WEIGHT

async def get_height(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        h = float(update.message.text.replace(',', '.'))
        if h < 50 or h > 250:
            await update.message.reply_text("Рост должен быть от 50 до 250 см:")
            return HEIGHT
        context.user_data['height'] = h

        await update.message.reply_text("Сколько тебе лет?")
        return AGE
    except ValueError:
        await update.message.reply_text("Число, пожалуйста:")
        return HEIGHT

async def get_age(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        age = int(update.message.text)
        if age <= 0 or age > 120:
            await update.message.reply_text("Возраст от 1 до 120:")
            return AGE
        context.user_data['age'] = age

        keyboard = [
            ['Сидячий', 'Легкая'],
            ['Умеренная', 'Высокая'],
            ['Экстремальная']
        ]
        reply_markup = ReplyKeyboardMarkup(keyboard, one_time_keyboard=True, resize_keyboard=True)

        await update.message.reply_text(
            "Выбери активность:\n\n"
            "💺 Сидячий - офис\n"
            "🚶 Легкая - 1-3 раза/нед\n"
            "🏃 Умеренная - 3-5 раз/нед\n"
            "🏋️ Высокая - почти каждый день\n"
            "🔥 Экстремальная - профспорт",
            reply_markup=reply_markup
        )
        return ACTIVITY
    except ValueError:
        await update.message.reply_text("Введи число:")
        return AGE

async def get_activity(update: Update, context: ContextTypes.DEFAULT_TYPE):
    act_input = update.message.text.lower()

    act_map = {
        'сидячий': 'сидячий',
        'сидячий образ жизни': 'сидячий',
        'легкая': 'легкая',
        'легкий': 'легкая',
        'умеренная': 'умеренная',
        'умеренный': 'умеренная',
        'высокая': 'высокая',
        'высокий': 'высокая',
        'экстремальная': 'экстремальная',
        'экстремальный': 'экстремальная'
    }

    context.user_data['activity_level'] = act_map.get(act_input, act_input)

    await update.message.reply_text(
        "Есть аллергии? 🚫\n"
        "Напиши через запятую или 'нет'\n"
        "(пример: орехи, лактоза, глютен)",
        reply_markup=ReplyKeyboardRemove()
    )
    return ALLERGIES

async def get_allergies(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        allergy_text = update.message.text.lower().strip()
        if allergy_text in ['нет', 'нету', 'no', 'n/a']:
            context.user_data['allergies'] = ''
        else:
            context.user_data['allergies'] = allergy_text

        data = context.user_data
        w = float(data['weight'])
        h = float(data['height'])
        age = int(data['age'])
        gender = data.get('gender', 'женский')
        activity = data.get('activity_level', 'сидячий')

        max_cal = calc_calories(w, h, age, gender, activity)
        context.user_data['max_calories'] = max_cal

        try:
            db.save_user_data(update.message.from_user.id, context.user_data)
        except Exception as e:
            logger.error(f"Ошибка БД: {e}")
            await update.message.reply_text("Ошибка при сохранении. Попробуй позже.")
            return ConversationHandler.END

        await update.message.reply_text(
            f"✅ Готово!\n\n"
            f"📊 Твои данные:\n"
            f"• Пол: {gender}\n"
            f"• Вес: {w} кг\n"
            f"• Рост: {h} см\n"
            f"• Возраст: {age}\n"
            f"• Активность: {activity}\n"
            f"• Аллергии: {context.user_data['allergies'] if context.user_data['allergies'] else 'нет'}\n\n"
            f"🔥 Норма калорий: {max_cal} ккал/день\n\n"
            f"Теперь можешь создать меню /menu"
        )
        return ConversationHandler.END

    except Exception as e:
        logger.error(f"Ошибка: {e}")
        await update.message.reply_text(
            f"Что-то пошло не так: {str(e)}\n"
            f"Начни заново /calculate"
        )
        return ConversationHandler.END

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        'Отменено. /calculate для заново',
        reply_markup=ReplyKeyboardRemove()
    )
    return ConversationHandler.END

async def show_my_data(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    user_data = db.get_user_data(user_id)

    if user_data:
        # (user_id, gender, weight, height, age, activity_level, max_calories, allergies)
        allrg = user_data[7] if user_data[7] else 'нет'
        
        await update.message.reply_text(
            f"📊 Твои данные:\n"
            f"• Пол: {user_data[1]}\n"
            f"• Вес: {user_data[2]} кг\n"
            f"• Рост: {user_data[3]} см\n"
            f"• Возраст: {user_data[4]}\n"
            f"• Активность: {user_data[5]}\n"
            f"• Аллергии: {allrg}\n"
            f"• Норма: {user_data[6]} ккал/день"
        )
    else:
        await update.message.reply_text(
            "Данные не найдены. Рассчитай калории /calculate"
        )

def generate_menu(user_id, max_cal, allergies_str):
    """Генерируем меню на неделю"""
    
    user_allerg = []
    if allergies_str and allergies_str.strip():
        user_allerg = [a.strip().lower() for a in allergies_str.split(',')]

    days = ['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пятница', 'Суббота', 'Воскресенье']
    meal_types = ['завтрак', 'обед', 'ужин', 'перекус']

    used = set()
    menu = []

    for day in days:
        day_cal = 0
        day_menu = []

        for meal_type in meal_types:
            if meal_type not in MEALS:
                continue

            # Ищем подходящие блюда
            suitable = [
                m for m in MEALS[meal_type]
                if m['name'] not in used
                and not any(alg.lower() in user_allerg for alg in m.get('allergens', []))
            ]

            # Если не найдены - берем без учета повторов
            if not suitable:
                suitable = [
                    m for m in MEALS[meal_type]
                    if not any(alg.lower() in user_allerg for alg in m.get('allergens', []))
                ]

            if not suitable:
                continue

            # Берем рандомное
            meal = random.choice(suitable)
            used.add(meal['name'])

            # Контроль калорий
            if day_cal + meal['calories'] <= max_cal * 1.2:
                day_cal += meal['calories']
                day_menu.append({
                    'day': day,
                    'meal_type': meal_type,
                    'dish': meal['name'],
                    'cal': meal['calories']
                })

        # Если день слишком пустой - добавим
        if day_cal < max_cal * 0.8:
            missing = [m for m in meal_types if m not in [x['meal_type'] for x in day_menu]]
            for meal_type in missing:
                possible = [
                    m for m in MEALS[meal_type]
                    if not any(alg.lower() in user_allerg for alg in m.get('allergens', []))
                ]
                if possible:
                    meal = random.choice(possible)
                    day_menu.append({
                        'day': day,
                        'meal_type': meal_type,
                        'dish': meal['name'],
                        'cal': meal['calories']
                    })
                    day_cal += meal['calories']

        menu.extend(day_menu)

    return menu

async def create_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.message.from_user.id
        user_data = db.get_user_data(user_id)

        if not user_data:
            await update.message.reply_text(
                "Сначала рассчитай калории /calculate"
            )
            return

        max_cal = int(user_data[6]) if user_data[6] else 2000
        allergies = user_data[7] if user_data[7] else ''

        menu = generate_menu(user_id, max_cal, allergies)

        if not menu:
            await update.message.reply_text(
                "Не получилось составить меню. Может быть слишком много аллергий?\n"
                "Попробуй /calculate и измени ограничения."
            )
            return

        # Очищаем старое
        cursor = db.conn.cursor()
        cursor.execute('DELETE FROM weekly_menu WHERE user_id = ?', (user_id,))

        # Записываем новое
        for item in menu:
            cursor.execute('''
                INSERT INTO weekly_menu (user_id, day, meal_type, dish_name, calories)
                VALUES (?, ?, ?, ?, ?)
            ''', (user_id, item['day'], item['meal_type'], item['dish'], item['cal']))

        db.conn.commit()

        # Выводим
        msg = "🍽️ Меню на неделю:\n"
        days_dict = {}
        for item in menu:
            if item['day'] not in days_dict:
                days_dict[item['day']] = []
            days_dict[item['day']].append(item)

        day_order = ['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пятница', 'Суббота', 'Воскресенье']

        total_cal = 0
        days_cnt = 0

        for day in day_order:
            if day in days_dict:
                days_cnt += 1
                msg += f"\n📅 {day}:\n"
                day_cal = 0

                for item in days_dict[day]:
                    msg += f"• {item['meal_type'].title()}: {item['dish']} ({item['cal']} ккал)\n"
                    day_cal += item['cal']

                total_cal += day_cal
                msg += f"📊 Итого: {day_cal} ккал\n"

        if days_cnt > 0:
            avg = total_cal / days_cnt
            msg += f"\n📈 Средняя в день: {int(avg)} ккал\n"
            msg += f"🔥 Твоя норма: {max_cal} ккал"

            diff = int(avg - max_cal)
            if diff > 0:
                msg += f"\n⚠️ Превышение: {diff} ккал"
            else:
                msg += f"\n✅ Все ок!"

        await update.message.reply_text(msg)

    except Exception as e:
        logger.error(f"Ошибка меню: {e}")
        await update.message.reply_text(
            f"Ошибка: {str(e)}\n"
            f"Попробуй еще раз или напиши разработчику."
        )

def main():
    TOKEN = "token"

    app = Application.builder().token(TOKEN).build()

    logger.info("Бот запускается...")

    # Команды
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("my_data", show_my_data))
    app.add_handler(CommandHandler("menu", create_menu))

    # Разговор для расчета
    conv = ConversationHandler(
        entry_points=[CommandHandler('calculate', calculate_calories_start)],
        states={
            GENDER: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_gender)],
            WEIGHT: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_weight)],
            HEIGHT: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_height)],
            AGE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_age)],
            ACTIVITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_activity)],
            ALLERGIES: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_allergies)],
        },
        fallbacks=[CommandHandler('cancel', cancel)],
        allow_reentry=True
    )

    app.add_handler(conv)

    logger.info("Бот готов!")

    try:
        app.run_polling()
    except Exception as e:
        logger.error(f"Ошибка запуска: {e}")

if __name__ == '__main__':
    main()