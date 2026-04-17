import logging
import random                                                                                     
import asyncio
from typing import Dict, List, Optional, Set
from enum import Enum
from dataclasses import dataclass, field

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ChatMemberAdministrator, ChatMemberOwner
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler, 
    ContextTypes, MessageHandler, filters, ChatMemberHandler
)

# ============ НАСТРОЙКИ ============
BOT_TOKEN = "8717912717:AAFDFk-C3s7BEtNUNwj7_IdjmNIx0Z6rk-I"

REGISTRATION_TIME = 180
NIGHT_TIME = 75
DAY_MIN_TIME = 180
DAY_MAX_TIME = 240
VOTING_TIME = 45

logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# ============ СИМВОЛЫ ============
class S:
    SNAKE = "🐍"
    SKULL = "💀"
    CROWN = "👑"
    DAGGER = "🗡️"
    SHIELD = "🛡️"
    POTION = "🧪"
    WAND = "✨"
    DROP = "💧"
    MOON = "🌙"
    SUN = "☀️"
    STAR = "⭐"
    FIRE = "🔥"
    ROSE = "🥀"
    OWL = "🦉"
    KEY = "🗝️"
    SCROLL = "📜"
    HOURGLASS = "⏳"
    CANDLE = "🕯️"
    FOG = "🌫️"
    MASK = "🎭"
    SCALE = "⚖️"
    BLOOD = "🩸"
    FEATHER = "🪶"
    CRYSTAL = "🔮"
    LOCK = "🔒"
    DOOR = "🚪"
    GHOST = "👻"
    SPARKLES = "✨"

class Role(Enum):
    DARK_LORD = "Тёмный Лорд"
    DEATH_EATER = "Пожиратель Смерти"
    HEALER = "Целитель"
    PUREBLOOD = "Чистокровный"
    HALFBLOOD = "Полукровка"
    ORDER = "Орден Феникса"

class GamePhase(Enum):
    WAITING = "waiting"
    NIGHT = "night"
    DAY = "day"
    VOTING = "voting"
    ENDED = "ended"

@dataclass
class Player:
    user_id: int
    username: str
    first_name: str
    role: Optional[Role] = None
    is_alive: bool = True
    is_protected: bool = False
    has_used_ability: bool = False
    voted_for: Optional[int] = None
    night_target: Optional[int] = None
    pureblood_used: bool = False

@dataclass
class Game:
    chat_id: int
    phase: GamePhase = GamePhase.WAITING
    players: Dict[int, Player] = field(default_factory=dict)
    day_count: int = 0
    killed_tonight: Optional[Player] = None
    registration_msg_id: Optional[int] = None
    day_timer_task: Optional[asyncio.Task] = None
    rules_message_id: Optional[int] = None

    def get_alive_players(self) -> List[Player]:
        return [p for p in self.players.values() if p.is_alive]
    
    def get_role_players(self, role: Role) -> List[Player]:
        return [p for p in self.get_alive_players() if p.role == role]
    
    def get_dark_side(self) -> List[Player]:
        return [p for p in self.get_alive_players() 
                if p.role in [Role.DARK_LORD, Role.DEATH_EATER]]

games: Dict[int, Game] = {}
chat_admins: Dict[int, Set[int]] = {}

# ============ УТИЛИТЫ ============
def italic(text: str) -> str:
    return f"_{text}_"

def bold(text: str) -> str:
    return f"*{text}*"

def quote(text: str, author: str = "") -> str:
    if author:
        return f'«{text}»\n— *{author}*'
    return f'«{text}»'

async def delete_message_after(message, seconds: int = 5):
    """Удаляет сообщение через указанное время"""
    await asyncio.sleep(seconds)
    try:
        await message.delete()
    except:
        pass

async def delete_command(update: Update):
    """Удаляет команду пользователя"""
    try:
        await update.message.delete()
    except:
        pass

# ← ИСПРАВЛЕНО: корректная проверка админа с обработкой пустого списка
def is_admin_chat(user_id: int, chat_id: int) -> bool:
    """Проверяет, является ли пользователь админом чата"""
    # Если список админов пустой, пробуем разрешить (для обратной совместимости)
    # или можно вернуть False для строгой безопасности
    if chat_id not in chat_admins:
        logger.warning(f"Admin list not loaded for chat {chat_id}")
        # Вариант 1: Строгий режим — только если точно знаем, что это админ
        # return False
        # Вариант 2: Разрешительный режим — если список не загружен, проверяем через API
        return None  # Вернём None, чтобы вызывающий код мог решить
    return user_id in chat_admins[chat_id]

# ← ДОБАВЛЕНО: асинхронная проверка админа через API если нет в кеше
async def check_is_admin(update: Update, context: ContextTypes.DEFAULT_TYPE) -> bool:
    """Проверяет, является ли пользователь админом, с загрузкой из API при необходимости"""
    user_id = update.effective_user.id
    chat_id = update.effective_chat.id
    
    # Проверяем кеш
    if chat_id in chat_admins:
        return user_id in chat_admins[chat_id]
    
    # Загружаем из API
    try:
        admins = await context.bot.get_chat_administrators(chat_id)
        admin_ids = {admin.user.id for admin in admins if isinstance(admin, (ChatMemberAdministrator, ChatMemberOwner))}
        chat_admins[chat_id] = admin_ids
        return user_id in admin_ids
    except Exception as e:
        logger.error(f"Failed to load admins: {e}")
        return False

