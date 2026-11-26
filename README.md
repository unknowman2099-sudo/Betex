import os
import re
import sqlite3
import time
import secrets
import urllib.parse
import urllib.request
import urllib.error
from datetime import date
from functools import wraps

from flask import Flask, request, redirect, session, render_template_string, flash, url_for
from werkzeug.security import generate_password_hash, check_password_hash
from markupsafe import escape

# ============================================================
# BETEX NEPAL - ORIGINAL SPORTS COMMUNITY PLATFORM
# Virtual points only. No real-money betting or gambling.
#
# Install:
#   pip install Flask
#
# Real SMS (Twilio Verify):
#   TWILIO_ACCOUNT_SID=...
#   TWILIO_AUTH_TOKEN=...
#   TWILIO_VERIFY_SERVICE_SID=...
#
# Required production settings:
#   BETEX_SECRET_KEY=<long-random-secret>
#   BETEX_ADMIN_KEY=<strong-admin-key>
#   BETEX_DB_PATH=/your/persistent/disk/betex.db  (recommended)
# ============================================================

app = Flask(__name__)
app.secret_key = os.environ.get("BETEX_SECRET_KEY", "dev-only-change-this-secret")
app.config.update(
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SAMESITE="Lax",
    SESSION_COOKIE_SECURE=os.environ.get("COOKIE_SECURE", "1") == "1",
    PERMANENT_SESSION_LIFETIME=60 * 60 * 24 * 7,
)

DB_PATH = os.environ.get("BETEX_DB_PATH", os.path.join("instance", "betex.db"))
ADMIN_KEY = os.environ.get("BETEX_ADMIN_KEY", "")
OTP_COOLDOWN_SECONDS = 60

CSS = """
<style>
:root{--bg:#071521;--panel:#12283a;--panel2:#0d2030;--text:#f4f8fb;--muted:#9eb2c2;--green:#00e6a0;--line:#294255}
body.light{--bg:#edf3f7;--panel:#fff;--panel2:#dce7ef;--text:#142333;--muted:#536779;--line:#c8d5df}
*{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--text);font-family:Arial,sans-serif;line-height:1.5}
header{background:var(--panel2);padding:16px 5%;display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap;border-bottom:1px solid var(--line)}
.logo{font-size:29px;font-weight:900;color:var(--green);text-decoration:none;letter-spacing:2px}
nav{background:var(--panel);padding:13px;display:flex;gap:15px;justify-content:center;flex-wrap:wrap;border-bottom:1px solid var(--line)}
nav a,a{color:var(--green);text-decoration:none}nav a{color:var(--text);font-size:14px}
.container{max-width:1050px;margin:22px auto;padding:0 14px;min-height:65vh}
.card{background:var(--panel);padding:19px;border:1px solid var(--line);border-radius:15px;margin:14px 0;box-shadow:0 8px 24px #00000012}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(210px,1fr));gap:12px}
h1,h2,h3{margin-top:4px}h2{color:var(--green)}input,textarea,select{width:100%;padding:13px;margin:7px 0 12px;border:1px solid var(--line);border-radius:9px;background:var(--bg);color:var(--text)}
textarea{min-height:90px}button,.btn{display:inline-block;background:var(--green);color:#06251c;border:0;padding:12px 16px;border-radius:9px;font-weight:bold;cursor:pointer}
button.full{width:100%}.muted{color:var(--muted)}.error{color:#ff7777}.success{color:var(--green)}
.pill{display:inline-block;padding:5px 10px;border-radius:20px;background:#28475a;color:#b8ffe8;font-size:12px}
.score{font-size:27px;font-weight:bold;color:var(--green)}footer{text-align:center;color:var(--muted);padding:25px}
table{width:100%;border-collapse:collapse;overflow-wrap:anywhere}td,th{padding:10px;border-bottom:1px solid var(--line);text-align:left}
small{color:var(--muted)}.notice{padding:12px;border-radius:9px;background:#263d4d;margin:10px 0}
@media(max-width:600px){header{justify-content:center;text-align:center}nav{gap:10px}nav a{font-size:13px}.card{padding:15px}}
</style>
"""

def db():
    folder = os.path.dirname(DB_PATH)
    if folder:
        os.makedirs(folder, exist_ok=True)
    con = sqlite3.connect(DB_PATH, timeout=20)
    con.row_factory = sqlite3.Row
    con.execute("PRAGMA foreign_keys=ON")
    con.execute("PRAGMA busy_timeout=20000")
    return con

