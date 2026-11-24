from flask import Flask, request, redirect, session, render_template_string
from werkzeug.security import generate_password_hash, check_password_hash
from markupsafe import escape
import sqlite3, random, time, re, os

app = Flask(__name__)
app.secret_key = os.environ.get("BETEX_SECRET_KEY", "CHANGE_THIS_SECRET_BEFORE_DEPLOYING")
DB = "betex.db"
ADMIN_KEY = os.environ.get("BETEX_ADMIN_KEY", "change-this-admin-key")

# ---------- DATABASE ----------
def db():
    con = sqlite3.connect(DB)
    con.row_factory = sqlite3.Row
    return con

def init_db():
    con = db()
    con.executescript("""
    CREATE TABLE IF NOT EXISTS users(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      phone TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      points INTEGER DEFAULT 1000,
      display_name TEXT DEFAULT '',
      favorite_team TEXT DEFAULT ''
    );
    CREATE TABLE IF NOT EXISTS news(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL, body TEXT NOT NULL, created_at TEXT DEFAULT CURRENT_TIMESTAMP
    );
    CREATE TABLE IF NOT EXISTS matches(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      sport TEXT NOT NULL, home_team TEXT NOT NULL, away_team TEXT NOT NULL,
      match_time TEXT NOT NULL, status TEXT DEFAULT 'Upcoming', score TEXT DEFAULT '-'
    );
    CREATE TABLE IF NOT EXISTS notifications(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL, message TEXT NOT NULL,
      created_at TEXT DEFAULT CURRENT_TIMESTAMP, seen INTEGER DEFAULT 0
    );
    CREATE TABLE IF NOT EXISTS chat(
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL, message TEXT NOT NULL,
      created_at TEXT DEFAULT CURRENT_TIMESTAMP
    );
    """)
    if con.execute("SELECT COUNT(*) FROM matches").fetchone()[0] == 0:
        con.executemany("""INSERT INTO matches
          (sport,home_team,away_team,match_time,status,score)
          VALUES(?,?,?,?,?,?)""", [
          ("Football","Team Alpha","Team Beta","Demo fixture","LIVE DEMO","1 - 0"),
          ("Cricket","Team Red","Team Blue","Demo fixture","UPCOMING","-"),
          ("Basketball","City Hoops","River Stars","Demo fixture","UPCOMING","-")
        ])
    if con.execute("SELECT COUNT(*) FROM news").fetchone()[0] == 0:
        con.execute("INSERT INTO news(title,body) VALUES(?,?)",
          ("Welcome to BETEX","Follow sports news, fixtures and community updates here."))
    con.commit()
    con.close()

# ---------- SHARED DESIGN ----------
CSS = """
<style>
:root{--bg:#091522;--panel:#192d40;--panel2:#14283b;--text:#f5f8fb;--muted:#9eb0bf;--green:#00e6a0}
body.light{--bg:#eef3f7;--panel:#fff;--panel2:#dce7ef;--text:#142333;--muted:#536779}
*{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--text);font-family:Arial,sans-serif}
header{background:var(--panel2);padding:17px;display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:10px}
.logo{font-size:29px;font-weight:bold;color:var(--green);text-decoration:none}
nav{background:var(--panel);padding:13px;display:flex;gap:15px;justify-content:center;flex-wrap:wrap}
nav a,a{color:var(--green);text-decoration:none}nav a{color:var(--text)}
.container{max-width:900px;margin:22px auto;padding:0 14px}
.card{background:var(--panel);padding:19px;border-radius:14px;margin:14px 0}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px}
h1,h2,h3{margin-top:5px}h2{color:var(--green)}
input,textarea,select{width:100%;padding:13px;margin:7px 0 12px;border:1px solid #536779;border-radius:8px;background:var(--bg);color:var(--text)}
textarea{min-height:90px}button,.btn{display:inline-block;background:var(--green);color:#06251c;border:0;padding:12px 16px;border-radius:8px;font-weight:bold;cursor:pointer}
button.full{width:100%}.muted{color:var(--muted)}.error{color:#ff7777}.success{color:var(--green)}
.pill{display:inline-block;padding:5px 9px;border-radius:20px;background:#28475a;color:#b8ffe8;font-size:12px}
.score{font-size:25px;font-weight:bold;color:var(--green)}
footer{text-align:center;color:var(--muted);padding:25px}
table{width:100%;border-collapse:collapse}td,th{padding:10px;border-bottom:1px solid #526272;text-align:left}
</style>
"""

