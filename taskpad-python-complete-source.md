# Taskpad (Python): complete source code

Every file in the project, in the order you would create them. The folder path is in each heading.

## `taskpad-python/README.md`

````markdown
# Taskpad (Python / Flask)

A closed-pilot site where members do short tasks and surveys, earn Sh.100 per task (plus a Sh.200 sign-up bonus), unlock paid tasks with an M-Pesa prompt (Sh.30), and withdraw to M-Pesa once they reach the minimum (Sh.2,500). The server only accepts new members while a reserve of your own money covers everything they could still withdraw.

## Folder layout
    app.py             API, accounts, ledger, reserve rules, M-Pesa (Daraja) calls
    tasks.json         the 24 tasks (id, category, price, questions); edit to change tasks
    public/index.html  the member site (Home, Wallet, Jobs)
    public/admin.html  your admin page: reserve, commitments, withdrawals
    smoke_test.py      self-check of the rules (no real M-Pesa needed)
    deploy/setup.sh    one-command setup for a fresh Ubuntu server (HTTPS, service, backups)
    .env.example       every setting, explained

## Run it on your computer
    python3 -m venv venv
    venv/bin/pip install -r requirements.txt
    cp .env.example .env        # fill it in; use `openssl rand -hex 32` for each secret
    venv/bin/python smoke_test.py
    venv/bin/python app.py      # http://localhost:3000  and  http://localhost:3000/admin.html

M-Pesa callbacks need a public HTTPS address. For sandbox testing run `ngrok http 3000` and put that address in PUBLIC_URL.

## Rules you can change (.env)
SIGNUP_BONUS, TASK_REWARD, MIN_WITHDRAW, MAX_MEMBERS, INVITE_CODE. The server refuses to start if MIN_WITHDRAW is higher than a member could ever earn (SIGNUP_BONUS + TASK_REWARD x number of tasks).

## How the reserve works
Add the money you have set aside on /admin.html. A new member is accepted only if the reserve covers: what members hold now + pending withdrawals + everything current members could still earn + the maximum a new member could earn. Unlock fees are not counted. Keep the number equal to the real money in your payout account.

## Going live
1. Registered business, KRA PIN, Paybill/Till, Daraja production approval (including B2C).
2. Point your domain's A record at the server, then: `sudo bash deploy/setup.sh your-domain.com` (runs the app with gunicorn behind Caddy HTTPS).
3. Set DARAJA_BASE=https://api.safaricom.co.ke and your production keys in `.env`.
4. Test with real small amounts: unlock, cancel, time out, withdraw.
5. Register with the Office of the Data Protection Commissioner and show clear terms on the sign-up page.

## Safety notes
Never share or commit `.env`. Back up the database (setup.sh does it nightly). Run gunicorn with one worker (the default in setup.sh); threads handle concurrency. A withdrawal that times out stays "pending" for you to check by hand. Prices live in tasks.json on the server, so users cannot change them.
````

## `taskpad-python/requirements.txt`

````text
flask>=3.0
bcrypt>=4.1
PyJWT>=2.8
requests>=2.31
python-dotenv>=1.0
gunicorn>=22.0
````

## `taskpad-python/.env.example`

````bash
# Copy to .env and fill in. Never commit .env.
PORT=3000
PUBLIC_URL=https://your-public-https-domain.example   # Safaricom must be able to reach this (use ngrok while testing)
JWT_SECRET=change-me-to-a-long-random-string
CALLBACK_SECRET=another-long-random-string-used-in-callback-urls

# Closed pilot controls
ADMIN_TOKEN=            # 16+ characters; opens /admin.html
INVITE_CODE=            # share only with pilot members; leave empty to allow anyone (while there is reserve)
MAX_MEMBERS=50

# Rewards and rules (KES)
SIGNUP_BONUS=200
TASK_REWARD=100
MIN_WITHDRAW=2500

# Daraja (sandbox by default; use https://api.safaricom.co.ke for live)
DARAJA_BASE=https://sandbox.safaricom.co.ke
DARAJA_KEY=
DARAJA_SECRET=

# STK Push (customer pays you)
SHORTCODE=174379
PASSKEY=
TXN_TYPE=CustomerPayBillOnline   # use CustomerBuyGoodsOnline for a Till number

# B2C (you pay customers: withdrawals)
B2C_SHORTCODE=
B2C_INITIATOR=
B2C_CREDENTIAL=   # initiator password encrypted with Safaricom's certificate
````

## `taskpad-python/app.py`

````python
"""Taskpad backend: Flask + SQLite + M-Pesa (Daraja).
Run locally:  python app.py      Production:  gunicorn -w 1 --threads 8 -b 127.0.0.1:3000 app:app
"""
import base64, hmac, json, os, sqlite3, time
from contextlib import closing, contextmanager
from datetime import datetime, timedelta, timezone
from functools import wraps

import bcrypt, jwt, requests
from dotenv import load_dotenv
from flask import Flask, g, jsonify, request, send_from_directory
from werkzeug.middleware.proxy_fix import ProxyFix

load_dotenv()
E = os.environ
HERE = os.path.dirname(os.path.abspath(__file__))
num = lambda k, d: int(E.get(k) or d)
CFG = dict(
    port=num('PORT', 3000), jwt=E.get('JWT_SECRET'), secret=E.get('CALLBACK_SECRET'), public=E.get('PUBLIC_URL', ''),
    max_members=num('MAX_MEMBERS', 50), signup=num('SIGNUP_BONUS', 200), reward=num('TASK_REWARD', 100),
    min_w=num('MIN_WITHDRAW', 2500), base=E.get('DARAJA_BASE') or 'https://sandbox.safaricom.co.ke')
if not CFG['jwt'] or not CFG['secret']:
    raise SystemExit('Set JWT_SECRET and CALLBACK_SECRET in .env')
if len(E.get('ADMIN_TOKEN') or '') < 16:
    raise SystemExit('Set ADMIN_TOKEN (16+ characters) in .env')

# Tasks live on the server (tasks.json): one source of truth for the catalog and prices.
with open(os.path.join(HERE, 'tasks.json'), encoding='utf-8') as f:
    TASKS = json.load(f)
PRICES = {t['id']: t['price'] for t in TASKS}
PER_MEMBER = CFG['signup'] + CFG['reward'] * len(TASKS)  # most one member can ever earn
if PER_MEMBER < CFG['min_w']:  # refuse to start if nobody could ever reach the withdrawal minimum
    raise SystemExit(f"MIN_WITHDRAW ({CFG['min_w']}) is higher than the most a member can earn ({PER_MEMBER}). Add tasks or lower it.")

# ---------------- database ----------------
DB_FILE = E.get('DB_FILE') or os.path.join(HERE, 'taskpad.db')
SCHEMA = """
create table if not exists users(id integer primary key, name text, phone text unique, pass text, created_at text default current_timestamp);
create table if not exists ledger(id integer primary key, user_id integer, type text, amount integer, ref text, created_at text default current_timestamp);
create unique index if not exists ledger_once on ledger(user_id, type, ref);
create table if not exists payments(checkout_id text primary key, user_id integer, task_id text, amount integer, phone text, status text default 'pending', receipt text, created_at text default current_timestamp);
create table if not exists unlocks(user_id integer, task_id text, primary key(user_id, task_id));
create table if not exists completions(user_id integer, task_id text, answers text, created_at text default current_timestamp, primary key(user_id, task_id));
create table if not exists reserve(id integer primary key, amount integer, note text, created_at text default current_timestamp);
create table if not exists withdrawals(id integer primary key, user_id integer, amount integer, phone text, status text default 'pending', conv_id text, created_at text default current_timestamp);
"""

def connect():
    c = sqlite3.connect(DB_FILE, timeout=10, isolation_level=None)
    c.row_factory = sqlite3.Row
    c.execute('pragma journal_mode=WAL')
    return c

with closing(connect()) as _c:
    _c.executescript(SCHEMA)

def rows(sql, *a):
    with closing(connect()) as c:
        return c.execute(sql, a).fetchall()

def val(sql, *a):
    r = rows(sql, *a)
    return r[0][0] if r else None

@contextmanager
def tx():
    """One atomic write transaction (takes the write lock up front, so checks and writes can't interleave)."""
    c = connect()
    try:
        c.execute('begin immediate')
        yield c
        c.execute('commit')
    except BaseException:
        try: c.execute('rollback')
        except sqlite3.Error: pass
        raise
    finally:
        c.close()

def add_ledger(c, uid, typ, amount, ref):
    c.execute('insert or ignore into ledger(user_id,type,amount,ref) values(?,?,?,?)', (uid, typ, amount, ref))