# ============ ЛОР РОЛЕЙ ============
ROLE_LORE = {
    Role.DARK_LORD: {
        "title": "Тёмный Лорд",
        "emoji": "👑",
        "quote": ("Я вернулся. Сильнее, чем прежде.", "Том Реддл"),
        "lore": "Верховный повелитель Пожирателей Смерти. Носитель Метки, сеятель страха, тот, чьё имя нельзя произносить вслух. Видит всех своих слуг через связь Метки.",
        "night_action": "Выбирает жертву для заклятия смерти. Решение окончательно и бесповоротно.",
        "secret_power": "Обладает тайным каналом связи с Пожирателями. Их разумы связаны Меткой.",
        "victory_condition": "Убить шпиона Ордена Феникса ИЛИ добиться равенства сил.",
        "risk": "Если Тёмный Лорд погибает — его сторона проигрывает немедленно.",
        "team": "Тёмная Сторона",
        "color": "зелёный"
    },
    Role.DEATH_EATER: {
        "title": "Пожиратель Смерти",
        "emoji": "💀",
        "quote": ("Мы были верны до последнего вздоха.", "Беллатриса Лестрейндж"),
        "lore": "Закалённый в боях маг, носящий Метку на левой руке. Готов отдать жизнь за своего повелителя. Видит Тёмного Лорда и других Пожирателей.",
        "night_action": "Может предложить жертву Лорду, но последнее слово всегда за ним.",
        "secret_power": "Тайная переписка с союзниками ночью. Защищает Лорда ценой собственной жизни.",
        "victory_condition": "Помочь Лорду победить любой ценой.",
        "risk": "Если Лорд погибает — вы все проиграете.",
        "team": "Тёмная Сторона",
        "color": "зелёный"
    },
    Role.HEALER: {
        "title": "Целитель",
        "emoji": "🧪",
        "quote": ("Исцеление — это искусство, требующее мудрости, а не только знаний.", "Мадам Помфри"),
        "lore": "Мастер зельеварения и целительных чар, хранитель жизней невинных. Его флаконы содержат эликсиры, способные отразить даже смертельное проклятие.",
        "night_action": "Выбирает одного жителя для защиты. Этот человек выживет, даже если Лорд выберет его жертвой.",
        "secret_power": "Нельзя защищать себя две ночи подряд — магия требует баланса.",
        "victory_condition": "Спасти как можно больше жизней и помочь изгнать Тёмных.",
        "risk": "Если защищаете другого, то не защищаете себя.",
        "team": "Светлая Сторона",
        "color": "синий"
    },
    Role.ORDER: {
        "title": "Орден Феникса",
        "emoji": "🔥",
        "quote": ("Счастье можно найти даже в самые тёмные времена, если не забывать обращаться к свету.", "Альбус Дамблдор"),
        "lore": "Тайный шпион Дамблдора, внедрённый среди слизеринцев. Обладает даром распознавать Пожирателей Смерти по их магической ауре.",
        "night_action": "Проверяет одного игрока: Пожиратель Смерти или нет.",
        "secret_power": "*Внимание:* Тёмного Лорда видит как обычного жителя! Его магия слишком сильна для ваших чар.",
        "victory_condition": "Найти всех Пожирателей и помочь их изгнать.",
        "risk": "Если вас убьют — Светлые потеряют свой главный инструмент.",
        "team": "Светлая Сторона",
        "color": "золотой"
    },
    Role.PUREBLOOD: {
        "title": "Чистокровный",
        "emoji": "💧",
        "quote": ("Сила кроется в крови, унаследованной от веков.", "Салазар Слизерин"),
        "lore": "Гордый носитель древней магии чистых кровей. Никто не подозревает, что в его жилах течёт сила, способная обмануть саму Смерть.",
        "night_action": "Спит, как и все обычные жители. Но его кровь хранит тайну.",
        "secret_power": "*Кость Судьбы:* Если изгоняют на голосовании — бросает кость. Шанс выжить: 10-50%. Если успех — голосование тайно отменяется, никто не узнает о спасении.",
        "victory_condition": "Выжить и помочь изгнать Тёмных.",
        "risk": "Если кость не сработает — вы погибаете как обычный житель.",
        "team": "Светлая Сторона",
        "color": "серебряный"
    },
    Role.HALFBLOOD: {
        "title": "Полукровка",
        "emoji": "⭐",
        "quote": ("Может, важно не то, как мы рождены, а то, каким нас делает наш выбор.", "Северус Снейп"),
        "lore": "Умный и проницательный, чей разум острее любого заклятия. Доказал, что происхождение не определяет достоинство. Его голос может перевернуть судьбы.",
        "night_action": "Ночью беззащитен, но днём его слово весит больше золота.",
        "secret_power": "Логика и интуиция — единственное оружие. Найдите ложь в словах Тёмных.",
        "victory_condition": "Разоблачить Пожирателей через дискуссию. Выжить любой ценой.",
        "risk": "Без ночных способностей вы уязвимы.",
        "team": "Светлая Сторона",
        "color": "бронзовый"
    }
}

def get_role_card(role: Role) -> str:
    info = ROLE_LORE[role]
    s = S
    
    card = f"""
{info['emoji']} *{info['title'].upper()}* {info['emoji']}

{quote(*info['quote'])}

{italic(info['lore'])}

{s.WAND} *Ночное действие:*
{info['night_action']}

{s.KEY} *Тайная сила:*
{info['secret_power']}

{s.CROWN} *Путь к победе:*
{info['victory_condition']}

{s.SKULL} *Опасность:*
{info['risk']}

{s.SHIELD} *Сторона:* {info['team']}
{s.DROP} *Цвет:* {info['color']}
"""
    return card

# ============ РАСПРЕДЕЛЕНИЕ РОЛЕЙ ============
def distribute_roles(count: int) -> List[Role]:
    if count < 6:
        raise ValueError("Минимум 6 игроков")
    
    roles = [Role.DARK_LORD, Role.DEATH_EATER, Role.HEALER, Role.ORDER]
    remaining = count - 4
    pure_count = remaining // 2
    half_count = remaining - pure_count
    
    roles.extend([Role.PUREBLOOD] * pure_count)
    roles.extend([Role.HALFBLOOD] * half_count)
    
    if count >= 10:
        roles.append(Role.DEATH_EATER)
        roles.remove(Role.PUREBLOOD)
    
    if count >= 15:
        roles.append(Role.HEALER)
        roles.remove(Role.HALFBLOOD)
    
    random.shuffle(roles)
    return roles

# ============ КОМАНДЫ ============
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    
    text = f"""
{s.SNAKE} *ТЕНИ ХОГВАРТСА* {s.SNAKE}

{quote("Амбиция — это мечта тех, кто не боится грязи.", "Салазар Слизерин")}

{s.FOG} *Тайная игра в подземельях замка...*

В эти стены проникла Тьма. Среди достойных Слизерина скрываются Пожиратели Смерти, готовые уничтожить всех, кто встанет на пути Тёмного Лорда. Но есть и те, кто противостоит им — Целители, способные спасти от смерти, и тайный Орден Феникса, ищущий врагов в тени.

*Ты готов войти в Тени?*

{s.DAGGER} */new_game* — открыть Тайную комнату
{s.OWL} */join* — принять приглашение
{s.SCROLL} */players* — узнать, кто уже здесь
{s.KEY} */rules* — изучить законы подземелий
{s.HOURGLASS} */status* — узнать ход игры

{s.ROSE} _Игры проводятся только по выходным_
{s.SNAKE} _Только для тех, кто не боится тьмы_
"""
    await update.message.reply_text(text, parse_mode='Markdown')
    await delete_command(update)