def init_db():
    con = db()
    con.executescript("""
    CREATE TABLE IF NOT EXISTS users(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      phone TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      points INTEGER NOT NULL DEFAULT 1000,
      display_name TEXT DEFAULT '',
      favorite_team TEXT DEFAULT '',
      created_at TEXT DEFAULT CURRENT_TIMESTAMP,
      is_active INTEGER NOT NULL DEFAULT 1
    );
    CREATE TABLE IF NOT EXISTS news(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL, body TEXT NOT NULL,
      created_at TEXT DEFAULT CURRENT_TIMESTAMP
    );
    CREATE TABLE IF NOT EXISTS matches(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      sport TEXT NOT NULL, home_team TEXT NOT NULL, away_team TEXT NOT NULL,
      match_time TEXT NOT NULL, status TEXT DEFAULT 'Upcoming', score TEXT DEFAULT '-'
    );
    CREATE TABLE IF NOT EXISTS notifications(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL, message TEXT NOT NULL,
      created_at TEXT DEFAULT CURRENT_TIMESTAMP, seen INTEGER DEFAULT 0,
      FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
    );
    CREATE TABLE IF NOT EXISTS chat(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL, message TEXT NOT NULL,
      created_at TEXT DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
    );
    CREATE TABLE IF NOT EXISTS checkins(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL, checkin_date TEXT NOT NULL,
      points INTEGER NOT NULL DEFAULT 25,
      UNIQUE(user_id,checkin_date),
      FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
    );
    CREATE TABLE IF NOT EXISTS quizzes(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      question TEXT NOT NULL, option_a TEXT NOT NULL, option_b TEXT NOT NULL,
      option_c TEXT NOT NULL, option_d TEXT NOT NULL,
      answer TEXT NOT NULL, active INTEGER NOT NULL DEFAULT 1
    );
    CREATE TABLE IF NOT EXISTS quiz_answers(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL, quiz_id INTEGER NOT NULL,
      answer TEXT NOT NULL, correct INTEGER NOT NULL,
      created_at TEXT DEFAULT CURRENT_TIMESTAMP,
      UNIQUE(user_id,quiz_id),
      FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE,
      FOREIGN KEY(quiz_id) REFERENCES quizzes(id) ON DELETE CASCADE
    );
    CREATE TABLE IF NOT EXISTS otp_limits(
      phone TEXT PRIMARY KEY, last_sent REAL NOT NULL, attempts INTEGER NOT NULL DEFAULT 0
    );
    """)
    if con.execute("SELECT COUNT(*) FROM matches").fetchone()[0] == 0:
        con.executemany("""INSERT INTO matches(sport,home_team,away_team,match_time,status,score)
        VALUES(?,?,?,?,?,?)""", [
            ("Football","Nepal Demo XI","Mountain United","Sample fixture","DEMO","1 - 0"),
            ("Cricket","Nepal Demo XI","Valley Stars","Sample fixture","UPCOMING","-"),
            ("Basketball","Kathmandu Hoops","Pokhara Stars","Sample fixture","UPCOMING","-")
        ])
    if con.execute("SELECT COUNT(*) FROM news").fetchone()[0] == 0:
        con.execute("INSERT INTO news(title,body) VALUES(?,?)",
                    ("Welcome to BETEX Nepal","Follow sports updates, fixtures and community news."))
    if con.execute("SELECT COUNT(*) FROM quizzes").fetchone()[0] == 0:
        con.execute("""INSERT INTO quizzes(question,option_a,option_b,option_c,option_d,answer)
        VALUES(?,?,?,?,?,?)""", ("How many players are on a football team on the field?",
        "9","10","11","12","C"))
    con.commit()
    con.close()

def sms_configured():
    return all(os.environ.get(k) for k in (
        "TWILIO_ACCOUNT_SID", "TWILIO_AUTH_TOKEN", "TWILIO_VERIFY_SERVICE_SID"))

def phone_e164(raw):
    digits = re.sub(r"\D", "", raw or "")
    if digits.startswith("977") and len(digits) == 13:
        return "+" + digits
    if len(digits) == 10 and digits[0] in "97":
        return "+977" + digits
    return None

