import json
import time
import requests
import websocket

# Symbols and timeframe mapping (Deriv granularity in seconds)
SYMBOLS = ["R_75", "R_100", "1HZ10V", "1HZ100V"]
TIMEFRAMES = {
    "H1": 3600,
    "H4": 14400
}

def get_candles(symbol, granularity, count=50):
    ws_url = "wss://ws.binaryws.com/websockets/v3?app_id=1089"
    try:
        ws = websocket.create_connection(ws_url, timeout=10)
        req = {
            "ticks_history": symbol,
            "adjust_start_time": 1,
            "count": count,
            "end": "latest",
            "style": "candles",
            "granularity": granularity
        }
        ws.send(json.dumps(req))
        res = ws.recv()
        ws.close()
        data = json.loads(res)
        return data.get("candles", [])
    except Exception as e:
        print(f"Error fetching {symbol} ({granularity}s): {e}")
        return []

def check_bos(symbol, tf_name, granularity):
    candles = get_candles(symbol, granularity)
    if not candles or len(candles) < 10:
        print(f"[{symbol} - {tf_name}] Insufficient data received.")
        return

    # Use closed candles (exclude current active candle at index -1)
    closed_candles = candles[:-1]
    current_close = closed_candles[-1]["close"]
    
    # Calculate previous structure high/low from prior candles
    prev_highs = [c["high"] for c in closed_candles[-6:-1]]
    prev_lows = [c["low"] for c in closed_candles[-6:-1]]
    
    structure_high = max(prev_highs)
    structure_low = min(prev_lows)

    print(f"[{symbol} - {tf_name}] Current Close: {current_close} | High: {structure_high} | Low: {structure_low}")

    if current_close > structure_high:
        print(f"🚨 [BULLISH BOS DETECTED] {symbol} ({tf_name}) broke high {structure_high} with close {current_close}!")
    elif current_close < structure_low:
        print(f"🚨 [BEARISH BOS DETECTED] {symbol} ({tf_name}) broke low {structure_low} with close {current_close}!")
    else:
        print(f"[{symbol} - {tf_name}] Market inside range. No BOS.")

def main():
    print("--- Starting Deriv H4 & H1 BOS Scan ---")
    for symbol in SYMBOLS:
        for tf_name, granularity in TIMEFRAMES.items():
            check_bos(symbol, tf_name, granularity)
            time.sleep(1)
    print("--- Scan Complete ---")

if __name__ == "__main__":
    main()
    