# Balance is always the sum of the ledger: signup_bonus + task_reward - withdrawal + withdrawal_refund
balance = lambda uid: val('select coalesce(sum(amount),0) from ledger where user_id=?', uid)

def refund(wid):
    with tx() as c:
        w = c.execute("select * from withdrawals where id=? and status='pending'", (wid,)).fetchone()
        if w:
            c.execute("update withdrawals set status='failed' where id=?", (wid,))
            add_ledger(c, w['user_id'], 'withdrawal_refund', w['amount'], f'r{wid}')

def mark_paid(c, p, receipt=''):
    c.execute("update payments set status='paid', receipt=? where checkout_id=? and status='pending'", (receipt, p['checkout_id']))
    c.execute('insert or ignore into unlocks values(?,?)', (p['user_id'], p['task_id']))

# ---------------- helpers ----------------
def norm(v):
    d = ''.join(ch for ch in str(v or '') if ch.isdigit())
    if len(d) == 10 and d[0] == '0' and d[1] in '17': d = '254' + d[1:]
    elif len(d) == 9 and d[0] in '17': d = '254' + d
    return d if len(d) == 12 and d.startswith('254') and d[3] in '17' else None

eq = lambda a, b: hmac.compare_digest(str(a).encode(), str(b).encode())
err = lambda code, msg: (jsonify(error=msg), code)
sign = lambda uid: jwt.encode({'uid': uid, 'exp': datetime.now(timezone.utc) + timedelta(days=30)}, CFG['jwt'], algorithm='HS256')
stamp = lambda: (datetime.now(timezone.utc) + timedelta(hours=3)).strftime('%Y%m%d%H%M%S')  # EAT
pw_bytes = lambda s: s.encode()[:72]

def me(uid):
    b = balance(uid)
    return {
        'name': val('select name from users where id=?', uid), 'balance': b,
        'earned': val("select coalesce(sum(amount),0) from ledger where user_id=? and type in ('signup_bonus','task_reward')", uid),
        'withdrawn': -val("select coalesce(sum(amount),0) from ledger where user_id=? and type in ('withdrawal','withdrawal_refund')", uid),
        'minWithdraw': CFG['min_w'], 'canWithdraw': b >= CFG['min_w'], 'reward': CFG['reward'],
        'unlocked': [r[0] for r in rows('select task_id from unlocks where user_id=?', uid)],
        'done': [r[0] for r in rows('select task_id from completions where user_id=?', uid)]}

