# =============================
# XAUUSD Trade Signal Bot 24/7
# Render / Cloud Ready
# =============================
import os
import time
import yfinance as yf
import pandas as pd
import ta
import requests
from datetime import datetime

# ===== CONFIG =====
SYMBOL = "GC=F"  # Gold futures
INTERVALS = ["5m", "15m", "1h", "4h"]
PERIOD = "7d"
CHECK_INTERVAL = 300  # seconds

# ===== TELEGRAM =====
BOT_TOKEN = os.getenv("8277096616:AAEe6LrJ1NFoxAfijFJCoRa4FPo4zFDe7hQ")
CHAT_ID = os.getenv("6838752300")

if not BOT_TOKEN or not CHAT_ID:
    raise Exception("Please set BOT_TOKEN and CHAT_ID as environment variables.")

def send_message(text):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": text}
    try:
        requests.post(url, data=payload)
    except Exception as e:
        print("❌ Telegram error:", e)

def fetch_data(symbol, interval):
    try:
        df = yf.download(symbol, period=PERIOD, interval=interval, progress=False)
        if df.empty:
            print(f"❌ No data for {symbol} @ {interval}")
            return None
        df = df.rename(str.lower, axis="columns")
        return df
    except Exception as e:
        print(f"❌ Error fetching data ({interval}):", e)
        return None

def generate_signal(df):
    df["ma_fast"] = ta.trend.sma_indicator(df["close"], window=10)
    df["ma_slow"] = ta.trend.sma_indicator(df["close"], window=30)
    
    prev = df.iloc[-2]
    last = df.iloc[-1]

    if prev["ma_fast"] < prev["ma_slow"] and last["ma_fast"] > last["ma_slow"]:
        return "BUY"
    elif prev["ma_fast"] > prev["ma_slow"] and last["ma_fast"] < last["ma_slow"]:
        return "SELL"
    else:
        return None

def check_signals():
    tally = {"BUY": 0, "SELL": 0}
    last_price = None

    for interval in INTERVALS:
        df = fetch_data(SYMBOL, interval)
        if df is None or "close" not in df:
            continue
        last_price = float(df["close"].iloc[-1])
        signal = generate_signal(df)
        if signal:
            tally[signal] += 1

    if tally["BUY"] >= 3:
        send_trade("BUY", last_price)
    elif tally["SELL"] >= 3:
        send_trade("SELL", last_price)

def send_trade(signal, price):
    text = f"""
📈 Trade Signal

Pair: XAUUSD
Entry: {price:.2f}
Side: {signal}

TP1: {price + 2 if signal=='BUY' else price - 2}
TP2: {price + 4 if signal=='BUY' else price - 4}
TP3: {price + 6 if signal=='BUY' else price - 6}
TP4: {price +10 if signal=='BUY' else price -10}
SL:  {price - 5 if signal=='BUY' else price +5}

⚠️ NOT FINANCIAL ADVICE
Copy-paste directly into MT5.
"""
    send_message(text)
    print(f"✅ {signal} Signal sent at {price:.2f}")

# ===== START BOT =====
start_text = f"🚀 Bot started for XAUUSD (TFs: {', '.join(INTERVALS)}) at {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
send_message(start_text)
print(start_text)

# ===== RUN 24/7 =====
while True:
    print(f"\n⏱️ {datetime.now().strftime('%Y-%m-%d %H:%M:%S')} — checking signals…")
    check_signals()
    time.sleep(CHECK_INTERVAL)