def page(content, title="BETEX"):
    theme = "light" if session.get("light") else ""
    return render_template_string("""
    <!doctype html><html><head><title>{{title}}</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    """ + CSS + """</head>
    <body class="{{theme}}">
    <header><a class="logo" href="/">BETEX</a><span>SPORTS COMMUNITY</span></header>
    <nav>
      <a href="/">Home</a><a href="/scores">Scores</a><a href="/schedule">Fixtures</a>
      <a href="/news">News</a><a href="/chat">Community</a>
      <a href="/dashboard">Dashboard</a><a href="/theme">Theme</a>
    </nav>
    <main class="container">{{content|safe}}</main>
    <footer>BETEX © 2026 · Sports entertainment demo · Virtual points only</footer>
    </body></html>
    """, content=content, title=title, theme=theme)

def current_user():
    uid = session.get("user_id")
    if not uid:
        return None
    con = db()
    user = con.execute("SELECT * FROM users WHERE id=?", (uid,)).fetchone()
    con.close()
    return user

def require_login():
    if not current_user():
        return redirect("/login")
    return None

def notify_all(message):
    con = db()
    ids = con.execute("SELECT id FROM users").fetchall()
    con.executemany("INSERT INTO notifications(user_id,message) VALUES(?,?)",
                    [(u["id"], message) for u in ids])
    con.commit()
    con.close()

# ---------- HOME ----------
@app.route("/")
def home():
    user = current_user()
    greeting = ("Welcome, " + escape(user["display_name"] or user["phone"]) if user else "Welcome to BETEX")
    content = f"""
    <div class="card"><span class="pill">SPORTS ENTERTAINMENT</span>
      <h1>{greeting}</h1><h2>Follow the games. Join the community.</h2>
      <p class="muted">Explore demo scores, fixtures, sports news and team updates.</p>
      <a class="btn" href="/scores">Explore scores</a>
      <a class="btn" href="/register">Create account</a>
    </div>
    <div class="grid">
      <div class="card"><h3>⚽ Football</h3><p class="muted">Fixtures and demo scores</p></div>
      <div class="card"><h3>🏏 Cricket</h3><p class="muted">Fixtures and demo scores</p></div>
      <div class="card"><h3>🏀 Basketball</h3><p class="muted">Sports updates</p></div>
    </div>
    <div class="card"><h2>BETEX features</h2>
      <p>Live-score-style demo · Profiles · News · Fixtures · Leaderboard · Favourites · Notifications · Admin · Themes · Community chat</p>
    </div>"""
    return page(content)

@app.route("/theme")
def theme():
    session["light"] = not session.get("light", False)
    return redirect(request.referrer or "/")

# ---------- PHONE REGISTRATION + DEMO OTP ----------
@app.route("/register", methods=["GET","POST"])
def register():
    error = ""
    if request.method == "POST":
        phone = request.form.get("phone","").strip()
        if not re.fullmatch(r"[0-9]{10}", phone):
            error = "Enter a valid 10-digit Nepal mobile number."
        else:
            con = db()
            exists = con.execute("SELECT id FROM users WHERE phone=?", (phone,)).fetchone()
            con.close()
            if exists:
                error = "This phone number is already registered."
            else:
                session["reg_phone"] = phone
                session["reg_otp"] = str(random.randint(100000,999999))
                session["otp_expiry"] = time.time() + 300
                session["otp_attempts"] = 0
                return redirect("/verify")
    return page(f"""<div class="card"><h2>Create BETEX account</h2>
      <p>Enter your Nepal mobile number (+977).</p><p class="error">{escape(error)}</p>
      <form method="post"><label>Phone number</label>
      <input name="phone" inputmode="numeric" maxlength="10" placeholder="98XXXXXXXX" required>
      <button class="full">Continue to demo OTP</button></form>
      <p class="muted">Local demo only: this version does not send SMS.</p>
      <a href="/login">Already registered? Login</a></div>""")

