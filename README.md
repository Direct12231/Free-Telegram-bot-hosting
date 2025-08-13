from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackContext, CallbackQueryHandler
import random
from datetime import datetime, timedelta
import asyncio
import json
import os

TOKEN = '8418206369:AAHwbp10g0jTP2ESh6hF6FwBLeinl4Aoauo'
PROFILES_FILE = "user_profiles.json"
WELCOME_IMAGE = "https://avatars.mds.yandex.net/i?id=a563f1942f2be1548d965e003a6005981a199397-5288900-images-thumbs&n=13"

# Словарь для хранения времени последнего использования
user_cooldowns = {}
# Словарь для хранения профилей пользователей
user_profiles = {}
# Список администраторов
ADMIN_IDS = [6775264881, 7522937903]

# Количество монет за каждую редкость
COINS_REWARD = {
    "common": 5,
    "rare": 10,
    "epic": 20,
    "legendary": 50,
    "ultra_legendary": 100
}

# Список товаров в магазине
SHOP_ITEMS = {
    "gacha_roll_voucher": {
        "name": "Ваучер на дополнительную попытку",
        "description": "Позволяет сделать дополнительную попытку вытягивания персонажа без ожидания кулдауна",
        "cost": 50,
        "emoji": "🎟️"
    },
    "common_character_card": {
        "name": "Карта персонажа (Обычная)",
        "description": "Получите случайного персонажа обычной редкости",
        "cost": 50,
        "emoji": "⭐",
        "rarity": "common"
    },
    "rare_character_card": {
        "name": "Карта персонажа (Редкая)",
        "description": "Получите случайного персонажа редкой редкости",
        "cost": 100,
        "emoji": "⭐⭐",
        "rarity": "rare"
    },
    "epic_character_card": {
        "name": "Карта персонажа (Эпическая)",
        "description": "Получите случайного персонажа эпической редкости",
        "cost": 200,
        "emoji": "⭐⭐⭐",
        "rarity": "epic"
    },
    "legendary_character_card": {
        "name": "Карта персонажа (Легендарная)",
        "description": "Получите случайного персонажа легендарной редкости",
        "cost": 500,
        "emoji": "⭐⭐⭐⭐",
        "rarity": "legendary"
    },
    "ultra_legendary_character_card": {
        "name": "Карта персонажа (Ультралегендарная)",
        "description": "Получите случайного персонажа ультралегендарной редкости",
        "cost": 1000,
        "emoji": "🌟🌟🌟🌟🌟",
        "rarity": "ultra_legendary"
    }
}

