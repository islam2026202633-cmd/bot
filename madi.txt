import asyncio, os, sqlite3, time, logging
from aiogram import Bot, Dispatcher, F, Router
from aiogram.filters import CommandStart, CommandObject, Command
from aiogram.types import (Message, CallbackQuery, LabeledPrice,
                           PreCheckoutQuery, InlineKeyboardButton)
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
import re

logging.basicConfig(level=logging.INFO)

# ===== الإعدادات (عبر متغيرات البيئة) =====
TOKEN ="9240806644:AAFEYXepErR6NNSIhZvrUgG4qTGGIB3GMnM"
ADMIN_ID =ADMIN_ID =ADMIN_ID = int(os.environ.get("ADMIN_ID", "0"))
CHANNEL = os.environ.get("CHANNEL", "8551242764")  # قناة الاشتراك الإ
DAILY, REF = 5, 10
PACKS = [5, 10, 50, 100, 200, 500, 1000, 2000]  # كل 5 نقاط = 2 نجوم

con = sqlite3.connect("bot.db", check_same_thread=False)
con.row_factory = sqlite3.Row
con.executescript("""
CREATE TABLE IF NOT EXISTS users(
  id INTEGER PRIMARY KEY, points INTEGER DEFAULT 0, ref_by INTEGER,
  ref_paid INTEGER DEFAULT 0, last_daily INTEGER DEFAULT 0, joined INTEGER);
CREATE TABLE IF NOT EXISTS ledger(
  id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER,
  delta INTEGER, reason TEXT, ts INTEGER);
CREATE TABLE IF NOT EXISTS payments(
  charge_id TEXT PRIMARY KEY, user_id INTEGER, points INTEGER, stars INTEGER, ts INTEGER);
CREATE TABLE IF NOT EXISTS stock(
  id INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT, tier TEXT, price INTEGER,
  payload TEXT, status TEXT DEFAULT 'available', buyer_id INTEGER, sold_ts INTEGER);
CREATE TABLE IF NOT EXISTS weekly_orders(
  id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, email TEXT,
  cost INTEGER, status TEXT DEFAULT 'pending', ts INTEGER);
""")
con.commit()
router = Router()


def add_points(uid, delta, reason):
    con.execute("UPDATE users SET points=points+? WHERE id=?", (delta, uid))
    con.execute("INSERT INTO ledger(user_id,delta,reason,ts) VALUES(?,?,?,?)",
                (uid, delta, reason, int(time.time())))
    con.commit()


def ensure_user(uid, ref):
    if ref and not con.execute("SELECT 1 FROM users WHERE id=?", (ref,)).fetchone():
        ref = None
    con.execute("INSERT OR IGNORE INTO users(id,ref_by,joined) VALUES(?,?,?)",
                (uid, ref, int(time.time())))
    con.commit()


async def is_member(bot, uid):
    try:
        m = await bot.get_chat_member(CHANNEL, uid)
        return m.status in ("member", "administrator", "creator")
    except Exception:
        return False


def main_kb():
    b = InlineKeyboardBuilder()
    for t, d in [("🏆 الجائزة الأسبوعية", "weekly"),
                 ("🎁 الجائزة اليومية", "daily"), ("🤝 الإحالة", "ref"),
                 ("💎 شراء حسابات", "accs"), ("💳 بطاقات Visa", "visa"),
                 ("⭐ شراء النقاط", "buy"), ("👤 حسابي", "me"), ("📜 السجل", "log")]:
        b.button(text=t, callback_data=d)
    b.adjust(2)
    return b.as_markup()


def sub_kb():
    b = InlineKeyboardBuilder()
    b.row(InlineKeyboardButton(text="📢 الاشتراك في القناة",
                               url=f"https://t.me/{CHANNEL.lstrip('@')}"))
    b.row(InlineKeyboardButton(text="✅ التحقق من الاشتراك", callback_data="check"))
    return b.as_markup()


async def after_verified(bot, uid):
    u = con.execute("SELECT ref_by FROM users WHERE id=?", (uid,)).fetchone()
    if u and u["ref_by"]:
        cur = con.execute("UPDATE users SET ref_paid=1 WHERE id=? AND ref_paid=0", (uid,))
        con.commit()
        if cur.rowcount:
            add_points(u["ref_by"], REF, f"referral:{uid}")
            try:
                await bot.send_message(u["ref_by"], f"🤝 إحالة ناجحة! +{REF} نقاط")
            except Exception:
                pass


