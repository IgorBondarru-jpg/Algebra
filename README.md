import logging
import random
import asyncio
import sys
import os
from threading import Lock
from typing import Dict, Any, Optional, List, Tuple
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes, ConversationHandler

# --- Настройка расширенного логирования ---
def setup_logging():
    """Настраивает расширенное логирование"""
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    
    # Форматтер с подробной информацией
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - [%(filename)s:%(lineno)d] - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    
    # Обработчик для консоли
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    
    # Обработчик для файла
    file_handler = logging.FileHandler('math_bot.log', encoding='utf-8')
    file_handler.setFormatter(formatter)
    
    # Добавляем обработчики
    logger.addHandler(console_handler)
    logger.addHandler(file_handler)
    
    return logger

# Инициализация логгера
logger = setup_logging()

# --- Конфигурация и Токен ---
BOT_TOKEN = os.getenv('BOT_TOKEN', "8013845766:AAG3TNwTVxDNH42443ldVBe0uYgY33GlzJQ")

# Состояния для ConversationHandler
CHOOSING_GRADE, SOLVING_PROBLEMS = range(2)

# --- База данных задач по классам ---
LESSONS = {
    "grade_1_2": {
        "name": "1-2 класс: Сложение и вычитание",
        "tasks": [
            {"question": "x + 5 = 8", "answer": "3", "hint": "Чтобы найти x, нужно из 8 вычесть 5: x = 8 - 5 = 3"},
            {"question": "x - 3 = 7", "answer": "10", "hint": "Чтобы найти x, нужно к 7 прибавить 3: x = 7 + 3 = 10"},
            {"question": "9 - x = 4", "answer": "5", "hint": "Чтобы найти x, нужно из 9 вычесть 4: x = 9 - 4 = 5"},
            {"question": "x + 7 = 15", "answer": "8", "hint": "Чтобы найти x, нужно из 15 вычесть 7: x = 15 - 7 = 8"},
            {"question": "12 - x = 8", "answer": "4", "hint": "Чтобы найти x, нужно из 12 вычесть 8: x = 12 - 8 = 4"}
        ]
    },
    "grade_3_4": {
        "name": "3-4 класс: Умножение и деление",
        "tasks": [
            {"question": "3 × x = 15", "answer": "5", "hint": "Чтобы найти x, нужно 15 разделить на 3: x = 15 ÷ 3 = 5"},
            {"question": "x ÷ 4 = 3", "answer": "12", "hint": "Чтобы найти x, нужно 3 умножить на 4: x = 3 × 4 = 12"},
            {"question": "24 ÷ x = 6", "answer": "4", "hint": "Чтобы найти x, нужно 24 разделить на 6: x = 24 ÷ 6 = 4"},
            {"question": "x × 7 = 42", "answer": "6", "hint": "Чтобы найти x, нужно 42 разделить на 7: x = 42 ÷ 7 = 6"},
            {"question": "36 ÷ x = 9", "answer": "4", "hint": "Чтобы найти x, нужно 36 разделить на 9: x = 36 ÷ 9 = 4"}
        ]
    },
    "grade_5_6": {
        "name": "5-6 класс: Дроби и пропорции",
        "tasks": [
            {"question": "x/3 = 4", "answer": "12", "hint": "Чтобы найти x, нужно 4 умножить на 3: x = 4 × 3 = 12"},
            {"question": "2x/5 = 6", "answer": "15", "hint": "Сначала умножаем обе части на 5: 2x = 30, затем делим на 2: x = 15"},
            {"question": "(x + 3)/2 = 5", "answer": "7", "hint": "Умножаем обе части на 2: x + 3 = 10, вычитаем 3: x = 7"},
            {"question": "3/4 × x = 9", "answer": "12", "hint": "Чтобы найти x, нужно 9 разделить на 3/4: x = 9 ÷ 3/4 = 9 × 4/3 = 12"},
            {"question": "x/2 + 3 = 7", "answer": "8", "hint": "Вычитаем 3: x/2 = 4, умножаем на 2: x = 8"}
        ]
    },
    "grade_7": {
        "name": "7 класс: Линейные уравнения",
        "tasks": [
            {"question": "2x + 5 = 13", "answer": "4", "hint": "Вычитаем 5: 2x = 8, делим на 2: x = 4"},
            {"question": "3(x - 4) = 15", "answer": "9", "hint": "Делим на 3: x - 4 = 5, прибавляем 4: x = 9"},
            {"question": "5x - 7 = 3x + 5", "answer": "6", "hint": "Переносим: 5x - 3x = 5 + 7, 2x = 12, x = 6"},
            {"question": "4 - 2x = 10", "answer": "-3", "hint": "Переносим: -2x = 10 - 4, -2x = 6, x = -3"},
            {"question": "(2x + 1)/3 = 3", "answer": "4", "hint": "Умножаем на 3: 2x + 1 = 9, вычитаем 1: 2x = 8, x = 4"}
        ]
    },
    "grade_8_9": {
        "name": "8-9 класс: Квадратные уравнения",
        "tasks": [
            {"question": "x² - 9 = 0", "answer": "3,-3", "hint": "x² = 9, x = √9 = ±3 (два корня: 3 и -3)"},
            {"question": "x² + 5x + 6 = 0", "answer": "-2,-3", "hint": "Раскладываем на множители: (x + 2)(x + 3) = 0, x = -2 или x = -3"},
            {"question": "2x² - 8x = 0", "answer": "0,4", "hint": "Выносим общий множитель: 2x(x - 4) = 0, x = 0 или x = 4"},
            {"question": "x² - 4x - 5 = 0", "answer": "5,-1", "hint": "Дискриминант D = 16 + 20 = 36, x = (4 ± 6)/2, x₁ = 5, x₂ = -1"},
            {"question": "3x² - 12 = 0", "answer": "2,-2", "hint": "Делим на 3: x² - 4 = 0, x² = 4, x = ±2"}
        ]
    }
}

