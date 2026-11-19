import os
import sqlite3
import datetime as dt
from zoneinfo import ZoneInfo
from telegram import Update, InlineKeyboardButton as Btn, InlineKeyboardMarkup as Markup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

TOKEN = os.environ["BOT_TOKEN"]          # токен от @BotFather
TZ = ZoneInfo("Asia/Bishkek")
REMIND_AT = dt.time(13, 0, tzinfo=TZ)

db = sqlite3.connect("habits.db", check_same_thread=False)
db.execute("""CREATE TABLE IF NOT EXISTS habits(
    id INTEGER PRIMARY KEY, user_id INT, name TEXT, kind TEXT,
    start TEXT, last_done TEXT, streak INT DEFAULT 0)""")
db.commit()


def today():
    return dt.datetime.now(TZ).date()


async def start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Привет! Я помогаю с привычками.\n\n"
        "/add Читать 10 минут — новая привычка\n"
        "/quit Курить — бросить плохую привычку\n"
        "/list — мои привычки и отметки"
    )


def add_habit(kind):
    async def handler(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        name = " ".join(ctx.args).strip()
        if not name:
            await update.message.reply_text("Напиши название после команды.")
            return
        db.execute(
            "INSERT INTO habits(user_id, name, kind, start) VALUES(?,?,?,?)",
            (update.effective_user.id, name, kind, today().isoformat()),
        )
        db.commit()
        await update.message.reply_text(f"Добавлено: {name}. Смотри /list")
    return handler


async def list_habits(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    rows = db.execute(
        "SELECT id, name, kind, start, streak FROM habits WHERE user_id=?",
        (update.effective_user.id,),
    ).fetchall()
    if not rows:
        await update.message.reply_text("Пока пусто. Добавь через /add или /quit")
        return
    for hid, name, kind, start_, streak in rows:
        if kind == "build":
            text = f"✅ {name}\n🔥 Серия: {streak} дн."
            btn = Btn("Сделал сегодня", callback_data=f"done:{hid}")
        else:
            days = (today() - dt.date.fromisoformat(start_)).days
            text = f"🚫 {name}\nДней без этого: {days}"
            btn = Btn("Сорвался", callback_data=f"slip:{hid}")
        await update.message.reply_text(text, reply_markup=Markup([[btn]]))


async def on_button(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    action, hid = q.data.split(":")
    row = db.execute("SELECT last_done, streak FROM habits WHERE id=?", (hid,)).fetchone()
    if not row:
        await q.answer("Привычка не найдена")
        return
    last, streak = row
    t = today()
    if action == "done":
        if last == t.isoformat():
            await q.answer("Сегодня уже отмечено")
            return
        streak = streak + 1 if last == (t - dt.timedelta(days=1)).isoformat() else 1
        db.execute("UPDATE habits SET last_done=?, streak=? WHERE id=?", (t.isoformat(), streak, hid))
        await q.answer(f"Отлично! Серия: {streak} 🔥", show_alert=True)
    else:
        db.execute("UPDATE habits SET start=? WHERE id=?", (t.isoformat(), hid))
        await q.answer("Бывает. Счётчик сброшен, начинаем заново 💪", show_alert=True)
    db.commit()


async def remind(ctx: ContextTypes.DEFAULT_TYPE):
    users = db.execute("SELECT DISTINCT user_id FROM habits").fetchall()
    for (uid,) in users:
        try:
            await ctx.bot.send_message(uid, "Как дела с привычками? Отметь прогресс: /list")
        except Exception:
            pass


def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("add", add_habit("build")))
    app.add_handler(CommandHandler("quit", add_habit("quit")))
    app.add_handler(CommandHandler("list", list_habits))
    app.add_handler(CallbackQueryHandler(on_button))
    app.job_queue.run_daily(remind, time=REMIND_AT)
    app.run_polling()


if __name__ == "__main__":
    main()