async def gate(cq: CallbackQuery, bot: Bot):
    if await is_member(bot, cq.from_user.id):
        return True
    await cq.message.answer("📢 يجب الاشتراك في القناة أولاً:", reply_markup=sub_kb())
    await cq.answer()
    return False


@router.message(CommandStart())
async def start(m: Message, command: CommandObject, bot: Bot):
    uid, ref = m.from_user.id, None
    a = command.args or ""
    if a.startswith("ref_") and a[4:].isdigit() and int(a[4:]) != uid:
        ref = int(a[4:])
    ensure_user(uid, ref)
    if not await is_member(bot, uid):
        return await m.answer("📢 اشترك في القناة ثم اضغط تحقق:", reply_markup=sub_kb())
    await after_verified(bot, uid)
    await m.answer("💎 مرحباً بك في بوت نقاط التشغيل", reply_markup=main_kb())


@router.callback_query(F.data == "check")
async def check(cq: CallbackQuery, bot: Bot):
    if not await is_member(bot, cq.from_user.id):
        return await cq.answer("❌ لم تشترك بعد", show_alert=True)
    await after_verified(bot, cq.from_user.id)
    await cq.message.answer("✅ تم التحقق", reply_markup=main_kb())
    await cq.answer()


@router.callback_query(F.data == "daily")
async def daily(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    uid, now = cq.from_user.id, int(time.time())
    cur = con.execute("UPDATE users SET last_daily=? WHERE id=? AND ?-last_daily>=86400",
                      (now, uid, now))
    con.commit()
    if cur.rowcount:
        add_points(uid, DAILY, "daily")
        return await cq.answer(f"🎁 حصلت على {DAILY} نقاط!", show_alert=True)
    last = con.execute("SELECT last_daily FROM users WHERE id=?", (uid,)).fetchone()[0]
    left = 86400 - (now - last)
    await cq.answer(f"⏳ المتبقي: {left // 3600} ساعة و{left % 3600 // 60} دقيقة", show_alert=True)


@router.callback_query(F.data == "ref")
async def ref(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    me = await bot.get_me()
    n = con.execute("SELECT COUNT(*) FROM users WHERE ref_by=? AND ref_paid=1",
                    (cq.from_user.id,)).fetchone()[0]
    await cq.message.answer(f"🤝 رابط إحالتك (+{REF} نقاط لكل صديق):\n"
                            f"https://t.me/{me.username}?start=ref_{cq.from_user.id}\n"
                            f"الإحالات الناجحة: {n}")
    await cq.answer()


@router.callback_query(F.data == "me")
async def me(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    uid = cq.from_user.id
    p = con.execute("SELECT points FROM users WHERE id=?", (uid,)).fetchone()[0]
    w = con.execute("SELECT status FROM weekly_orders WHERE user_id=? ORDER BY id DESC LIMIT 1",
                    (uid,)).fetchone()
    wtxt = WSTAT.get(w["status"], "-") if w else "لا يوجد طلب"
    await cq.message.answer(f"👤 رصيدك: {p} نقطة 💎\n🏆 الجائزة الأسبوعية: {wtxt}")
    await cq.answer()


@router.callback_query(F.data == "log")
async def log(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    rows = con.execute("SELECT delta,reason,ts FROM ledger WHERE user_id=? ORDER BY id DESC LIMIT 10",
                       (cq.from_user.id,)).fetchall()
    txt = "\n".join(f"{'+' if r['delta'] > 0 else ''}{r['delta']} | {r['reason'].split(':')[0]} | "
                    f"{time.strftime('%Y-%m-%d', time.gmtime(r['ts']))}" for r in rows)
    await cq.message.answer("📜 آخر العمليات:\n" + (txt or "لا توجد عمليات"))
    await cq.answer()


@router.callback_query(F.data == "buy")
async def buy(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    b = InlineKeyboardBuilder()
    for n in PACKS:
        b.button(text=f"{n} نقطة = {n * 2 // 5} ⭐", callback_data=f"pay_{n}")
    b.adjust(2)
    await cq.message.answer("⭐ اختر الباقة:", reply_markup=b.as_markup())
    await cq.answer()


@router.callback_query(F.data.startswith("pay_"))
async def pay(cq: CallbackQuery, bot: Bot):
    n = int(cq.data[4:])
    if n not in PACKS:
        return await cq.answer()
    await bot.send_invoice(cq.from_user.id, title=f"{n} نقطة",
                           description=f"شراء {n} نقطة تشغيل", payload=f"pts:{n}",
                           currency="XTR", provider_token="",
                           prices=[LabeledPrice(label=f"{n} نقطة", amount=n * 2 // 5)])
    await cq.answer()


@router.pre_checkout_query()
async def pre(q: PreCheckoutQuery):
    ok = q.invoice_payload.startswith("pts:") and int(q.invoice_payload[4:]) in PACKS
    await q.answer(ok=ok, error_message=None if ok else "باقة غير صالحة")


@router.message(F.successful_payment)
async def paid(m: Message):
    sp = m.successful_payment
    n = int(sp.invoice_payload[4:])
    try:  # charge_id فريد: يمنع إضافة النقاط مرتين
        con.execute("INSERT INTO payments VALUES(?,?,?,?,?)",
                    (sp.telegram_payment_charge_id, m.from_user.id, n,
                     sp.total_amount, int(time.time())))
        con.commit()
    except sqlite3.IntegrityError:
        return
    add_points(m.from_user.id, n, "stars_purchase")
    await m.answer(f"✅ تمت إضافة {n} نقطة إلى رصيدك")


# ===== المخزون: حسابات + Visa (يبدأ فارغاً) =====
TIERS = {"bronze": ("🥉 البرونزي", 200), "silver": ("🥈 الفضي", 350),
         "gold": ("🥇 الذهبي", 500), "platinum": ("💠 البلاتيني", 1000),
         "diamond": ("💎 الماسي", 2000)}
VISA_PRICE = 80
TEST_CARD = ("🧪 بطاقة اختبار (للتجارب البرمجية فقط)\n"
             "الرقم: 4242 4242 4242 4242\nأي تاريخ مستقبلي وأي CVC\n"
             "⚠️ ليست بطاقة دفع حقيقية.")


def count(kind, tier):
    return con.execute("SELECT COUNT(*) FROM stock WHERE kind=? AND tier=? AND status='available'",
                       (kind, tier)).fetchone()[0]


def price_of(kind, tier):
    return TIERS[tier][1] if kind == "account" else VISA_PRICE


def buy_item(uid, kind, tier):
    """شراء ذري: خصم الرصيد + تعليم العنصر مباعاً في معاملة واحدة."""
    con.execute("BEGIN IMMEDIATE")
    try:
        it = con.execute("SELECT id,price,payload FROM stock WHERE kind=? AND tier=? "
                         "AND status='available' ORDER BY id LIMIT 1", (kind, tier)).fetchone()
        if not it:
            con.rollback()
            return "empty", None
        cur = con.execute("UPDATE users SET points=points-? WHERE id=? AND points>=?",
                          (it["price"], uid, it["price"]))
        if cur.rowcount == 0:
            con.rollback()
            return "funds", None
        now = int(time.time())
        con.execute("UPDATE stock SET status='sold',buyer_id=?,sold_ts=? WHERE id=?",
                    (uid, now, it["id"]))
        con.execute("INSERT INTO ledger(user_id,delta,reason,ts) VALUES(?,?,?,?)",
                    (uid, -it["price"], f"buy:{kind}:{it['id']}", now))
        con.commit()
        return "ok", it
    except Exception:
        con.rollback()
        raise


@router.callback_query(F.data == "accs")
async def accs(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    b = InlineKeyboardBuilder()
    for k, (name, price) in TIERS.items():
        b.button(text=f"{name} - {price} نقطة ({count('account', k)})",
                 callback_data=f"ask_account_{k}")
    b.adjust(1)
    await cq.message.answer("💎 باقات الحسابات:", reply_markup=b.as_markup())
    await cq.answer()


@router.callback_query(F.data == "visa")
async def visa(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    b = InlineKeyboardBuilder()
    b.button(text=f"💳 طلب بطاقة Visa - {VISA_PRICE} نقطة ({count('visa', 'real')})",
             callback_data="ask_visa_real")
    b.button(text="🧪 بطاقة اختبار", callback_data="testcard")
    b.adjust(1)
    await cq.message.answer("💳 بطاقات Visa:", reply_markup=b.as_markup())
    await cq.answer()


@router.callback_query(F.data == "testcard")
async def testcard(cq: CallbackQuery):
    await cq.message.answer(TEST_CARD)
    await cq.answer()


@router.callback_query(F.data.startswith("ask_"))
async def ask(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    _, kind, tier = cq.data.split("_", 2)
    if kind not in ("account", "visa") or (kind == "account" and tier not in TIERS):
        return await cq.answer()
    if count(kind, tier) == 0:
        msg = "لا توجد حسابات متاحة حاليًا" if kind == "account" else "لا توجد بطاقات متاحة حاليًا"
        return await cq.answer(msg, show_alert=True)
    b = InlineKeyboardBuilder()
    b.button(text=f"✅ تأكيد الشراء ({price_of(kind, tier)} نقطة)",
             callback_data=f"ok_{kind}_{tier}")
    await cq.message.answer("هل تريد إتمام الشراء؟", reply_markup=b.as_markup())
    await cq.answer()


@router.callback_query(F.data.startswith("ok_"))
async def confirm(cq: CallbackQuery, bot: Bot):
    if not await gate(cq, bot):
        return
    _, kind, tier = cq.data.split("_", 2)
    if kind not in ("account", "visa") or (kind == "account" and tier not in TIERS):
        return await cq.answer()
    status, it = buy_item(cq.from_user.id, kind, tier)
    if status == "empty":
        return await cq.answer("لا توجد عناصر متاحة حاليًا", show_alert=True)
    if status == "funds":
        return await cq.answer("❌ رصيدك غير كافٍ", show_alert=True)
    await cq.message.edit_reply_markup(reply_markup=None)
    await cq.message.answer(f"✅ تم الشراء (رقم #{it['id']}):\n\n{it['payload']}")
    await cq.answer()


@router.message(Command("addacc", "addvisa"), F.from_user.id == ADMIN_ID)
async def add_stock(m: Message, command: CommandObject):
    lines = (command.args or "").split("\n")
    if command.command == "addacc":
        kind, tier, items = "account", lines[0].strip().lower(), lines[1:]
        if tier not in TIERS:
            return await m.answer("الصيغة:\n/addacc gold\nحساب1\nحساب2 (سطر لكل حساب)\n"
                                  "الباقات: " + ", ".join(TIERS))
    else:
        kind, tier, items = "visa", "real", lines
    items = [i.strip() for i in items if i.strip()]
    if not items:
        return await m.answer("لم تُرسل أي عناصر.")
    price = price_of(kind, tier)
    con.executemany("INSERT INTO stock(kind,tier,price,payload) VALUES(?,?,?,?)",
                    [(kind, tier, price, i) for i in items])
    con.commit()
    await m.answer(f"✅ أُضيف {len(items)} عنصر.")


@router.message(Command("stock"), F.from_user.id == ADMIN_ID)
async def stock_cmd(m: Message):
    rows = con.execute("SELECT id,kind,tier,status FROM stock ORDER BY id DESC LIMIT 30").fetchall()
    txt = "\n".join(f"#{r['id']} {r['kind']}/{r['tier']} - {r['status']}" for r in rows)
    await m.answer(txt or "المخزون فارغ.")


@router.message(Command("disable"), F.from_user.id == ADMIN_ID)
async def disable(m: Message, command: CommandObject):
    if command.args and command.args.isdigit():
        con.execute("UPDATE stock SET status='disabled' WHERE id=? AND status='available'",
                    (int(command.args),))
        con.commit()
        return await m.answer("تم التعطيل ✅")
    await m.answer("الصيغة: /disable ID")


# ===== الجائزة الأسبوعية =====
WEEKLY_COST = 120
WSTAT = {"pending": "قيد المعالجة ⏳", "done": "مكتمل ✅", "rejected": "مرفوض ❌"}
EMAIL_RE = re.compile(r"^[\w.+-]+@[\w-]+\.[\w.-]+$")


class WeeklyForm(StatesGroup):
    email = State()


def has_pending(uid):
    return con.execute("SELECT 1 FROM weekly_orders WHERE user_id=? AND status='pending'",
                       (uid,)).fetchone() is not None


@router.callback_query(F.data == "weekly")
async def weekly(cq: CallbackQuery, bot: Bot, state: FSMContext):
    if not await gate(cq, bot):
        return
    uid = cq.from_user.id
    if has_pending(uid):
        return await cq.answer("لديك طلب قيد المعالجة بالفعل", show_alert=True)
    p = con.execute("SELECT points FROM users WHERE id=?", (uid,)).fetchone()[0]
    if p < WEEKLY_COST:
        return await cq.answer(f"❌ رصيدك {p} والمطلوب {WEEKLY_COST} نقطة", show_alert=True)
    await state.set_state(WeeklyForm.email)
    await cq.message.answer(
        f"🏆 تفعيل الجائزة الأسبوعية ({WEEKLY_COST} نقطة)\n\n"
        "أرسل بريدك الإلكتروني الآن.\n"
        "⚠️ لا ترسل كلمة المرور أو أي رمز تحقق أبداً.\n"
        "للإلغاء: /cancel")
    await cq.answer()


@router.message(Command("cancel"))
async def cancel(m: Message, state: FSMContext):
    await state.clear()
    await m.answer("تم الإلغاء.", reply_markup=main_kb())


@router.message(WeeklyForm.email, F.text, ~F.text.startswith("/"))
async def weekly_email(m: Message, state: FSMContext, bot: Bot):
    email = m.text.strip()
    if not EMAIL_RE.match(email):
        return await m.answer("❌ البريد غير صالح، أعد إرساله أو اكتب /cancel")
    uid, now, oid, res = m.from_user.id, int(time.time()), None, None
    con.execute("BEGIN IMMEDIATE")
    try:
        if has_pending(uid):
            res = "dup"
        elif con.execute("UPDATE users SET points=points-? WHERE id=? AND points>=?",
                         (WEEKLY_COST, uid, WEEKLY_COST)).rowcount == 0:
            res = "funds"
        else:
            oid = con.execute("INSERT INTO weekly_orders(user_id,email,cost,status,ts) "
                              "VALUES(?,?,?,?,?)", (uid, email, WEEKLY_COST, "pending", now)).lastrowid
            con.execute("INSERT INTO ledger(user_id,delta,reason,ts) VALUES(?,?,?,?)",
                        (uid, -WEEKLY_COST, f"weekly:{oid}", now))
            res = "ok"
        con.commit() if res == "ok" else con.rollback()
    except Exception:
        con.rollback()
        raise
    await state.clear()
    if res == "dup":
        return await m.answer("لديك طلب قيد المعالجة بالفعل.")
    if res == "funds":
        return await m.answer("❌ رصيدك غير كافٍ.")
    await m.answer(f"✅ تم إنشاء طلبك رقم #{oid} وحالته: قيد المعالجة ⏳", reply_markup=main_kb())
    if ADMIN_ID:
        b = InlineKeyboardBuilder()
        b.button(text="✅ قبول", callback_data=f"wk_ok_{oid}")
        b.button(text="❌ رفض", callback_data=f"wk_no_{oid}")
        await bot.send_message(ADMIN_ID, f"🏆 طلب جائزة أسبوعية #{oid}\n"
                               f"المستخدم: {uid}\nالبريد: {email}", reply_markup=b.as_markup())


@router.callback_query(F.data.startswith("wk_"), F.from_user.id == ADMIN_ID)
async def weekly_admin(cq: CallbackQuery, bot: Bot):
    _, act, oid = cq.data.split("_")
    oid = int(oid)
    o = con.execute("SELECT * FROM weekly_orders WHERE id=?", (oid,)).fetchone()
    if not o:
        return await cq.answer("الطلب غير موجود", show_alert=True)
    new = "done" if act == "ok" else "rejected"
    cur = con.execute("UPDATE weekly_orders SET status=? WHERE id=? AND status='pending'", (new, oid))
    con.commit()
    if not cur.rowcount:
        return await cq.answer("تمت معالجته مسبقاً", show_alert=True)
    if new == "rejected":
        add_points(o["user_id"], o["cost"], f"weekly_refund:{oid}")
    try:
        await bot.send_message(o["user_id"], f"🏆 طلبك #{oid}: {WSTAT[new]}" +
                               (f"\nأُعيدت لك {o['cost']} نقطة." if new == "rejected" else ""))
    except Exception:
        pass
    await cq.message.edit_text(f"{cq.message.text}\n\nالحالة: {WSTAT[new]}")
    await cq.answer()


# ===== إدارة بسيطة =====
@router.message(Command("stats"), F.from_user.id == ADMIN_ID)
async def stats(m: Message):
    u = con.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    s = con.execute("SELECT COALESCE(SUM(stars),0) FROM payments").fetchone()[0]
    await m.answer(f"👥 المستخدمون: {u}\n⭐ إجمالي النجوم: {s}")


@router.message(Command("give"), F.from_user.id == ADMIN_ID)
async def give(m: Message, command: CommandObject):
    try:
        uid, n = map(int, command.args.split())
        add_points(uid, n, "admin")
        await m.answer("تم ✅")
    except Exception:
        await m.answer("الصيغة: /give USER_ID POINTS")


async def main():
    bot = Bot(TOKEN)
    dp = Dispatcher()
    dp.include_router(router)
    await dp.start_polling(bot)
from aiogram.types import LabeledPrice, PreCheckoutQuery
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.context import FSMContext
import sqlite3

con = sqlite3.connect("shop.db")
con.execute("CREATE TABLE IF NOT EXISTS shop_products (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, price INTEGER, photo TEXT)")
con.commit()

class AddProduct(StatesGroup):
    name = State()
    price = State()
    photo = State()

@router.message(Command("admin"))
async def admin_panel(m: Message):
    if m.from_user.id != ADMIN_ID:
        return await m.answer("مش انت الادمن")
    kb = InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="➕ اضافة سلعة", callback_data="add_product")]])
    await m.answer("لوحة الادمن:", reply_markup=kb)

@router.callback_query(F.data == "add_product")
async def add_start(cq: CallbackQuery, state: FSMContext):
    await state.set_state(AddProduct.name)
    await cq.message.answer("ابعث اسم السلعة:")
    await cq.answer()

@router.message(AddProduct.name)
async def add_name(m: Message, state: FSMContext):
    await state.update_data(name=m.text)
    await state.set_state(AddProduct.price)
    await m.answer("ابعث السعر بالنجوم، مثلا 50:")

@router.message(AddProduct.price)
async def add_price(m: Message, state: FSMContext):
    try:
        price = int(m.text)
    except:
        return await m.answer("ابعث رقم برك")
    await state.update_data(price=price)
    await state.set_state(AddProduct.photo)
    await m.answer("ابعث صورة السلعة:")

@router.message(AddProduct.photo, F.photo)
async def add_photo(m: Message, state: FSMContext):
    data = await state.get_data()
    con.execute("INSERT INTO shop_products (name, price, photo) VALUES (?,?,?)", (data["name"], data["price"], m.photo[-1].file_id))
    con.commit()
    await state.clear()
    await m.answer(f"تم اضافة السلعة ✅ {data['name']} - {data['price']} ⭐")

@router.message(Command("shop"))
async def shop(m: Message):
    rows = con.execute("SELECT id, name, price, photo FROM shop_products").fetchall()
    if not rows:
        return await m.answer("المتجر فارغ")
    for pid, name, price, photo in rows:
        kb = InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text=f"شراء - {price} ⭐", callback_data=f"buy_{pid}")]])
        await m.answer_photo(photo, caption=f"{name}\nالسعر: {price} ⭐", reply_markup=kb)

@router.callback_query(F.data.startswith("buy_"))
async def buy_product(cq: CallbackQuery):
    pid = int(cq.data.split("_")[1])
    row = con.execute("SELECT name, price FROM shop_products WHERE id=?", (pid,)).fetchone()
    name, price = row
    await bot.send_invoice(chat_id=cq.from_user.id, title=name, description=f"شراء {name}", payload=f"shop_{pid}", provider_token="", currency="XTR", prices=[LabeledPrice(label=name, amount=int(price))])

@router.pre_checkout_query()
async def pre(q: PreCheckoutQuery):
    await q.answer(ok=True)

@router.message(F.successful_payment)
async def ok_pay(m: Message):
    await m.answer(f"تم الدفع ✅ {m.successful_payment.total_amount} ⭐")

if __name__ == "__main__":
    asyncio.run(main())