# Потокобезопасное хранилище статистики
class UserStatsManager:
    def __init__(self):
        self._user_stats = {}
        self._lock = Lock()
        logger.info("📊 Менеджер статистики инициализирован")
    
    def get_user_stats(self, user_id: int) -> Dict[str, Any]:
        """Получает статистику пользователя (потокобезопасно)"""
        with self._lock:
            if user_id not in self._user_stats:
                self._user_stats[user_id] = {
                    'total_problems': 0,
                    'correct_answers': 0,
                    'grades_accuracy': {},
                    'sessions_completed': 0,
                    'user_name': None,
                    'current_streak': 0,
                    'max_streak': 0,
                    'last_active': None
                }
                logger.debug(f"📝 Создана новая статистика для пользователя {user_id}")
            return self._user_stats[user_id].copy()
    
    def update_user_stats(self, user_id: int, user_name: str, total_problems: int, 
                         correct_answers: int, grade_name: str, accuracy: float, 
                         sessions_completed: int, is_correct: bool = False) -> None:
        """Обновляет статистику пользователя (потокобезопасно)"""
        with self._lock:
            try:
                if user_id not in self._user_stats:
                    self._user_stats[user_id] = {
                        'total_problems': 0,
                        'correct_answers': 0,
                        'grades_accuracy': {},
                        'sessions_completed': 0,
                        'user_name': user_name,
                        'current_streak': 0,
                        'max_streak': 0,
                        'last_active': None
                    }
                
                stats = self._user_stats[user_id]
                old_total = stats['total_problems']
                old_correct = stats['correct_answers']
                
                stats['total_problems'] += total_problems
                stats['correct_answers'] += correct_answers
                
                # Обновляем серии правильных ответов
                if is_correct:
                    stats['current_streak'] += 1
                    stats['max_streak'] = max(stats['max_streak'], stats['current_streak'])
                else:
                    stats['current_streak'] = 0
                
                stats['grades_accuracy'][grade_name] = accuracy
                stats['sessions_completed'] += sessions_completed
                stats['user_name'] = user_name
                stats['last_active'] = asyncio.get_event_loop().time()
                
                logger.info(f"📊 Обновлена статистика пользователя {user_id}: "
                          f"+{total_problems} задач, +{correct_answers} правильных, "
                          f"серия: {stats['current_streak']}")
                          
            except Exception as e:
                logger.error(f"❌ Ошибка при обновлении статистики пользователя {user_id}: {e}")
                raise
    
    def get_all_stats(self) -> Dict[int, Dict[str, Any]]:
        """Получает копию всей статистики (для отладки)"""
        with self._lock:
            logger.debug("📋 Запрошена полная статистика")
            return self._user_stats.copy()

# Глобальный менеджер статистики
user_stats_manager = UserStatsManager()