# Полный список персонажей по редкости с URL изображений
CHARACTERS = {
    "common": {
        "emoji": "⭐",
        "chance": 48,
        "list": [
            {"name": "Нобара Кугисаки", "image": "https://avatars.mds.yandex.net/i?id=c32dd1811f3225bd2a25d53c454d9748dd5c3a0a-9181211-images-thumbs&n=13"},
            {"name": "Маки Дзэнин (без проклятой энергии)", "image": "https://avatars.mds.yandex.net/i?id=25f21bcb866bf4d06106f9a7e28fc3a2e2a04b73-5249548-images-thumbs&n=13"},
            {"name": "Панда", "image": "https://avatars.mds.yandex.net/i?id=d9683004a29d2fe642d6d0c46f64f4c671154054-5874721-images-thumbs&n=13"},
            {"name": "Тодо Аой", "image": "https://avatars.mds.yandex.net/i?id=3cd6b748dc313c169e1ee7229efec813042f51aa-5297681-images-thumbs&n=13"},
            {"name": "Юдзи Итадори (базовый)", "image": "https://avatars.mds.yandex.net/i?id=f893f87d7696ef840c7add0367e79c49ef7cbc8b-3924502-images-thumbs&n=13"},
            {"name": "Мегуми Фушигуро (базовый)", "image": "https://avatars.mds.yandex.net/i?id=4c88e92981989b004b9d7294297f4ad4094905d7-5322694-images-thumbs&n=13"},
            {"name": "Камо Норитоши", "image": "https://avatars.mds.yandex.net/i?id=3ba86d57b06c422cc8f60d0a2368ad6f373f5a28-5887690-images-thumbs&n=13"},
            {"name": "Касами Мива", "image": "https://avatars.mds.yandex.net/i?id=e8ff25e84a1354f675c5697861c44f689cc685a6-4481820-images-thumbs&n=13"},
            {"name": "Момо Нишимия", "image": "https://avatars.mds.yandex.net/i?id=0f6354bd3b886902d88d73ffc04c30147e4bca3d-5355514-images-thumbs&n=13"},
            {"name": "Кокчи Муто", "image": "https://avatars.mds.yandex.net/i?id=c90949d6951d6b7125b9b62452c85d4bf515ec06-6262780-images-thumbs&n=13"}
        ]
    },
    "rare": {
        "emoji": "⭐⭐",
        "chance": 30,
        "list": [
            {"name": "Мегуми Фушигуро (с Договором)", "image": "https://avatars.mds.yandex.net/i?id=0fd9039784d20da7fed7a12116d3d1f176d38632-5232501-images-thumbs&n=13"},
            {"name": "Юта Оккотсу", "image": "https://avatars.mds.yandex.net/i?id=7473a960d19c395e41fed2af9206ae603c2360d8-5177089-images-thumbs&n=13"},
            {"name": "Кириара", "image": "https://avatars.mds.yandex.net/i?id=18c3d0615f959c12be8d064d45235ff7a90d3580-4760093-images-thumbs&n=13"},
            {"name": "Махито (ранняя форма)", "image": "https://avatars.mds.yandex.net/i?id=b229607263577b138ea9b8c0b59cfe9e65950d27-9234735-images-thumbs&n=13"},
            {"name": "Дзёго", "image": "https://avatars.mds.yandex.net/i?id=8d3bbd7e0ceb69ded4c04dca1204b014a9824f87-5855963-images-thumbs&n=13"},
            {"name": "Нанмин Кэнтаку", "image": "https://avatars.mds.yandex.net/i?id=bba18632f3e43a2d57e5e2c594a611796cb259ef-4350140-images-thumbs&n=13"},
            {"name": "Май Зенин", "image": "https://avatars.mds.yandex.net/i?id=4c24e326dd2ea80bf1ad0dc946eec81e0eb3e4fa-16290826-images-thumbs&n=13"},
            {"name": "Тохе Нанао", "image": "https://avatars.mds.yandex.net/i?id=e8ff25e84a1354f675c5697861c44f689cc685a6-4481820-images-thumbs&n=13"},
            {"name": "Юдзи Итадори (с проклятой энергией)", "image": "https://avatars.mds.yandex.net/i?id=1895d62b5a9469841f34aba5ea5d9b7d49b37c37-7755608-images-thumbs&n=13"},
            {"name": "Аои Тодо (с Бумерангом)", "image": "https://avatars.mds.yandex.net/i?id=6df5733611275e4e3331d5bc5465e90007883f01-12609997-images-thumbs&n=13"}
        ]
    },
    "epic": {
        "emoji": "⭐⭐⭐",
        "chance": 15,
        "list": [
            {"name": "Годжо Сатору", "image": "https://avatars.mds.yandex.net/i?id=12eda419c761d9cde31c0f5af76a791afe71d5a6-12506585-images-thumbs&n=13"},
            {"name": "Сукуна (1 палец)", "image": "https://avatars.mds.yandex.net/i?id=6619aeed3f0ad7074e37b8bf3c2840a9bd80c565-4281183-images-thumbs&n=13"},
            {"name": "Тоджи", "image": "https://avatars.mds.yandex.net/i?id=61302d2ed0274a183bc771d6c308e75189d827ba-5477180-images-thumbs&n=13"},
            {"name": "Махито (полная форма)", "image": "https://avatars.mds.yandex.net/i?id=9bc0413f072df6213d26c499d5955e3d7db3b180-12151887-images-thumbs&n=13"},
            {"name": "Ураме", "image": "https://avatars.mds.yandex.net/i?id=d94f579526f03636bc6240e9f354e6f6b46d30b5-5650815-images-thumbs&n=13"},
            {"name": "Куро", "image": "https://avatars.mds.yandex.net/i?id=3ba86d57b06c422cc8f60d0a2368ad6f373f5a28-5887690-images-thumbs&n=13"},
            {"name": "Джого (с проклятой энергией)", "image": "https://avatars.mds.yandex.net/i?id=601ba2bc8e32141e6f4e9ccf447650d10560fb53-5887740-images-thumbs&n=13"},
            {"name": "Маки Дзэнин (пробуждение)", "image": "https://avatars.mds.yandex.net/i?id=0998e719bc2b55384b4e6d9fb232437eb84bdf2e-10126262-images-thumbs&n=13"},
            {"name": "Юдзи Итадори (Черная Вспышка)", "image": "https://avatars.mds.yandex.net/i?id=b220f4a3306d2bce53896e5e7ccf15cb00528f5e-4884261-images-thumbs&n=13"},
            {"name": "Мегуми Фушигуро (Домен: Химеровое Садоводство)", "image": "https://avatars.mds.yandex.net/i?id=41e067fd7a6ddada1b145294adff49a39f6102be-12422391-images-thumbs&n=13"}
        ]
    },
    "legendary": {
        "emoji": "⭐⭐⭐⭐",
        "chance": 3,
        "list": [
            {"name": "Рёмен Сукуна (полная сила)", "image": "https://avatars.mds.yandex.net/i?id=c3e59bca913d6234a6d6db1f6a6fbdf539501b31-9049934-images-thumbs&n=13"},
            {"name": "Годжо (Limitless Purple)", "image": "https://avatars.mds.yandex.net/i?id=aacdacaae629b6c11e7179633ba53ba8d8a5352f-9107157-images-thumbs&n=13"},
            {"name": "Кендзяку", "image": "https://avatars.mds.yandex.net/i?id=275ef5c5db38280ceb278702195b4bbe556dd3f1-5235884-images-thumbs&n=13"},
            {"name": "Юта Оккотсу (полная мощь)", "image": "https://avatars.mds.yandex.net/i?id=b5b67cdec5737537450b9f2504d2070acd9d094a-5263409-images-thumbs&n=13"},
            {"name": "Мегуми (Домен: Полная Луна)", "image": "https://avatars.mds.yandex.net/i?id=ae069877c01ca8b106f2f13de8246386bb09792f-5354411-images-thumbs&n=13"},
            {"name": "Сукуна (Огненный Домен)", "image": "https://avatars.mds.yandex.net/i?id=e644b7b729ccb82aaeb792af5e92db4ccafa9693-5161973-images-thumbs&n=13"},
            {"name": "Махито (Слияние)", "image": "https://avatars.mds.yandex.net/i?id=70a13ea47a9132a2f16c14c69e93669c39fffa01-5544858-images-thumbs&n=13"},
            {"name": "Годжо Сатору (Пустой Фиолетовый)", "image": "https://avatars.mds.yandex.net/i?id=f0c7368ee6c010cbbb9f16812cb08aa2debb91fa-5660528-images-thumbs&n=13"},
            {"name": "Юдзи Итадори (Пробуждение Сукуны)", "image": "https://avatars.mds.yandex.net/i?id=9cd46473e034944a8995425f32f70d4d3e1e9583-6978923-images-thumbs&n=13"},
            {"name": "Тоджи (Обратное Проклятие)", "image": "https://avatars.mds.yandex.net/i?id=829ab4f0895f057b7db4d3991cce005ff14946d3-4662453-images-thumbs&n=13"}
        ]
    },
    "ultra_legendary": {
        "emoji": "🌟🌟🌟🌟🌟",
        "chance": 2,
        "list": [
            {"name": "Рёмен Сукуна (Истинная форма)", "image": "https://avatars.mds.yandex.net/i?id=5402d0ab13840a844a7a156b6931bc2567528a55-5210329-images-thumbs&n=13"},
            {"name": "Годжо Сатору (Полная мощь)", "image": "https://avatars.mds.yandex.net/i?id=da173df7ad0d4b27f79409f015ea26aad633d4a5-4430150-images-thumbs&n=13"},
            {"name": "Юта Оккотсу (Режим берсерка)", "image": "https://avatars.mds.yandex.net/i?id=035104cbe20dd7be5e10ad145c33035bb58c30ba-6978923-images-thumbs&n=13"},
            {"name": "Махито (Совершенная форма)", "image": "https://avatars.mds.yandex.net/i?id=a66b421f2e70c94987b1b165ee926b27e1650cfc-4815936-images-thumbs&n=13"},
            {"name": "Тоджи (Пробуждение)", "image": "https://avatars.mds.yandex.net/i?id=e5bc7825c70498d881640607d955a2e6504f39440c019017-5327407-images-thumbs&n=13"},
            {"name": "Юдзи Итадори (Контроль Сукуны)", "image": "https://avatars.mds.yandex.net/i?id=9cd46473e034944a8995425f32f70d4d3e1e9583-6978923-images-thumbs&n=13"},
            {"name": "Мегуми Фушигуро (Полная луна)", "image": "https://avatars.mds.yandex.net/i?id=970b76f84b362636ab5534f79345718e16db7819-10471467-images-thumbs&n=13"},
            {"name": "Кендзяку (Все техники)", "image": "https://avatars.mds.yandex.net/i?id=a8861e9a8bffd3eab30690b7a3e0d3ef6718f1f6-5283109-images-thumbs&n=13"},
            {"name": "Ураме (Древний дух)", "image": "https://avatars.mds.yandex.net/i?id=3a05448546ad40f102d11801ae271289fe286de7-8439682-images-thumbs&n=13"},
            {"name": "Маки Дзэнин (Безграничная сила)", "image": "https://avatars.mds.yandex.net/i?id=38b1977fd18fd9439268a13a371aaf4c24fbe508-2375701-images-thumbs&n=13"}
        ]
    }
}

def load_profiles():
    """Загружает профили из JSON-файла"""
    global user_profiles
    if os.path.exists(PROFILES_FILE):
        try:
            with open(PROFILES_FILE, 'r', encoding='utf-8') as f:
                loaded_data = json.load(f)
                # Преобразуем ключи обратно в int
                user_profiles = {int(k): v for k, v in loaded_data.items()}
        except Exception as e:
            print(f"Ошибка при загрузке профилей: {e}")
            user_profiles = {}
    else:
        user_profiles = {}

