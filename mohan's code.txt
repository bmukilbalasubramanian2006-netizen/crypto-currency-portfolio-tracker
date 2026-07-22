from flask import Flask, render_template_string, request, redirect, session
import sqlite3, requests
import re 
from werkzeug.security import generate_password_hash, check_password_hash
from plyer import notification

app = Flask(__name__)
app.secret_key = "your_secret_key"

# === DB Initialization ===
def init_db():
    with sqlite3.connect("crypto.db") as con:
        cur = con.cursor()
        cur.execute("""CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE,
            password TEXT
        )""")
        cur.execute("""CREATE TABLE IF NOT EXISTS portfolio (
            user_id INTEGER,
            coin_id TEXT,
            amount REAL,
            alert_above REAL,
            alert_below REAL
        )""")
init_db()

def fetch_price(coin_id):
    try:
        url = f"https://api.coingecko.com/api/v3/simple/price?ids={coin_id}&vs_currencies=usd"
        data = requests.get(url).json()
        return data[coin_id]['usd']
    except:
        return None

def send_notification(title, message):
    try:
        notification.notify(title=title, message=message, timeout=10)
    except Exception as e:
        print("Notification Error:", e)

# === HTML Templates ===
login_template = """<!DOCTYPE html><html><head><title>{{ 'Register' if is_register else 'Login' }}</title>
<style>body{background:linear-gradient(to bottom,#e0f7fa,#80deea);font-family:'Segoe UI',sans-serif;margin:0}.topbar{background:#006064;color:#fff;padding:10px;text-align:center;font-size:18px}.container{background:#fff;max-width:400px;margin:80px auto;padding:30px;border-radius:10px;box-shadow:0 4px 12px rgba(0,0,0,0.2)}h2{text-align:center;color:#006064}input{width:100%;padding:12px;margin:10px 0;border:1px solid #006064;border-radius:5px}button{background:#004d40;color:#fff;border:none;padding:12px;width:100%;border-radius:5px;cursor:pointer}button:hover{background:#00251a}a{display:block;text-align:center;margin-top:15px;color:#006064;text-decoration:none}</style>
</head><body><div class="topbar">💰 Crypto Portfolio Tracker</div>
<div class="container"><h2>{{ 'Register' if is_register else 'Login' }}</h2>
<form method="POST">
<input name="email" type="email" placeholder="Gmail ID" required>
<input name="password" type="password" placeholder="Password" required>
<button type="submit">{{ 'Register' if is_register else 'Login' }}</button>
</form>
{% if is_register %}
<a href="/login">Already have an account? Login</a>
{% else %}
<a href="/register">New user? Register here</a> <br>
{% endif %}
</div></body></html>"""

portfolio_template = """<!DOCTYPE html><html><head><title>Crypto Portfolio</title>
<style>body{margin:0;font-family:'Segoe UI',sans-serif;background:linear-gradient(to right,#e1f5fe,#b3e5fc)}.topnav{background:#006064;overflow:hidden;color:#fff;padding:15px 20px;display:flex;justify-content:space-between;align-items:center}.topnav h1{margin:0;font-size:20px}.topnav a{color:#fff;text-decoration:none;background:#004d40;padding:8px 14px;border-radius:5px}.topnav a:hover{background:#00251a}.container{background:#fff;max-width:900px;margin:30px auto;padding:30px;border-radius:10px;box-shadow:0 0 12px rgba(0,0,0,0.2)}h2,h3{color:#006064;text-align:center}form{display:flex;flex-wrap:wrap;gap:10px;justify-content:center}input{padding:10px;border:1px solid #006064;border-radius:5px}button{background:#d32f2f;color:#fff;border:none;padding:10px 20px;border-radius:5px;cursor:pointer}table{width:100%;margin-top:20px;border-collapse:collapse}th{background:#00796b;color:#fff;padding:10px}td{background:#e0f2f1;text-align:center;padding:10px;border:1px solid #ccc}</style>
</head><body><div class="topnav"><h1>Welcome {{ email }}</h1><a href="/logout">Logout</a></div>
<div class="container"><h2>Crypto Portfolio Tracker</h2>
<form method="POST" action="/add">
<input name="coin_id" placeholder="Coin ID (e.g. bitcoin)" required>
<input name="amount" type="number" step="any" placeholder="Amount" required>
<input name="alert_above" type="number" step="any" placeholder="Alert Above ($)">
<input name="alert_below" type="number" step="any" placeholder="Alert Below ($)">
<button>Add Coin</button></form>
<table><tr><th>Coin</th><th>Price</th><th>Amount</th><th>Value</th></tr>{{ table|safe }}</table>
<h3>Total: ${{ '%.2f'|format(total) }}</h3></div><a href="/users">View Registered Users</a></body></html>"""

