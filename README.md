
import ccxt
import pandas as pd
import numpy as np
import time
from datetime import datetime
import requests

# ตั้งค่า Binance Futures API
exchange = ccxt.binance({'enableRateLimit': True, 'options': {'defaultType': 'future'}})

# ฟังก์ชันส่งข้อความไปยัง Telegram
def send_telegram_message(message, bot_token, chat_id):
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    data = {"chat_id": chat_id, "text": message}
    try:
        requests.post(url, data=data)
    except requests.exceptions.RequestException as e:
        print(f"❗ Error sending message to Telegram: {e}")

# เปลี่ยนค่าต่อไปนี้ให้เป็นข้อมูลของคุณ
bot_token = "YOUR_BOT_TOKEN"
chat_id = "YOUR_CHAT_ID"

# ดึงข้อมูลแท่งเทียน
def fetch_ohlcv(symbol, timeframe='1m', limit=100):
    for _ in range(5):
        try:
            data = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
            if data:
                return data
        except Exception as e:
            print(f"❗ Error fetching data: {e}")
            time.sleep(5)
    return None

# คำนวณ Indicator
def calculate_indicators(df):
    # MACD (เร็วขึ้น)
    df['EMA9'] = df['close'].ewm(span=9, adjust=False).mean()
    df['EMA21'] = df['close'].ewm(span=21, adjust=False).mean()
    df['MACD'] = df['EMA9'] - df['EMA21']
    df['MACD_signal'] = df['MACD'].ewm(span=7, adjust=False).mean()

    # RSI (เร็วขึ้น)
    delta = df['close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=3).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=3).mean()
    rs = gain / loss
    df['RSI'] = 100 - (100 / (1 + rs))

    # Bollinger Bands (เข้มงวดขึ้น)
    df['SMA20'] = df['close'].rolling(window=20).mean()
    df['stddev'] = df['close'].rolling(window=20).std()
    df['upper_band'] = df['SMA20'] + (df['stddev'] * 2.5)  # ใช้ 2.5 เท่าเพื่อให้เข้มงวดขึ้น
    df['lower_band'] = df['SMA20'] - (df['stddev'] * 2.5)

    # Volume Filter
    df['Volume_MA'] = df['volume'].rolling(window=10).mean()

    return df

# ตรวจจับสัญญาณซื้อขาย
def get_signal(df):
    signal = None
    icon = "⚪"

    # เงื่อนไข BUY
    if (
        df['MACD'].iloc[-2] < df['MACD_signal'].iloc[-2] and df['MACD'].iloc[-1] > df['MACD_signal'].iloc[-1]  # MACD ตัดขึ้น
        and df['RSI'].iloc[-1] > 40  # RSI ไม่ต่ำเกินไป
        and df['close'].iloc[-2] < df['lower_band'].iloc[-2] and df['close'].iloc[-1] > df['lower_band'].iloc[-1]  # ทะลุ BB แล้วกลับเข้าไป
        and df['volume'].iloc[-1] > df['Volume_MA'].iloc[-1]  # Volume สูงกว่าค่าเฉลี่ย
    ):
        signal = "BUY"
        icon = "🟢"

    # เงื่อนไข SELL
    elif (
        df['MACD'].iloc[-2] > df['MACD_signal'].iloc[-2] and df['MACD'].iloc[-1] < df['MACD_signal'].iloc[-1]  # MACD ตัดลง
        and df['RSI'].iloc[-1] < 60  # RSI ไม่สูงเกินไป
        and df['close'].iloc[-2] > df['upper_band'].iloc[-2] and df['close'].iloc[-1] < df['upper_band'].iloc[-1]  # ทะลุ BB แล้วกลับเข้าไป
        and df['volume'].iloc[-1] > df['Volume_MA'].iloc[-1]  # Volume สูงกว่าค่าเฉลี่ย
    ):
        signal = "SELL"
        icon = "🔴"

    return signal, icon

# เปิด-ปิดออเดอร์
def open_position(symbol, position, price, position_type):
    message = f"{datetime.now()} - 📈 Open {position_type} position at {price:.2f}"
    print(message)
    send_telegram_message(message, bot_token, chat_id)

def close_position(symbol, position, price, position_type, open_price):
    profit = (price - open_price) * abs(position) if position_type == 'long' else (open_price - price) * abs(position)
    message = f"{datetime.now()} - ❌ Close {position_type} at {price:.2f} | Profit: {profit:.2f}"
    print(message)
    send_telegram_message(message, bot_token, chat_id)
    return profit

# กลยุทธ์ MACD + RSI + BB + Volume
def trade_macd():
    symbol = 'BTC/USDT'
    timeframe = '1m'  # ลด Timeframe
    balance = 100
    position = 0
    trade_size = 0.001  # ลดขนาดเพื่อลดความเสี่ยง
    position_type = None
    last_signal = None
    total_profit = 0
    total_trades = 0
    total_wins = 0
    open_price = 0

    while True:
        ohlcv = fetch_ohlcv(symbol, timeframe, limit=100)
        if not ohlcv:
            time.sleep(10)
            continue

        df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
        df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')

        df = calculate_indicators(df)
        signal, icon = get_signal(df)
        price = df['close'].iloc[-1]

        print(f"{datetime.now()} - Price: {price:.2f} | MACD: {df['MACD'].iloc[-1]:.4f} | RSI: {df['RSI'].iloc[-1]:.2f} | {icon} {signal}")

        if signal != last_signal and signal is not None:
            if signal == "BUY":
                if position < 0:
                    profit = close_position(symbol, position, price, position_type, open_price)
                    total_profit += profit
                    total_trades += 1
                    total_wins += 1 if profit > 0 else 0
                    position = 0
                    position_type = None
                if position == 0:
                    open_position(symbol, trade_size, price, 'long')
                    open_price = price
                    position = trade_size
                    position_type = 'long'

            elif signal == "SELL":
                if position > 0:
                    profit = close_position(symbol, position, price, position_type, open_price)
                    total_profit += profit
                    total_trades += 1
                    total_wins += 1 if profit > 0 else 0
                    position = 0
                    position_type = None
                if position == 0:
                    open_position(symbol, trade_size, price, 'short')
                    open_price = price
                    position = -trade_size
                    position_type = 'short'

            last_signal = signal

        time.sleep(5)  # ลดเวลาหน่วงเพื่อให้ทำงานเร็วขึ้น

trade_macd()