class BotMessages:
    """Класс для хранения текстовых сообщений бота"""
    
    WELCOME = (
        "👋 Привет, {user_name}!\n\n"
        "🤖 Я - математический бот для изучения уравнений!\n"
        "От простых задач начальной школы до сложных уравнений 9 класса.\n\n"
        "🎯 Выбери, с какого уровня начать:"
    )
    
    HELP = """
📖 <b>Помощь по использованию бота</b>

🔹 <b>Основные команды:</b>
/start - начать работу с ботом
/help - показать эту справку  
/menu - показать главное меню
/problems - начать решать задачи
/stats - показать статистику

🔹 <b>Как решать задачи:</b>
1. Выберите класс из предложенных вариантов
2. Решайте уравнения, вводя ответы
3. После каждого ответа появляется новая задача
4. Для помощи нажмите "💡 Подсказка"
5. Для смены класса нажмите "🔄 Сменить класс"

🔹 <b>Формат ответов:</b>
• Простые уравнения: число (5)
• Квадратные уравнения: через запятую ("2,3" или "2,-3")
• Дроби: десятичные (0.5) или обычные (1/2)
• Отрицательные числа: с минусом (-3)

💡 <b>Совет:</b> Не бойтесь ошибаться! Каждая ошибка - это шаг к пониманию.
    """
    
    MENU = """
📋 <b>Главное меню команд</b>

🎯 <b>Основные команды:</b>
/start - Начать работу с ботом
/menu - Показать это меню
/help - Полная справка
/problems - Начать решать задачи  
/stats - Показать статистику

📚 <b>Уровни сложности:</b>
• 1-2 класс: Сложение и вычитание
• 3-4 класс: Умножение и деление
• 5-6 класс: Дроби и пропорции
• 7 класс: Линейные уравнения
• 8-9 класс: Квадратные уравнения

⭐ <b>Особенности:</b>
• Подсказки к каждой задаче
• Подробная статистика прогресса
• Рекомендации по улучшению
• Серии правильных ответов
    """
    
    ERROR_GENERIC = "❌ Произошла непредвиденная ошибка. Попробуйте снова."
    ERROR_NO_ACTIVE_TASK = "❌ Нет активной задачи. Используйте /start для начала."
    ERROR_TASKS_COMPLETED = "✅ Все задачи завершены! Используйте /start для нового раздела."

# --- Улучшенная проверка ответов ---
class AnswerValidator:
    """Класс для проверки правильности ответов"""
    
    @staticmethod
    def normalize_answer(answer: str) -> str:
        """Нормализует ответ: убирает пробелы, заменяет символы"""
        return answer.replace(' ', '').replace('х', 'x').replace(',', '.').lower().strip()
    
    @staticmethod
    def parse_number(value: str) -> Optional[float]:
        """Парсит число из строки с обработкой ошибок"""
        try:
            # Безопасная оценка выражений
            if any(op in value for op in ['/', '*', '+', '-']):
                # Запрещаем опасные операции
                if any(danger in value for danger in ['import', 'exec', 'eval', '__']):
                    return None
                return eval(value)
            return float(value)
        except (ValueError, SyntaxError, ZeroDivisionError, NameError):
            return None
    
    @staticmethod
    def compare_single_answers(user_ans: str, correct_ans: str) -> bool:
        """Сравнивает одиночные ответы"""
        user_norm = AnswerValidator.normalize_answer(user_ans)
        correct_norm = AnswerValidator.normalize_answer(correct_ans)
        
        # Прямое сравнение строк
        if user_norm == correct_norm:
            return True
        
        # Попытка численного сравнения
        user_num = AnswerValidator.parse_number(user_norm)
        correct_num = AnswerValidator.parse_number(correct_norm)
        
        if user_num is not None and correct_num is not None:
            return abs(user_num - correct_num) < 0.0001
        
        return False
    
    @staticmethod
    def compare_multiple_answers(user_ans: str, correct_ans: str) -> bool:
        """Сравнивает ответы с несколькими корнями"""
        try:
            user_parts = [AnswerValidator.normalize_answer(ans) for ans in user_ans.split(',')]
            correct_parts = [AnswerValidator.normalize_answer(ans) for ans in correct_ans.split(',')]
            
            if len(user_parts) != len(correct_parts):
                return False
            
            # Сортируем для сравнения (для квадратных уравнений порядок может быть любым)
            user_sorted = sorted(user_parts)
            correct_sorted = sorted(correct_parts)
            
            for u, c in zip(user_sorted, correct_sorted):
                if not AnswerValidator.compare_single_answers(u, c):
                    return False
            return True
            
        except Exception as e:
            logger.warning(f"⚠️ Ошибка при сравнении множественных ответов: {e}")
            return False
    
    @staticmethod
    def is_correct_answer(user_answer: str, correct_answer: str) -> Tuple[bool, str]:
        """Проверяет правильность ответа с возвратом диагностики"""
        try:
            user_clean = user_answer.strip()
            correct_clean = correct_answer.strip()
            
            if not user_clean:
                return False, "Пустой ответ"
            
            # Для ответов с несколькими корнями
            if ',' in correct_clean:
                is_correct = AnswerValidator.compare_multiple_answers(user_clean, correct_clean)
                return is_correct, "Множественные корни"
            
            # Для одиночных ответов
            is_correct = AnswerValidator.compare_single_answers(user_clean, correct_clean)
            return is_correct, "Одиночный ответ"
            
        except Exception as e:
            logger.error(f"❌ Ошибка при проверке ответа: {e}")
            return False, f"Ошибка проверки: {e}"