users_template = """<!DOCTYPE html>
<html>
<head>
    <title>List of Users</title>
    <style>
        body {
            background-color: #d3d3d3;
            font-family: 'Segoe UI', sans-serif;
        }
        table {
            border-collapse: collapse;
            width: 60%;
            margin: 40px auto;
            box-shadow: 0 4px 12px rgba(0,0,0,0.2);
        }
        th {
            background-color: #006400;
            color: #fff;
            padding: 12px;
            text-align: left;
        }
        td {
            background-color: #50c878;
            padding: 10px;
            border: 1px solid #444;
            color: #000;
        }
        h1 {
            text-align: center;
            margin-top: 30px;
            color: #006400;
        }
    </style>
</head>
<body>
    <h1>Registered Users</h1>
    <table>
        <tr>
            <th>UserID</th>
            <th>Username</th>
        </tr>
        {% for user in users %}
        <tr>
            <td>{{ user[0] }}</td>
            <td>{{ user[1] }}</td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>"""

# === Routes ===
@app.route('/')
def home():
    if 'user_id' not in session:
        return redirect('/login')
    user_id = session['user_id']
    email = session['email']
    with sqlite3.connect("crypto.db") as con:
        cur = con.cursor()
        cur.execute("SELECT coin_id, amount, alert_above, alert_below FROM portfolio WHERE user_id = ?", (user_id,))
        rows = cur.fetchall()

    table = ""
    total = 0
    for coin_id, amount, alert_above, alert_below in rows:
        price = fetch_price(coin_id)
        if price:
            value = price * amount
            total += value
            table += f"<tr><td>{coin_id}</td><td>${price:.2f}</td><td>{amount}</td><td>${value:.2f}</td></tr>"
            if alert_above and price > alert_above:
                send_notification(f"{coin_id.title()} 🚀 Above ${alert_above}", f"Now: ${price:.2f}")
            if alert_below and price < alert_below:
                send_notification(f"{coin_id.title()} 📉 Below ${alert_below}", f"Now: ${price:.2f}")
        else:
            table += f"<tr><td>{coin_id}</td><td>Error</td><td>{amount}</td><td>N/A</td></tr>"
    return render_template_string(portfolio_template, email=email, table=table, total=total)

@app.route('/add', methods=["POST"])
def add():
    if 'user_id' not in session:
        return redirect('/login')
    user_id = session['user_id']
    coin_id = request.form['coin_id'].lower()
    try:
        amount = float(request.form['amount'])
        alert_above = float(request.form.get('alert_above') or 0)
        alert_below = float(request.form.get('alert_below') or 0)
    except:
        return "Invalid input"
    with sqlite3.connect("crypto.db") as con:
        cur = con.cursor()
        cur.execute("SELECT amount FROM portfolio WHERE user_id = ? AND coin_id = ?", (user_id, coin_id))
        row = cur.fetchone()
        if row:
            cur.execute("UPDATE portfolio SET amount = amount + ?, alert_above = ?, alert_below = ? WHERE user_id = ? AND coin_id = ?",
                        (amount, alert_above, alert_below, user_id, coin_id))
        else:
            cur.execute("INSERT INTO portfolio (user_id, coin_id, amount, alert_above, alert_below) VALUES (?, ?, ?, ?, ?)",
                        (user_id, coin_id, amount, alert_above, alert_below))
    return redirect('/')

@app.route('/register', methods=["GET", "POST"])
def register():
    if request.method == 'POST':
        email = request.form['email']
        password = request.form['password']

        # ✅ Email check
        if not email.endswith("@gmail.com"):
            return "Only Gmail allowed!"

        # ✅ Password complexity check
        pattern = r'^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@#$^&*_\-]).{8,}$'
        if not re.match(pattern, password):
            return """Password does not meet the requirements:
            - Password must be at least 8 characters
            - Include at least one uppercase
            - Include at least one lowercase
            - Include at least one digit
            - Include at least one special character (@#$^&*-_)"""

        # 🔒 Secure password storage
        hashed_password = generate_password_hash(password)

        # 🗃️ Database operations
        try:
            with sqlite3.connect("crypto.db") as con:
                cur = con.cursor()
                cur.execute("INSERT INTO users (email, password) VALUES (?, ?)", (email, hashed_password))
                con.commit()
        except sqlite3.IntegrityError:
            return "Email already exists!"

        return redirect('/login')
    
    return render_template_string(login_template, is_register=True)

@app.route("/users")
def list_users():
    conn = sqlite3.connect("crypto.db")
    cursor = conn.cursor()
    cursor.execute("SELECT id,email FROM users")
    users = cursor.fetchall()
    conn.close()
    return render_template_string(users_template, users=users)


@app.route('/login', methods=["GET", "POST"])
def login():
    if request.method == 'POST':
        email = request.form['email']
        password = request.form['password']
        with sqlite3.connect("crypto.db") as con:
            cur = con.cursor()
            cur.execute("SELECT id, password FROM users WHERE email = ?", (email,))
            row = cur.fetchone()
            if row and check_password_hash(row[1], password):
                session['user_id'] = row[0]
                session['email'] = email
                return redirect('/')
            else:
                return "Invalid credentials"
    return render_template_string(login_template, is_register=False)

@app.route('/logout')
def logout():
    session.clear()
    return redirect('/login')

if __name__ == '__main__':
    app.run(debug=True)