import os
import re
import logging
import httpx
import asyncio
import glob
import tempfile
from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
import instaloader

# إعداد السجلات
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

TOKEN = os.getenv("BOT_TOKEN", "8974184806:AAHM1YBZKpSR4b3n7kI-xvEDpZ2Ga4L9m4g")

users_db = set()
L = instaloader.Instaloader(
    download_pictures=False,
    download_videos=True,
    download_video_thumbnails=False,
    save_metadata=False,
    compress_json=False
)

def save_user(user_id: int):
    users_db.add(user_id)

def clean_url(url: str) -> str:
    return url.split('?')[0].strip()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    save_user(update.effective_user.id)
    
    # إضافة لوحة مفاتيح تفاعلية لتسهيل معرفة عدد المشتركين
    keyboard = [["📊 عدد المشتركين"]]
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

    msg = (
        "أهلاً بك! 👋\n\n"
        "أنا بوت تحميل الفيديوهات بدون علامة مائية.\n"
        "يمكنك إرسال رابط فيديو من:\n"
        "• TikTok 🎵\n"
        "• Instagram 📸\n"
        "• Facebook 📘\n"
        "• Twitter / X 🐦\n\n"
        "أرسل الرابط وسأقوم بمعالجته فوراً."
    )
    await update.message.reply_text(msg, reply_markup=reply_markup)

async def users_count(update: Update, context: ContextTypes.DEFAULT_TYPE):
    count = len(users_db)
    await update.message.reply_text(f"📊 **إحصائيات البوت:**\nإجمالي عدد المستخدمين النشطين: {count}")

async def download_tiktok(url: str, client: httpx.AsyncClient):
    try:
        api_res = await client.post("https://www.tikwm.com/api/", data={'url': url, 'hd': 1})
        res_data = api_res.json()
        if res_data.get('code') == 0 and 'data' in res_data:
            video_url = res_data['data'].get('play') or res_data['data'].get('wmplay')
            if video_url and not video_url.startswith("http"):
                video_url = "https://www.tikwm.com" + video_url
            return video_url
    except Exception as e:
        logging.error(f"TikTok error: {e}")
    return None

async def download_twitter(url: str, client: httpx.AsyncClient):
    cleaned = clean_url(url)
    try:
        res = await client.get(f"https://twitsave.com/info?url={cleaned}")
        if res.status_code == 200:
            matches = re.findall(r'https://[^\s"<]+\.mp4[^\s"<]*', res.text)
            if matches:
                return matches[0]
            
            matches_dl = re.findall(r'href="(https://download\.twitsave\.com/[^"]+)"', res.text)
            if matches_dl:
                return matches_dl[0]
            
            matches_tw = re.findall(r'href="(https://video\.twimg\.com/[^"]+)"', res.text)
            if matches_tw:
                return matches_tw[0]
    except Exception as e:
        logging.error(f"Twitter error: {e}")
    return None

def fetch_insta_info(url: str, target_dir: str):
    try:
        shortcode_match = re.search(r'/(?:p|reel|reels)/([^/?#&]+)', url)
        if not shortcode_match:
            return None, None
        shortcode = shortcode_match.group(1)
        
        post = instaloader.Post.from_shortcode(L.context, shortcode)
        direct_url = post.video_url
        
        L.download_post(post, target=target_dir)
        
        mp4_files = glob.glob(os.path.join(target_dir, "*.mp4"))
        local_path = mp4_files[0] if mp4_files else None
        
        return direct_url, local_path
    except Exception as e:
        logging.error(f"Instaloader Fetch Error: {e}")
    return None, None

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    save_user(user_id)

    text = update.message.text.strip() if update.message.text else ""

    if text.startswith("/start"):
        await start(update, context)
        return
    elif text.startswith("/users") or text == "المستخدمين" or text == "📊 عدد المشتركين":
        await users_count(update, context)
        return

    platforms = ["tiktok.com", "instagram.com", "facebook.com", "fb.watch", "twitter.com", "x.com"]
    if not any(p in text for p in platforms):
        await update.message.reply_text("الرجاء إرسال رابط صحيح من (تيك توك، انستقرام، فيسبوك، أو تويتر).")
        return

    msg = await update.message.reply_text("جاري معالجة الفيديو...")

    try:
        cleaned_link = clean_url(text)

        # 1. TikTok
        if "tiktok.com" in text:
            headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
            async with httpx.AsyncClient(follow_redirects=True, headers=headers, timeout=35.0) as client:
                video_url = await download_tiktok(text, client)

            if video_url:
                await update.message.reply_video(video=video_url, caption="تم التحميل بنجاح! ✨")
                await msg.delete()
            else:
                await msg.edit_text("تعذر جلب الفيديو، تأكد من أن الرابط صحيح وأن الحساب عام.")

        # 2. Twitter / X
        elif "twitter.com" in text or "x.com" in text:
            headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
            async with httpx.AsyncClient(follow_redirects=True, headers=headers, timeout=35.0) as client:
                video_url = await download_twitter(text, client)

            if video_url:
                await update.message.reply_video(video=video_url, caption="تم التحميل بنجاح! ✨")
                await msg.delete()
            else:
                await msg.edit_text("تعذر جلب الفيديو من تويتر، تأكد أن التغريدة تحتوي على فيديو.")

        # 3. Instagram
        elif "instagram.com" in text:
            loop = asyncio.get_running_loop()
            
            download_dir = os.path.join(tempfile.gettempdir(), f"insta_bot_{update.message.message_id}")
            os.makedirs(download_dir, exist_ok=True)

            direct_url, video_path = await loop.run_in_executor(None, fetch_insta_info, cleaned_link, download_dir)

            sent_successfully = False

            if direct_url:
                try:
                    await update.message.reply_video(video=direct_url, caption="تم التحميل بنجاح! ✨")
                    sent_successfully = True
                except Exception as e:
                    logging.info(f"Direct URL send failed, falling back to local file: {e}")

            if not sent_successfully and video_path and os.path.exists(video_path):
                try:
                    with open(video_path, 'rb') as vf:
                        await update.message.reply_document(
                            document=vf, 
                            filename="instagram_video.mp4",
                            caption="تم التحميل بنجاح! ✨"
                        )
                    sent_successfully = True
                except Exception as upload_err:
                    logging.error(f"Telegram Upload Exception: {upload_err}")

            if sent_successfully:
                await msg.delete()
            else:
                await msg.edit_text("تعذر جلب الفيديو من إنستغرام، تأكد أن الحساب عام وأن الرابط صحيح.")

            try:
                for f in os.listdir(download_dir):
                    os.remove(os.path.join(download_dir, f))
                os.rmdir(download_dir)
            except Exception:
                pass

    except Exception as e:
        logging.error(f"Error handling request: {e}")
        await msg.edit_text("حدث خطأ أثناء معالجة الفيديو. حاول مجدداً لاحقاً.")

def main():
    if not TOKEN:
        raise ValueError("لم يتم العثور على توكن البوت!")

    app = ApplicationBuilder().token(TOKEN).read_timeout(300).write_timeout(300).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("users", users_count))
    app.add_handler(MessageHandler(filters.TEXT, handle_message))

    print("البوت يعمل الآن بنجاح...")
    app.run_polling(drop_pending_updates=True)

if __name__ == '__main__':
    main()