def money():
    """Reserve = money you set aside. It must cover everything members could still withdraw."""
    deposited = val('select coalesce(sum(amount),0) from reserve')
    paid_out = val("select coalesce(sum(amount),0) from withdrawals where status='paid'")
    pending = val("select coalesce(sum(amount),0) from withdrawals where status='pending'")
    balances = val('select coalesce(sum(amount),0) from ledger')
    members = val('select count(*) from users')
    potential = CFG['reward'] * (len(TASKS) * members - val('select count(*) from completions'))
    available = deposited - paid_out
    exposure = balances + pending + potential
    headroom = available - exposure
    room = max(0, min(headroom // PER_MEMBER, CFG['max_members'] - members))
    return dict(deposited=deposited, paidOut=paid_out, available=available, balances=balances, pending=pending,
                potential=potential, exposure=exposure, headroom=headroom, members=members, perMember=PER_MEMBER,
                maxMembers=CFG['max_members'], room=room, open=room > 0)

_hits = {}
def limited(f):
    @wraps(f)
    def w(*a, **k):
        now, ip = time.time(), request.remote_addr or '?'
        hits = [t for t in _hits.get(ip, []) if now - t < 900]
        if len(hits) >= 20: return err(429, 'Too many attempts. Try again in a few minutes.')
        _hits[ip] = hits + [now]
        return f(*a, **k)
    return w

def auth(f):
    @wraps(f)
    def w(*a, **k):
        try: g.uid = jwt.decode(request.headers.get('Authorization', '').replace('Bearer ', ''), CFG['jwt'], algorithms=['HS256'])['uid']
        except Exception: return err(401, 'Please log in.')
        return f(*a, **k)
    return w

def admin(f):
    @wraps(f)
    def w(*a, **k):
        return f(*a, **k) if eq(request.headers.get('x-admin-token', ''), E['ADMIN_TOKEN']) else err(401, 'Admin only.')
    return w

# ---------------- Daraja ----------------
_tok = {'v': None, 'exp': 0}
def daraja_token():
    if _tok['v'] and time.time() < _tok['exp']: return _tok['v']
    r = requests.get(CFG['base'] + '/oauth/v1/generate', params={'grant_type': 'client_credentials'},
                     auth=(E.get('DARAJA_KEY', ''), E.get('DARAJA_SECRET', '')), timeout=15)
    r.raise_for_status()
    _tok.update(v=r.json()['access_token'], exp=time.time() + 3000)
    return _tok['v']

def daraja(path, body):
    r = requests.post(CFG['base'] + path, json=body, headers={'Authorization': 'Bearer ' + daraja_token()}, timeout=30)
    try: return r.json()
    except ValueError: return {}

# ---------------- app ----------------
app = Flask(__name__, static_folder='public', static_url_path='')
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1)
app.config['MAX_CONTENT_LENGTH'] = 20 * 1024

@app.get('/')
def index(): return send_from_directory(app.static_folder, 'index.html')

@app.get('/api/tasks')
def tasks():
    return jsonify(tasks=TASKS, rules=dict(signup=CFG['signup'], reward=CFG['reward'], minW=CFG['min_w'],
                                           open=money()['open'], invite=bool(E.get('INVITE_CODE'))))

class Closed(Exception): pass

@app.post('/api/register')
@limited
def register():
    b = request.get_json(silent=True) or {}
    name, phone, pw = str(b.get('name') or '').strip()[:60], norm(b.get('phone')), str(b.get('password') or '')
    if not name or not phone or len(pw) < 8:
        return err(400, 'Enter your name, a valid Safaricom number, and a password of 8+ characters.')
    if E.get('INVITE_CODE') and not eq(str(b.get('invite') or '').strip(), E['INVITE_CODE']):
        return err(403, 'You need a valid invite code to join.')
    if val('select 1 from users where phone=?', phone): return err(409, 'That number is already registered.')
    h = bcrypt.hashpw(pw_bytes(pw), bcrypt.gensalt()).decode()
    try:
        with tx() as c:  # account and signup bonus are created together or not at all
            if not money()['open']: raise Closed()
            uid = c.execute('insert into users(name,phone,pass) values(?,?,?)', (name, phone, h)).lastrowid
            add_ledger(c, uid, 'signup_bonus', CFG['signup'], 'signup')
    except Closed: return err(403, 'Sign-ups are closed for now.')
    except sqlite3.IntegrityError: return err(409, 'That number is already registered.')
    return jsonify(token=sign(uid), **me(uid))

@app.post('/api/login')
@limited
def login():
    b = request.get_json(silent=True) or {}
    u = rows('select * from users where phone=?', norm(b.get('phone')) or '')
    if not u or not bcrypt.checkpw(pw_bytes(str(b.get('password') or '')), u[0]['pass'].encode()):
        return err(401, 'Wrong phone number or password.')
    return jsonify(token=sign(u[0]['id']), **me(u[0]['id']))

@app.get('/api/me')
@auth
def get_me(): return jsonify(me(g.uid))

# Pay to unlock: sends the STK push prompt to the customer's phone
@app.post('/api/stk-push')
@auth
def stk_push():
    b = request.get_json(silent=True) or {}
    task_id, phone = b.get('taskId'), norm(b.get('phone'))
    price = PRICES.get(task_id)
    if not price: return err(400, 'This task is free or does not exist.')
    if not phone: return err(400, 'Enter a valid Safaricom number.')
    if val('select 1 from unlocks where user_id=? and task_id=?', g.uid, task_id): return err(409, 'Already unlocked.')
    ts, sc = stamp(), E.get('SHORTCODE', '')
    try:
        r = daraja('/mpesa/stkpush/v1/processrequest', {
            'BusinessShortCode': sc, 'Password': base64.b64encode((sc + E.get('PASSKEY', '') + ts).encode()).decode(), 'Timestamp': ts,
            'TransactionType': E.get('TXN_TYPE') or 'CustomerPayBillOnline', 'Amount': price, 'PartyA': phone, 'PartyB': sc,
            'PhoneNumber': phone, 'CallBackURL': f"{CFG['public']}/api/mpesa/callback/{CFG['secret']}",
            'AccountReference': 'Taskpad', 'TransactionDesc': 'Unlock ' + task_id})
        if r.get('ResponseCode') != '0':
            return err(502, r.get('errorMessage') or r.get('ResponseDescription') or 'M-Pesa rejected the request.')
        with tx() as c:
            c.execute('insert into payments(checkout_id,user_id,task_id,amount,phone) values(?,?,?,?,?)',
                      (r['CheckoutRequestID'], g.uid, task_id, price, phone))
        return jsonify(checkoutId=r['CheckoutRequestID'])
    except Exception as e:
        app.logger.exception(e)
        return err(502, 'Could not reach M-Pesa.')

# Safaricom calls this when the customer approves or cancels. The secret in the URL stops forged calls.
@app.post('/api/mpesa/callback/<k>')
def stk_callback(k):
    ok = jsonify(ResultCode=0, ResultDesc='ok')
    if not eq(k, CFG['secret']): return ok
    cb = ((request.get_json(silent=True) or {}).get('Body') or {}).get('stkCallback')
    if not cb: return ok
    with tx() as c:
        p = c.execute('select * from payments where checkout_id=?', (cb.get('CheckoutRequestID'),)).fetchone()
        if p and p['status'] == 'pending':
            it = {i.get('Name'): i.get('Value') for i in ((cb.get('CallbackMetadata') or {}).get('Item') or [])}
            if cb.get('ResultCode') == 0 and float(it.get('Amount') or 0) >= p['amount']:
                mark_paid(c, p, str(it.get('MpesaReceiptNumber') or ''))
            else:
                c.execute("update payments set status='failed' where checkout_id=?", (p['checkout_id'],))
    return ok

def settle(p):
    """If Safaricom's callback is lost, ask Safaricom directly so a customer who paid is never locked out."""
    age = (datetime.now(timezone.utc) - datetime.strptime(p['created_at'], '%Y-%m-%d %H:%M:%S').replace(tzinfo=timezone.utc)).total_seconds()
    if p['status'] != 'pending' or age < 15: return p['status']
    try:
        ts, sc = stamp(), E.get('SHORTCODE', '')
        r = daraja('/mpesa/stkpushquery/v1/query', {
            'BusinessShortCode': sc, 'Password': base64.b64encode((sc + E.get('PASSKEY', '') + ts).encode()).decode(),
            'Timestamp': ts, 'CheckoutRequestID': p['checkout_id']})
        if 'ResultCode' not in r: return 'pending'  # still being processed
        with tx() as c:
            if str(r['ResultCode']) == '0':
                mark_paid(c, p); return 'paid'
            c.execute("update payments set status='failed' where checkout_id=? and status='pending'", (p['checkout_id'],))
        return 'failed'
    except Exception:
        return 'pending'

@app.get('/api/payment-status')
@auth
def payment_status():
    r = rows('select * from payments where checkout_id=? and user_id=?', request.args.get('id', ''), g.uid)
    return jsonify(status=settle(r[0]) if r else 'failed')

# Complete a task: one reward per task per user
@app.post('/api/tasks/<task_id>/complete')
@auth
def complete(task_id):
    if task_id not in PRICES: return err(404, 'Unknown task.')
    if PRICES[task_id] > 0 and not val('select 1 from unlocks where user_id=? and task_id=?', g.uid, task_id):
        return err(402, 'Unlock this task first.')
    answers = json.dumps((request.get_json(silent=True) or {}).get('answers') or [])[:5000]
    try:
        with tx() as c:
            c.execute('insert into completions(user_id,task_id,answers) values(?,?,?)', (g.uid, task_id, answers))
            add_ledger(c, g.uid, 'task_reward', CFG['reward'], task_id)
    except sqlite3.IntegrityError:
        return err(409, 'You already completed this task.')
    return jsonify(me(g.uid))

class Fail(Exception): pass

# Withdraw: only once balance >= MIN_WITHDRAW. Debited first, refunded automatically if the payout fails.
@app.post('/api/withdraw')
@auth
def withdraw():
    try: amt = int(float((request.get_json(silent=True) or {}).get('amount') or 0))
    except (TypeError, ValueError): amt = 0
    phone = val('select phone from users where id=?', g.uid)
    try:
        with tx() as c:
            bal = c.execute('select coalesce(sum(amount),0) v from ledger where user_id=?', (g.uid,)).fetchone()['v']
            if bal < CFG['min_w']: raise Fail('min')
            if amt < CFG['min_w'] or amt > bal: raise Fail('amt')
            wid = c.execute('insert into withdrawals(user_id,amount,phone) values(?,?,?)', (g.uid, amt, phone)).lastrowid
            add_ledger(c, g.uid, 'withdrawal', -amt, f'w{wid}')
    except Fail as e:
        return err(400, f"You can withdraw once your balance reaches KES {CFG['min_w']}." if str(e) == 'min'
                   else 'Enter an amount from the minimum up to your balance.')
    try:
        r = daraja('/mpesa/b2c/v1/paymentrequest', {
            'InitiatorName': E.get('B2C_INITIATOR'), 'SecurityCredential': E.get('B2C_CREDENTIAL'), 'CommandID': 'BusinessPayment',
            'Amount': amt, 'PartyA': E.get('B2C_SHORTCODE'), 'PartyB': phone, 'Remarks': 'Taskpad withdrawal', 'Occasion': f'w{wid}',
            'QueueTimeOutURL': f"{CFG['public']}/api/mpesa/b2c/timeout/{CFG['secret']}",
            'ResultURL': f"{CFG['public']}/api/mpesa/b2c/result/{CFG['secret']}"})
        if r.get('ResponseCode') != '0': raise RuntimeError('rejected')
        with tx() as c:
            c.execute('update withdrawals set conv_id=? where id=?', (r.get('OriginatorConversationID'), wid))
        return jsonify(status='pending', **me(g.uid))
    except Exception as e:
        app.logger.exception(e)
        refund(wid)
        return err(502, 'Withdrawal could not be sent. Your balance was restored.')

@app.post('/api/mpesa/b2c/result/<k>')
def b2c_result(k):
    ok = jsonify(ResultCode=0, ResultDesc='ok')
    r = (request.get_json(silent=True) or {}).get('Result')
    if not eq(k, CFG['secret']) or not r: return ok
    w = rows('select * from withdrawals where conv_id=?', r.get('OriginatorConversationID'))
    if w and w[0]['status'] == 'pending':
        if r.get('ResultCode') == 0:
            with tx() as c: c.execute("update withdrawals set status='paid' where id=?", (w[0]['id'],))
        else:
            refund(w[0]['id'])
    return ok

# On timeout the withdrawal stays 'pending' so you can reconcile it by hand before refunding.
@app.post('/api/mpesa/b2c/timeout/<k>')
def b2c_timeout(k): return jsonify(ResultCode=0)

# ---------------- admin ----------------
@app.get('/api/admin/summary')
@admin
def admin_summary():
    w = rows('select w.id, w.amount, w.status, u.name from withdrawals w join users u on u.id=w.user_id order by w.id desc limit 30')
    return jsonify(**money(), minW=CFG['min_w'], withdrawals=[dict(r) for r in w])

@app.post('/api/admin/reserve')
@admin
def admin_reserve():
    b = request.get_json(silent=True) or {}
    try: amt = int(float(b.get('amount') or 0))
    except (TypeError, ValueError): amt = 0
    if amt <= 0: return err(400, 'Enter an amount above 0.')
    with tx() as c:
        c.execute('insert into reserve(amount,note) values(?,?)', (amt, str(b.get('note') or '')[:100]))
    return jsonify(money())

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=CFG['port'])
````

## `taskpad-python/tasks.json`