def twilio_request(path, fields):
    sid = os.environ["TWILIO_ACCOUNT_SID"]
    token = os.environ["TWILIO_AUTH_TOKEN"]
    url = "https://verify.twilio.com/v2/Services/" + os.environ["TWILIO_VERIFY_SERVICE_SID"] + path
    data = urllib.parse.urlencode(fields).encode()
    req = urllib.request.Request(url, data=data, method="POST")
    import base64
    auth = base64.b64encode((sid + ":" + token).encode()).decode()
    req.add_header("Authorization", "Basic " + auth)
    req.add_header("Content-Type", "application/x-www-form-urlencoded")
    try:
        with urllib.request.urlopen(req, timeout=15) as response:
            import json
            return json.loads(response.read().decode())
    except urllib.error.HTTPError as exc:
        detail = exc.read().decode(errors="ignore")[:500]
        app.logger.warning("SMS provider error: %s", detail)
        raise RuntimeError("SMS provider rejected the request. Check your Twilio Verify setup and phone permissions.")
    except Exception:
        app.logger.exception("SMS provider request failed")
        raise RuntimeError("Could not contact the SMS provider. Please try again later.")

def send_otp(phone):
    if not sms_configured():
        raise RuntimeError("Real SMS is not configured. Add the three Twilio environment variables first.")
    con = db()
    row = con.execute("SELECT last_sent FROM otp_limits WHERE phone=?", (phone,)).fetchone()
    if row and time.time() - row["last_sent"] < OTP_COOLDOWN_SECONDS:
        con.close()
        raise RuntimeError("Please wait 60 seconds before requesting another code.")
    con.execute("""INSERT INTO otp_limits(phone,last_sent,attempts) VALUES(?,?,0)
      ON CONFLICT(phone) DO UPDATE SET last_sent=excluded.last_sent,attempts=0""",
      (phone,time.time()))
    con.commit()
    con.close()
    twilio_request("/Verifications", {"To": phone, "Channel": "sms"})

def check_otp(phone, code):
    if not sms_configured():
        raise RuntimeError("Real SMS is not configured.")
    con = db()
    row = con.execute("SELECT attempts FROM otp_limits WHERE phone=?", (phone,)).fetchone()
    attempts = row["attempts"] if row else 0
    if attempts >= 5:
        con.close()
        raise RuntimeError("Too many attempts. Request a new code later.")
    con.execute("UPDATE otp_limits SET attempts=attempts+1 WHERE phone=?", (phone,))
    con.commit()
    con.close()
    result = twilio_request("/VerificationCheck", {"To": phone, "Code": code})
    return result.get("status") == "approved"

def page(content, title="BETEX Nepal"):
    theme = "light" if session.get("light") else ""
    return render_template_string("""
    <!doctype html><html><head><title>{{title}}</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="theme-color" content="#071521">
    """ + CSS + """</head><body class="{{theme}}">
    <header><a class="logo" href="/">BETEX</a><span>NEPAL · SPORTS COMMUNITY</span></header>
    <nav>
      <a href="/">Home</a><a href="/scores">Scores</a><a href="/schedule">Fixtures</a>
      <a href="/news">News</a><a href="/chat">Community</a><a href="/leaderboard">Leaderboard</a>
      <a href="/dashboard">Dashboard</a><a href="/theme">Theme</a>
    </nav>
    <main class="container">{{content|safe}}</main>
    <footer>BETEX Nepal © 2026 · Sports community · Virtual points only</footer>
    </body></html>""", content=content, title=title, theme=theme)

def current_user():
    uid = session.get("user_id")
    if not uid:
        return None
    con = db()
    user = con.execute("SELECT * FROM users WHERE id=? AND is_active=1", (uid,)).fetchone()
    con.close()
    return user

def login_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        if not current_user():
            return redirect(url_for("login"))
        return fn(*args, **kwargs)
    return wrapper

def admin_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        if not session.get("is_admin"):
            return redirect(url_for("admin"))
        return fn(*args, **kwargs)
    return wrapper

def notify_all(message):
    con = db()
    con.executemany("INSERT INTO notifications(user_id,message) VALUES(?,?)",
        [(r["id"], message) for r in con.execute("SELECT id FROM users WHERE is_active=1").fetchall()])
    con.commit()
    con.close()