@app.route("/verify", methods=["GET","POST"])
def verify():
    if "reg_phone" not in session:
        return redirect("/register")
    error = ""
    if request.method == "POST":
        code = request.form.get("otp","").strip()
        session["otp_attempts"] = session.get("otp_attempts",0) + 1
        if session["otp_attempts"] > 5:
            session.pop("reg_phone",None); session.pop("reg_otp",None)
            return redirect("/register")
        if time.time() > session.get("otp_expiry",0):
            error = "OTP expired. Register again."
        elif code == session.get("reg_otp"):
            session["otp_verified"] = True
            return redirect("/create-password")
        else:
            error = "Incorrect OTP."
    demo = escape(session["reg_otp"])
    return page(f"""<div class="card"><h2>Verify phone</h2>
      <div class="card"><p class="muted">DEMO OTP (not an SMS)</p><div class="score">{demo}</div></div>
      <p class="error">{escape(error)}</p>
      <form method="post"><input name="otp" inputmode="numeric" maxlength="6" placeholder="6-digit OTP" required>
      <button class="full">Verify OTP</button></form></div>""")

@app.route("/create-password", methods=["GET","POST"])
def create_password():
    if not session.get("otp_verified") or "reg_phone" not in session:
        return redirect("/register")
    error = ""
    if request.method == "POST":
        pw = request.form.get("password","")
        confirm = request.form.get("confirm","")
        if len(pw) < 8:
            error = "Use at least 8 characters."
        elif pw != confirm:
            error = "Passwords do not match."
        else:
            con = db()
            try:
                con.execute("INSERT INTO users(phone,password) VALUES(?,?)",
                            (session["reg_phone"], generate_password_hash(pw)))
                con.commit()
                uid = con.execute("SELECT id FROM users WHERE phone=?",
                                  (session["reg_phone"],)).fetchone()["id"]
                con.close()
                session.clear()
                session["user_id"] = uid
                return redirect("/dashboard")
            except sqlite3.IntegrityError:
                con.close()
                error = "That phone number already has an account."
    return page(f"""<div class="card"><h2>Create password</h2><p class="error">{escape(error)}</p>
      <form method="post"><input type="password" name="password" minlength="8" placeholder="Password" required>
      <input type="password" name="confirm" placeholder="Confirm password" required>
      <button class="full">Create account</button></form></div>""")

@app.route("/login", methods=["GET","POST"])
def login():
    error = ""
    if request.method == "POST":
        phone = request.form.get("phone","").strip()
        pw = request.form.get("password","")
        con = db()
        user = con.execute("SELECT id,password FROM users WHERE phone=?", (phone,)).fetchone()
        con.close()
        if user and check_password_hash(user["password"],pw):
            session.clear()
            session["user_id"] = user["id"]
            return redirect("/dashboard")
        error = "Incorrect phone number or password."
    return page(f"""<div class="card"><h2>Login to BETEX</h2><p class="error">{escape(error)}</p>
      <form method="post"><input name="phone" inputmode="numeric" maxlength="10" placeholder="10-digit phone number" required>
      <input name="password" type="password" placeholder="Password" required>
      <button class="full">Login</button></form><a href="/register">Create account</a></div>""")

# ---------- PROFILE + DASHBOARD ----------
@app.route("/dashboard")
def dashboard():
    user = current_user()
    if not user: return redirect("/login")
    con = db()
    notes = con.execute("SELECT * FROM notifications WHERE user_id=? ORDER BY id DESC LIMIT 8",
                        (user["id"],)).fetchall()
    con.close()
    note_html = "".join(f"<li>{escape(n['message'])} <small class='muted'>· {escape(n['created_at'])}</small></li>" for n in notes) or "<li>No notifications yet.</li>"
    return page(f"""<div class="card"><h2>Your dashboard</h2>
      <p>Phone: +977 {escape(user['phone'])}</p>
      <p>Virtual points</p><div class="score">{user['points']}</div>
      <p class="muted">Demo points have no cash value.</p>
      <a class="btn" href="/profile">Edit profile</a> <a class="btn" href="/logout">Logout</a>
    </div><div class="card"><h2>Notifications</h2><ul>{note_html}</ul></div>""")