````json
[{"id":"t1","title":"Mobile money habits","cat":"Money & mobile","type":"Survey","mins":3,"price":0,"qs":[{"q":"How often do you use mobile money?","t":"choice","o":["Daily","Weekly","Monthly","Rarely"]},{"q":"What would make it better for you?","t":"text"},{"q":"How satisfied are you with it today?","t":"rate"}]},{"id":"t2","title":"Saving habits","cat":"Money & mobile","type":"Survey","mins":3,"price":30,"qs":[{"q":"Where do you keep most of your savings?","t":"choice","o":["Mobile wallet","Bank","Sacco or chama","At home"]},{"q":"What is your main savings goal?","t":"text"},{"q":"How easy is it to save each month?","t":"rate"}]},{"id":"t3","title":"Paying bills","cat":"Money & mobile","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How do you pay most bills?","t":"choice","o":["M-Pesa","Bank transfer","Cash","Cards"]},{"q":"Which bill is hardest to manage?","t":"text"},{"q":"How stressful is paying bills?","t":"rate"}]},{"id":"t4","title":"Loan apps","cat":"Money & mobile","type":"Survey","mins":3,"price":30,"qs":[{"q":"Have you used a mobile loan app?","t":"choice","o":["Often","Once or twice","Never","Not sure"]},{"q":"What do you wish loan apps explained better?","t":"text"},{"q":"How much do you trust them?","t":"rate"}]},{"id":"t5","title":"Side income","cat":"Money & mobile","type":"Survey","mins":3,"price":30,"qs":[{"q":"Do you earn from a side hustle?","t":"choice","o":["Yes, regularly","Sometimes","Not now","Planning to"]},{"q":"What side hustle would you start?","t":"text"},{"q":"How easy is it to find extra work?","t":"rate"}]},{"id":"t6","title":"Budgeting","cat":"Money & mobile","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How do you track spending?","t":"choice","o":["App","Notebook","In my head","I don't"]},{"q":"What do you spend most on?","t":"text"},{"q":"How in control of your money do you feel?","t":"rate"}]},{"id":"t7","title":"Online shopping","cat":"Shopping & brands","type":"Survey","mins":3,"price":0,"qs":[{"q":"How often do you shop online?","t":"choice","o":["Weekly","Monthly","Rarely","Never"]},{"q":"What stops you from buying online?","t":"text"},{"q":"How much do you trust online shops?","t":"rate"}]},{"id":"t8","title":"Name this product","cat":"Shopping & brands","type":"Quick task","mins":2,"price":30,"qs":[{"q":"Which product would you buy?","t":"choice","o":["Cold water bottle","Solar lantern","Phone stand","Lunch box"]},{"q":"Suggest a short name for the one you chose.","t":"text"},{"q":"How likely are you to buy it?","t":"rate"}]},{"id":"t9","title":"Delivery experience","cat":"Shopping & brands","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How long will you wait for a delivery?","t":"choice","o":["Same day","1-2 days","3-5 days","A week"]},{"q":"What went wrong with your last delivery?","t":"text"},{"q":"How happy are you with delivery services?","t":"rate"}]},{"id":"t10","title":"Groceries","cat":"Shopping & brands","type":"Survey","mins":3,"price":30,"qs":[{"q":"Where do you buy groceries?","t":"choice","o":["Supermarket","Open market","Kiosk","Online"]},{"q":"What makes you pick a shop?","t":"text"},{"q":"How fair do you find food prices?","t":"rate"}]},{"id":"t11","title":"Ad feedback","cat":"Shopping & brands","type":"Quick task","mins":2,"price":30,"qs":[{"q":"Where do you notice ads most?","t":"choice","o":["Social media","Radio","TV","Billboards"]},{"q":"Describe an ad you remember.","t":"text"},{"q":"How much do ads influence you?","t":"rate"}]},{"id":"t12","title":"Packaging","cat":"Shopping & brands","type":"Quick task","mins":2,"price":30,"qs":[{"q":"What matters most in packaging?","t":"choice","o":["Looks","Price","Recycling","Easy to open"]},{"q":"Which package annoys you most?","t":"text"},{"q":"How important is eco-friendly packaging?","t":"rate"}]},{"id":"t13","title":"Morning routine","cat":"Everyday life","type":"Quick task","mins":2,"price":0,"qs":[{"q":"What time do you wake up?","t":"choice","o":["Before 5am","5-7am","7-9am","After 9am"]},{"q":"What do you do first each morning?","t":"text"},{"q":"How rested do you feel?","t":"rate"}]},{"id":"t14","title":"Commute","cat":"Everyday life","type":"Survey","mins":3,"price":30,"qs":[{"q":"How do you get to work or school?","t":"choice","o":["Matatu","Boda boda","Walk","Own vehicle"]},{"q":"What would improve your commute?","t":"text"},{"q":"How tiring is your commute?","t":"rate"}]},{"id":"t15","title":"Food and meals","cat":"Everyday life","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How many meals do you eat a day?","t":"choice","o":["1","2","3","4 or more"]},{"q":"What is your favourite meal?","t":"text"},{"q":"How healthy is your diet?","t":"rate"}]},{"id":"t16","title":"Sleep","cat":"Everyday life","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How many hours do you sleep?","t":"choice","o":["Under 5","5-6","7-8","Over 8"]},{"q":"What keeps you up at night?","t":"text"},{"q":"How well do you sleep?","t":"rate"}]},{"id":"t17","title":"Free time","cat":"Everyday life","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How do you spend free time?","t":"choice","o":["Phone","Sports","Friends","Rest"]},{"q":"What hobby would you like to try?","t":"text"},{"q":"How much free time do you get?","t":"rate"}]},{"id":"t18","title":"Community","cat":"Everyday life","type":"Survey","mins":3,"price":30,"qs":[{"q":"How often do you join community events?","t":"choice","o":["Often","Sometimes","Rarely","Never"]},{"q":"What would improve your area?","t":"text"},{"q":"How safe do you feel in your area?","t":"rate"}]},{"id":"t19","title":"Favourite apps","cat":"Tech & apps","type":"Survey","mins":3,"price":0,"qs":[{"q":"Which apps do you use most?","t":"choice","o":["Social","Finance","Shopping","Learning"]},{"q":"What frustrates you in apps?","t":"text"},{"q":"How likely are you to recommend your favourite app?","t":"rate"}]},{"id":"t20","title":"Website first impressions","cat":"Tech & apps","type":"Survey","mins":3,"price":30,"qs":[{"q":"What do you check first on a website?","t":"choice","o":["Price","Reviews","Contact","Design"]},{"q":"What would you change on a site you used today?","t":"text"},{"q":"How easy are most sites to use?","t":"rate"}]},{"id":"t21","title":"Internet access","cat":"Tech & apps","type":"Quick task","mins":2,"price":30,"qs":[{"q":"How do you get online?","t":"choice","o":["Mobile data","Home Wi-Fi","Office or school","Cyber cafe"]},{"q":"What makes internet access hard?","t":"text"},{"q":"How reliable is your connection?","t":"rate"}]},{"id":"t22","title":"Privacy","cat":"Tech & apps","type":"Survey","mins":3,"price":30,"qs":[{"q":"Do you read privacy policies?","t":"choice","o":["Always","Sometimes","Rarely","Never"]},{"q":"What worries you about sharing data?","t":"text"},{"q":"How safe do you feel online?","t":"rate"}]},{"id":"t23","title":"Phone features","cat":"Tech & apps","type":"Quick task","mins":2,"price":30,"qs":[{"q":"Which phone feature matters most?","t":"choice","o":["Battery","Camera","Storage","Price"]},{"q":"What phone problem do you face most?","t":"text"},{"q":"How happy are you with your phone?","t":"rate"}]},{"id":"t24","title":"Learning online","cat":"Tech & apps","type":"Survey","mins":3,"price":30,"qs":[{"q":"Have you learned a skill online?","t":"choice","o":["Yes","Trying now","Not yet","No time"]},{"q":"What skill would you learn next?","t":"text"},{"q":"How useful is online learning?","t":"rate"}]}]
````

## `taskpad-python/public/index.html`