@app.after_request
def security_headers(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    return response

@app.route("/")
def home():
    user = current_user()
    greeting = ("Welcome, " + escape(user["display_name"] or ("User " + user["phone"][-4:]))
                if user else "Welcome to BETEX Nepal")
    content = f"""
    <div class="card"><span class="pill">NEPAL SPORTS COMMUNITY</span>
      <h1>{greeting}</h1><h2>Follow the games. Join the community.</h2>
      <p class="muted">Sports updates, sample scores, fixtures, quizzes and community.</p>
      <a class="btn" href="/scores">Explore scores</a> <a class="btn" href="/register">Create account</a>
    </div>
    <div class="grid">
      <div class="card"><h3>⚽ Football</h3><p class="muted">Fixtures and score updates</p></div>
      <div class="card"><h3>🏏 Cricket</h3><p class="muted">Fixtures and score updates</p></div>
      <div class="card"><h3>🏀 Basketball</h3><p class="muted">Sports community</p></div>
      <div class="card"><h3>🏆 Quiz competitions</h3><p class="muted">Earn virtual points</p><a href="/quiz">Play quiz</a></div>
    </div>
    <div class="card"><h2>BETEX features</h2>
      <p>Phone OTP · Profiles · News · Fixtures · Leaderboard · Favourites · Notifications · Daily check-in · Quizzes · Admin · Themes · Community chat</p>
      <p class="muted">All points are virtual and have no cash value. No real-money betting or gambling.</p>
    </div>"""
    return page(content)

@app.route("/theme")
def theme():
    session["light"] = not session.get("light", False)
    return redirect(request.referrer or "/")

# Registration: real Twilio Verify SMS
@app.route("/register", methods=["GET","POST"])
def register():
    error = ""
    if request.method == "POST":
        phone = phone_e164(request.form.get("phone"))
        if not phone:
            error = "Enter a valid Nepal mobile number, e.g. 98XXXXXXXX."
        else:
            con = db()
            exists = con.execute("SELECT id FROM users WHERE phone=?", (phone,)).fetchone()
            con.close()
            if exists:
                error = "This phone number is already registered. Please log in."
            else:
                try:
                    send_otp(phone)
                    session.clear()
                    session["pending_phone"] = phone
                    session["otp_purpose"] = "register"
                    return redirect("/verify")
                except RuntimeError as exc:
                    error = str(exc)
    return page(f"""<div class="card"><h2>Create BETEX account</h2>
      <p>Enter your Nepal mobile number. We'll send a real SMS verification code.</p>
      <p class="error">{escape(error)}</p>
      <form method="post"><label>Phone number</label>
      <input name="phone" inputmode="tel" maxlength="16" placeholder="98XXXXXXXX" required>
      <button class="full">Send SMS code</button></form>
      <p class="muted">SMS requires configured Twilio Verify credentials.</p>
      <a href="/login">Already registered? Login</a></div>""")

@app.route("/verify", methods=["GET","POST"])
def verify():
    phone = session.get("pending_phone")
    if not phone:
        return redirect("/register")
    error = ""
    if request.method == "POST":
        code = request.form.get("otp","").strip()
        if not re.fullmatch(r"\d{4,10}", code):
            error = "Enter the verification code from SMS."
        else:
            try:
                if check_otp(phone, code):
                    session["phone_verified"] = True
                    if session.get("otp_purpose") == "login":
                        con = db()
                        row = con.execute("SELECT id FROM users WHERE phone=?", (phone,)).fetchone()
                        con.close()
                        if row:
                            session.clear()
                            session["user_id"] = row["id"]
                            session.permanent = True
                            return redirect("/dashboard")
                        error = "Account not found. Please register."
                    else:
                        return redirect("/create-password")
                else:
                    error = "Incorrect or expired code. Try again."
            except RuntimeError as exc:
                error = str(exc)
    return page(f"""<div class="card"><h2>Verify your phone</h2>
      <p>We sent an SMS verification code to <b>{escape(phone)}</b>.</p>
      <p class="error">{escape(error)}</p>
      <form method="post"><input name="otp" inputmode="numeric" maxlength="10" placeholder="SMS verification code" required>
      <button class="full">Verify code</button></form>
      <form method="post" action="/resend-otp"><button class="full">Resend code</button></form>
    </div>""")

@app.route("/resend-otp", methods=["POST"])
def resend_otp():
    phone = session.get("pending_phone")
    if not phone:
        return redirect("/register")
    try:
        send_otp(phone)
        flash("A new SMS code was requested.")
    except RuntimeError as exc:
        flash(str(exc))
    return redirect("/verify")

@app.route("/create-password", methods=["GET","POST"])
def create_password():
    phone = session.get("pending_phone")
    if not phone or not session.get("phone_verified") or session.get("otp_purpose") != "register":
        return redirect("/register")
    error = ""
    if request.method == "POST":
        pw = request.form.get("password","")
        confirm = request.form.get("confirm","")
        if len(pw) < 10:
            error = "Use at least 10 characters for your password."
        elif pw != confirm:
            error = "Passwords do not match."
        else:
            con = db()
            try:
                cur = con.execute("INSERT INTO users(phone,password) VALUES(?,?)",
                                  (phone, generate_password_hash(pw)))
                con.commit()
                uid = cur.lastrowid
                con.close()
                session.clear()
                session["user_id"] = uid
                session.permanent = True
                return redirect("/dashboard")
            except sqlite3.IntegrityError:
                con.close()
                error = "That phone number already has an account. Please log in."
    return page(f"""<div class="card"><h2>Create password</h2><p class="error">{escape(error)}</p>
      <form method="post"><input type="password" name="password" minlength="10" placeholder="Password (10+ characters)" required>
      <input type="password" name="confirm" placeholder="Confirm password" required>
      <button class="full">Create account</button></form></div>""")

@app.route("/login", methods=["GET","POST"])
def login():
    error = ""
    if request.method == "POST":
        phone = phone_e164(request.form.get("phone"))
        pw = request.form.get("password","")
        if not phone:
            error = "Enter a valid Nepal mobile number."
        else:
            con = db()
            user = con.execute("SELECT id,password FROM users WHERE phone=? AND is_active=1", (phone,)).fetchone()
            con.close()
            if user and check_password_hash(user["password"], pw):
                try:
                    send_otp(phone)
                    session.clear()
                    session["pending_phone"] = phone
                    session["otp_purpose"] = "login"
                    session["phone_verified"] = False
                    return redirect("/verify")
                except RuntimeError as exc:
                    error = str(exc)
            else:
                error = "Incorrect phone number or password."
    return page(f"""<div class="card"><h2>Login to BETEX</h2>
      <p>Login requires your password and a real SMS verification code.</p>
      <p class="error">{escape(error)}</p>
      <form method="post"><input name="phone" inputmode="tel" maxlength="16" placeholder="98XXXXXXXX" required>
      <input name="password" type="password" placeholder="Password" required>
      <button class="full">Continue with SMS verification</button></form>
      <a href="/register">Create account</a></div>""")

@app.route("/dashboard")
@login_required
def dashboard():
    user = current_user()
    con = db()
    notes = con.execute("SELECT * FROM notifications WHERE user_id=? ORDER BY id DESC LIMIT 8",
                        (user["id"],)).fetchall()
    checkin = con.execute("SELECT id FROM checkins WHERE user_id=? AND checkin_date=?",
                          (user["id"], date.today().isoformat())).fetchone()
    con.close()
    note_html = "".join(f"<li>{escape(n['message'])} <small>· {escape(n['created_at'])}</small></li>" for n in notes) or "<li>No notifications yet.</li>"
    return page(f"""<div class="card"><h2>Your dashboard</h2>
      <p>Phone: {escape(user['phone'])}</p><p>Virtual points</p><div class="score">{user['points']}</div>
      <p class="muted">Demo points have no cash value.</p>
      <a class="btn" href="/profile">Edit profile</a> <a class="btn" href="/checkin">Daily check-in</a>
      <a class="btn" href="/logout">Logout</a></div>
      <div class="card"><h2>Notifications</h2><ul>{note_html}</ul></div>
      <div class="grid"><div class="card"><h3>Daily mission</h3><p>Check in today to earn virtual points.</p>
      <a href="/checkin">Open mission</a></div><div class="card"><h3>Quiz challenge</h3><p>Answer sports questions.</p><a href="/quiz">Play quiz</a></div></div>""")

@app.route("/profile", methods=["GET","POST"])
@login_required
def profile():
    user = current_user()
    if request.method == "POST":
        name = request.form.get("name","").strip()[:40]
        team = request.form.get("team","").strip()[:60]
        con = db()
        con.execute("UPDATE users SET display_name=?,favorite_team=? WHERE id=?",
                    (name,team,user["id"]))
        con.commit()
        con.close()
        return redirect("/profile")
    return page(f"""<div class="card"><h2>Your profile</h2>
      <form method="post"><label>Display name</label>
      <input name="name" maxlength="40" value="{escape(user['display_name'])}" placeholder="Your name">
      <label>Favourite team</label><input name="team" maxlength="60" value="{escape(user['favorite_team'])}" placeholder="Favourite team">
      <button class="full">Save profile</button></form></div>""")

@app.route("/checkin")
@login_required
def checkin():
    user = current_user()
    con = db()
    try:
        con.execute("INSERT INTO checkins(user_id,checkin_date,points) VALUES(?,?,25)",
                    (user["id"], date.today().isoformat()))
        con.execute("UPDATE users SET points=points+25 WHERE id=?", (user["id"],))
        con.execute("INSERT INTO notifications(user_id,message) VALUES(?,?)",
                    (user["id"], "Daily check-in complete: +25 virtual points."))
        con.commit()
        message = "Check-in successful! You earned 25 virtual points."
    except sqlite3.IntegrityError:
        message = "You've already checked in today. Come back tomorrow."
    con.close()
    return page(f"""<div class="card"><h2>Daily check-in</h2><p>{escape(message)}</p>
      <p class="score">+25 points</p><a class="btn" href="/dashboard">Back to dashboard</a></div>""")

@app.route("/scores")
def scores():
    con = db()
    rows = con.execute("SELECT * FROM matches ORDER BY id DESC").fetchall()
    con.close()
    items = ""
    for m in rows:
        items += f"""<div class="card"><span class="pill">{escape(m['sport'])} · {escape(m['status'])}</span>
        <h3>{escape(m['home_team'])} vs {escape(m['away_team'])}</h3>
        <div class="score">{escape(m['score'])}</div><p class="muted">{escape(m['match_time'])}</p></div>"""
    return page("<h2>Scoreboard</h2><p class='muted'>Sample/demo scores only; not live verified data.</p>"+(items or "<p>No matches yet.</p>"))

@app.route("/schedule")
def schedule():
    con = db()
    rows = con.execute("SELECT * FROM matches ORDER BY id DESC").fetchall()
    con.close()
    items = "".join(f"<tr><td>{escape(m['sport'])}</td><td>{escape(m['home_team'])} vs {escape(m['away_team'])}</td><td>{escape(m['match_time'])}</td><td>{escape(m['status'])}</td></tr>" for m in rows)
    return page("<div class='card'><h2>Match schedule</h2><table><tr><th>Sport</th><th>Match</th><th>Time</th><th>Status</th></tr>"+items+"</table></div>")

@app.route("/favorites")
@login_required
def favorites():
    user = current_user()
    return page(f"""<div class="card"><h2>Favourite team</h2>
      <p>{escape(user['favorite_team'] or 'No favourite team selected yet.')}</p>
      <a class="btn" href="/profile">Choose or update team</a></div>""")

@app.route("/news")
def news():
    con = db()
    rows = con.execute("SELECT * FROM news ORDER BY id DESC").fetchall()
    con.close()
    items = "".join(f"<div class='card'><h3>{escape(n['title'])}</h3><p>{escape(n['body'])}</p><small>{escape(n['created_at'])}</small></div>" for n in rows)
    return page("<h2>Sports news</h2>"+(items or "<p>No news yet.</p>"))

@app.route("/chat", methods=["GET","POST"])
@login_required
def chat():
    user = current_user()
    if request.method == "POST":
        msg = request.form.get("message","").strip()[:500]
        if msg:
            con = db()
            con.execute("INSERT INTO chat(user_id,message) VALUES(?,?)", (user["id"],msg))
            con.commit()
            con.close()
        return redirect("/chat")
    con = db()
    rows = con.execute("""SELECT chat.message,chat.created_at,users.display_name,users.phone
        FROM chat JOIN users ON users.id=chat.user_id ORDER BY chat.id DESC LIMIT 50""").fetchall()
    con.close()
    items = ""
    for r in reversed(rows):
        who = r["display_name"] or ("User " + r["phone"][-4:])
        items += f"<div class='card'><b>{escape(who)}</b><p>{escape(r['message'])}</p><small>{escape(r['created_at'])}</small></div>"
    return page("""<div class="card"><h2>Community chat</h2>
      <form method="post"><textarea name="message" maxlength="500" placeholder="Share a sports-related message..." required></textarea>
      <button class="full">Send message</button></form><p class="muted">Be respectful. Do not share personal information.</p></div>""" + items)

@app.route("/leaderboard")
def leaderboard():
    con = db()
    rows = con.execute("SELECT display_name,phone,points FROM users WHERE is_active=1 ORDER BY points DESC LIMIT 20").fetchall()
    con.close()
    items = "".join(f"<tr><td>{i}</td><td>{escape(u['display_name'] or ('User '+u['phone'][-4:]))}</td><td>{u['points']}</td></tr>" for i,u in enumerate(rows,1))
    return page("<div class='card'><h2>Virtual points leaderboard</h2><table><tr><th>Rank</th><th>User</th><th>Points</th></tr>"+items+"</table><p class='muted'>Points have no cash value.</p></div>")

@app.route("/notifications")
@login_required
def notifications():
    user = current_user()
    con = db()
    rows = con.execute("SELECT * FROM notifications WHERE user_id=? ORDER BY id DESC", (user["id"],)).fetchall()
    con.execute("UPDATE notifications SET seen=1 WHERE user_id=?", (user["id"],))
    con.commit()
    con.close()
    items = "".join(f"<div class='card'>{escape(n['message'])}<p class='muted'>{escape(n['created_at'])}</p></div>" for n in rows)
    return page("<h2>Your notifications</h2>"+(items or "<p>No notifications.</p>"))

@app.route("/quiz", methods=["GET","POST"])
@login_required
def quiz():
    user = current_user()
    con = db()
    if request.method == "POST":
        qid = request.form.get("quiz_id","")
        answer = request.form.get("answer","").upper()
        q = con.execute("SELECT * FROM quizzes WHERE id=? AND active=1", (qid,)).fetchone()
        if not q:
            con.close()
            return page("<div class='card'><h2>Quiz unavailable</h2></div>")
        correct = int(answer == q["answer"])
        try:
            con.execute("INSERT INTO quiz_answers(user_id,quiz_id,answer,correct) VALUES(?,?,?,?)",
                        (user["id"],q["id"],answer,correct))
            if correct:
                con.execute("UPDATE users SET points=points+10 WHERE id=?", (user["id"],))
                con.execute("INSERT INTO notifications(user_id,message) VALUES(?,?)",
                            (user["id"], "Correct quiz answer: +10 virtual points."))
            con.commit()
            result = "Correct! You earned 10 virtual points." if correct else "Not quite. Better luck next time."
        except sqlite3.IntegrityError:
            result = "You've already answered this quiz."
        con.close()
        return page(f"<div class='card'><h2>Quiz result</h2><p>{escape(result)}</p><a class='btn' href='/quiz'>Back to quiz</a></div>")
    q = con.execute("SELECT * FROM quizzes WHERE active=1 ORDER BY id DESC LIMIT 1").fetchone()
    answered = con.execute("SELECT 1 FROM quiz_answers WHERE user_id=? AND quiz_id=?",
                           (user["id"], q["id"])).fetchone() if q else None
    con.close()
    if not q:
        return page("<div class='card'><h2>No active quiz</h2><p>Check back soon.</p></div>")
    if answered:
        return page("<div class='card'><h2>Quiz completed</h2><p>You have already answered today's available quiz.</p></div>")
    options = "".join(f"<label><input style='width:auto' type='radio' name='answer' value='{letter}' required> {letter}. {escape(q[col])}</label><br>" for letter,col in [("A","option_a"),("B","option_b"),("C","option_c"),("D","option_d")])
    return page(f"""<div class="card"><h2>Sports quiz</h2><p>{escape(q['question'])}</p>
      <form method="post"><input type="hidden" name="quiz_id" value="{q['id']}">{options}
      <button class="full">Submit answer</button></form><p class="muted">Correct answers earn virtual points only.</p></div>""")

# Admin routes
@app.route("/admin", methods=["GET","POST"])
def admin():
    if not session.get("is_admin"):
        error = ""
        if request.method == "POST":
            if ADMIN_KEY and secrets.compare_digest(request.form.get("key",""), ADMIN_KEY):
                session.clear()
                session["is_admin"] = True
                return redirect("/admin")
            error = "Incorrect admin key. Set BETEX_ADMIN_KEY in hosting environment."
        return page(f"""<div class="card"><h2>Admin login</h2><p class="error">{escape(error)}</p>
          <form method="post"><input type="password" name="key" placeholder="Admin key" required>
          <button class="full">Admin login</button></form>
          <p class="muted">Configure BETEX_ADMIN_KEY before deployment.</p></div>""")
    con = db()
    users = con.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    matches = con.execute("SELECT COUNT(*) FROM matches").fetchone()[0]
    quiz_count = con.execute("SELECT COUNT(*) FROM quizzes").fetchone()[0]
    con.close()
    return page(f"""<div class="card"><h2>BETEX Admin</h2>
      <div class="grid"><div><h3>Users</h3><div class="score">{users}</div></div>
      <div><h3>Fixtures</h3><div class="score">{matches}</div></div>
      <div><h3>Quizzes</h3><div class="score">{quiz_count}</div></div></div>
      <h3>Publish sports news</h3><form method="post" action="/admin/news">
      <input name="title" maxlength="120" placeholder="News title" required>
      <textarea name="body" maxlength="2000" placeholder="News details" required></textarea><button class="full">Publish news</button></form>
      <h3>Add demo fixture</h3><form method="post" action="/admin/match">
      <input name="sport" placeholder="Sport" required><input name="home" placeholder="Home team" required>
      <input name="away" placeholder="Away team" required><input name="time" placeholder="Date/time label" required>
      <input name="status" placeholder="UPCOMING / DEMO" value="UPCOMING"><input name="score" placeholder="Score" value="-">
      <button class="full">Add fixture</button></form>
      <h3>Add quiz</h3><form method="post" action="/admin/quiz">
      <input name="question" placeholder="Question" required>
      <input name="a" placeholder="Option A" required><input name="b" placeholder="Option B" required>
      <input name="c" placeholder="Option C" required><input name="d" placeholder="Option D" required>
      <select name="answer"><option value="A">Correct: A</option><option value="B">Correct: B</option><option value="C">Correct: C</option><option value="D">Correct: D</option></select>
      <button class="full">Publish quiz</button></form>
      <a class="btn" href="/admin/logout">Admin logout</a></div>""")

@app.route("/admin/news", methods=["POST"])
@admin_required
def admin_news():
    title = request.form.get("title","").strip()[:120]
    body = request.form.get("body","").strip()[:2000]
    if title and body:
        con = db()
        con.execute("INSERT INTO news(title,body) VALUES(?,?)", (title,body))
        con.commit()
        con.close()
        notify_all("New sports news: " + title)
    return redirect("/admin")

@app.route("/admin/match", methods=["POST"])
@admin_required
def admin_match():
    values = [request.form.get(k,"").strip() for k in ("sport","home","away","time","status","score")]
    if all(values[:4]):
        con = db()
        con.execute("""INSERT INTO matches(sport,home_team,away_team,match_time,status,score)
          VALUES(?,?,?,?,?,?)""", values)
        con.commit()
        con.close()
        notify_all("A new fixture was added: " + values[1] + " vs " + values[2])
    return redirect("/admin")

@app.route("/admin/quiz", methods=["POST"])
@admin_required
def admin_quiz():
    values = [request.form.get(k,"").strip() for k in ("question","a","b","c","d")]
    answer = request.form.get("answer","A").upper()
    if all(values) and answer in ("A","B","C","D"):
        con = db()
        con.execute("""INSERT INTO quizzes(question,option_a,option_b,option_c,option_d,answer)
          VALUES(?,?,?,?,?,?)""", (*values,answer))
        con.commit()
        con.close()
        notify_all("A new sports quiz is available.")
    return redirect("/admin")

@app.route("/admin/logout")
def admin_logout():
    session.pop("is_admin",None)
    return redirect("/")

@app.route("/logout")
def logout():
    session.clear()
    return redirect("/")

@app.errorhandler(404)
def not_found(_):
    return page("<div class='card'><h2>Page not found</h2><a href='/'>Return home</a></div>"), 404

# Initialize database for Gunicorn/Render as well as local execution.
init_db()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)), debug=False)