@app.route("/profile", methods=["GET","POST"])
def profile():
    user = current_user()
    if not user: return redirect("/login")
    if request.method == "POST":
        name = request.form.get("name","").strip()[:40]
        team = request.form.get("team","").strip()[:60]
        con = db()
        con.execute("UPDATE users SET display_name=?,favorite_team=? WHERE id=?",
                    (name,team,user["id"]))
        con.commit(); con.close()
        return redirect("/profile")
    return page(f"""<div class="card"><h2>Your profile</h2>
      <form method="post"><label>Display name</label>
      <input name="name" maxlength="40" value="{escape(user['display_name'])}" placeholder="Your name">
      <label>Favourite team</label>
      <input name="team" maxlength="60" value="{escape(user['favorite_team'])}" placeholder="Favourite team">
      <button class="full">Save profile</button></form></div>""")

# ---------- SCORES + FIXTURES ----------
@app.route("/scores")
def scores():
    con = db(); rows = con.execute("SELECT * FROM matches ORDER BY id DESC").fetchall(); con.close()
    items = ""
    for m in rows:
        items += f"""<div class="card"><span class="pill">{escape(m['sport'])} · {escape(m['status'])}</span>
        <h3>{escape(m['home_team'])} vs {escape(m['away_team'])}</h3>
        <div class="score">{escape(m['score'])}</div><p class="muted">{escape(m['match_time'])}</p></div>"""
    return page("<h2>Scoreboard</h2><p class='muted'>Scores shown here are demo/sample data, not live verified scores.</p>"+(items or "<p>No matches yet.</p>"))

@app.route("/schedule")
def schedule():
    con = db(); rows = con.execute("SELECT * FROM matches ORDER BY id DESC").fetchall(); con.close()
    items = ""
    for m in rows:
        items += f"<tr><td>{escape(m['sport'])}</td><td>{escape(m['home_team'])} vs {escape(m['away_team'])}</td><td>{escape(m['match_time'])}</td><td>{escape(m['status'])}</td></tr>"
    return page("<div class='card'><h2>Match schedule</h2><table><tr><th>Sport</th><th>Match</th><th>Time</th><th>Status</th></tr>"+items+"</table></div>")

# ---------- FAVOURITES ----------
@app.route("/favorites")
def favorites():
    user = current_user()
    if not user: return redirect("/login")
    return page(f"""<div class="card"><h2>Favourite team</h2>
      <p>{escape(user['favorite_team'] or 'No favourite team selected yet.')}</p>
      <a class="btn" href="/profile">Choose or update team</a></div>""")

# ---------- NEWS ----------
@app.route("/news")
def news():
    con = db(); rows = con.execute("SELECT * FROM news ORDER BY id DESC").fetchall(); con.close()
    items = ""
    for n in rows:
        items += f"<div class='card'><h3>{escape(n['title'])}</h3><p>{escape(n['body'])}</p><small class='muted'>{escape(n['created_at'])}</small></div>"
    return page("<h2>Sports news</h2>"+(items or "<p>No news yet.</p>"))

# ---------- COMMUNITY CHAT ----------
@app.route("/chat", methods=["GET","POST"])
def chat():
    user = current_user()
    if not user: return redirect("/login")
    if request.method == "POST":
        msg = request.form.get("message","").strip()
        if msg:
            con = db()
            con.execute("INSERT INTO chat(user_id,message) VALUES(?,?)",
                        (user["id"],msg[:500]))
            con.commit(); con.close()
        return redirect("/chat")
    con = db()
    rows = con.execute("""SELECT chat.message,chat.created_at,users.display_name,users.phone
                          FROM chat JOIN users ON users.id=chat.user_id
                          ORDER BY chat.id DESC LIMIT 50""").fetchall()
    con.close()
    items = ""
    for r in reversed(rows):
        who = r["display_name"] or ("User " + r["phone"][-4:])
        items += f"<div class='card'><b>{escape(who)}</b><p>{escape(r['message'])}</p><small class='muted'>{escape(r['created_at'])}</small></div>"
    return page("""<div class="card"><h2>Community chat</h2>
      <form method="post"><textarea name="message" maxlength="500" placeholder="Share a sports-related message..." required></textarea>
      <button class="full">Send message</button></form>
      <p class="muted">Be respectful. Do not share personal information.</p></div>""" + items)