````html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Taskpad</title>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
<style>
:root{--bg:#eef3ee;--card:#fff;--ink:#12261b;--muted:#5f7468;--line:#dbe5dc;--g1:#0e3a24;--g2:#1c6a40;--lime:#c9f47f;--lime2:#a6e85b;--brand:#2a7a4c;--bad:#c0322b;color-scheme:light;box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
html{scroll-padding-top:env(safe-area-inset-top,0px)}*{box-sizing:border-box}[hidden]{display:none!important}
body{margin:0;background:var(--bg);color:var(--ink);font:16px/1.5 Inter,system-ui,-apple-system,"Segoe UI",sans-serif}
.top{position:sticky;top:env(safe-area-inset-top,0px);z-index:4;background:rgba(255,255,255,.88);backdrop-filter:blur(10px);border-bottom:1px solid var(--line);padding:10px 16px}
.topin{display:flex;align-items:center;gap:8px;max-width:960px;margin:0 auto}.grow{flex:1}
.logo{width:44px;height:44px;border-radius:14px;background:linear-gradient(145deg,var(--g2),var(--g1));color:var(--lime);display:grid;place-items:center;font-weight:800;font-size:1.2rem;box-shadow:0 6px 16px rgba(14,58,36,.3);flex:none}
.brand b{display:block;line-height:1.1;font-weight:800;font-size:1.05rem}.brand small{color:var(--muted);font-size:.72rem}
.pill{display:inline-flex;align-items:center;gap:8px;padding:8px 14px;border-radius:99px;border:1px solid var(--line);background:#fff;font:inherit;font-size:.82rem;font-weight:600;color:var(--ink);white-space:nowrap}
button.pill{cursor:pointer}.pill.ok{background:#eaf7ee;color:var(--brand);border-color:#cfe9d7}
.dot{width:8px;height:8px;border-radius:50%;background:#2fbf71}.av{width:26px;height:26px;border-radius:50%;background:linear-gradient(145deg,var(--lime),var(--lime2));display:grid;place-items:center;font-size:.75rem;font-weight:800;color:var(--g1)}
@media(max-width:720px){.hide-s{display:none}}@media(max-width:560px){.brand small{display:none}}
.wrap{max-width:960px;margin:0 auto;padding:26px 16px 130px}
.eyebrow{font-size:.72rem;font-weight:800;letter-spacing:.14em;text-transform:uppercase;color:var(--brand)}.eyebrow.lt{color:rgba(255,255,255,.72)}
h1{font-size:clamp(2.2rem,7vw,3.5rem);line-height:1.02;letter-spacing:-.04em;font-weight:800;margin:8px 0 10px}
h2{font-size:1.9rem;letter-spacing:-.03em;font-weight:800;margin:8px 0 14px}h3{margin:6px 0 8px;font-size:1.15rem;font-weight:800;letter-spacing:-.02em}
.sub{color:var(--muted);margin:0 0 22px;max-width:52ch}.grid{display:grid;gap:16px}@media(min-width:720px){.grid{grid-template-columns:1fr 1fr}}
.card{background:var(--card);border-radius:28px;padding:22px;box-shadow:0 14px 40px rgba(20,50,30,.08)}.card p{color:var(--muted);margin:4px 0}
.track{height:8px;background:#e3ece4;border-radius:99px;overflow:hidden;margin:12px 0 6px}.track i{display:block;height:100%;background:linear-gradient(90deg,var(--g2),var(--lime2));border-radius:99px;transition:width .4s}
.bcard{background:linear-gradient(160deg,var(--g2),var(--g1));color:#fff;border-radius:28px;padding:22px;box-shadow:0 18px 44px rgba(14,58,36,.35)}
.bhead{display:flex;justify-content:space-between;align-items:center;gap:8px}.chip{font-size:.72rem;font-weight:700;padding:5px 10px;border-radius:99px;background:rgba(255,255,255,.14);border:1px solid rgba(255,255,255,.25)}
.big{font-size:clamp(2.6rem,10vw,3.6rem);font-weight:800;letter-spacing:-.04em;line-height:1.05;margin:14px 0 6px}.bsub{margin:0;opacity:.8}
.mini{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin:18px 0}.mini div{background:rgba(255,255,255,.1);border:1px solid rgba(255,255,255,.18);border-radius:18px;padding:12px}
.mini small{display:block;opacity:.75;font-size:.75rem}.mini b{color:var(--lime);font-size:1.25rem}
.lime{width:100%;padding:15px;border:0;border-radius:16px;background:linear-gradient(180deg,var(--lime),var(--lime2));color:var(--g1);font:inherit;font-weight:800;cursor:pointer}
.list{display:grid;gap:10px}.cat{margin:18px 0 2px}
.task{display:flex;align-items:center;gap:14px;background:var(--card);border:1px solid var(--line);border-radius:20px;padding:16px;text-align:left;width:100%;color:inherit;font:inherit;cursor:pointer}
.task:hover{border-color:var(--g2)}.task .t{flex:1;min-width:0}.task b{display:block;font-weight:700}.task small{color:var(--muted)}
.tag{font-size:.8rem;font-weight:700;padding:5px 11px;border-radius:99px;white-space:nowrap}.tag.free{background:#eaf7ee;color:var(--brand)}.tag.paid{background:#fff4e0;color:#a35a00}.tag.done{background:var(--g1);color:var(--lime)}.tag.open{background:var(--lime);color:var(--g1)}
.nav{position:fixed;left:50%;transform:translateX(-50%);bottom:calc(14px + env(safe-area-inset-bottom,0px));width:min(92vw,420px);display:flex;gap:6px;padding:8px;border-radius:28px;background:rgba(255,255,255,.82);backdrop-filter:blur(14px);border:1px solid #fff;box-shadow:0 12px 40px rgba(20,50,30,.2);z-index:6}
.nav button{flex:1;display:flex;align-items:center;justify-content:center;gap:8px;padding:12px 8px;border:0;border-radius:20px;background:transparent;color:var(--muted);font:inherit;font-weight:700;font-size:.85rem;cursor:pointer}
.nav button.on{background:linear-gradient(180deg,var(--lime),var(--lime2));color:var(--g1)}.nav svg{width:20px;height:20px;fill:none;stroke:currentColor;stroke-width:1.8;stroke-linecap:round;stroke-linejoin:round}
.task:focus-visible,button:focus-visible,input:focus-visible{outline:3px solid var(--g2);outline-offset:2px}
.ov{position:fixed;inset:0;background:rgba(10,24,16,.55);display:none;align-items:flex-end;justify-content:center;z-index:10}.ov.on{display:flex}
.sheet{background:var(--card);width:100%;max-width:520px;max-height:92%;overflow:auto;border-radius:28px 28px 0 0;padding:24px 20px calc(24px + env(safe-area-inset-bottom,0px));animation:up .22s ease-out}
@keyframes up{from{transform:translateY(30px);opacity:0}to{transform:none;opacity:1}}@media(min-width:600px){.ov{align-items:center}.sheet{border-radius:28px}}
.sheet h2{font-size:1.5rem;margin:0 0 4px}.sheet p{margin:0 0 14px;color:var(--muted)}
label{display:block;font-weight:600;margin:14px 0 6px}
input[type=text],input[type=tel],input[type=password]{width:100%;padding:13px 14px;border:1px solid var(--line);border-radius:14px;background:var(--bg);color:var(--ink);font:inherit}
.opt{display:flex;gap:10px;align-items:center;padding:12px;border:1px solid var(--line);border-radius:14px;margin-bottom:8px;cursor:pointer}.opt:has(input:checked){border-color:var(--g2);background:#eaf7ee}
.rate{display:flex;gap:8px}.rate label{margin:0;flex:1;text-align:center;padding:11px 0;border:1px solid var(--line);border-radius:14px;cursor:pointer;font-weight:700}.rate label:has(input:checked){background:var(--g1);color:var(--lime);border-color:var(--g1)}
.rate input{position:absolute;opacity:0}.opt input{accent-color:var(--g2)}
.btn{width:100%;margin-top:18px;padding:15px;border:0;border-radius:16px;background:var(--g1);color:#fff;font:inherit;font-weight:700;cursor:pointer}.btn.ghost{background:transparent;color:var(--muted);margin-top:6px}
.err{color:var(--bad);min-height:1.3em;margin:8px 0 0;font-size:.92rem}.center{text-align:center}
.pulse{width:72px;height:72px;border-radius:50%;background:var(--g2);margin:8px auto 18px;animation:pl 1.4s ease-out infinite}
@keyframes pl{0%{box-shadow:0 0 0 0 rgba(28,106,64,.5)}100%{box-shadow:0 0 0 34px transparent}}
@media(prefers-reduced-motion:reduce){.pulse,.sheet{animation:none}}
.sm{padding:6px 12px;border:1px solid var(--line);border-radius:99px;background:#fff;font:inherit;font-weight:600;font-size:.8rem;cursor:pointer}
</style>
</head>
<body>
<header class="top" id="top" hidden><div class="topin"><div class="logo">T</div><div class="brand"><b>Taskpad</b><small>Tasks and surveys, paid out</small></div><span class="grow"></span><span class="pill ok hide-s"><i class="dot"></i>Account active</span><span class="pill">Balance: <b id="pbal"></b></span><span class="pill hide-s"><span class="av" id="uav"></span><b id="uname"></b></span><button class="pill" id="lo">Log out</button></div></header>
<main class="wrap">
<section class="view" id="v-landing"><span class="eyebrow">Tasks and surveys</span><h1>Taskpad</h1><p class="sub">Complete short tasks and surveys, watch your balance grow, and withdraw to M-Pesa once you reach the minimum.</p></section>
<section class="view" id="v-home" hidden><span class="eyebrow">Your workspace</span><h1 id="hello"></h1><p class="sub">Your earnings, tasks, and account activity in one place.</p><div class="grid"><div class="card" id="prog"></div><div id="hcard"></div></div></section>
<section class="view" id="v-wallet" hidden><span class="eyebrow">Wallet</span><h1>Your money</h1><p class="sub">Your balance and how to get it to your phone.</p><div class="grid"><div id="wcard"></div><div class="card" id="wrules"></div></div></section>
<section class="view" id="v-jobs" hidden><span class="eyebrow">Tasks and surveys</span><h2>Pick a task</h2><div class="list" id="list"></div></section>
</main>
<nav class="nav" id="nav" hidden><button data-v="home"><svg viewBox="0 0 24 24"><path d="M3 11l9-8 9 8"/><path d="M5 10v10h14V10"/></svg>Home</button><button data-v="wallet"><svg viewBox="0 0 24 24"><rect x="3" y="6" width="18" height="13" rx="3"/><path d="M3 10h18"/><circle cx="16.5" cy="14.5" r="1"/></svg>Wallet</button><button data-v="jobs"><svg viewBox="0 0 24 24"><rect x="3" y="7" width="18" height="13" rx="3"/><path d="M9 7V5h6v2"/><path d="M3 13h18"/></svg>Jobs</button></nav>
<div class="ov" id="ov" role="dialog" aria-modal="true"><div class="sheet" id="sheet"></div></div>
<script>

// ---------- CONFIG ----------
// API_URL points at the Taskpad backend that serves this page.
const CONFIG = { API_URL: "/api", CURRENCY: "Sh.", POLL_MS: 3000, TIMEOUT_MS: 60000 };

let TASKS=[],RULES={signup:200,reward:100,minW:2500};

// ---------- STATE ----------
const TK="taskpad:token";
let S={unlocked:[],done:{}},ME=null,authed=false;
const getTok=()=>{try{return localStorage.getItem(TK)||""}catch(e){return ""}};
const setTok=v=>{try{v?localStorage.setItem(TK,v):localStorage.removeItem(TK)}catch(e){}};
const $=id=>document.getElementById(id);
const esc=s=>String(s).replace(/[&<>"]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;"}[c]));
async function api(path,opts={}){
  const r=await fetch(CONFIG.API_URL+path,{method:opts.body?"POST":"GET",headers:{"Content-Type":"application/json",Authorization:"Bearer "+getTok()},body:opts.body?JSON.stringify(opts.body):undefined});
  const d=await r.json().catch(()=>({}));
  if(r.status===401&&authed)logout();
  if(!r.ok)throw new Error(d.error||"Something went wrong.");
  return d;
}
function setMe(m){ME=m;S.unlocked=m.unlocked;S.done=Object.fromEntries(m.done.map(id=>[id,1]));render()}
function logout(){setTok("");authed=false;ME=null;S={unlocked:[],done:{}};render();openAuth("login")}

// ---------- LIST ----------
const fmt=n=>"Sh."+Number(n).toLocaleString();
let VIEW="home";
function bcard(){
  if(!ME)return"";
  const left=Math.max(0,ME.minWithdraw-ME.balance);
  return '<div class="bcard"><div class="bhead"><span class="eyebrow lt">Available balance</span><span class="chip">M-Pesa payouts</span></div><div class="big">'+fmt(ME.balance)+'</div><p class="bsub">'+(ME.canWithdraw?'You can withdraw now.':fmt(left)+' more to reach the '+fmt(ME.minWithdraw)+' minimum.')+'</p><div class="mini"><div><small>Amount earned</small><b>'+fmt(ME.earned)+'</b></div><div><small>Amount withdrawn</small><b>'+fmt(ME.withdrawn)+'</b></div></div><button class="lime" data-act="withdraw">Withdraw</button></div>';
}
function render(){
  const n=Object.keys(S.done).length,T=TASKS.length,fee=Math.max(0,...TASKS.map(t=>t.price));
  $("top").hidden=!ME;$("nav").hidden=!ME;
  const vid=ME?"v-"+VIEW:"v-landing";
  document.querySelectorAll(".view").forEach(v=>v.hidden=v.id!==vid);
  document.querySelectorAll("#nav button").forEach(b=>b.classList.toggle("on",b.dataset.v===VIEW));
  if(ME){
    $("pbal").textContent=fmt(ME.balance);$("uname").textContent=ME.name.split(" ")[0];$("uav").textContent=ME.name.charAt(0).toUpperCase();
    $("hello").textContent="Welcome back, "+ME.name.split(" ")[0];
    $("prog").innerHTML='<span class="eyebrow">Your progress</span><h3>'+n+' of '+T+' tasks done</h3><div class="track"><i style="width:'+(T?n/T*100:0)+'%"></i></div><p>Each completed task pays '+fmt(ME.reward)+'.</p><div class="track"><i style="width:'+Math.min(100,ME.balance/ME.minWithdraw*100)+'%"></i></div><p>'+fmt(ME.balance)+' of '+fmt(ME.minWithdraw)+' toward your first withdrawal.</p>';
    $("hcard").innerHTML=bcard();$("wcard").innerHTML=bcard();
    $("wrules").innerHTML='<span class="eyebrow">Good to know</span><h3>How withdrawals work</h3><p>Money is sent to the M-Pesa number you registered with. Withdrawals open at '+fmt(ME.minWithdraw)+', and every completed task adds '+fmt(ME.reward)+'. Paid tasks cost '+fmt(fee)+' each to unlock.</p>';
  }
  let last="";
  $("list").innerHTML=TASKS.map(t=>{
    const done=S.done[t.id],open=t.price===0||S.unlocked.includes(t.id);
    const tag=done?'<span class="tag done">Done</span>':open?'<span class="tag '+(t.price?'open':'free')+'">'+(t.price?'Unlocked':'Free')+'</span>':'<span class="tag paid">'+fmt(t.price)+' to unlock</span>';
    const h3=t.cat!==last?(last=t.cat,'<h3 class="cat">'+esc(t.cat)+'</h3>'):'';
    return h3+'<button class="task" data-id="'+t.id+'"><div class="t"><b>'+esc(t.title)+'</b><small>'+t.type+' · '+t.mins+' min</small></div>'+tag+'</button>';
  }).join("");
}
$("nav").onclick=e=>{const b=e.target.closest("button");if(b){VIEW=b.dataset.v;render();scrollTo(0,0)}};
document.addEventListener("click",e=>{if(e.target.closest("[data-act=withdraw]"))openWithdraw()});
$("list").addEventListener("click",e=>{
  const b=e.target.closest(".task");if(!b)return;
  if(!authed)return openAuth("login");
  const t=TASKS.find(x=>x.id===b.dataset.id);
  if(S.done[t.id])return showDone(t);
  (t.price===0||S.unlocked.includes(t.id))?openTask(t):openPay(t);
});
const show=h=>{$("sheet").innerHTML=h;$("ov").classList.add("on")};
const hide=()=>{$("ov").classList.remove("on");active=null};
$("ov").addEventListener("click",e=>{if(e.target.id==="ov"&&!paying&&authed)hide()});
document.addEventListener("keydown",e=>{if(e.key==="Escape"&&!paying&&authed)hide()});

// ---------- TASK RUNNER ----------
function openTask(t){
  const body=t.qs.map((q,i)=>{
    let f="";
    if(q.t==="choice")f=q.o.map(o=>'<label class="opt"><input type="radio" name="q'+i+'" value="'+esc(o)+'"> '+esc(o)+'</label>').join("");
    else if(q.t==="rate")f='<div class="rate">'+[1,2,3,4,5].map(n=>'<label><input type="radio" name="q'+i+'" value="'+n+'">'+n+'</label>').join("")+'</div>';
    else f='<input type="text" name="q'+i+'" maxlength="200" autocomplete="off">';
    return '<label>'+esc(q.q)+'</label>'+f;
  }).join("");
  show('<h2>'+esc(t.title)+'</h2><p>'+t.type+' · about '+t.mins+' min</p><form id="f" onsubmit="return false">'+body+'<p class="err" id="err"></p><button class="btn" id="go">Submit answers</button><button class="btn ghost" type="button" onclick="hide()">Close</button></form>');
  $("go").onclick=()=>{
    const ans=[];
    for(let i=0;i<t.qs.length;i++){
      const el=document.querySelector('[name=q'+i+']:checked')||document.querySelector('input[type=text][name=q'+i+']');
      const v=el?el.value.trim():"";
      if(!v){$("err").textContent="Answer every question to submit.";return}
      ans.push(v);
    }
    $("go").disabled=true;
    api("/tasks/"+t.id+"/complete",{body:{answers:ans}}).then(r=>{setMe(r);
      show('<div class="center"><h2>Task complete</h2><p>You earned '+CONFIG.CURRENCY+' '+r.reward+'.</p><button class="btn" onclick="hide()">Back to tasks</button></div>')}).catch(e=>{$("go").disabled=false;$("err").textContent=e.message});
  };
}
function showDone(t){show('<div class="center"><h2>Already done</h2><p>You finished "'+esc(t.title)+'".</p><button class="btn" onclick="hide()">Back to tasks</button></div>')}

// ---------- PAYMENT ----------
let paying=false,active=null;
function normPhone(v){
  let d=String(v).replace(/\D/g,"");
  if(/^0[17]\d{8}$/.test(d))d="254"+d.slice(1);
  else if(/^[17]\d{8}$/.test(d))d="254"+d;
  return /^254[17]\d{8}$/.test(d)?d:null;
}
function openPay(t){
  active=t;
  show('<h2>Unlock "'+esc(t.title)+'"</h2><p>Pay '+CONFIG.CURRENCY+' '+t.price+' once to open this task.</p><label for="ph">M-Pesa phone number</label><input id="ph" type="tel" inputmode="tel" placeholder="0712 345 678" autocomplete="tel"><p class="err" id="err"></p><button class="btn" id="pay">Send payment request</button><button class="btn ghost" type="button" onclick="hide()">Cancel</button>');
  $("pay").onclick=()=>{
    const p=normPhone($("ph").value);
    if(!p){$("err").textContent="Enter a valid Safaricom number, like 0712 345 678.";return}
    startPay(t,p);
  };
}
async function startPay(t,phone){
  paying=true;
  show('<div class="center"><div class="pulse"></div><h2>Check your phone</h2><p>We sent a payment request for '+CONFIG.CURRENCY+' '+t.price+' to +'+phone+'. Enter your M-Pesa PIN to approve it.</p><p class="err" id="err"></p></div>');
  try{
    const res=await initiatePayment(phone,t);
    paying=false;
    if(res==="paid"){
      api("/me").then(setMe).catch(()=>{S.unlocked.push(t.id);render()});
      show('<div class="center"><h2>Payment received</h2><p>"'+esc(t.title)+'" is unlocked.</p><button class="btn" id="st">Start task</button></div>');
      $("st").onclick=()=>openTask(t);
    }else{
      const msg=res==="timeout"?"We didn't get an approval in time.":"The payment was cancelled or failed.";
      show('<div class="center"><h2>Not paid</h2><p>'+msg+' You were not charged.</p><button class="btn" id="re">Try again</button><button class="btn ghost" onclick="hide()">Close</button></div>');
      $("re").onclick=()=>openPay(t);
    }
  }catch(e){
    paying=false;
    show('<div class="center"><h2>Couldn\'t send request</h2><p>'+esc(e.message)+'</p><button class="btn" id="re">Try again</button><button class="btn ghost" onclick="hide()">Close</button></div>');
    $("re").onclick=()=>openPay(t);
  }
}
// Returns "paid" | "failed" | "timeout"
async function initiatePayment(phone,t){
  const {checkoutId}=await api("/stk-push",{body:{phone,taskId:t.id}});
  const end=Date.now()+CONFIG.TIMEOUT_MS;
  while(Date.now()<end){
    await new Promise(r=>setTimeout(r,CONFIG.POLL_MS));
    const s=await api("/payment-status?id="+encodeURIComponent(checkoutId));
    if(s.status==="paid")return "paid";
    if(s.status==="failed")return "failed";
  }
  return "timeout";
}

// ---------- ACCOUNT ----------
function openAuth(mode){
  const reg=mode==="register",C=CONFIG.CURRENCY;
  if(reg&&!RULES.open)return show('<h2>Sign-ups are closed</h2><p>This pilot is full for now. Check back later.</p><button class="btn ghost" id="sw">I already have an account</button>'),$("sw").onclick=()=>openAuth("login");
  show('<h2>'+(reg?"Create your account":"Log in")+'</h2><p>'+(reg?"Get "+C+" "+RULES.signup+" when you join, then "+C+" "+RULES.reward+" for each task you complete. You can withdraw once your balance reaches "+C+" "+RULES.minW+" ("+Math.ceil((RULES.minW-RULES.signup)/RULES.reward)+" tasks). Paid tasks cost "+C+" "+Math.max(0,...TASKS.map(t=>t.price))+" each to unlock.":"Welcome back.")+'</p>'+(reg?'<label for="nm">Name</label><input id="nm" type="text" autocomplete="name">'+(RULES.invite?'<label for="iv">Invite code</label><input id="iv" type="text" autocomplete="off">':''):'')+'<label for="ph">Phone number</label><input id="ph" type="tel" inputmode="tel" placeholder="0712 345 678" autocomplete="tel"><label for="pw">Password</label><input id="pw" type="password" placeholder="At least 8 characters" autocomplete="'+(reg?"new-password":"current-password")+'"><p class="err" id="err"></p><button class="btn" id="ok">'+(reg?"Create account":"Log in")+'</button><button class="btn ghost" id="sw">'+(reg?"I already have an account":"Create an account")+'</button>');
  $("sw").onclick=()=>openAuth(reg?"login":"register");
  $("ok").onclick=async()=>{
    $("ok").disabled=true;$("err").textContent="";
    try{
      const d=await api(reg?"/register":"/login",{body:{name:reg?$("nm").value:undefined,invite:reg&&RULES.invite?$("iv").value:undefined,phone:$("ph").value,password:$("pw").value}});
      setTok(d.token);authed=true;setMe(d);
      if(reg)show('<div class="center"><h2>Welcome, '+esc(d.name)+'</h2><p>'+C+' '+d.balance+' is in your account. You can withdraw once your balance reaches '+C+' '+d.minWithdraw+'.</p><button class="btn" onclick="hide()">Start earning</button></div>');else hide();
    }catch(e){$("ok").disabled=false;$("err").textContent=e.message}
  };
}
function openWithdraw(){
  if(!ME)return;const C=CONFIG.CURRENCY;
  if(!ME.canWithdraw)return show('<h2>Not yet</h2><p>You can withdraw once your balance reaches '+C+' '+ME.minWithdraw+'. You have '+C+' '+ME.balance+'. Keep completing tasks: about '+Math.ceil((ME.minWithdraw-ME.balance)/ME.reward)+' more to go.</p><button class="btn" onclick="hide()">Back to tasks</button>');
  show('<h2>Withdraw to M-Pesa</h2><p>Available: '+C+' '+ME.balance+'. Minimum: '+C+' '+ME.minWithdraw+'. Money is sent to the number you registered with.</p><label for="am">Amount</label><input id="am" type="tel" inputmode="numeric" value="'+ME.balance+'"><p class="err" id="err"></p><button class="btn" id="go2">Withdraw</button><button class="btn ghost" onclick="hide()">Cancel</button>');
  $("go2").onclick=async()=>{
    $("go2").disabled=true;
    try{const d=await api("/withdraw",{body:{amount:$("am").value}});setMe(d);
      show('<div class="center"><h2>Withdrawal sent</h2><p>Your M-Pesa payment is on its way. Your balance is now '+C+' '+d.balance+'.</p><button class="btn" onclick="hide()">Done</button></div>');
    }catch(e){$("go2").disabled=false;$("err").textContent=e.message}
  };
}
$("lo").onclick=logout;
(async()=>{try{const d=await api("/tasks");TASKS=d.tasks;RULES=d.rules}catch(e){}render();if(getTok()){try{authed=true;setMe(await api("/me"));return}catch(e){authed=false}}openAuth("register")})();
</script>
</body>
</html>
````

## `taskpad-python/public/admin.html`

````html
<!DOCTYPE html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Taskpad admin</title>
<style>:root{color-scheme:light dark;font:16px/1.5 system-ui,sans-serif}body{max-width:640px;margin:0 auto;padding:24px 16px}h1{margin:0 0 16px}dl{display:grid;grid-template-columns:1fr auto;gap:6px 16px;margin:0 0 20px}dt{opacity:.7}dd{margin:0;font-weight:700;text-align:right}input,button{font:inherit;padding:10px 12px;border-radius:8px;border:1px solid GrayText}button{cursor:pointer}.row{display:flex;gap:8px;margin:8px 0}.row input{flex:1;min-width:0}.bad{color:#c0322b}table{width:100%;border-collapse:collapse;font-size:.9rem}td,th{text-align:left;padding:4px 6px;border-bottom:1px solid GrayText}[hidden]{display:none!important}</style></head><body>
<h1>Taskpad admin</h1>
<div class="row"><input id="tk" type="password" placeholder="Admin token" autocomplete="off"><button id="in">Open</button></div>
<p id="msg" class="bad"></p>
<div id="main" hidden>
<dl id="d"></dl>
<h2>Add to reserve</h2>
<p>Record money you have set aside in your payout account. Sign-ups close automatically when the reserve cannot cover every shilling members could still earn.</p>
<div class="row"><input id="am" type="number" min="1" placeholder="Amount (KES)"><input id="nt" placeholder="Note (optional)"><button id="ad">Add</button></div>
<h2>Recent withdrawals</h2><table id="w"></table>
</div>
<script>
const $=id=>document.getElementById(id);let T=sessionStorage.getItem('at')||'';
const esc=s=>String(s).replace(/[&<>"]/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
const K=n=>'KES '+Number(n).toLocaleString();
async function call(p,b){const r=await fetch('/api/admin/'+p,{method:b?'POST':'GET',headers:{'Content-Type':'application/json','x-admin-token':T},body:b?JSON.stringify(b):undefined});const d=await r.json();if(!r.ok)throw new Error(d.error||'Failed');return d}
function show(d){$('main').hidden=false;$('msg').textContent='';
 const rows=[['Reserve available',K(d.available)],['Owed to members now',K(d.balances+d.pending)],['Still earnable by current members',K(d.potential)],['Total commitments',K(d.exposure)],['Headroom (reserve minus commitments)',K(d.headroom)],['Members',d.members+' of '+d.maxMembers],['More members the reserve can cover',d.room],['Sign-ups',d.open?'Open':'Closed']];
 $('d').innerHTML=rows.map(r=>'<dt>'+r[0]+'</dt><dd>'+r[1]+'</dd>').join('');
 $('w').innerHTML='<tr><th>#</th><th>Member</th><th>Amount</th><th>Status</th></tr>'+(d.withdrawals||[]).map(w=>'<tr><td>'+w.id+'</td><td>'+esc(w.name)+'</td><td>'+K(w.amount)+'</td><td>'+esc(w.status)+'</td></tr>').join('')}
async function load(){try{show(await call('summary'))}catch(e){$('main').hidden=true;$('msg').textContent=e.message}}
$('in').onclick=()=>{T=$('tk').value;sessionStorage.setItem('at',T);load()};
$('ad').onclick=async()=>{try{await call('reserve',{amount:$('am').value,note:$('nt').value});$('am').value='';$('nt').value='';load()}catch(e){$('msg').textContent=e.message}};
if(T)load();
</script></body></html>
````

## `taskpad-python/deploy/setup.sh`

````bash
#!/usr/bin/env bash
# One-shot setup for a fresh Ubuntu 22.04/24.04 server.
# 1) Point your domain's A record at this server's IP first (Caddy needs it to issue the HTTPS certificate).
# 2) Copy the taskpad-backend folder here, including a filled-in .env.
# 3) Run:  sudo bash deploy/setup.sh yourdomain.com
set -euo pipefail
DOMAIN=${1:?Usage: sudo bash deploy/setup.sh yourdomain.com}
SRC=$(cd "$(dirname "$0")/.." && pwd)
APP=/opt/taskpad; DATA=/var/lib/taskpad; BAK=/var/backups/taskpad
[ -f "$SRC/.env" ] || { echo "Missing $SRC/.env (copy .env.example and fill it in)"; exit 1; }
grep -q "^PUBLIC_URL=https://$DOMAIN" "$SRC/.env" || echo "WARNING: PUBLIC_URL in .env should be https://$DOMAIN"

apt-get update -y
apt-get install -y curl ufw rsync sqlite3 build-essential python3 python3-venv python3-pip debian-keyring debian-archive-keyring apt-transport-https gnupg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor --yes -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' > /etc/apt/sources.list.d/caddy-stable.list
apt-get update -y && apt-get install -y caddy

id taskpad >/dev/null 2>&1 || useradd --system --home "$APP" --shell /usr/sbin/nologin taskpad
mkdir -p "$APP" "$DATA" "$BAK"
rsync -a --delete --exclude venv --exclude __pycache__ --exclude node_modules --exclude '*.db*' --exclude deploy "$SRC"/ "$APP"/
python3 -m venv "$APP/venv" && "$APP/venv/bin/pip" install --quiet -r "$APP/requirements.txt"
chown -R taskpad:taskpad "$APP" "$DATA"; chmod 600 "$APP/.env"

cat > /etc/systemd/system/taskpad.service <<UNIT
[Unit]
Description=Taskpad
After=network.target
[Service]
User=taskpad
WorkingDirectory=$APP
Environment=DB_FILE=$DATA/taskpad.db
ExecStart=$APP/venv/bin/gunicorn -w 1 --threads 8 -b 127.0.0.1:3000 app:app
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
UNIT
cat > /etc/caddy/Caddyfile <<CADDY
$DOMAIN {
    reverse_proxy localhost:3000
}
CADDY
cat > /etc/cron.d/taskpad-backup <<CRON
15 2 * * * root sqlite3 $DATA/taskpad.db ".backup '$BAK/taskpad-\$(date +\%F).db'" && find $BAK -name '*.db' -mtime +30 -delete
CRON

ufw allow OpenSSH; ufw allow 80/tcp; ufw allow 443/tcp; ufw --force enable
systemctl daemon-reload; systemctl enable --now taskpad; systemctl restart caddy
sleep 3; systemctl --no-pager status taskpad | head -5
echo "Done. Open https://$DOMAIN and https://$DOMAIN/admin.html. Logs: journalctl -u taskpad -f"
````

## `taskpad-python/smoke_test.py`

````python
"""Quick self-check (no real M-Pesa): python smoke_test.py"""
import os, tempfile
os.environ.update(JWT_SECRET='x', CALLBACK_SECRET='y', PUBLIC_URL='http://x', ADMIN_TOKEN='adminadminadmin1',
                  INVITE_CODE='PILOT1', MAX_MEMBERS='5', DB_FILE=os.path.join(tempfile.mkdtemp(), 't.db'))
from app import app
c, AD = app.test_client(), {'x-admin-token': 'adminadminadmin1'}
def reg(n, inv='PILOT1'): return c.post('/api/register', json={'name': 'T' + n, 'phone': '07220001' + n, 'password': 'password1', 'invite': inv})
def check(label, cond):
    print(('PASS  ' if cond else 'FAIL  ') + label)
    if not cond: raise SystemExit(1)
t = c.get('/api/tasks').get_json()
check('24 tasks, minimum 2500', len(t['tasks']) == 24 and t['rules']['minW'] == 2500)
check('sign-ups closed with no reserve', reg('11').status_code == 403)
check('wrong invite refused', reg('12', 'NOPE').get_json()['error'].startswith('You need a valid invite'))
check('admin needs token', c.get('/api/admin/summary').status_code == 401)
check('reserve of 5000 allows exactly 1 member', c.post('/api/admin/reserve', json={'amount': 5000}, headers=AD).get_json()['room'] == 1)
r = reg('13').get_json(); H = {'Authorization': 'Bearer ' + r['token']}
check('member gets 200 bonus', r['balance'] == 200 and r['earned'] == 200)
check('second sign-up closed (reserve used)', reg('14').status_code == 403)
check('duplicate phone refused', reg('13').status_code in (403, 409))
check('login works', c.post('/api/login', json={'phone': '0722000113', 'password': 'password1'}).status_code == 200)
check('bad password refused', c.post('/api/login', json={'phone': '0722000113', 'password': 'nope'}).status_code == 401)
d = c.post('/api/tasks/t1/complete', json={'answers': ['a']}, headers=H).get_json()
check('free task pays 100', d['reward'] == 100 and d['balance'] == 300 and d['earned'] == 300)
check('same task twice refused', c.post('/api/tasks/t1/complete', json={'answers': ['a']}, headers=H).status_code == 409)
check('locked task refused', c.post('/api/tasks/t2/complete', json={'answers': ['a']}, headers=H).status_code == 402)
check('withdraw below minimum refused', c.post('/api/withdraw', json={'amount': 300}, headers=H).status_code == 400)
s = c.get('/api/admin/summary', headers=AD).get_json()
check('headroom unchanged by earning (2400)', s['headroom'] == 2400 and s['exposure'] == 2600)
check('unknown payment reports failed', c.get('/api/payment-status?id=nope', headers=H).get_json()['status'] == 'failed')
check('site page served', b'<title>Taskpad' in c.get('/').data)
print('All checks passed.')
````
