import time
import pandas as pd
from binance.client import Client
from ta.momentum import RSIIndicator
from binance.enums import *

# === CONFIGURAZIONE TESTNET ===
API_KEY = 'la_tua_testnet_api_key'
API_SECRET = 'la_tua_testnet_api_secret'

client = Client(API_KEY, API_SECRET)
client.API_URL = 'https://testnet.binance.vision/api'

# === PARAMETRI BOT ===
symbol = 'BTCUSDT'
interval = '1m'
quantity = 0.001
rsi_buy = 30
rsi_sell = 70

def get_klines(symbol, interval, limit=100):
    klines = client.get_klines(symbol=symbol, interval=interval, limit=limit)
    df = pd.DataFrame(klines, columns=[
        'timestamp', 'open', 'high', 'low', 'close', 'volume',
        'close_time', 'quote_asset_volume', 'number_of_trades',
        'taker_buy_base', 'taker_buy_quote', 'ignore'
    ])
    df['close'] = df['close'].astype(float)
    return df

def get_rsi(df, period=14):
    rsi = RSIIndicator(close=df['close'], window=period)
    df['rsi'] = rsi.rsi()
    return df

def place_order(order_type):
    try:
        if order_type == 'buy':
            order = client.order_market_buy(symbol=symbol, quantity=quantity)
            print("ORDINE BUY ESEGUITO:", order)
        elif order_type == 'sell':
            order = client.order_market_sell(symbol=symbol, quantity=quantity)
            print("ORDINE SELL ESEGUITO:", order)
    except Exception as e:
        print("Errore ordine:", e)

def run_bot():
    bought = False
    print("Saldo iniziale USDT:", client.get_asset_balance(asset='USDT'))
    while True:
        try:
            df = get_klines(symbol, interval)
            df = get_rsi(df)
            latest_rsi = df['rsi'].iloc[-1]
            print(f"[RSI: {latest_rsi:.2f}] Stato: {'IN POSIZIONE' if bought else 'PRONTO'}")

            if latest_rsi < rsi_buy and not bought:
                place_order('buy')
                bought = True

            elif latest_rsi > rsi_sell and bought:
                place_order('sell')
                bought = False

            time.sleep(60)

        except Exception as e:
            print("Errore bot:", e)
            time.sleep(60)

run_bot()