# ---------- LEADERBOARD ----------
@app.route("/leaderboard")
def leaderboard():
    con = db()
    rows = con.execute("SELECT display_name,phone,points FROM users ORDER BY points DESC LIMIT 20").fetchall()
    con.close()
    items = ""
    for i,u in enumerate(rows,1):
        name = u["display_name"] or ("User " + u["phone"][-4:])
        items += f"<tr><td>{i}</td><td>{escape(name)}</td><td>{u['points']}</td></tr>"
    return page("<div class='card'><h2>Virtual points leaderboard</h2><table><tr><th>Rank</th><th>User</th><th>Points</th></tr>"+items+"</table><p class='muted'>Points are for demo participation only and have no cash value.</p></div>")

# ---------- NOTIFICATIONS ----------
@app.route("/notifications")
def notifications():
    user = current_user()
    if not user: return redirect("/login")
    con = db()
    rows = con.execute("SELECT * FROM notifications WHERE user_id=? ORDER BY id DESC",
                       (user["id"],)).fetchall()
    con.execute("UPDATE notifications SET seen=1 WHERE user_id=?", (user["id"],))
    con.commit(); con.close()
    items = "".join(f"<div class='card'>{escape(n['message'])}<p class='muted'>{escape(n['created_at'])}</p></div>" for n in rows)
    return page("<h2>Your notifications</h2>"+(items or "<p>No notifications.</p>"))

# ---------- ADMIN ----------
@app.route("/admin", methods=["GET","POST"])
def admin():
    if not session.get("is_admin"):
        error = ""
        if request.method == "POST":
            if request.form.get("key","") == ADMIN_KEY and ADMIN_KEY != "change-this-admin-key":
                session["is_admin"] = True
                return redirect("/admin")
            error = "Admin key is incorrect, or set BETEX_ADMIN_KEY before using admin."
        return page(f"""<div class="card"><h2>Admin login</h2><p class="error">{escape(error)}</p>
          <form method="post"><input type="password" name="key" placeholder="Admin key" required>
          <button class="full">Admin login</button></form>
          <p class="muted">Set BETEX_ADMIN_KEY in your hosting environment before deployment.</p></div>""")
    con = db()
    users = con.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    matches = con.execute("SELECT COUNT(*) FROM matches").fetchone()[0]
    con.close()
    return page(f"""<div class="card"><h2>Admin dashboard</h2>
      <p>Registered users: <b>{users}</b></p><p>Listed matches: <b>{matches}</b></p>
      <h3>Publish sports news</h3>
      <form method="post" action="/admin/news"><input name="title" placeholder="News title" required>
      <textarea name="body" placeholder="News details" required></textarea><button class="full">Publish news</button></form>
      <h3>Add a demo fixture</h3>
      <form method="post" action="/admin/match">
      <input name="sport" placeholder="Sport (Football/Cricket)" required>
      <input name="home" placeholder="Home team" required><input name="away" placeholder="Away team" required>
      <input name="time" placeholder="Date/time label" required>
      <input name="status" placeholder="UPCOMING / DEMO" value="UPCOMING">
      <input name="score" placeholder="Score" value="-">
      <button class="full">Add fixture</button></form>
      <a class="btn" href="/admin/logout">Admin logout</a></div>""")

@app.route("/admin/news", methods=["POST"])
def admin_news():
    if not session.get("is_admin"): return redirect("/admin")
    title = request.form.get("title","").strip()[:120]
    body = request.form.get("body","").strip()[:2000]
    if title and body:
        con = db(); con.execute("INSERT INTO news(title,body) VALUES(?,?)",(title,body)); con.commit(); con.close()
        notify_all("New sports news: " + title)
    return redirect("/admin")

@app.route("/admin/match", methods=["POST"])
def admin_match():
    if not session.get("is_admin"): return redirect("/admin")
    values = [request.form.get(k,"").strip() for k in ("sport","home","away","time","status","score")]
    if all(values[:4]):
        con = db()
        con.execute("""INSERT INTO matches(sport,home_team,away_team,match_time,status,score)
                       VALUES(?,?,?,?,?,?)""", values)
        con.commit(); con.close()
        notify_all("A new demo fixture was added: " + values[1] + " vs " + values[2])
    return redirect("/admin")

@app.route("/admin/logout")
def admin_logout():
    session.pop("is_admin",None)
    return redirect("/")

@app.route("/logout")
def logout():
    session.clear()
    return redirect("/")

if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT",5000)), debug=False)
