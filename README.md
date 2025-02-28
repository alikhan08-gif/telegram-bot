from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes
import sqlite3
import os

# Databasega ulanamiz (Agar mavjud bo'lmasa, avtomatik yaratadi)
conn = sqlite3.connect("mydatabase.db")
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS referrals (
    user_id INTEGER PRIMARY KEY,
    referrer_id INTEGER
)
""")
conn.commit()

# TOKEN
TOKEN = '7529217803:AAEK9Lr076vke9ituZV5rrRgzmZXUvR8Z9g'
CHANNEL_URL = 'https://t.me/mykimyo'
CHANNEL_USERNAME = '@mykimyo'
REFERRAL_COUNT = 5

user_data = {}

# /start komandasi
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    keyboard = [
        [InlineKeyboardButton("🔗 Kanalga a'zo bo'lish", url=CHANNEL_URL)],
        [InlineKeyboardButton("✅ Tasdiqlash", callback_data='check_membership')],
        [InlineKeyboardButton("📲 Taklif qilish", callback_data='invite_friends')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    user_id = update.message.from_user.id
    args = context.args

    if args:
        referrer_id = args[0]
        if referrer_id.isdigit() and int(referrer_id) != user_id:
            try:
                cursor.execute("INSERT INTO referrals (user_id, referrer_id) VALUES (?, ?)", (user_id, int(referrer_id)))
                conn.commit()
            except:
                pass

    await update.message.reply_text(
        'Kanalga a\'zo bo\'ling va qaytib "Tasdiqlash" tugmasini bosing.',
        reply_markup=reply_markup
    )

# Obuna tekshirish
async def check_membership(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user = update.callback_query.from_user
    already_subscribed = await is_user_subscribed(user.id, context)

    if already_subscribed:
        await update.callback_query.answer('Siz kanalga a\'zo bo\'lgansiz.')
        await give_referral_link(update, context)
    else:
        await update.callback_query.answer('Iltimos, avval kanalga a\'zo bo\'ling.')

# Referral havola berish
async def give_referral_link(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user_id = update.callback_query.from_user.id
    referral_link = f'https://t.me/kitob_yuklovchi_bot?start={user_id}'
    keyboard = [
        [InlineKeyboardButton("📲 Taklif qilish", callback_data='invite_friends')],
        [InlineKeyboardButton("✅ Taklif qildim", callback_data='confirm_invite')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.callback_query.message.reply_text(
        f'Bu sizning referral havolangiz: {referral_link}\n5 ta do\'stingizni taklif qiling va "Taklif qildim" tugmasini bosing.',
        reply_markup=reply_markup
    )

# Do'st taklif qilish
async def invite_friends(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.callback_query.answer()
    await update.callback_query.message.reply_text('Do\'stlaringizni taklif qilish uchun yuqoridagi havolani ulashing.')

# Referral tasdiqlash
async def confirm_invite(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user_id = update.callback_query.from_user.id
    cursor.execute("SELECT COUNT(*) FROM referrals WHERE referrer_id = ?", (user_id,))
    count = cursor.fetchone()[0]

    if count >= REFERRAL_COUNT:
        await send_book(update, context, user_id)
    else:
        await update.callback_query.answer(
            f'Iltimos, barcha do\'stlaringizni taklif qiling. Siz {count} ta do\'stni taklif qilgansiz.'
        )

# Kitobni yuborish funksiyasi
async def send_book(update: Update, context: ContextTypes.DEFAULT_TYPE, user_id: int) -> None:
    SECRET_CHANNEL_ID = -1002320322668  # Maxfiy kanalingiz ID
    MESSAGE_IDS = [2, 11, 10, 9, 8]  # Forward qilinadigan kitob xabarlarining ID'lari

    try:
        for message_id in MESSAGE_IDS:
            # Har bir xabarni forward qilish
            await context.bot.forward_message(
                chat_id=user_id,
                from_chat_id=SECRET_CHANNEL_ID,
                message_id=message_id
            )
            print(f"Xabar {message_id} muvaffaqiyatli yuborildi.")  # Konsolga natija chiqarish
    except Exception as e:
        await context.bot.send_message(chat_id=user_id, text="Kechirasiz, kitobni forward qilishda xatolik yuz berdi.")
        print(f"Xatolik: {e}")
        # Obuna tekshirish yordamchisi
async def is_user_subscribed(user_id: int, context: ContextTypes.DEFAULT_TYPE) -> bool:
    try:
        member = await context.bot.get_chat_member(CHANNEL_USERNAME, user_id)
        return member.status in ["member", "administrator", "creator"]
    except Exception as e:
        return False

# Asosiy funksiyalarni ishga tushirish
def main() -> None:
    application = Application.builder().token(TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(check_membership, pattern='check_membership'))
    application.add_handler(CallbackQueryHandler(invite_friends, pattern='invite_friends'))
    application.add_handler(CallbackQueryHandler(confirm_invite, pattern='confirm_invite'))

    application.run_polling()

if __name__ == '__main__':
    main()