# --- Обработчики ошибок ---
async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обрабатывает ошибки в боте"""
    try:
        # Логируем ошибку
        logger.error(f"🚨 Ошибка в обработчике: {context.error}", exc_info=context.error)
        
        # Отправляем сообщение пользователю
        if update and update.effective_message:
            await update.effective_message.reply_text(
                BotMessages.ERROR_GENERIC,
                parse_mode='HTML'
            )
    except Exception as e:
        logger.critical(f"💥 Критическая ошибка в обработчике ошибок: {e}")

# --- Универсальные обработчики команд ---

async def universal_start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Универсальный обработчик команды /start"""
    try:
        user = update.effective_user
        logger.info(f"🚀 Команда /start от пользователя {user.id} ({user.first_name})")
        
        keyboard = [
            [InlineKeyboardButton(LESSONS["grade_1_2"]["name"], callback_data="grade_1_2")],
            [InlineKeyboardButton(LESSONS["grade_3_4"]["name"], callback_data="grade_3_4")],
            [InlineKeyboardButton(LESSONS["grade_5_6"]["name"], callback_data="grade_5_6")],
            [InlineKeyboardButton(LESSONS["grade_7"]["name"], callback_data="grade_7")],
            [InlineKeyboardButton(LESSONS["grade_8_9"]["name"], callback_data="grade_8_9")],
            [InlineKeyboardButton("📊 Моя статистика", callback_data="show_stats")],
            [InlineKeyboardButton("❓ Помощь", callback_data="help_from_menu")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)

        await update.message.reply_html(
            BotMessages.WELCOME.format(user_name=user.mention_html()),
            reply_markup=reply_markup,
        )
        
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в universal_start: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)
        return ConversationHandler.END

