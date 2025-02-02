import random
from datetime import datetime
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
import logging
import os

TOKEN = "8138636341:AAGxJdvvYL6h74t2WigcrtcTE94I0b7zkls"
BOT_USERNAME = "@SayyedqBot"

LOG_FILE = "chat_log.txt"

user_stats = {}

user_titles = {}

group_members = []

ranks = ["Novice", "Intermediate", "Expert", "Master", "کیوت"]


def get_formatted_time(timestamp: datetime) -> str:
    return timestamp.strftime("%Y-%m-%d %H:%M:%S")


def get_username(update: Update) -> str:
    username = update.message.from_user.username
    return username if username else "Unknown"


def write_to_log(username, message, reply_text, timestamp):
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(f"{get_formatted_time(timestamp)} - {username}: {message} => {reply_text}\n")


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    reply_text = "سلام! به ربات مدیریت گروه خوش آمدید!"
    await update.message.reply_text(reply_text)
    username = get_username(update)
    write_to_log(username, update.message.text, reply_text, update.message.date)


async def set_title(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if len(context.args) < 2:
        await update.message.reply_text("لطفاً یک کاربر و لقب وارد کنید. مثال: /set_title @username گیج")
        return

    username = context.args[0]
    title = ' '.join(context.args[1:])

    # پیدا کردن کاربری که به عنوان لقب‌گیرنده انتخاب شده
    user_id = None
    for user in group_members:
        if user['username'] == username.lstrip('@'):  # از @ در username صرفنظر می‌کنیم
            user_id = user['id']
            break

    if not user_id:
        await update.message.reply_text(f"کاربری با نام {username} پیدا نشد.")
        return

    # تنظیم لقب کاربر در دیکشنری
    user_titles[user_id] = title
    # ارسال پاسخ به همراه لینک به پروفایل کاربر
    reply_text = f"لقب {title} به کاربر [{username}](tg://user?id={user_id}) اختصاص داده شد."

    await update.message.reply_text(reply_text, parse_mode="Markdown")

    # ثبت در لاگ
    write_to_log(get_username(update), update.message.text, f"لقب {title} به {username} اختصاص داده شد.",
                 update.message.date)


# Message handler برای پاسخ خودکار
async def reply_to_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.message.chat.type in ['group', 'supergroup']:
        message_text = update.message.text

        # اگر پیام "لقب" باشد و کاربری که به آن ریپلای شده، لقب داشته باشد
        if message_text.lower() == "لقب":
            if update.message.reply_to_message:
                replied_user_id = update.message.reply_to_message.from_user.id
                if replied_user_id in user_titles:
                    user_title = user_titles[replied_user_id]
                    username = update.message.reply_to_message.from_user.username
                    reply_text = f"لقب کاربر [{username}](tg://user?id={replied_user_id}): {user_title}"
                else:
                    reply_text = "برای این کاربر لقبی تعیین نشده است."
            else:
                reply_text = "لطفاً یک کاربر را ریپلای کنید."

        elif "سلام" in message_text.lower().split():
            answer_hello = ["سلام", "سلام چطوری", "سلام دلقک", "باز این اومد", "سلام خوشگله", "کصخل اومد"]
            reply_text = random.choice(answer_hello)

        elif "بای" in message_text.lower().split():
            answer_goodbye = ["بای بای", "جیش بوس لالا", "شببخیر دلقک", "بری دیگه بر نگردی", "شببخیر خوب بخوابی",
                              "خداروشکر بالاخره این رفت"]
            reply_text = random.choice(answer_goodbye)

        elif message_text == "سیدبات":
            answer_calling = ["جان", "جونم", "بنال", "چیه", "زیاد صدام میزنی کراشی؟", "من زن دارم"]
            reply_text = random.choice(answer_calling)

        elif message_text.startswith("کی "):
            new_message = message_text[len("کی "):].strip()
            random_member = random.choice(group_members)
            reply_text = f"{new_message} " + f"{random_member['name']}"

        elif message_text == "تگ":
            reply_text = "هر کی تگ کنه کونیه"

        elif "خوبی" in message_text:
            reply_text = "به تو چه"

        elif message_text == "قهرم":
            reply_text = "دلقکو"

        elif "کیوت" in message_text.lower():
            reply_text = "کیوت فقط مبین"

        elif message_text == "بکیرم":
            reply_text = (
                "اصطلاحی که شما دوست عزیز بکار بردید زمانی استفاده میشوند که شما از ماتحت سوخته اید و فشاری شده اید."
                "\n\n روابط عمومی سیدبات")

        elif message_text == "من خدام":
            reply_text = "خفه ضعیفه"

        elif message_text == "دلقک":
            reply_text = "قیافته"

        elif message_text == "کونی":
            reply_text = "چی مینالی تس باقه"

        elif message_text.startswith("تنظیم لقب به"):
            if update.message.reply_to_message:
                username = update.message.reply_to_message.from_user.username
                if username:
                    title = message_text[len("تنظیم لقب به "):]
                    user_id = update.message.reply_to_message.from_user.id
                    user_titles[user_id] = title
                    reply_text = f"کاربر [{username}](tg://user?id={user_id}) به {title} ملقب شد."
                    # ریپلای به کاربر با لقب جدید
                    try:
                        await update.message.reply_to_message.reply(f"چطوری {title}")
                    except AttributeError:
                        # در صورت بروز خطا (در صورتی که ویژگی reply موجود نباشد)، پاسخ به خود پیام داده می‌شود.
                        await update.message.reply_text(f"چطوری {title}")
                else:
                    reply_text = "کاربر نام کاربری ندارد."
            else:
                reply_text = "لطفاً یک کاربر را ریپلای کنید."

        else:
            reply_text = None

        # ارسال پاسخ به پیام
        if reply_text:
            await update.message.reply_text(reply_text, parse_mode="Markdown")

        # ثبت در لاگ
        write_to_log(get_username(update), update.message.text, reply_text, update.message.date)

        # ذخیره آیدی و نام کاربر در لیست اعضای گروه
        name = update.message.from_user.name
        user_id = update.message.from_user.id
        username = update.message.from_user.username if update.message.from_user.username else "Unknown"
        if not any(user['id'] == user_id for user in group_members):
            group_members.append({'id': user_id, 'username': username, 'name': name})


# تابع اصلی برای راه‌اندازی ربات
def main() -> None:
    application = Application.builder().token(TOKEN).build()

    # ثبت دستورات
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("set_title", set_title))  # اضافه کردن لقب
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, reply_to_message))  # پاسخ به پیام‌ها

    logging.info("Bot is starting...")
    application.run_polling()


if __name__ == "__main__":
    # بررسی ایجاد فایل لاگ
    if not os.path.exists(LOG_FILE):
        with open(LOG_FILE, "w", encoding="utf-8") as f:
            f.write("Log file created.\n")

    main()