def save_profiles():
    """Сохраняет профили в JSON-файл"""
    try:
        with open(PROFILES_FILE, 'w', encoding='utf-8') as f:
            # Преобразуем ключи в строки для JSON
            json.dump({str(k): v for k, v in user_profiles.items()}, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"Ошибка при сохранении профилей: {e}")

def escape_markdown(text: str) -> str:
    """Экранирует специальные символы для MarkdownV2"""
    escape_chars = r'\_*[]()~`>#+-=|{}.!'
    return ''.join(f'\\{char}' if char in escape_chars else char for char in text)

async def start(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /start"""
    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    
    # Инициализация профиля, если его нет
    if user_id not in user_profiles:
        user_profiles[user_id] = {
            "username": username,
            "characters": [],
            "total_rolls": 0,
            "ultra_legendary_count": 0,
            "legendary_count": 0,
            "epic_count": 0,
            "rare_count": 0,
            "common_count": 0,
            "coins": 0,
            "favorite_character": None,
            "inventory": {"gacha_roll_voucher": 0}
        }
        save_profiles()
    
    chances_text = "\n".join(
        f"{data['emoji']} {escape_markdown(rarity.capitalize())} - {data['chance']}%"
        for rarity, data in CHARACTERS.items()
    )
    
    keyboard = [
        [
            InlineKeyboardButton("🎲 Вытянуть персонажа", callback_data="roll_gacha"),
            InlineKeyboardButton("👤 Мой профиль", callback_data="show_profile"),
            InlineKeyboardButton("💰 JJK Coins", callback_data="show_coins")
        ],
        [
            InlineKeyboardButton("🏆 Топ игроков", callback_data="show_top_menu"),
            InlineKeyboardButton("🃏 Показать карту", callback_data="show_card"),
            InlineKeyboardButton("❤️ Выбрать любимого персонажа", callback_data="choose_favorite_menu")
        ],
        [
            InlineKeyboardButton("🛒 Магазин", callback_data="show_shop")
        ]
    ]
    
    # Добавляем кнопку админ-панели для администраторов
    if user_id in ADMIN_IDS:
        keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
    
    welcome_text = (
        f"{escape_markdown('🔥 Добро пожаловать в GACHA JJK! 🔥')}\n\n"
        f"{escape_markdown('Шансы выпадения персонажей:')}\n{chances_text}\n\n"
        f"{escape_markdown('Нажми кнопку, чтобы попробовать удачу:')}\n"
        f"{escape_markdown('(можно использовать раз в час или с ваучером)')}"
    )
    
    try:
        await context.bot.send_photo(
            chat_id=update.effective_chat.id,
            photo=WELCOME_IMAGE,
            caption=welcome_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
    except Exception as e:
        print(f"Ошибка при отправке фото в /start: {e}")
        await update.message.reply_text(
            welcome_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def shop_command(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /shop"""
    user_id = update.effective_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await update.message.reply_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    shop_text = f"{escape_markdown('🛒 Добро пожаловать в магазин JJK!')}\n\n"
    shop_text += f"{escape_markdown('Ваш баланс:')} {profile['coins']} JJK Coins\n\n"
    shop_text += f"{escape_markdown('Доступные товары:')}\n"
    
    keyboard = []
    for item_id, item in SHOP_ITEMS.items():
        shop_text += (
            f"{item['emoji']} {escape_markdown(item['name'])} - {item['cost']} монет\n"
            f"  {escape_markdown(item['description'])}\n\n"
        )
        keyboard.append([InlineKeyboardButton(
            f"{item['emoji']} Купить {item['name']} ({item['cost']} монет)",
            callback_data=f"buy_item_{item_id}"
        )])
    
    keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")])
    
    await update.message.reply_text(
        shop_text,
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def show_shop(update: Update, context: CallbackContext) -> None:
    """Показывает магазин (обработчик кнопки)"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка show_shop для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    shop_text = f"{escape_markdown('🛒 Добро пожаловать в магазин JJK!')}\n\n"
    shop_text += f"{escape_markdown('Ваш баланс:')} {profile['coins']} JJK Coins\n\n"
    shop_text += f"{escape_markdown('Доступные товары:')}\n"
    
    keyboard = []
    for item_id, item in SHOP_ITEMS.items():
        shop_text += (
            f"{item['emoji']} {escape_markdown(item['name'])} - {item['cost']} монет\n"
            f"  {escape_markdown(item['description'])}\n\n"
        )
        keyboard.append([InlineKeyboardButton(
            f"{item['emoji']} Купить {item['name']} ({item['cost']} монет)",
            callback_data=f"buy_item_{item_id}"
        )])
    
    keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")])
    
    try:
        await query.edit_message_text(
            shop_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
    except Exception as e:
        print(f"Ошибка при редактировании сообщения в show_shop: {e}")
        await query.message.reply_text(
            shop_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def buy_item(update: Update, context: CallbackContext) -> None:
    """Обработчик покупки товара"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка buy_item для пользователя {query.from_user.id}, item: {query.data}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    item_id = query.data.split("_")[-1]
    if item_id not in SHOP_ITEMS:
        await query.edit_message_text(
            escape_markdown("❌ Товар не найден."),
            parse_mode="HTML"
        )
        return
    
    item = SHOP_ITEMS[item_id]
    if profile["coins"] < item["cost"]:
        await query.edit_message_text(
            escape_markdown(f"❌ Недостаточно JJK Coins для покупки {item['name']}. "
                          f"Требуется {item['cost']} монет, у вас {profile['coins']} монет."),
            parse_mode="HTML"
        )
        return
    
    # Списываем монеты
    profile["coins"] -= item["cost"]
    
    # Обработка покупки
    if item_id == "gacha_roll_voucher":
        profile["inventory"][item_id] = profile["inventory"].get(item_id, 0) + 1
        purchase_message = f"✅ Вы успешно купили {item['name']} за {item['cost']} JJK Coins!\n"
        purchase_message += f"Ваш текущий баланс: {profile['coins']} монет"
    else:
        # Покупка карты персонажа
        rarity = item["rarity"]
        character_data = random.choice(CHARACTERS[rarity]["list"])
        profile["characters"].append(character_data)
        profile[f"{rarity}_count"] += 1
        purchase_message = (
            f"✅ Вы успешно купили {item['name']} за {item['cost']} JJK Coins!\n"
            f"Получен персонаж: {item['emoji']} {escape_markdown(character_data['name'])} ({rarity.upper()})\n"
            f"Ваш текущий баланс: {profile['coins']} монет"
        )
    
    save_profiles()
    
    await query.edit_message_text(
        escape_markdown(purchase_message),
        parse_mode="HTML"
    )
    
    # Возвращаемся в магазин через 2 секунды
    await asyncio.sleep(2)
    await show_shop(update, context)

async def choose_favorite_menu(update: Update, context: CallbackContext) -> None:
    """Показывает меню выбора любимого персонажа"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка choose_favorite_menu для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    if not profile["characters"]:
        await query.edit_message_text(
            escape_markdown("❌ У вас нет персонажей в коллекции. Используйте /start для получения первого персонажа."),
            parse_mode="HTML"
        )
        return
    
    # Создаем клавиатуру с персонажами пользователя
    keyboard = []
    for i in range(0, len(profile["characters"]), 2):
        row = []
        for j in range(2):
            if i + j < len(profile["characters"]):
                character = profile["characters"][i + j]
                rarity = next(
                    r for r in CHARACTERS 
                    if any(c["name"] == character["name"] for c in CHARACTERS[r]["list"])
                )
                emoji = CHARACTERS[rarity]["emoji"]
                button_text = f"{emoji} {character['name']}"
                callback_data = f"set_favorite_{i + j}"
                row.append(InlineKeyboardButton(button_text, callback_data=callback_data))
        if row:
            keyboard.append(row)
    
    keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")])
    
    try:
        await query.edit_message_text(
            escape_markdown("❤️ Выберите любимого персонажа из вашей коллекции:"),
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
    except Exception as e:
        print(f"Ошибка при редактировании сообщения в choose_favorite_menu: {e}")
        await query.message.reply_text(
            escape_markdown("❤️ Выберите любимого персонажа из вашей коллекции:"),
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def choose_favorite(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /choose_favorite"""
    user_id = update.effective_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await update.message.reply_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    if not profile["characters"]:
        await update.message.reply_text(
            escape_markdown("❌ У вас нет персонажей в коллекции. Используйте /start для получения первого персонажа."),
            parse_mode="HTML"
        )
        return
    
    # Создаем клавиатуру с персонажами пользователя
    keyboard = []
    for i in range(0, len(profile["characters"]), 2):
        row = []
        for j in range(2):
            if i + j < len(profile["characters"]):
                character = profile["characters"][i + j]
                rarity = next(
                    r for r in CHARACTERS 
                    if any(c["name"] == character["name"] for c in CHARACTERS[r]["list"])
                )
                emoji = CHARACTERS[rarity]["emoji"]
                button_text = f"{emoji} {character['name']}"
                callback_data = f"set_favorite_{i + j}"
                row.append(InlineKeyboardButton(button_text, callback_data=callback_data))
        if row:
            keyboard.append(row)
    
    keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")])
    
    await update.message.reply_text(
        escape_markdown("❤️ Выберите любимого персонажа из вашей коллекции:"),
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def set_favorite(update: Update, context: CallbackContext) -> None:
    """Устанавливает выбранного персонажа как любимого"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка set_favorite для пользователя {query.from_user.id}, data: {query.data}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    character_index = int(query.data.split("_")[-1])
    
    if character_index < 0 or character_index >= len(profile["characters"]):
        await query.edit_message_text(
            escape_markdown("❌ Неверный выбор персонажа."),
            parse_mode="HTML"
        )
        return
    
    profile["favorite_character"] = profile["characters"][character_index]
    save_profiles()
    character_data = profile["favorite_character"]
    rarity = next(
        r for r in CHARACTERS 
        if any(c["name"] == character_data["name"] for c in CHARACTERS[r]["list"])
    )
    emoji = CHARACTERS[rarity]["emoji"]
    
    try:
        await query.edit_message_text(
            escape_markdown(f"{emoji} {character_data['name']} теперь ваш любимый персонаж!"),
            parse_mode="HTML"
        )
    except Exception as e:
        print(f"Ошибка при редактировании сообщения в set_favorite: {e}")
        await query.message.reply_text(
            escape_markdown(f"{emoji} {character_data['name']} теперь ваш любимый персонаж!"),
            parse_mode="HTML"
        )
    
    await asyncio.sleep(2)
    await back_to_main(update, context)

async def coins_command(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /coins"""
    user_id = update.effective_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await update.message.reply_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    coins_text = (
        f"💰 Ваш баланс JJK Coins: {profile['coins']}\n\n"
        f"Способы получения монет:\n"
        f"🎲 Вытягивание персонажей:\n"
        f"⭐ Обычный: {COINS_REWARD['common']} монет\n"
        f"⭐⭐ Редкий: {COINS_REWARD['rare']} монет\n"
        f"⭐⭐⭐ Эпический: {COINS_REWARD['epic']} монет\n"
        f"⭐⭐⭐⭐ Легендарный: {COINS_REWARD['legendary']} монет\n"
        f"🌟🌟🌟🌟🌟 Ультралегендарный: {COINS_REWARD['ultra_legendary']} монет"
    )
    
    keyboard = [
        [InlineKeyboardButton("🛒 Магазин", callback_data="show_shop"),
         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    await update.message.reply_text(
        coins_text,
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def show_coins(update: Update, context: CallbackContext) -> None:
    """Показывает информацию о монетах пользователя"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка show_coins для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    coins_text = (
        f"💰 Ваш баланс JJK Coins: {profile['coins']}\n\n"
        f"Способы получения монет:\n"
        f"🎲 Вытягивание персонажей:\n"
        f"⭐ Обычный: {COINS_REWARD['common']} монет\n"
        f"⭐⭐ Редкий: {COINS_REWARD['rare']} монет\n"
        f"⭐⭐⭐ Эпический: {COINS_REWARD['epic']} монет\n"
        f"⭐⭐⭐⭐ Легендарный: {COINS_REWARD['legendary']} монет\n"
        f"🌟🌟🌟🌟🌟 Ультралегендарный: {COINS_REWARD['ultra_legendary']} монет"
    )
    
    keyboard = [
        [InlineKeyboardButton("🛒 Магазин", callback_data="show_shop"),
         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    try:
        await query.edit_message_text(
            coins_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
    except Exception as e:
        print(f"Ошибка при редактировании сообщения в show_coins: {e}")
        await query.message.reply_text(
            coins_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def profile_command(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /profile"""
    user_id = update.effective_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await update.message.reply_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    characters_list = profile["characters"][:5]
    characters_text = "\n".join(
        f"{CHARACTERS[next(r for r in CHARACTERS if any(c['name'] == char['name'] for c in CHARACTERS[r]['list']))]['emoji']} {escape_markdown(char['name'])}"
        for char in characters_list
    )
    
    if len(profile["characters"]) > 5:
        characters_text += f"\n...и ещё {len(profile['characters']) - 5} персонажей"
    
    inventory_text = "\n".join(
        f"{SHOP_ITEMS[item_id]['emoji']} {escape_markdown(SHOP_ITEMS[item_id]['name'])}: {count}"
        for item_id, count in profile["inventory"].items() if count > 0
    ) or "Пусто"
    
    profile_text = (
        f"👤 Профиль игрока: {escape_markdown(profile['username'])}\n\n"
        f"💰 JJK Coins: {profile.get('coins', 0)}\n"
        f"🎲 Всего попыток: {profile['total_rolls']}\n"
        f"💎 Ультралегендарных: {profile.get('ultra_legendary_count', 0)}\n"
        f"✨ Легендарных: {profile['legendary_count']}\n"
        f"🌕 Эпических: {profile['epic_count']}\n"
        f"🌗 Редких: {profile['rare_count']}\n"
        f"🌑 Обычных: {profile['common_count']}\n\n"
        f"📜 Персонажи:\n{characters_text}\n\n"
        f"🎒 Инвентарь:\n{inventory_text}"
    )
    
    keyboard = [
        [InlineKeyboardButton("🛒 Магазин", callback_data="show_shop"),
         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    if user_id in ADMIN_IDS:
        keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
    
    if profile.get("favorite_character"):
        favorite_char = profile["favorite_character"]
        rarity = next(
            r for r in CHARACTERS 
            if any(c["name"] == favorite_char["name"] for c in CHARACTERS[r]["list"])
        )
        emoji = CHARACTERS[rarity]["emoji"]
        favorite_text = (
            f"\n\n❤️ Любимый персонаж: {emoji} {escape_markdown(favorite_char['name'])}\n"
            f"⚡ Редкость: {escape_markdown(rarity.upper())}"
        )
        profile_text += favorite_text
        
        try:
            await context.bot.send_photo(
                chat_id=update.effective_chat.id,
                photo=favorite_char["image"],
                caption=profile_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
        except Exception as e:
            print(f"Ошибка при отправке фото профиля: {e}")
            await update.message.reply_text(
                escape_markdown(f"❌ Не удалось загрузить изображение любимого персонажа\n\n{profile_text}"),
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
    else:
        profile_text += f"\n\n❤️ Любимый персонаж: не выбран"
        await update.message.reply_text(
            profile_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def show_profile(update: Update, context: CallbackContext) -> None:
    """Показывает профиль пользователя"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка show_profile для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    characters_list = profile["characters"][:5]
    characters_text = "\n".join(
        f"{CHARACTERS[next(r for r in CHARACTERS if any(c['name'] == char['name'] for c in CHARACTERS[r]['list']))]['emoji']} {escape_markdown(char['name'])}"
        for char in characters_list
    )
    
    if len(profile["characters"]) > 5:
        characters_text += f"\n...и ещё {len(profile['characters']) - 5} персонажей"
    
    inventory_text = "\n".join(
        f"{SHOP_ITEMS[item_id]['emoji']} {escape_markdown(SHOP_ITEMS[item_id]['name'])}: {count}"
        for item_id, count in profile["inventory"].items() if count > 0
    ) or "Пусто"
    
    profile_text = (
        f"👤 Профиль игрока: {escape_markdown(profile['username'])}\n\n"
        f"💰 JJK Coins: {profile.get('coins', 0)}\n"
        f"🎲 Всего попыток: {profile['total_rolls']}\n"
        f"💎 Ультралегендарных: {profile.get('ultra_legendary_count', 0)}\n"
        f"✨ Легендарных: {profile['legendary_count']}\n"
        f"🌕 Эпических: {profile['epic_count']}\n"
        f"🌗 Редких: {profile['rare_count']}\n"
        f"🌑 Обычных: {profile['common_count']}\n\n"
        f"📜 Персонажи:\n{characters_text}\n\n"
        f"🎒 Инвентарь:\n{inventory_text}"
    )
    
    keyboard = [
        [InlineKeyboardButton("🛒 Магазин", callback_data="show_shop"),
         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    if user_id in ADMIN_IDS:
        keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
    
    if profile.get("favorite_character"):
        favorite_char = profile["favorite_character"]
        rarity = next(
            r for r in CHARACTERS 
            if any(c["name"] == favorite_char["name"] for c in CHARACTERS[r]["list"])
        )
        emoji = CHARACTERS[rarity]["emoji"]
        favorite_text = (
            f"\n\n❤️ Любимый персонаж: {emoji} {escape_markdown(favorite_char['name'])}\n"
            f"⚡ Редкость: {escape_markdown(rarity.upper())}"
        )
        profile_text += favorite_text
        
        try:
            await context.bot.send_photo(
                chat_id=query.message.chat_id,
                photo=favorite_char["image"],
                caption=profile_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
            await query.message.delete()
        except Exception as e:
            print(f"Ошибка при отправке фото профиля: {e}")
            await query.edit_message_text(
                escape_markdown(f"❌ Не удалось загрузить изображение любимого персонажа\n\n{profile_text}"),
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
    else:
        profile_text += f"\n\n❤️ Любимый персонаж: не выбран"
        try:
            await query.edit_message_text(
                profile_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
        except Exception as e:
            print(f"Ошибка при редактировании сообщения в show_profile: {e}")
            await query.message.reply_text(
                profile_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )

async def back_to_main(update: Update, context: CallbackContext) -> None:
    """Возвращает в главное меню"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка back_to_main для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    chances_text = "\n".join(
        f"{data['emoji']} {escape_markdown(rarity.capitalize())} - {data['chance']}%"
        for rarity, data in CHARACTERS.items()
    )
    
    keyboard = [
        [
            InlineKeyboardButton("🎲 Вытянуть персонажа", callback_data="roll_gacha"),
            InlineKeyboardButton("👤 Мой профиль", callback_data="show_profile"),
            InlineKeyboardButton("💰 JJK Coins", callback_data="show_coins")
        ],
        [
            InlineKeyboardButton("🏆 Топ игроков", callback_data="show_top_menu"),
            InlineKeyboardButton("🃏 Показать карту", callback_data="show_card"),
            InlineKeyboardButton("❤️ Выбрать любимого персонажа", callback_data="choose_favorite_menu")
        ],
        [
            InlineKeyboardButton("🛒 Магазин", callback_data="show_shop")
        ]
    ]
    
    if user_id in ADMIN_IDS:
        keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
    
    welcome_text = (
        f"{escape_markdown('🔥 Добро пожаловать в GACHA JJK! 🔥')}\n\n"
        f"{escape_markdown('Шансы выпадения персонажей:')}\n{chances_text}\n\n"
        f"{escape_markdown('Нажми кнопку, чтобы попробовать удачу:')}\n"
        f"{escape_markdown('(можно использовать раз в час или с ваучером)')}"
    )
    
    try:
        await context.bot.send_photo(
            chat_id=query.message.chat_id,
            photo=WELCOME_IMAGE,
            caption=welcome_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
        await query.message.delete()
    except Exception as e:
        print(f"Ошибка при отправке фото в back_to_main: {e}")
        await query.edit_message_text(
            welcome_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def gacha_roll(update: Update, context: CallbackContext) -> None:
    """Генерация случайного персонажа с ограничением по времени или ваучером"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка gacha_roll для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    current_time = datetime.now()
    is_admin = user_id in ADMIN_IDS
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    # Проверка кулдауна и ваучеров (если пользователь не админ)
    use_voucher = False
    if not is_admin and user_id in user_cooldowns:
        last_use = user_cooldowns[user_id]
        time_left = (last_use + timedelta(hours=1)) - current_time
        
        if time_left.total_seconds() > 0:
            if profile["inventory"].get("gacha_roll_voucher", 0) > 0:
                # Пользователь может использовать ваучер
                keyboard = [
                    [InlineKeyboardButton("🎟️ Использовать ваучер", callback_data="use_voucher_gacha"),
                     InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
                ]
                minutes_left = time_left.seconds // 60
                seconds_left = time_left.seconds % 60
                await query.edit_message_text(
                    escape_markdown(
                        f"⏳ Подождите ещё {minutes_left} минут {seconds_left} секунд\n"
                        f"⌛ Последняя попытка была в {last_use.strftime('%H:%M:%S')}\n\n"
                        f"🎟️ У вас есть ваучеры ({profile['inventory'].get('gacha_roll_voucher', 0)}). "
                        f"Использовать ваучер для мгновенной попытки?"
                    ),
                    parse_mode="HTML",
                    reply_markup=InlineKeyboardMarkup(keyboard)
                )
                return
            else:
                minutes_left = time_left.seconds // 60
                seconds_left = time_left.seconds % 60
                await query.edit_message_text(
                    escape_markdown(
                        f"⏳ Подождите ещё {minutes_left} минут {seconds_left} секунд\n"
                        f"⌛ Последняя попытка была в {last_use.strftime('%H:%M:%S')}\n\n"
                        f"🛒 Вы можете купить ваучер в магазине!"
                    ),
                    parse_mode="HTML",
                    reply_markup=InlineKeyboardMarkup([
                        [InlineKeyboardButton("🛒 Магазин", callback_data="show_shop"),
                         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
                    ])
                )
                return
    
    # Если пользователь использует ваучер
    if query.data == "use_voucher_gacha":
        profile["inventory"]["gacha_roll_voucher"] -= 1
        use_voucher = True
    
    # Обновляем время последнего использования (если пользователь не админ и не использует ваучер)
    if not is_admin and not use_voucher:
        user_cooldowns[user_id] = current_time
    
    # Логика выбора персонажа
    rand = random.randint(1, 100)
    rarity = "common"
    for rar, data in CHARACTERS.items():
        if rand <= data["chance"]:
            rarity = rar
            break
        rand -= data["chance"]

    character_data = random.choice(CHARACTERS[rarity]["list"])
    character_name = escape_markdown(character_data["name"])
    character_image = character_data["image"]
    emoji = CHARACTERS[rarity]["emoji"]
    
    profile["characters"].append(character_data)
    profile["total_rolls"] += 1
    
    coins_reward = COINS_REWARD[rarity]
    profile["coins"] = profile.get("coins", 0) + coins_reward
    
    if rarity == "ultra_legendary":
        profile["ultra_legendary_count"] += 1
    elif rarity == "legendary":
        profile["legendary_count"] += 1
    elif rarity == "epic":
        profile["epic_count"] += 1
    elif rarity == "rare":
        profile["rare_count"] += 1
    else:
        profile["common_count"] += 1
    
    save_profiles()
    
    admin_note = "\nГЕЙ НАЙДЕН: теперь ты пидор" if is_admin else ""
    voucher_note = "\n🎟️ Использован ваучер!" if use_voucher else ""
    result_text = (
        f"{emoji} {escape_markdown('Ты получил:')} {character_name}\n"
        f"⚡ {escape_markdown('Редкость:')} {escape_markdown(rarity.upper())}\n"
        f"💰 {escape_markdown('Получено JJK Coins:')} {coins_reward}\n"
        f"🎲 {escape_markdown('Шанс выпадения:')} {CHARACTERS[rarity]['chance']}%{admin_note}{voucher_note}\n\n"
    )
    
    if not is_admin and not use_voucher:
        result_text += (
            f"⏳ {escape_markdown('Следующая попытка будет доступна в')} "
            f"{escape_markdown((current_time + timedelta(hours=1)).strftime('%H:%M:%S'))}\n"
        )
    
    result_text += f"🎮 {escape_markdown('Для новой попытки нажми /start')}"
    if profile["inventory"].get("gacha_roll_voucher", 0) > 0:
        result_text += f"\n🎟️ Осталось ваучеров: {profile['inventory']['gacha_roll_voucher']}"

    try:
        await context.bot.send_photo(
            chat_id=query.message.chat_id,
            photo=character_image,
            caption=result_text,
            parse_mode="HTML"
        )
        
        keyboard = [
            [
                InlineKeyboardButton("🎲 Вытянуть персонажа", callback_data="roll_gacha"),
                InlineKeyboardButton("👤 Мой профиль", callback_data="show_profile"),
                InlineKeyboardButton("💰 JJK Coins", callback_data="show_coins")
            ],
            [
                InlineKeyboardButton("🏆 Топ игроков", callback_data="show_top_menu"),
                InlineKeyboardButton("🃏 Показать карту", callback_data="show_card"),
                InlineKeyboardButton("❤️ Выбрать любимого персонажа", callback_data="choose_favorite_menu")
            ],
            [
                InlineKeyboardButton("🛒 Магазин", callback_data="show_shop")
            ]
        ]
        if user_id in ADMIN_IDS:
            keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
        
        await query.edit_message_reply_markup(reply_markup=InlineKeyboardMarkup(keyboard))
        
    except Exception as e:
        print(f"Ошибка при отправке фото в gacha_roll: {e}")
        await query.edit_message_text(
            escape_markdown(
                f"❌ Не удалось загрузить изображение\n\n"
                f"{emoji} Ты получил: {character_name}\n"
                f"⚡ Редкость: {rarity.upper()}\n"
                f"💰 Получено JJK Coins: {coins_reward}\n"
                f"🎲 Шанс выпадения: {CHARACTERS[rarity]['chance']}%{admin_note}{voucher_note}"
            ),
            parse_mode="HTML"
        )

async def use_voucher_gacha(update: Update, context: CallbackContext) -> None:
    """Обработчик использования ваучера для гacha roll"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка use_voucher_gacha для пользователя {query.from_user.id}")
    await gacha_roll(update, context)

async def top_command(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /top"""
    await show_top_players(update, context)

async def show_top_menu(update: Update, context: CallbackContext) -> None:
    """Показывает меню топа игроков"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка show_top_menu для пользователя {query.from_user.id}")
    await show_top_players(update, context, is_callback=True)

async def show_top_players(update: Update, context: CallbackContext, is_callback: bool = False) -> None:
    """Функция для отображения топа игроков по монетам"""
    sorted_users = sorted(
        user_profiles.items(),
        key=lambda x: x[1].get("coins", 0),
        reverse=True
    )
    
    top_text = "🏆 Топ игроков по JJK Coins 🏆\n\n"
    for i, (user_id, profile) in enumerate(sorted_users[:10], 1):
        username = escape_markdown(profile["username"])
        coins = profile.get("coins", 0)
        top_text += f"{i}. {username} - {coins} монет\n"
    
    current_user_id = update.effective_user.id if is_callback else update.message.from_user.id
    current_profile = user_profiles.get(current_user_id)
    
    if current_profile:
        current_username = escape_markdown(current_profile["username"])
        current_coins = current_profile.get("coins", 0)
        current_position = next(
            (i+1 for i, (user_id, _) in enumerate(sorted_users) if user_id == current_user_id),
            None
        )
        
        if current_position is None:
            top_text += f"\nВаше место: не в топе\n{current_username} - {current_coins} монет"
        elif current_position > 10:
            top_text += f"\nВаше место: {current_position}\n{current_username} - {current_coins} монет"
    
    keyboard = [
        [InlineKeyboardButton("🛒 Магазин", callback_data="show_shop"),
         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    if current_user_id in ADMIN_IDS:
        keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
    
    try:
        if is_callback:
            await update.callback_query.edit_message_text(
                top_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
        else:
            await update.message.reply_text(
                top_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
    except Exception as e:
        print(f"Ошибка при отправке топа игроков: {e}")
        await (update.callback_query.message.reply_text if is_callback else update.message.reply_text)(
            top_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def card_command(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /card"""
    user_id = update.effective_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await update.message.reply_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    if not profile["characters"]:
        await update.message.reply_text(
            escape_markdown("❌ У вас нет персонажей в коллекции. Используйте /start для получения первого персонажа."),
            parse_mode="HTML"
        )
        return
    
    character_data = random.choice(profile["characters"])
    rarity = next(
        r for r in CHARACTERS 
        if any(c["name"] == character_data["name"] for c in CHARACTERS[r]["list"])
    )
    
    emoji = CHARACTERS[rarity]["emoji"]
    character_name = escape_markdown(character_data["name"])
    character_image = character_data["image"]
    
    card_text = (
        f"{emoji} {escape_markdown('Карта персонажа:')} {character_name}\n"
        f"⚡ {escape_markdown('Редкость:')} {escape_markdown(rarity.upper())}\n"
        f"🎮 {escape_markdown('Всего персонажей в коллекции:')} {len(profile['characters'])}\n\n"
        f"🔄 Используйте /card снова для просмотра другой карты"
    )
    
    try:
        await context.bot.send_photo(
            chat_id=update.effective_chat.id,
            photo=character_image,
            caption=card_text,
            parse_mode="HTML"
        )
    except Exception as e:
        print(f"Ошибка при отправке фото в card_command: {e}")
        await update.message.reply_text(
            escape_markdown(
                f"❌ Не удалось загрузить изображение\n\n"
                f"{emoji} Карта персонажа: {character_name}\n"
                f"⚡ Редкость: {rarity.upper()}\n"
                f"🎮 Всего персонажей в коллекции: {len(profile['characters'])}"
            ),
            parse_mode="HTML"
        )

async def show_card(update: Update, context: CallbackContext) -> None:
    """Показывает карту персонажа (обработчик кнопки)"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка show_card для пользователя {query.from_user.id}")
    
    user_id = query.from_user.id
    profile = user_profiles.get(user_id, None)
    
    if not profile:
        await query.edit_message_text(
            escape_markdown("❌ Профиль не найден. Нажмите /start для создания."),
            parse_mode="HTML"
        )
        return
    
    if not profile["characters"]:
        await query.edit_message_text(
            escape_markdown("❌ У вас нет персонажей в коллекции. Используйте /start для получения первого персонажа."),
            parse_mode="HTML"
        )
        return
    
    character_data = random.choice(profile["characters"])
    rarity = next(
        r for r in CHARACTERS 
        if any(c["name"] == character_data["name"] for c in CHARACTERS[r]["list"])
    )
    
    emoji = CHARACTERS[rarity]["emoji"]
    character_name = escape_markdown(character_data["name"])
    character_image = character_data["image"]
    
    card_text = (
        f"{emoji} {escape_markdown('Карта персонажа:')} {character_name}\n"
        f"⚡ {escape_markdown('Редкость:')} {escape_markdown(rarity.upper())}\n"
        f"🎮 {escape_markdown('Всего персонажей в коллекции:')} {len(profile['characters'])}\n\n"
        f"🔄 Нажмите кнопку снова для просмотра другой карты"
    )
    
    keyboard = [
        [InlineKeyboardButton("🃏 Показать другую карту", callback_data="show_card"),
         InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    if user_id in ADMIN_IDS:
        keyboard.append([InlineKeyboardButton("🔧 Админ-панель", callback_data="show_admin_panel")])
    
    try:
        await context.bot.send_photo(
            chat_id=query.message.chat_id,
            photo=character_image,
            caption=card_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
        await query.message.delete()
    except Exception as e:
        print(f"Ошибка при отправке фото в show_card: {e}")
        await query.edit_message_text(
            escape_markdown(
                f"❌ Не удалось загрузить изображение\n\n"
                f"{emoji} Карта персонажа: {character_name}\n"
                f"⚡ Редкость: {rarity.upper()}\n"
                f"🎮 Всего персонажей в коллекции: {len(profile['characters'])}"
            ),
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def admin_panel_command(update: Update, context: CallbackContext) -> None:
    """Обработчик команды /panel для админов"""
    user_id = update.effective_user.id
    if user_id not in ADMIN_IDS:
        await update.message.reply_text(
            escape_markdown("❌ Доступ запрещен. Эта команда только для администраторов."),
            parse_mode="HTML"
        )
        return
    
    await show_admin_panel(update, context)

async def show_admin_panel(update: Update, context: CallbackContext) -> None:
    """Показывает админ-панель"""
    query = update.callback_query
    if query:
        await query.answer()
        print(f"Обработка show_admin_panel для пользователя {query.from_user.id}")
    else:
        print(f"Обработка admin_panel_command для пользователя {update.effective_user.id}")
    
    user_id = query.from_user.id if query else update.effective_user.id
    if user_id not in ADMIN_IDS:
        await (query.edit_message_text if query else update.message.reply_text)(
            escape_markdown("❌ Доступ запрещен. Эта команда только для администраторов."),
            parse_mode="HTML"
        )
        return
    
    admin_text = escape_markdown("🔧 Админ-панель\n\nВыберите действие:")
    keyboard = [
        [InlineKeyboardButton("💰 Выдать монеты", callback_data="admin_give_coins")],
        [InlineKeyboardButton("💸 Забрать монеты", callback_data="admin_take_coins")],
        [InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]
    ]
    
    try:
        if query:
            await query.edit_message_text(
                admin_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
        else:
            await update.message.reply_text(
                admin_text,
                parse_mode="HTML",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )
    except Exception as e:
        print(f"Ошибка при отображении админ-панели: {e}")
        await (query.message.reply_text if query else update.message.reply_text)(
            admin_text,
            parse_mode="HTML",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

async def admin_give_coins(update: Update, context: CallbackContext) -> None:
    """Обработчик выдачи монет"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка admin_give_coins для пользователя {query.from_user.id}")
    
    if query.from_user.id not in ADMIN_IDS:
        await query.edit_message_text(
            escape_markdown("❌ Доступ запрещен. Эта команда только для администраторов."),
            parse_mode="HTML"
        )
        return
    
    # Список пользователей для выбора
    keyboard = []
    for user_id, profile in user_profiles.items():
        username = escape_markdown(profile["username"])
        keyboard.append([InlineKeyboardButton(
            f"{username} (ID: {user_id})",
            callback_data=f"admin_give_coins_user_{user_id}"
        )])
    
    keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="show_admin_panel")])
    
    await query.edit_message_text(
        escape_markdown("💰 Выберите пользователя для выдачи монет:"),
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def admin_take_coins(update: Update, context: CallbackContext) -> None:
    """Обработчик изъятия монет"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка admin_take_coins для пользователя {query.from_user.id}")
    
    if query.from_user.id not in ADMIN_IDS:
        await query.edit_message_text(
            escape_markdown("❌ Доступ запрещен. Эта команда только для администраторов."),
            parse_mode="HTML"
        )
        return
    
    # Список пользователей для выбора
    keyboard = []
    for user_id, profile in user_profiles.items():
        username = escape_markdown(profile["username"])
        keyboard.append([InlineKeyboardButton(
            f"{username} (ID: {user_id})",
            callback_data=f"admin_take_coins_user_{user_id}"
        )])
    
    keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="show_admin_panel")])
    
    await query.edit_message_text(
        escape_markdown("💸 Выберите пользователя для изъятия монет:"),
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def admin_select_coins_amount(update: Update, context: CallbackContext) -> None:
    """Обработчик выбора количества монет"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка admin_select_coins_amount для пользователя {query.from_user.id}, data: {query.data}")
    
    if query.from_user.id not in ADMIN_IDS:
        await query.edit_message_text(
            escape_markdown("❌ Доступ запрещен. Эта команда только для администраторов."),
            parse_mode="HTML"
        )
        return
    
    data_parts = query.data.split("_")
    action = data_parts[1]  # give or take
    target_user_id = int(data_parts[-1])
    
    if target_user_id not in user_profiles:
        await query.edit_message_text(
            escape_markdown("❌ Пользователь не найден."),
            parse_mode="HTML"
        )
        return
    
    username = escape_markdown(user_profiles[target_user_id]["username"])
    action_text = "выдачи" if action == "give" else "изъятия"
    
    keyboard = [
        [InlineKeyboardButton("10", callback_data=f"admin_{action}_coins_{target_user_id}_10"),
         InlineKeyboardButton("50", callback_data=f"admin_{action}_coins_{target_user_id}_50"),
         InlineKeyboardButton("100", callback_data=f"admin_{action}_coins_{target_user_id}_100")],
        [InlineKeyboardButton("500", callback_data=f"admin_{action}_coins_{target_user_id}_500"),
         InlineKeyboardButton("1000", callback_data=f"admin_{action}_coins_{target_user_id}_1000")],
        [InlineKeyboardButton("🔙 Назад", callback_data=f"admin_{action}_coins")]
    ]
    
    await query.edit_message_text(
        escape_markdown(f"💰 Выберите количество монет для {action_text} пользователю {username}:"),
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def admin_process_coins(update: Update, context: CallbackContext) -> None:
    """Обработчик финальной обработки выдачи/изъятия монет"""
    query = update.callback_query
    await query.answer()
    print(f"Обработка admin_process_coins для пользователя {query.from_user.id}, data: {query.data}")
    
    if query.from_user.id not in ADMIN_IDS:
        await query.edit_message_text(
            escape_markdown("❌ Доступ запрещен. Эта команда только для администраторов."),
            parse_mode="HTML"
        )
        return
    
    data_parts = query.data.split("_")
    action = data_parts[1]  # give or take
    target_user_id = int(data_parts[-2])
    amount = int(data_parts[-1])
    
    if target_user_id not in user_profiles:
        await query.edit_message_text(
            escape_markdown("❌ Пользователь не найден."),
            parse_mode="HTML"
        )
        return
    
    profile = user_profiles[target_user_id]
    username = escape_markdown(profile["username"])
    
    if action == "give":
        profile["coins"] = profile.get("coins", 0) + amount
        action_text = "выдано"
    else:  # take
        new_coins = profile.get("coins", 0) - amount
        if new_coins < 0:
            await query.edit_message_text(
                escape_markdown(f"❌ Нельзя забрать {amount} монет у {username}. У пользователя только {profile['coins']} монет."),
                parse_mode="HTML"
            )
            return
        profile["coins"] = new_coins
        action_text = "изъято"
    
    save_profiles()
    
    await query.edit_message_text(
        escape_markdown(f"✅ Успешно {action_text} {amount} JJK Coins пользователю {username}!\nТекущий баланс: {profile['coins']} монет"),
        parse_mode="HTML"
    )
    
    # Уведомляем пользователя
    try:
        await context.bot.send_message(
            chat_id=target_user_id,
            text=escape_markdown(f"💰 Ваш баланс JJK Coins изменён администратором!\n{action_text.capitalize()} {amount} монет.\nТекущий баланс: {profile['coins']} монет"),
            parse_mode="HTML"
        )
    except Exception as e:
        print(f"Ошибка при отправке уведомления пользователю {target_user_id}: {e}")
    
    # Возвращаемся в админ-панель через 2 секунды
    await asyncio.sleep(2)
    await show_admin_panel(update, context)

def remove_expired_cooldowns():
    """Очистка устаревших записей о кд"""
    current_time = datetime.now()
    global user_cooldowns
    user_cooldowns = {
        user_id: last_use 
        for user_id, last_use in user_cooldowns.items()
        if (current_time - last_use) < timedelta(hours=2)
    }

def main() -> None:
    """Запуск бота"""
    load_profiles()
    remove_expired_cooldowns()
    
    application = Application.builder().token(TOKEN).build()
    
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("profile", profile_command))
    application.add_handler(CommandHandler("coins", coins_command))
    application.add_handler(CommandHandler("top", top_command))
    application.add_handler(CommandHandler("card", card_command))
    application.add_handler(CommandHandler("shop", shop_command))
    application.add_handler(CommandHandler("choose_favorite", choose_favorite))
    application.add_handler(CommandHandler("panel", admin_panel_command))
    
    application.add_handler(CallbackQueryHandler(gacha_roll, pattern="^roll_gacha$"))
    application.add_handler(CallbackQueryHandler(use_voucher_gacha, pattern="^use_voucher_gacha$"))
    application.add_handler(CallbackQueryHandler(show_profile, pattern="^show_profile$"))
    application.add_handler(CallbackQueryHandler(show_coins, pattern="^show_coins$"))
    application.add_handler(CallbackQueryHandler(show_top_menu, pattern="^show_top_menu$"))
        application.add_handler(CallbackQueryHandler(show_card, pattern="^show_card$"))
    application.add_handler(CallbackQueryHandler(back_to_main, pattern="^back_to_main$"))
    application.add_handler(CallbackQueryHandler(choose_favorite_menu, pattern="^choose_favorite_menu$"))
    application.add_handler(CallbackQueryHandler(set_favorite, pattern="^set_favorite_"))
    application.add_handler(CallbackQueryHandler(show_shop, pattern="^show_shop$"))
    application.add_handler(CallbackQueryHandler(buy_item, pattern="^buy_item_"))
    application.add_handler(CallbackQueryHandler(show_admin_panel, pattern="^show_admin_panel$"))
    application.add_handler(CallbackQueryHandler(admin_give_coins, pattern="^admin_give_coins$"))
    application.add_handler(CallbackQueryHandler(admin_take_coins, pattern="^admin_take_coins$"))
    application.add_handler(CallbackQueryHandler(admin_select_coins_amount, pattern="^admin_give_coins_user_"))
    application.add_handler(CallbackQueryHandler(admin_select_coins_amount, pattern="^admin_take_coins_user_"))
    application.add_handler(CallbackQueryHandler(admin_process_coins, pattern="^admin_give_coins_\\d+_\\d+$"))
    application.add_handler(CallbackQueryHandler(admin_process_coins, pattern="^admin_take_coins_\\d+_\\d+$"))

    print("Бот запущен!")
    application.run_polling()
