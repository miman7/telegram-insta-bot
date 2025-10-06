# bot.py
import os
import re
import tempfile
import shutil
import logging
import asyncio
from pathlib import Path
from yt_dlp import YoutubeDL
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
from dotenv import load_dotenv

# بارگذاری متغیرهای محیطی از .env (در لوکال مفید است)
load_dotenv()

# تنظیمات
TOKEN = os.getenv("TELEGRAM_TOKEN")
ALLOWED_USERNAMES = [u.strip() for u in os.getenv("ALLOWED_USERNAMES", "").split(",") if u.strip()]
ALLOWED_CHAT_IDS = [int(x) for x in os.getenv("ALLOWED_CHAT_IDS", "").split(",") if x.strip().isdigit()]
MAX_FILE_SIZE_MB = int(os.getenv("MAX_FILE_SIZE_MB", "50"))
MAX_FILE_SIZE = MAX_FILE_SIZE_MB * 1024 * 1024

# لاگ ساده
logging.basicConfig(
    format="%(asctime)s - %(levelname)s - %(message)s", level=logging.INFO
)
logger = logging.getLogger(__name__)

# regex برای پیدا کردن لینک اینستاگرام
INSTAGRAM_URL_RE = re.compile(
    r"(https?://(?:www\.)?instagram\.com/[^\s]+|https?://instagr\.am/[^\s]+)", re.IGNORECASE
)

def is_allowed_user(user):
    if not user:
        return False
    if user.username and user.username in ALLOWED_USERNAMES:
        return True
    if user.id and user.id in ALLOWED_CHAT_IDS:
        return True
    return False

def download_instagram_to_dir(url, tmpdir):
    """
    با yt-dlp دانلود می‌کند و فایل‌ها را در tmpdir می‌گذارد.
    خروجی: لیستی از مسیر فایل‌های دانلودشده (ترتیب الفبایی نام فایل).
    """
    ydl_opts = {
        'outtmpl': os.path.join(tmpdir, '%(id)s.%(ext)s'),
        'quiet': True,
        'no_warnings': True,
        'ignoreerrors': False,
        'format': 'best',
        'noplaylist': False,  # اجازه دانلود sidecar / multiple items
        'merge_output_format': 'mp4',
    }
    with YoutubeDL(ydl_opts) as ydl:
        info = ydl.extract_info(url, download=True)
    files = sorted([os.path.join(tmpdir, f) for f in os.listdir(tmpdir)])
    return files, info

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("سلام! لینک پست اینستاگرام را ارسال کنید. من فایل‌ها را برایتان می‌فرستم.")

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = ("راهنما:\n"
            "- فقط لینک اینستاگرام بفرستید.\n"
            "- فقط کاربران مجاز می‌توانند استفاده کنند.\n"
            "- در صورت خطا، پیغام خطا را همانجا می‌بینید.")
    await update.message.reply_text(text)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    chat_id = update.message.chat_id

    # بررسی مجاز بودن
    if not is_allowed_user(user):
        await update.message.reply_text("❌ شما مجاز به استفاده از این ربات نیستید.")
        return

    text = update.message.text or ""
    m = INSTAGRAM_URL_RE.search(text)
    if not m:
        await update.message.reply_text("لینک معتبر اینستاگرام ارسال نشده است. لطفاً لینک را ارسال کنید.")
        return

    url = m.group(1)
    await update.message.reply_text("🔗 لینک دریافت شد. شروع به دانلود می‌کنم...")

    # کار دانلود را در executor انجام می‌دهیم چون yt-dlp مسدودکننده است
    loop = asyncio.get_event_loop()
    tmpdir_obj = tempfile.TemporaryDirectory(prefix="insta_dl_")
    tmpdir = tmpdir_obj.name

    try:
        await update.message.reply_text("⏳ در حال دانلود از اینستاگرام...")
        files, info = await loop.run_in_executor(None, download_instagram_to_dir, url, tmpdir)

        if not files:
            await update.message.reply_text("⚠️ هیچ فایلی دانلود نشد. شاید پست خصوصی باشد یا خطایی رخ داده.")
            return

        await update.message.reply_text(f"✅ دانلود تمام شد. {len(files)} فایل آماده ارسال است.")

        # ارسال فایل‌ها یکی‌یکی و گزارش وضعیت
        for idx, filepath in enumerate(files, start=1):
            fname = os.path.basename(filepath)
            fsize = os.path.getsize(filepath)
            await update.message.reply_text(f"📤 ارسال فایل {idx}/{len(files)}: {fname} ({round(fsize/1024/1024,2)} MB)")

            if fsize > MAX_FILE_SIZE:
                await update.message.reply_text(
                    f"⚠️ فایل {fname} بزرگ‌تر از {MAX_FILE_SIZE_MB}MB است و قابل ارسال نیست."
                )
                continue

            ext = Path(fname).suffix.lower()
            try:
                with open(filepath, "rb") as f:
                    if ext in [".jpg", ".jpeg", ".png", ".webp", ".gif"]:
                        await context.bot.send_photo(chat_id=chat_id, photo=f)
                    elif ext in [".mp4", ".mkv", ".webm", ".mov"]:
                        await context.bot.send_video(chat_id=chat_id, video=f)
                    else:
                        await context.bot.send_document(chat_id=chat_id, document=f)
                await update.message.reply_text(f"✅ ارسال {fname} انجام شد.")
            except Exception as send_err:
                logger.exception("send error")
                await update.message.reply_text(f"❌ خطا هنگام ارسال {fname}: {send_err}")

    except Exception as e:
        logger.exception("download/send error")
        await update.message.reply_text(f"⚠️ خطا: {e}")
    finally:
        # پاک‌سازی
        try:
            tmpdir_obj.cleanup()
        except Exception:
            pass

def main():
    if not TOKEN:
        print("ERROR: TELEGRAM_TOKEN مشخص نشده. لطفاً متغیر محیطی را تنظیم کنید.")
        return

    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_message))

    # لاگ روی کنسول
    logger.info("Bot is starting. Run polling...")
    app.run_polling()

if __name__ == "__main__":
    main()