async def rules(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    
    chat_id = update.effective_chat.id
    
    keyboard = [
        [
            InlineKeyboardButton(f"👑 Тёмный Лорд", callback_data="role_dark_lord"),
            InlineKeyboardButton(f"💀 Пожиратель", callback_data="role_death_eater")
        ],
        [
            InlineKeyboardButton(f"🧪 Целитель", callback_data="role_healer"),
            InlineKeyboardButton(f"🔥 Орден", callback_data="role_order")
        ],
        [
            InlineKeyboardButton(f"💧 Чистокровный", callback_data="role_pureblood"),
            InlineKeyboardButton(f"⭐ Полукровка", callback_data="role_halfblood")
        ]
    ]
    
    text = f"""
{s.SNAKE} *ЗАКОНЫ ПОДЗЕМЕЛИЙ* {s.SNAKE}

{quote("Тьма возвращается, и нам нужно выбрать сторону...", "Альбус Дамблдор")}

{s.MOON} *Как ведётся игра в Тенях*

Каждый цикл состоит из Ночи и Дня. Ночью маги действуют тайно, днём — открыто. Никто не знает наверняка, кто есть кто. Все врут, все подозревают, доверие — роскошь, которую никто не может себе позволить.

{s.CANDLE} *Ночь — время теней*
Подземелья погружаются во мрак. Пока честные жители спят, пробуждаются Тёмные.
• *Тёмный Лорд* выбирает жертву для заклятия смерти
• *Целитель* тайно защищает одного от гибели
• *Орден Феникса* проверяет подозреваемого
• *Пожиратели* советуют Лорду, кто будет следующей жертвой

{s.SUN} *День — время правосудия*
Хогвартс просыпается. Если кто-то пал ночью — все узнают об этом. Живые собираются в Большом зале, но вместо завтрака — обсуждение. Кто врёт? Кто действительно видел то, о чём говорит? Кто слишком быстро обвиняет других?

{s.SCALE} *Суд — время голосования*
После обсуждения наступает час расплаты. Каждый живой отдаёт голос против того, кого подозревает. Тот, кто наберёт больше всего голосов, *изгоняется из Хогвартса*. Его роль открывается всем — было ли подозрение верным?

{s.FIRE} *Победа и поражение*

*Тёмные побеждают,* если:
• Убьют Орден Феникса (шпион Дамблдора)
• Их станет столько же или больше, чем Светлых

*Светлые побеждают,* если:
• Изгонят Тёмного Лорда
• Изгонят всех Пожирателей Смерти

{s.HOURGLASS} *Время течёт иначе в магии...*

• *Регистрация — 3 минуты* — время для собрания достойных
• *Ночь — 75 секунд* — молчание в чате, все действия в личных сообщениях
• *День — 3-4 минуты* — обсуждение, обвинения, защита
• *Голосование — 45 секунд* — мгновенное решение судьбы

{s.KEY} *Нажми на роль ниже, чтобы узнать её секреты...*
"""
    msg = await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
    
    if chat_id in games:
        games[chat_id].rules_message_id = msg.message_id
    
    await delete_command(update)

async def role_card_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    data = query.data
    if not data.startswith("role_"):
        return
    
    role_map = {
        "role_dark_lord": Role.DARK_LORD,
        "role_death_eater": Role.DEATH_EATER,
        "role_healer": Role.HEALER,
        "role_order": Role.ORDER,
        "role_pureblood": Role.PUREBLOOD,
        "role_halfblood": Role.HALFBLOOD
    }
    
    role = role_map.get(data)
    if not role:
        return
    
    card_text = get_role_card(role)
    
    keyboard = [[InlineKeyboardButton("◀️ Вернуться к законам", callback_data="back_to_rules")]]
    
    await query.edit_message_text(
        card_text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

async def back_to_rules(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    if query.data != "back_to_rules":
        return
    
    s = S
    
    keyboard = [
        [
            InlineKeyboardButton(f"👑 Тёмный Лорд", callback_data="role_dark_lord"),
            InlineKeyboardButton(f"💀 Пожиратель", callback_data="role_death_eater")
        ],
        [
            InlineKeyboardButton(f"🧪 Целитель", callback_data="role_healer"),
            InlineKeyboardButton(f"🔥 Орден", callback_data="role_order")
        ],
        [
            InlineKeyboardButton(f"💧 Чистокровный", callback_data="role_pureblood"),
            InlineKeyboardButton(f"⭐ Полукровка", callback_data="role_halfblood")
        ]
    ]
    
    text = f"""
{s.SNAKE} *ЗАКОНЫ ПОДЗЕМЕЛИЙ* {s.SNAKE}

{quote("Тьма возвращается, и нам нужно выбрать сторону...", "Альбус Дамблдор")}

{s.MOON} *Как ведётся игра в Тенях*

Каждый цикл состоит из Ночи и Дня. Ночью маги действуют тайно, днём — открыто. Никто не знает наверняка, кто есть кто. Все врут, все подозревают, доверие — роскошь, которую никто не может себе позволить.

{s.CANDLE} *Ночь — время теней*
Подземелья погружаются во мрак. Пока честные жители спят, пробуждаются Тёмные.
• *Тёмный Лорд* выбирает жертву для заклятия смерти
• *Целитель* тайно защищает одного от гибели
• *Орден Феникса* проверяет подозреваемого
• *Пожиратели* советуют Лорду, кто будет следующей жертвой

{s.SUN} *День — время правосудия*
Хогвартс просыпается. Если кто-то пал ночью — все узнают об этом. Живые собираются в Большом зале, но вместо завтрака — обсуждение. Кто врёт? Кто действительно видел то, о чём говорит? Кто слишком быстро обвиняет других?

{s.SCALE} *Суд — время голосования*
После обсуждения наступает час расплаты. Каждый живой отдаёт голос против того, кого подозревает. Тот, кто наберёт больше всего голосов, *изгоняется из Хогвартса*. Его роль открывается всем — было ли подозрение верным?

{s.FIRE} *Победа и поражение*

*Тёмные побеждают,* если:
• Убьют Орден Феникса (шпион Дамблдора)
• Их станет столько же или больше, чем Светлых

*Светлые побеждают,* если:
• Изгонят Тёмного Лорда
• Изгонят всех Пожирателей Смерти

{s.HOURGLASS} *Время течёт иначе в магии...*

• *Регистрация — 3 минуты* — время для собрания достойных
• *Ночь — 75 секунд* — молчание в чате, все действия в личных сообщениях
• *День — 3-4 минуты* — обсуждение, обвинения, защита
• *Голосование — 45 секунд* — мгновенное решение судьбы

{s.KEY} *Нажми на роль ниже, чтобы узнать её секреты...*
"""
    
    await query.edit_message_text(
        text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

# ← ИСПРАВЛЕНО: используем async проверку админа
async def new_game(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    
    user = update.effective_user
    chat = update.effective_chat
    
    # Используем новую async проверку
    is_admin = await check_is_admin(update, context)
    
    if not is_admin:
        msg = await update.message.reply_text(
            f"{s.SKULL} *В доступе отказано*\n\n"
            f"{user.first_name}… Ты не хранитель этих стен. "
            f"Только администраторы могут открывать Тайную комнату.",
            parse_mode='Markdown'
        )
        asyncio.create_task(delete_message_after(msg, 5))
        await delete_command(update)
        return
    
    chat_id = chat.id
    if chat_id in games and games[chat_id].phase != GamePhase.ENDED:
        msg = await update.message.reply_text(
            f"{s.SNAKE} *Тайная комната уже открыта*\n\n"
            f"Игра идёт. Завершите текущую (/end), чтобы начать новую.",
            parse_mode='Markdown'
        )
        asyncio.create_task(delete_message_after(msg, 5))
        await delete_command(update)
        return
    
    game = Game(chat_id=chat_id)
    games[chat_id] = game
    
    keyboard = [[InlineKeyboardButton(f"{s.SNAKE} ВСТУПИТЬ В ТЕНИ {s.SNAKE}", callback_data="join_game")]]
    
    msg = await update.message.reply_text(
        f"""
{s.SNAKE} *ТЕНИ ОТКРЫТЫ* {s.SNAKE}

{quote("Слизерин принимает только сильнейших", "Сортировочная Шляпа")}

{s.FOG} Тайная комната открыла свои двери...

*Время сбора:* 3 минуты
*Желающих войти:* 0

*Минимум для начала:* 6 магов

{s.ROSE} Нажми кнопку ниже, чтобы принять приглашение...
""",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )
    
    game.registration_msg_id = msg.message_id
    asyncio.create_task(registration_timer(chat_id, context, REGISTRATION_TIME))
    await delete_command(update)

async def registration_timer(chat_id: int, context: ContextTypes.DEFAULT_TYPE, seconds: int):
    s = S
    game = games.get(chat_id)
    if not game:
        return
    
    remaining = seconds
    while remaining > 0:
        await asyncio.sleep(15)
        remaining -= 15
        
        if chat_id not in games or games[chat_id].phase != GamePhase.WAITING:
            return
        
        try:
            player_list = "\n".join([f"• {p.first_name}" for p in game.players.values()])
            if not player_list:
                player_list = "_Пока никто не осмелился переступить порог..._"
            
            keyboard = [[InlineKeyboardButton(f"{s.SNAKE} ВСТУПИТЬ {s.SNAKE}", callback_data="join_game")]]
            
            await context.bot.edit_message_text(
                f"""
{s.SNAKE} *ТЕНИ ХОГВАРТСА* {s.SNAKE}

{s.HOURGLASS} *До закрытия:* {max(0, remaining)} секунд
*Вошедшие в Тени:* {len(game.players)}

{player_list}

{quote("Ждём достойных…")}

*Минимум:* 6 магов
""",
                chat_id=chat_id,
                message_id=game.registration_msg_id,
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode='Markdown'
            )
        except:
            pass
    
    if len(game.players) < 6:
        await context.bot.send_message(
            chat_id=chat_id, 
            text=f"""
{s.ROSE} *Недостаточно жертв…*

Тьма отступает. Тайная комната закрывается до следующих выходных.
"""
        )
        del games[chat_id]
        return
    
    await start_game_logic(chat_id, context)

# ← ВАЖНО: здесь НЕТ проверки на админа — любой может вступить!
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    query = update.callback_query
    await query.answer()
    
    chat_id = query.message.chat_id
    user = query.from_user
    
    if chat_id not in games:
        return
    
    game = games[chat_id]
    
    if query.data == "join_game":
        # ЛЮБОЙ пользователь может вступить, не только админ!
        if user.id in game.players:
            await query.answer("Ты уже в Тенях!", show_alert=True)
            return
        
        if game.phase != GamePhase.WAITING:
            await query.answer("Врата закрыты!", show_alert=True)
            return
        
        player = Player(user_id=user.id, username=user.username or f"user_{user.id}", first_name=user.first_name or "Аноним")
        game.players[user.id] = player
        
        await query.answer(f"Добро пожаловать в Тени, {player.first_name}!")
        
        try:
            player_list = "\n".join([f"• {p.first_name}" for p in game.players.values()])
            keyboard = [[InlineKeyboardButton(f"{s.SNAKE} ВСТУПИТЬ {s.SNAKE}", callback_data="join_game")]]
            
            await query.edit_message_text(
                f"""
{s.SNAKE} *ТЕНИ ХОГВАРТСА* {s.SNAKE}

*Тайная комната открыта*
*Вошедшие:* {len(game.players)}

{player_list}

{quote("Слизерин ждёт…")}

*Минимум:* 6 магов
""",
                reply_markup=InlineKeyboardMarkup(keyboard),
                parse_mode='Markdown'
            )
        except:
            pass

async def start_game_logic(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    s = S
    game = games[chat_id]
    players_list = list(game.players.values())
    count = len(players_list)
    
    roles = distribute_roles(count)
    for i, player in enumerate(players_list):
        player.role = roles[i]
    
    failed = []
    dark_side_ids = []
    
    for player in players_list:
        try:
            text = get_role_assignment_text(player.role, game)
            await context.bot.send_message(chat_id=player.user_id, text=text, parse_mode='Markdown')
            
            if player.role in [Role.DARK_LORD, Role.DEATH_EATER]:
                dark_side_ids.append(player.user_id)
                
        except:
            failed.append(player.first_name)
    
    if failed:
        await context.bot.send_message(
            chat_id=chat_id, 
            text=f"""
{s.SKULL} *Совы не долетели...*

Не удалось доставить послание: {", ".join(failed)}

Запусти меня в личных сообщениях и нажми /start, чтобы открыть почтовое отделение.
"""
        )
        return
    
    if dark_side_ids:
        await create_dark_chat(context, game, dark_side_ids)
    
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"""
{s.SKULL}{s.SKULL}{s.SKULL} *ИГРА НАЧАЛАСЬ* {s.SKULL}{s.SKULL}{s.SKULL}

*Собрано в Тенях:* {count} магов

{quote("Тьма возвращается…", "Сивилла Трелони")}

{s.SNAKE} Тёмная сторона пробуждается...
{s.FIRE} Орден готовится к бою...

{s.CANDLE} *Молчание в чате!*
Первая ночь наступает через 10 секунд...
Проверь личные сообщения от совы...
"""
    )
    await asyncio.sleep(10)
    await night_phase(chat_id, context)

async def create_dark_chat(context, game: Game, dark_ids: List[int]):
    s = S
    
    dark_players = [game.players[uid] for uid in dark_ids]
    
    for player_id in dark_ids:
        player = game.players[player_id]
        teammates = [p for p in dark_players if p.user_id != player_id]
        teammates_str = "\n".join([f"{s.SKULL} {p.first_name} — {p.role.value}" for p in teammates])
        
        await context.bot.send_message(
            chat_id=player_id,
            text=f"""
{s.SKULL}{s.SKULL}{s.SKULL} *ТАЙНЫЙ КАНАЛ* {s.SKULL}{s.SKULL}{s.SKULL}

*Твои союзники:*
{teammates_str}

{s.KEY} *Как связь работает:*
Во время *НОЧИ* пиши мне сообщения — я перешлю их твоим союзникам. Все послания анонимны, никто не узнает, кто именно написал.

{s.LOCK} *Помни:* Общение только ночью. Днём — молчание.
"""
        )

def get_role_assignment_text(role: Role, game: Game) -> str:
    s = S
    info = ROLE_LORE[role]
    
    if role == Role.DARK_LORD:
        dark = [p for p in game.players.values() if p.role in [Role.DARK_LORD, Role.DEATH_EATER]]
        teammates = "\n".join([f"{s.SKULL} {p.first_name}" for p in dark if p.role == Role.DEATH_EATER])
        
        return f"""
{s.SKULL}{s.SKULL}{s.SKULL} *ТЫ — ТЁМНЫЙ ЛОРД* {s.SKULL}{s.SKULL}{s.SKULL}

{quote(*info['quote'])}

{italic(info['lore'])}

*Твои Пожиратели:*
{teammates}

{s.DAGGER} *Твоя миссия:*
Выбирай жертву каждую ночь. Тот, кого защитит Целитель, выживет — остальные падут. Убей Орден Феникса, и победа твоя.

{s.KEY} *Тайная связь:*
Пиши мне ночью — сообщения уйдут союзникам через Метку.

{s.SKULL} *НИКОМУ НЕ ГОВОРИ СВОЮ РОЛЬ!*
Если тебя раскроют — Светлые изгонят тебя днём.
"""
    
    if role == Role.DEATH_EATER:
        dark = [p for p in game.players.values() if p.role in [Role.DARK_LORD, Role.DEATH_EATER]]
        lord = next((p for p in dark if p.role == Role.DARK_LORD), None)
        
        return f"""
{s.SKULL}{s.SKULL}{s.SKULL} *ТЫ — ПОЖИРАТЕЛЬ СМЕРТИ* {s.SKULL}{s.SKULL}{s.SKULL}

{quote(*info['quote'])}

{italic(info['lore'])}

*Твой Лорд:* {lord.first_name if lord else '???'}

{s.DAGGER} *Твоя миссия:*
Советуй Лорду, кого убить. Защищай его ценой собственной жизни. Если Лорд погибнет — вы все проиграете.

{s.KEY} *Тайная связь:*
Пиши мне ночью — Лорд и другие Пожиратели получат через Метку.

{s.SKULL} *НИКОМУ НЕ ГОВОРИ СВОЮ РОЛЬ!*
Доверяй только Метке.
"""
    
    if role == Role.HEALER:
        return f"""
{s.POTION}{s.POTION}{s.POTION} *ТЫ — ЦЕЛИТЕЛЬ* {s.POTION}{s.POTION}{s.POTION}

{quote(*info['quote'])}

{italic(info['lore'])}

{s.SHIELD} *Твоя миссия:*
Каждую ночь защищай одного мага от смерти. Если Лорд выберет твоего подопечного жертвой — тот выживет.

{s.KEY} *Важное правило:*
Нельзя защищать себя две ночи подряд! Магия требует жертв.

{s.SKULL} *НИКОМУ НЕ ГОВОРИ СВОЮ РОЛЬ!*
Если Тёмные узнают, кто Целитель — ты станешь их первой жертвой.
"""
    
    if role == Role.ORDER:
        return f"""
{s.FIRE}{s.FIRE}{s.FIRE} *ТЫ — ОРДЕН ФЕНИКСА* {s.FIRE}{s.FIRE}{s.FIRE}

{quote(*info['quote'])}

{italic(info['lore'])}

{s.WAND} *Твоя миссия:*
Проверяй одного игрока ночью. Я скажу: Пожиратель Смерти или нет. Найди их всех, чтобы Светлые могли изгнать Тёмных.

{s.KEY} *Критически важно:*
Тёмного Лорда ты видишь как обычного жителя! Его магия слишком сильна. Не перепутай...

{s.SKULL} *НИКОМУ НЕ ГОВОРИ СВОЮ РОЛЬ!*
Ты — главная цель Тёмных. Если тебя убьют — Светлые потеряют шанс на победу.
"""
    
    if role == Role.PUREBLOOD:
        return f"""
{s.DROP}{s.DROP}{s.DROP} *ТЫ — ЧИСТОКРОВНЫЙ* {s.DROP}{s.DROP}{s.DROP}

{quote(*info['quote'])}

{italic(info['lore'])}

{s.CRYSTAL} *Твоя скрытая сила — Кость Судьбы:*

Если тебя выбирают на голосовании — ты автоматически бросаешь кость. Шанс выжить: *10-50%*.

{s.SHIELD} *Если выпадает успех:*
Голосование тайно отменяется. Все думают, что просто не состоялось. Никто не узнает, что у тебя есть эта сила. Ты продолжаешь играть.

{s.SKULL} *Если кость подведёт:*
Ты изгнан. Твоя роль открыта.

{s.LOCK} *Веди себя как обычный житель.*
Не выдавай, что у тебя есть защита. Пусть все думают, что ты простой Полукровка.
"""
    
    return f"""
{s.STAR}{s.STAR}{s.STAR} *ТЫ — ПОЛУКРОВКА* {s.STAR}{s.STAR}{s.STAR}

{quote(*info['quote'])}

{italic(info['lore'])}

{s.WAND} *Твоя миссия:*
Ночью ты спишь, но днём твой голос решает всё. Логика, интуиция, наблюдательность — твоё оружие. Найди ложь в словах Тёмных и убеди остальных.

{s.KEY} *Твоя сила:*
Ты невидим для ночних чар. Целитель может спасти тебя, но только если догадается, что ты в опасности.

{s.SKULL} *НИКОМУ НЕ ГОВОРИ СВОЮ РОЛЬ!*
Пусть все думают, что ты можешь быть кем угодно.
"""

# ============ НОЧЬ ============
async def night_phase(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    s = S
    game = games[chat_id]
    game.phase = GamePhase.NIGHT
    game.day_count += 1
    game.killed_tonight = None
    
    for p in game.players.values():
        p.night_target = None
        p.is_protected = False
    
    alive = game.get_alive_players()
    tasks = []
    
    lord = next((p for p in alive if p.role == Role.DARK_LORD), None)
    if lord:
        tasks.append(send_lord_choice(context, lord, game))
    
    for eater in [p for p in alive if p.role == Role.DEATH_EATER]:
        tasks.append(send_eater_choice(context, eater, game))
    
    for healer in [p for p in alive if p.role == Role.HEALER]:
        tasks.append(send_healer_choice(context, healer, game))
    
    for order in [p for p in alive if p.role == Role.ORDER]:
        tasks.append(send_order_choice(context, order, game))
    
    if tasks:
        await asyncio.gather(*tasks, return_exceptions=True)
    
    dark_side = game.get_dark_side()
    for dark_player in dark_side:
        try:
            await context.bot.send_message(
                chat_id=dark_player.user_id,
                text=f"{s.MOON} *НОЧЬ {game.day_count}*\n\nТы можешь общаться с союзниками через Метку.\nПросто пиши мне сообщения."
            )
        except:
            pass
    
    night_atmosphere = [
        "Подземелья погружаются во тьму… Факелы гаснут один за другим.",
        "Тёмная магия пробуждается… Воздух становится тяжёлым и холодным.",
        "В коридорах слышны шёпоты… Кто-то произносит запретные заклинания.",
        "Нечто движется в тени… Зелёный свет мелькает за поворотом."
    ]
    
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"""
{s.MOON}{s.MOON}{s.MOON} *НОЧЬ {game.day_count}* {s.MOON}{s.MOON}{s.MOON}

{s.HOURGLASS} *75 секунд тишины*

{quote(random.choice(night_atmosphere))}

{s.SNAKE} Пожиратели выходят на охоту…
{s.POTION} Целитель готовит эликсиры…
{s.FIRE} Орден следит из тени…

{s.SKULL} *МОЛЧАНИЕ В ЧАТЕ!*
Нарушителей ждёт немедленное изгнание.
Проверь личные сообщения от совы...
"""
    )
    await asyncio.sleep(NIGHT_TIME)
    await process_night_results(chat_id, context)

async def send_lord_choice(context, player: Player, game: Game):
    s = S
    alive = [p for p in game.get_alive_players() if p.user_id != player.user_id]
    keyboard = [[InlineKeyboardButton(f"{s.SKULL} {p.first_name}", callback_data=f"kill_{p.user_id}")] for p in alive]
    await context.bot.send_message(
        chat_id=player.user_id,
        text=f"{s.MOON} *НОЧЬ {game.day_count}*\n\n{s.DAGGER} *Выбери жертву для Авады Кедавры:*",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

async def send_eater_choice(context, player: Player, game: Game):
    s = S
    alive = [p for p in game.get_alive_players() if p.user_id != player.user_id]
    keyboard = [[InlineKeyboardButton(f"{s.SNAKE} {p.first_name}", callback_data=f"suggest_{p.user_id}")] for p in alive]
    await context.bot.send_message(
        chat_id=player.user_id,
        text=f"{s.MOON} *НОЧЬ {game.day_count}*\n\n{s.DAGGER} *Посоветуй Лорду жертву:*",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

async def send_healer_choice(context, player: Player, game: Game):
    s = S
    alive = game.get_alive_players()
    keyboard = [[InlineKeyboardButton(f"{s.POTION} {p.first_name}" + (" (ты)" if p == player else ""), callback_data=f"heal_{p.user_id}")] for p in alive]
    await context.bot.send_message(
        chat_id=player.user_id,
        text=f"{s.MOON} *НОЧЬ {game.day_count}*\n\n{s.SHIELD} *Кого защитить от Тьмы?*",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

async def send_order_choice(context, player: Player, game: Game):
    s = S
    alive = [p for p in game.get_alive_players() if p.user_id != player.user_id]
    keyboard = [[InlineKeyboardButton(f"{s.WAND} {p.first_name}", callback_data=f"check_{p.user_id}")] for p in alive]
    await context.bot.send_message(
        chat_id=player.user_id,
        text=f"{s.MOON} *НОЧЬ {game.day_count}*\n\n{s.FIRE} *Кого проверить?*",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )

async def process_night_results(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    s = S
    game = games[chat_id]
    
    dark_lord = next((p for p in game.get_alive_players() if p.role == Role.DARK_LORD), None)
    
    if dark_lord and dark_lord.night_target:
        target = game.players.get(dark_lord.night_target)
        if target:
            healers = game.get_role_players(Role.HEALER)
            healed = any(h.night_target == target.user_id for h in healers)
            if not healed:
                target.is_alive = False
                game.killed_tonight = target
    
    if game.killed_tonight:
        victim = game.killed_tonight
        death_scenes = [
            f"Зелёная вспышка озарила коридор… *{victim.first_name}* упал, не издав ни звука.",
            f"Кто-то нашёл тело в Трофейной комнате… *{victim.first_name}* не подавал признаков жизни.",
            f"На стенах появились слова кровью… *{victim.first_name}* стал жертвой Тьмы."
        ]
        
        await context.bot.send_message(
            chat_id=chat_id,
            text=f"""
{s.SKULL}{s.SKULL}{s.SKULL} *ЖЕРТВА НОЧИ* {s.SKULL}{s.SKULL}{s.SKULL}

{random.choice(death_scenes)}

*{victim.first_name}* пал от рук Тёмных…

{s.MASK} *Роль:* {victim.role.value}

{s.ROSE} *Покойся с миром...*
"""
        )
    else:
        await context.bot.send_message(
            chat_id=chat_id,
            text=f"""
{s.SNAKE}{s.SNAKE}{s.SNAKE} *НОЧЬ ПРОШЛА БЕЗ ЖЕРТВ* {s.SNAKE}{s.SNAKE}{s.SNAKE}

{quote("Целитель успел вовремя… Или Тёмные промахнулись?")}

{s.FOG} Тишина в подземельях нарушена только шорохом сов...
"""
        )
    
    winner = check_winner(game)
    if winner:
        await end_game(chat_id, context, winner)
        return
    
    await asyncio.sleep(3)
    await day_phase(chat_id, context)

# ============ ДЕНЬ ============
async def day_phase(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    s = S
    game = games[chat_id]
    game.phase = GamePhase.DAY
    
    alive = game.get_alive_players()
    alive_text = "\n".join([f"• {p.first_name}" for p in alive])
    day_duration = random.randint(DAY_MIN_TIME, DAY_MAX_TIME)
    
    day_scenes = [
        "Солнце встало над Хогвартсом, но тепло не принесло...",
        "Утренний туман окутал замок. Кто-то не дожил до рассвета...",
        "Птицы замолкли. Даже они чувствуют, что в замке Тьма...",
        "Новый день — новые подозрения. Доверие — умерло вчера ночью..."
    ]
    
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"""
{s.SUN}{s.SUN}{s.SUN} *ДЕНЬ {game.day_count}* {s.SUN}{s.SUN}{s.SUN}

{quote(random.choice(day_scenes))}

*Выжившие слизеринцы:*
{alive_text}

{s.HOURGLASS} *Время обсуждения:* {day_duration // 60} мин {day_duration % 60} сек

{s.DAGGER} *Кто Пожиратель? Кто лжёт?*
{s.SCALE} *Решайте, кого изгнать до следующей ночи...*
"""
    )
    game.day_timer_task = asyncio.create_task(day_timer(chat_id, context, day_duration))

async def day_timer(chat_id: int, context: ContextTypes.DEFAULT_TYPE, seconds: int):
    s = S
    await asyncio.sleep(seconds)
    if chat_id not in games or games[chat_id].phase != GamePhase.DAY:
        return
    await context.bot.send_message(
        chat_id=chat_id, 
        text=f"{s.SKULL} *Время обсуждения истекло…*\n\n{s.HOURGLASS} Начинается голосование!",
        parse_mode='Markdown'
    )
    await start_voting(chat_id, context)

# ============ ГОЛОСОВАНИЕ ============
async def start_voting(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    s = S
    game = games[chat_id]
    game.phase = GamePhase.VOTING
    
    for p in game.players.values():
        p.voted_for = None
    
    alive = game.get_alive_players()
    keyboard = [[InlineKeyboardButton(f"{s.DAGGER} {p.first_name}", callback_data=f"vote_{p.user_id}")] for p in alive]
    keyboard.append([InlineKeyboardButton(f"{s.SKULL} Воздержаться", callback_data="vote_none")])
    
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"""
{s.SCALE}{s.SCALE}{s.SCALE} *СУД НАД ПОДОЗРЕВАЕМЫМ* {s.SCALE}{s.SCALE}{s.SCALE}

{quote("Кого выгоняем из Хогвартса?")}

{s.HOURGLASS} *45 секунд на решение!*

{s.MASK} *Помни:* Если ошибёшься — ночью умрёт ещё кто-то...
""",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )
    await asyncio.sleep(VOTING_TIME)
    await process_voting(chat_id, context)

async def process_voting(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    s = S
    game = games[chat_id]
    
    votes = {}
    for player in game.get_alive_players():
        if player.voted_for and player.voted_for != -1:
            votes[player.voted_for] = votes.get(player.voted_for, 0) + 1
    
    if not votes:
        await context.bot.send_message(
            chat_id=chat_id, 
            text=f"{s.SKULL} *Никто не осмелился проголосовать…*\n\nТьма смеётся над вашей нерешительностью.",
            parse_mode='Markdown'
        )
        await night_phase(chat_id, context)
        return
    
    max_votes = max(votes.values())
    candidates = [uid for uid, count in votes.items() if count == max_votes]
    vote_stats = "\n".join([f"• {game.players[uid].first_name}: {count} голосов" for uid, count in sorted(votes.items(), key=lambda x: -x[1])])
    
    if len(candidates) > 1:
        await context.bot.send_message(
            chat_id=chat_id,
            text=f"""
{s.SCALE} *Результаты голосования:*
{vote_stats}

{s.SKULL} *Ничья!* Стороны слишком равны.
Никто не изгнан. Тьма продолжает охоту...
"""
        )
        await night_phase(chat_id, context)
        return
    
    target_id = candidates[0]
    target = game.players[target_id]
    
    # Чистокровный — скрытая механика
    if target.role == Role.PUREBLOOD and target.is_alive and not target.pureblood_used:
        survival_chance = random.randint(10, 50)
        roll = random.randint(1, 100)
        
        try:
            result_text = f"{s.STAR} *ТЫ ВЫЖИЛ!* Судьба дала тебе шанс." if roll <= survival_chance else f"{s.SKULL} *ТЫ ПОГИБ…* Кость не сработала."
            await context.bot.send_message(
                chat_id=target.user_id,
                text=f"""
{s.CRYSTAL}{s.CRYSTAL}{s.CRYSTAL} *КОСТЬ СУДЬБЫ БРОШЕНА* {s.CRYSTAL}{s.CRYSTAL}{s.CRYSTAL}

Тебя выбрали на голосовании!

*Шанс выжить:* {survival_chance}%
*Выпало:* {roll}

{result_text}
"""
            )
        except:
            pass
        
        if roll <= survival_chance:
            target.pureblood_used = True
            
            fake_reasons = [
                "Голосование признано недействительным… Кто-то вмешался?",
                "Вмешались силы, неподвластные нам… Магия сработала тайно.",
                "Судьба изменила своё решение… Кость была брошена.",
                "Загадочные силы защитили изгнанника… Никто не понял, что произошло."
            ]
            
            await context.bot.send_message(
                chat_id=chat_id,
                text=f"""
{s.FOG} *{random.choice(fake_reasons)}*

{s.SKULL} Голосование отменяется. Никто не изгнан.
{s.LOCK} Что произошло — остаётся тайной...
"""
            )
            await night_phase(chat_id, context)
            return
    
    target.is_alive = False
    
    exile_scenes = [
        f"*{target.first_name}* выведен за ворота Хогвартса. Ворота закрылись за ним навсегда.",
        f"Толпа решила судьбу *{target.first_name}*. Авада Кедавра была бы быстрее...",
        f"*{target.first_name}* покинул замок в сопровождении дементоров. Или это были просто стражники?"
    ]
    
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"""
{s.SCALE} *Результаты голосования:*
{vote_stats}

{random.choice(exile_scenes)}

{s.MASK} *Роль изгнанного:* {target.role.value}

{s.SKULL} *Ещё один покинул Тени...*
"""
    )
    
    winner = check_winner(game)
    if winner:
        await end_game(chat_id, context, winner)
    else:
        await night_phase(chat_id, context)

def check_winner(game: Game) -> Optional[str]:
    alive = game.get_alive_players()
    dark = [p for p in alive if p.role in [Role.DARK_LORD, Role.DEATH_EATER]]
    order = [p for p in alive if p.role == Role.ORDER]
    
    if not any(p.role == Role.DARK_LORD for p in dark):
        return "light"
    if not order:
        return "dark"
    if len(dark) >= len([p for p in alive if p not in dark]):
        return "dark"
    return None

async def end_game(chat_id: int, context: ContextTypes.DEFAULT_TYPE, winner: str):
    s = S
    game = games[chat_id]
    game.phase = GamePhase.ENDED
    
    if winner == "dark":
        text = f"""
{s.SKULL}{s.SKULL}{s.SKULL} *ТЁМНЫЕ ПОБЕДИЛИ!* {s.SKULL}{s.SKULL}{s.SKULL}

{quote("Да здравствует Тёмный Лорд! Мы были верны до конца!", "Пожиратели Смерти")}

{s.BLOOD} *Победители — Сторона Тени:*
"""
        winners = [p for p in game.players.values() if p.role in [Role.DARK_LORD, Role.DEATH_EATER]]
    else:
        text = f"""
{s.FIRE}{s.FIRE}{s.FIRE} *СВЕТЛЫЕ ПОБЕДИЛИ!* {s.FIRE}{s.FIRE}{s.FIRE}

{quote("Хогвартс спасён… Но какой ценой?", "Орден Феникса")}

{s.STAR} *Победители — Сторона Света:*
"""
        winners = [p for p in game.players.values() if p.role not in [Role.DARK_LORD, Role.DEATH_EATER]]
    
    for w in winners:
        status = f"{s.SUN} Жив" if w.is_alive else f"{s.ROSE} Пал"
        text += f"{status} — {w.first_name}\n"
    
    text += f"\n{s.SCROLL} *Все участники и их роли:*\n"
    for p in game.players.values():
        status_icon = s.ROSE if not p.is_alive else s.SUN
        text += f"{status_icon} {p.first_name} — {p.role.value}\n"
    
    text += f"\n{s.SNAKE} *Игра окончена. Тайная комната закрывается...*"
    
    await context.bot.send_message(chat_id=chat_id, text=text, parse_mode='Markdown')
    
    await asyncio.sleep(60)
    if chat_id in games:
        del games[chat_id]

# ============ КОЛБЭКИ ============
async def night_callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    data = query.data
    
    game = None
    player = None
    for g in games.values():
        if user_id in g.players:
            game = g
            player = g.players[user_id]
            break
    
    if not game or game.phase != GamePhase.NIGHT:
        await query.answer("Сейчас не ночь!", show_alert=True)
        return
    
    await query.answer("Заклинание произнесено!")
    s = S
    
    if data.startswith("kill_"):
        player.night_target = int(data.split("_")[1])
        await query.edit_message_text(f"{s.SKULL} *Жертва выбрана.* Авада Кедавра будет произнесена...")
        target = game.players.get(player.night_target)
        for eater in game.get_role_players(Role.DEATH_EATER):
            await context.bot.send_message(chat_id=eater.user_id, text=f"{s.SNAKE} Лорд выбрал: *{target.first_name if target else '???'}*", parse_mode='Markdown')
    
    elif data.startswith("suggest_"):
        await query.edit_message_text(f"{s.SNAKE} *Предложение отправлено Лорду.*")
    
    elif data.startswith("heal_"):
        player.night_target = int(data.split("_")[1])
        target = game.players.get(player.night_target)
        await query.edit_message_text(f"{s.POTION} *Защищаем {target.first_name if target else '???'}*\nПусть эликсир подействует...")
    
    elif data.startswith("check_"):
        target_id = int(data.split("_")[1])
        target = game.players.get(target_id)
        if target:
            result = f"{s.SKULL} *ПОЖИРАТЕЛЬ СМЕРТИ!*" if target.role == Role.DEATH_EATER else f"{s.SNAKE} Обычный житель"
            await query.edit_message_text(f"{s.FIRE} *{target.first_name}:*\n{result}")

async def vote_callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    data = query.data
    
    game = None
    player = None
    for g in games.values():
        if user_id in g.players:
            game = g
            player = g.players[user_id]
            break
    
    if not game or game.phase != GamePhase.VOTING:
        await query.answer("Не время голосовать!", show_alert=True)
        return
    
    if not player.is_alive:
        await query.answer("Ты мёртв!", show_alert=True)
        return
    
    s = S
    if data == "vote_none":
        player.voted_for = -1
        await query.answer("Воздержался")
        await query.edit_message_text(f"{s.SKULL} *Воздержался*\nПусть судьба решает сама...")
    else:
        target_id = int(data.split("_")[1])
        player.voted_for = target_id
        target = game.players.get(target_id)
        await query.answer(f"Голос против {target.first_name if target else '???'}")
        await query.edit_message_text(f"{s.DAGGER} *Голос против {target.first_name if target else '???'}*\nПусть это будет правильный выбор...")

# ============ МОЛЧАНИЕ И ЧАТ УБИЙЦ ============
async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    
    # Проверяем личные сообщения убийцам
    if not update.effective_chat or update.effective_chat.type == "private":
        for game in games.values():
            if user_id not in game.players:
                continue
            
            player = game.players[user_id]
            
            if player.role not in [Role.DARK_LORD, Role.DEATH_EATER]:
                await update.message.reply_text("Ты не можешь использовать этот канал связи.")
                return
            
            if game.phase != GamePhase.NIGHT:
                await update.message.reply_text("Связь через Метку работает только ночью!")
                return
            
            dark_side = game.get_dark_side()
            message_text = update.message.text
            
            for teammate in dark_side:
                if teammate.user_id != user_id:
                    try:
                        await context.bot.send_message(
                            chat_id=teammate.user_id,
                            text=f'{S.SKULL} *Голос Метки:* {message_text}',
                            parse_mode='Markdown'
                        )
                    except:
                        pass
            
            await update.message.reply_text(f"{S.SKULL} *Послание ушло союзникам через Метку.*")
            return
    
    # Молчание в чате ночью
    chat_id = update.effective_chat.id
    if chat_id not in games:
        return
    
    game = games[chat_id]
    user = update.effective_user
    
    if user.is_bot:
        return
    
    if game.phase == GamePhase.NIGHT:
        try:
            await update.message.delete()
        except:
            pass
        return
    
    if user.id not in game.players:
        try:
            await update.message.delete()
        except:
            pass

# ============ ВСПОМОГАТЕЛЬНЫЕ ============
async def players_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    
    chat_id = update.effective_chat.id
    if chat_id not in games:
        msg = await update.message.reply_text(f"{s.SKULL} *Тайная комната закрыта.*\nНет активной игры.", parse_mode='Markdown')
        asyncio.create_task(delete_message_after(msg, 3))
        asyncio.create_task(delete_message_after(update.message, 1))
        return
    
    game = games[chat_id]
    alive = game.get_alive_players()
    dead = [p for p in game.players.values() if not p.is_alive]
    
    text = f"{s.SNAKE} *СОБРАНИЕ В ТЕНЯХ* {s.SNAKE}\n\n"
    text += f"*{s.SUN} Живые ({len(alive)}):*\n"
    for p in alive:
        text += f"• {p.first_name}\n"
    
    if dead:
        text += f"\n*{s.ROSE} Павшие ({len(dead)}):*\n"
        for p in dead:
            text += f"○ ~~{p.first_name}~~ ({p.role.value})\n"
    
    await update.message.reply_text(text, parse_mode='Markdown')
    await delete_command(update)

async def game_status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    
    chat_id = update.effective_chat.id
    if chat_id not in games:
        msg = await update.message.reply_text(f"{s.SKULL} *Тайная комната пуста.*", parse_mode='Markdown')
        asyncio.create_task(delete_message_after(msg, 3))
        asyncio.create_task(delete_message_after(update.message, 1))
        return
    
    game = games[chat_id]
    phases = {
        GamePhase.WAITING: f"{s.DOOR} Открытие Тайной комнаты",
        GamePhase.NIGHT: f"{s.MOON} Ночь — время теней",
        GamePhase.DAY: f"{s.SUN} День — время правосудия",
        GamePhase.VOTING: f"{s.SCALE} Суд над подозреваемыми"
    }
    
    await update.message.reply_text(
        f"{s.SNAKE} *ХОД ИГРЫ* {s.SNAKE}\n\n"
        f"{phases.get(game.phase, '???')}\n"
        f"День: {game.day_count}\n"
        f"Живых: {len(game.get_alive_players())}/{len(game.players)}",
        parse_mode='Markdown'
    )
    await delete_command(update)

# ← ИСПРАВЛЕНО: используем async проверку админа
async def end_game_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    s = S
    
    user = update.effective_user
    chat = update.effective_chat
    
    is_admin = await check_is_admin(update, context)
    
    if not is_admin:
        msg = await update.message.reply_text(f"{s.SKULL} *Только хранители этих стен могут закрыть Тайную комнату!*", parse_mode='Markdown')
        asyncio.create_task(delete_message_after(msg, 5))
        asyncio.create_task(delete_message_after(update.message, 1))
        return
    
    if chat.id in games:
        del games[chat.id]
        msg = await update.message.reply_text(f"{s.SKULL} *Тайная комната закрыта.*\nИгра завершена принудительно.", parse_mode='Markdown')
        asyncio.create_task(delete_message_after(msg, 5))
        await delete_command(update)
    else:
        msg = await update.message.reply_text(f"{s.SKULL} *Нет открытых дверей.*", parse_mode='Markdown')
        asyncio.create_task(delete_message_after(msg, 3))
        await delete_command(update)

# ============ АДМИНЫ ============
async def update_chat_admins(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat = update.effective_chat
    if not chat:
        return
    try:
        admins = await chat.get_administrators()
        admin_ids = {admin.user.id for admin in admins if isinstance(admin, (ChatMemberAdministrator, ChatMemberOwner))}
        chat_admins[chat.id] = admin_ids
        logger.info(f"Updated admins for chat {chat.id}: {len(admin_ids)} admins")
    except Exception as e:
        logger.error(f"Failed to update admins: {e}")

# ============ MAIN ============
def main():
    s = S
    application = Application.builder().token(BOT_TOKEN).build()
    
    application.add_handler(ChatMemberHandler(update_chat_admins, ChatMemberHandler.MY_CHAT_MEMBER))
    
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("new_game", new_game))
    application.add_handler(CommandHandler("rules", rules))
    application.add_handler(CommandHandler("players", players_list))
    application.add_handler(CommandHandler("status", game_status))
    application.add_handler(CommandHandler("end", end_game_cmd))
    
    # Обработчики кнопок
    application.add_handler(CallbackQueryHandler(button_handler, pattern="^join_game$"))
    application.add_handler(CallbackQueryHandler(role_card_handler, pattern="^role_"))
    application.add_handler(CallbackQueryHandler(back_to_rules, pattern="^back_to_rules$"))
    application.add_handler(CallbackQueryHandler(night_callback_handler, pattern="^kill_|^suggest_|^heal_|^check_"))
    application.add_handler(CallbackQueryHandler(vote_callback_handler, pattern="^vote_"))
    
    # Обработчик сообщений
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, message_handler))
    
    print(f"{s.SNAKE}{s.SNAKE}{s.SNAKE} БОТ ЗАПУЩЕН {s.SNAKE}{s.SNAKE}{s.SNAKE}")
    application.run_polling()

if __name__ == "__main__":
    main()