async def universal_help(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Универсальный обработчик команды /help"""
    try:
        logger.info(f"📖 Команда /help от пользователя {update.effective_user.id}")
        
        keyboard = [
            [InlineKeyboardButton("🚀 Начать решать", callback_data="change_grade")],
            [InlineKeyboardButton("📋 Главное меню", callback_data="menu_from_query")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await update.message.reply_text(BotMessages.HELP, reply_markup=reply_markup, parse_mode='HTML')
        
    except Exception as e:
        logger.error(f"❌ Ошибка в universal_help: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)

async def universal_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Универсальный обработчик команды /menu"""
    try:
        logger.info(f"📋 Команда /menu от пользователя {update.effective_user.id}")
        
        keyboard = [
            [InlineKeyboardButton("🚀 Начать решать задачи", callback_data="change_grade")],
            [InlineKeyboardButton("📊 Моя статистика", callback_data="show_stats")],
            [InlineKeyboardButton("❓ Помощь", callback_data="help_from_menu")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await update.message.reply_text(BotMessages.MENU, reply_markup=reply_markup, parse_mode='HTML')
        
    except Exception as e:
        logger.error(f"❌ Ошибка в universal_menu: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)

async def universal_problems(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Универсальный обработчик команды /problems"""
    try:
        logger.info(f"🎯 Команда /problems от пользователя {update.effective_user.id}")
        return await universal_start(update, context)
    except Exception as e:
        logger.error(f"❌ Ошибка в universal_problems: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)
        return ConversationHandler.END

async def universal_stats(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Универсальный обработчик команды /stats"""
    try:
        logger.info(f"📊 Команда /stats от пользователя {update.effective_user.id}")
        await show_user_stats(update, context)
    except Exception as e:
        logger.error(f"❌ Ошибка в universal_stats: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)

# --- Обработчики Callback-запросов ---

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Обрабатывает нажатия кнопок"""
    try:
        query = update.callback_query
        await query.answer()
        
        data = query.data
        logger.info(f"🔘 Нажата кнопка: {data} пользователем {query.from_user.id}")
        
        # Обработка выбора класса
        if data in LESSONS:
            return await handle_grade_selection(update, context, data)
        
        # Обработка других действий
        handlers = {
            "hint": show_hint,
            "change_grade": start_from_query,
            "back_to_task": send_task,
            "show_stats": lambda u, c: show_user_stats(u, c, from_main_menu=True),
            "stats": lambda u, c: show_user_stats(u, c, from_results=True),
            "start_over": start_from_query,
            "back_to_results": lambda u, c: show_final_results(u, c, from_stats=True),
            "help_from_menu": show_help_from_menu,
            "menu_from_query": menu_from_query,
        }
        
        if data in handlers:
            return await handlers[data](update, context)
        
        logger.warning(f"⚠️ Неизвестный callback_data: {data}")
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в button_handler: {e}")
        await update.callback_query.message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def handle_grade_selection(update: Update, context: ContextTypes.DEFAULT_TYPE, grade_key: str) -> int:
    """Обрабатывает выбор класса"""
    try:
        user = update.callback_query.from_user
        logger.info(f"🎓 Пользователь {user.id} выбрал класс: {grade_key}")
        
        context.user_data.update({
            'current_grade': grade_key,
            'task_index': 0,
            'score': 0,
            'problems_solved': 0,
            'current_grade_name': LESSONS[grade_key]["name"]
        })
        
        await send_task(update, context)
        return SOLVING_PROBLEMS
        
    except Exception as e:
        logger.error(f"❌ Ошибка в handle_grade_selection: {e}")
        await update.callback_query.message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def start_from_query(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Запускает выбор класса из callback query"""
    try:
        query = update.callback_query
        logger.info(f"🔄 Пользователь {query.from_user.id} запустил выбор класса")
        
        keyboard = [
            [InlineKeyboardButton(LESSONS["grade_1_2"]["name"], callback_data="grade_1_2")],
            [InlineKeyboardButton(LESSONS["grade_3_4"]["name"], callback_data="grade_3_4")],
            [InlineKeyboardButton(LESSONS["grade_5_6"]["name"], callback_data="grade_5_6")],
            [InlineKeyboardButton(LESSONS["grade_7"]["name"], callback_data="grade_7")],
            [InlineKeyboardButton(LESSONS["grade_8_9"]["name"], callback_data="grade_8_9")],
            [InlineKeyboardButton("📊 Моя статистика", callback_data="show_stats")],
            [InlineKeyboardButton("📋 Меню команд", callback_data="menu_from_query")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)

        await query.edit_message_text(
            "🎯 Выбери класс для решения уравнений:",
            reply_markup=reply_markup,
        )
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в start_from_query: {e}")
        await update.callback_query.message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def send_task(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Отправляет задачу из выбранного класса"""
    try:
        query = update.callback_query if hasattr(update, 'callback_query') and update.callback_query else None
        chat_id = update.effective_chat.id
        
        grade_key = context.user_data['current_grade']
        task_index = context.user_data['task_index']
        tasks = LESSONS[grade_key]["tasks"]
        
        if task_index >= len(tasks):
            logger.info(f"✅ Пользователь завершил раздел {grade_key}")
            await show_final_results(update, context)
            return CHOOSING_GRADE
        
        task = tasks[task_index]
        
        keyboard = [
            [InlineKeyboardButton("💡 Подсказка", callback_data="hint")],
            [InlineKeyboardButton("🔄 Сменить класс", callback_data="change_grade")],
            [InlineKeyboardButton("📊 Статистика", callback_data="show_stats")],
            [InlineKeyboardButton("📋 Меню", callback_data="menu_from_query")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        # Создаем прогресс-бар
        progress = "🟩" * task_index + "⬜" * (len(tasks) - task_index)
        
        message_text = (
            f"📚 {LESSONS[grade_key]['name']}\n"
            f"📊 Прогресс: {progress}\n"
            f"📝 Задача {task_index + 1}/{len(tasks)}:\n\n"
            f"🧮 <b>{task['question']}</b>\n\n"
            f"💭 <i>Введите ответ ниже...</i>"
        )
        
        if query:
            await query.edit_message_text(message_text, reply_markup=reply_markup, parse_mode='HTML')
        else:
            await context.bot.send_message(chat_id=chat_id, text=message_text, reply_markup=reply_markup, parse_mode='HTML')
        
        logger.debug(f"📝 Отправлена задача {task_index + 1} для класса {grade_key}")
        return SOLVING_PROBLEMS
        
    except Exception as e:
        logger.error(f"❌ Ошибка в send_task: {e}")
        await update.effective_message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def show_hint(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Показывает подсказку к текущей задаче"""
    try:
        query = update.callback_query
        
        grade_key = context.user_data['current_grade']
        task_index = context.user_data['task_index']
        task = LESSONS[grade_key]["tasks"][task_index]
        
        logger.info(f"💡 Пользователь {query.from_user.id} запросил подсказку для задачи {task_index + 1}")
        
        keyboard = [
            [InlineKeyboardButton("↩️ К задаче", callback_data="back_to_task")],
            [InlineKeyboardButton("🔄 Сменить класс", callback_data="change_grade")],
            [InlineKeyboardButton("📊 Статистика", callback_data="show_stats")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            f"💡 Подсказка к задаче №{task_index + 1}:\n\n"
            f"{task['hint']}\n\n"
            f"🧮 Уравнение: <b>{task['question']}</b>\n\n"
            f"<i>Попробуйте решить или вернитесь к задаче</i>",
            reply_markup=reply_markup,
            parse_mode='HTML'
        )
        return SOLVING_PROBLEMS
        
    except Exception as e:
        logger.error(f"❌ Ошибка в show_hint: {e}")
        await update.callback_query.message.reply_text(BotMessages.ERROR_GENERIC)
        return SOLVING_PROBLEMS

async def show_final_results(update: Update, context: ContextTypes.DEFAULT_TYPE, from_stats=False) -> int:
    """Показывает финальные результаты"""
    try:
        query = update.callback_query if hasattr(update, 'callback_query') and update.callback_query else None
        chat_id = update.effective_chat.id
        
        score = context.user_data.get('score', 0)
        total = context.user_data.get('problems_solved', 0)
        accuracy = (score / total * 100) if total > 0 else 0
        grade_name = context.user_data.get('current_grade_name', '')
        
        logger.info(f"🏁 Пользователь завершил раздел с результатом: {score}/{total} ({accuracy:.1f}%)")
        
        # Обновляем статистику
        if not from_stats:
            user = update.effective_user
            user_stats_manager.update_user_stats(
                user_id=user.id,
                user_name=user.first_name,
                total_problems=total,
                correct_answers=score,
                grade_name=grade_name,
                accuracy=accuracy,
                sessions_completed=1,
                is_correct=False
            )
        
        # Определяем оценку
        if accuracy >= 90:
            grade_emoji, grade_text = "🏆", "Отлично!"
        elif accuracy >= 70:
            grade_emoji, grade_text = "👍", "Хорошо!"
        elif accuracy >= 50:
            grade_emoji, grade_text = "👌", "Неплохо!"
        else:
            grade_emoji, grade_text = "💪", "Продолжайте тренироваться!"
        
        keyboard = [
            [InlineKeyboardButton("🔄 Новый класс", callback_data="change_grade")],
            [InlineKeyboardButton("📊 Статистика", callback_data="stats")],
            [InlineKeyboardButton("📋 Меню", callback_data="menu_from_query")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        message_text = (
            f"🎉 Поздравляю! Задачи раздела решены!\n\n"
            f"📊 <b>Результаты:</b>\n"
            f"• Решено задач: {total}\n"
            f"• Правильных ответов: {score}\n"
            f"• Точность: {accuracy:.1f}%\n"
            f"• Оценка: {grade_emoji} {grade_text}\n\n"
            f"Выберите следующий раздел:"
        )
        
        if query:
            await query.edit_message_text(message_text, reply_markup=reply_markup, parse_mode='HTML')
        else:
            await context.bot.send_message(chat_id=chat_id, text=message_text, reply_markup=reply_markup, parse_mode='HTML')
        
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в show_final_results: {e}")
        await update.effective_message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def show_user_stats(update: Update, context: ContextTypes.DEFAULT_TYPE, from_main_menu=False, from_results=False) -> int:
    """Показывает статистику пользователя"""
    try:
        user = update.effective_user
        stats = user_stats_manager.get_user_stats(user.id)
        
        total_problems = stats['total_problems']
        correct_answers = stats['correct_answers']
        overall_accuracy = (correct_answers / total_problems * 100) if total_problems > 0 else 0
        sessions_completed = stats['sessions_completed']
        current_streak = stats['current_streak']
        max_streak = stats['max_streak']
        
        logger.info(f"📊 Показана статистика пользователя {user.id}")
        
        # Создаем текст статистики
        stats_text = f"📊 <b>Статистика {user.first_name}</b>\n\n"
        
        stats_text += f"📈 <b>Общая статистика:</b>\n"
        stats_text += f"• Всего решено: {total_problems}\n"
        stats_text += f"• Правильных: {correct_answers}\n"
        stats_text += f"• Точность: {overall_accuracy:.1f}%\n"
        stats_text += f"• Сессий: {sessions_completed}\n"
        stats_text += f"• Серия: {current_streak} (рекорд: {max_streak})\n\n"
        
        stats_text += f"🎓 <b>По классам:</b>\n"
        if stats['grades_accuracy']:
            for grade_name, accuracy in stats['grades_accuracy'].items():
                emoji = "🏆" if accuracy >= 90 else "👍" if accuracy >= 70 else "👌" if accuracy >= 50 else "💪"
                stats_text += f"• {grade_name}: {accuracy:.1f}% {emoji}\n"
        else:
            stats_text += "• Пока нет данных\n\n"
        
        # Рекомендации
        stats_text += f"\n💡 <b>Рекомендации:</b>\n"
        if overall_accuracy >= 80:
            stats_text += "Отлично! Попробуйте более сложные классы 🚀"
        elif overall_accuracy >= 60:
            stats_text += "Хорошо! Продолжайте тренироваться 💪"
        else:
            stats_text += "Практика - ключ к успеху! Начните с 1-2 класса 📚"
        
        # Настройка кнопок
        if from_results:
            keyboard = [
                [InlineKeyboardButton("↩️ Назад", callback_data="back_to_results")],
                [InlineKeyboardButton("🎯 Продолжить", callback_data="change_grade")],
            ]
        elif from_main_menu:
            keyboard = [
                [InlineKeyboardButton("🎯 Начать", callback_data="change_grade")],
                [InlineKeyboardButton("🔄 Меню", callback_data="start_over")],
            ]
        else:
            keyboard = [
                [InlineKeyboardButton("🎯 Продолжить", callback_data="change_grade")],
                [InlineKeyboardButton("🔄 Заново", callback_data="start_over")],
            ]
        
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        if hasattr(update, 'callback_query') and update.callback_query:
            await update.callback_query.edit_message_text(stats_text, reply_markup=reply_markup, parse_mode='HTML')
        else:
            await context.bot.send_message(
                chat_id=update.effective_chat.id,
                text=stats_text,
                reply_markup=reply_markup,
                parse_mode='HTML'
            )
        
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в show_user_stats: {e}")
        await update.effective_message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def show_help_from_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Показывает справку из меню"""
    try:
        query = update.callback_query
        logger.info(f"❓ Пользователь {query.from_user.id} запросил справку из меню")
        
        keyboard = [
            [InlineKeyboardButton("🚀 Начать", callback_data="change_grade")],
            [InlineKeyboardButton("📋 Меню", callback_data="menu_from_query")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(BotMessages.HELP, reply_markup=reply_markup, parse_mode='HTML')
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в show_help_from_menu: {e}")
        await update.callback_query.message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def menu_from_query(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Показывает меню из callback query"""
    try:
        query = update.callback_query
        logger.info(f"📋 Пользователь {query.from_user.id} запросил меню")
        
        keyboard = [
            [InlineKeyboardButton("🚀 Начать решать", callback_data="change_grade")],
            [InlineKeyboardButton("📊 Статистика", callback_data="show_stats")],
            [InlineKeyboardButton("❓ Помощь", callback_data="help_from_menu")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(BotMessages.MENU, reply_markup=reply_markup, parse_mode='HTML')
        return CHOOSING_GRADE
        
    except Exception as e:
        logger.error(f"❌ Ошибка в menu_from_query: {e}")
        await update.callback_query.message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

# --- Обработчик ответов на задачи ---

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Обрабатывает текстовые ответы на задачи"""
    try:
        user = update.effective_user
        user_answer = update.message.text.strip()
        
        logger.info(f"📝 Ответ от пользователя {user.id}: '{user_answer}'")

        # Проверяем активную задачу
        if 'current_grade' not in context.user_data or 'task_index' not in context.user_data:
            logger.warning(f"⚠️ Нет активной задачи у пользователя {user.id}")
            await update.message.reply_text(BotMessages.ERROR_NO_ACTIVE_TASK)
            return CHOOSING_GRADE

        grade_key = context.user_data['current_grade']
        task_index = context.user_data['task_index']
        
        if task_index >= len(LESSONS[grade_key]["tasks"]):
            logger.info(f"✅ Пользователь {user.id} завершил все задачи раздела {grade_key}")
            await update.message.reply_text(BotMessages.ERROR_TASKS_COMPLETED)
            return CHOOSING_GRADE
            
        correct_answer = LESSONS[grade_key]["tasks"][task_index]["answer"]
        
        # Проверяем ответ с улучшенным валидатором
        is_correct, check_type = AnswerValidator.is_correct_answer(user_answer, correct_answer)
        
        logger.info(f"🔍 Проверка ответа пользователя {user.id}: "
                   f"правильный={is_correct}, тип={check_type}, "
                   f"ответ='{user_answer}', эталон='{correct_answer}'")

        # Обновляем счетчики
        context.user_data['problems_solved'] = context.user_data.get('problems_solved', 0) + 1
        
        if is_correct:
            context.user_data['score'] = context.user_data.get('score', 0) + 1
            result_message = (
                f"✅ <b>Верно!</b>\n"
                f"Ответ: <b>{correct_answer}</b>\n\n"
                f"📊 Прогресс: {context.user_data['score']}✅ из {context.user_data['problems_solved']} задач"
            )
            result_emoji = "🎉"
            
            # Добавляем сообщение о серии, если есть
            user_stats = user_stats_manager.get_user_stats(user.id)
            if user_stats['current_streak'] > 1:
                result_message += f"\n🔥 Серия правильных ответов: {user_stats['current_streak']}"
                
        else:
            result_message = (
                f"❌ <b>Пока нет.</b>\n"
                f"Ваш ответ: {user_answer}\n"
                f"Правильный: <b>{correct_answer}</b>\n\n"
                f"📊 Прогресс: {context.user_data['score']}✅ из {context.user_data['problems_solved']} задач"
            )
            result_emoji = "💪"

        # Отправляем результат
        await update.message.reply_text(
            f"{result_emoji} {result_message}",
            parse_mode='HTML'
        )

        # Обновляем статистику серии
        user_stats_manager.update_user_stats(
            user_id=user.id,
            user_name=user.first_name,
            total_problems=1,
            correct_answers=1 if is_correct else 0,
            grade_name=context.user_data.get('current_grade_name', ''),
            accuracy=0,
            sessions_completed=0,
            is_correct=is_correct
        )

        # Небольшая пауза перед следующей задачей
        await asyncio.sleep(1)

        # Следующая задача
        context.user_data['task_index'] += 1
        await send_task(update, context)
        
        return SOLVING_PROBLEMS
        
    except Exception as e:
        logger.error(f"❌ Ошибка в handle_message: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)
        return CHOOSING_GRADE

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    """Отменяет текущую операцию"""
    try:
        user = update.effective_user
        logger.info(f"🛑 Пользователь {user.id} отменил операцию")
        
        await update.message.reply_text(
            "Решение задач прервано. Используйте /start чтобы начать заново!"
        )
        return ConversationHandler.END
        
    except Exception as e:
        logger.error(f"❌ Ошибка в cancel: {e}")
        await update.message.reply_text(BotMessages.ERROR_GENERIC)
        return ConversationHandler.END

def main() -> None:
    """Запуск бота с улучшенной обработкой ошибок"""
    try:
        logger.info("🤖 Запуск математического бота...")
        
        # Проверка токена
        if not BOT_TOKEN or BOT_TOKEN == "your_bot_token_here":
            logger.critical("❌ Токен бота не настроен!")
            return

        application = Application.builder().token(BOT_TOKEN).build()

        # Добавляем обработчик ошибок
        application.add_error_handler(error_handler)

        # Обычные обработчики команд
        application.add_handler(CommandHandler("help", universal_help))
        application.add_handler(CommandHandler("menu", universal_menu))
        application.add_handler(CommandHandler("problems", universal_problems))
        application.add_handler(CommandHandler("stats", universal_stats))
        
        # ConversationHandler
        conv_handler = ConversationHandler(
            entry_points=[CommandHandler("start", universal_start)],
            states={
                CHOOSING_GRADE: [
                    CallbackQueryHandler(
                        button_handler, 
                        pattern="^(grade_1_2|grade_3_4|grade_5_6|grade_7|grade_8_9|change_grade|show_stats|start_over|back_to_results|help_from_menu|menu_from_query)$"
                    )
                ],
                SOLVING_PROBLEMS: [
                    CallbackQueryHandler(
                        button_handler, 
                        pattern="^(hint|change_grade|back_to_task|show_stats|menu_from_query)$"
                    ),
                    MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message)
                ],
            },
            fallbacks=[CommandHandler("cancel", cancel)],
        )
        
        application.add_handler(conv_handler)

        # Запуск бота
        logger.info("✅ Математический бот успешно запущен!")
        logger.info("📚 Доступные команды: /start, /menu, /help, /problems, /stats")
        logger.info("🔒 Безопасная проверка ответов активирована")
        logger.info("📝 Подробное логирование включено")
        
        application.run_polling(
            allowed_updates=Update.ALL_TYPES,
            drop_pending_updates=True
        )
        
    except Exception as e:
        logger.critical(f"💥 Критическая ошибка при запуске бота: {e}")
    finally:
        logger.info("🛑 Бот остановлен")

if __name__ == '__main__':
    main()
