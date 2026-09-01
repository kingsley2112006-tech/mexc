# mexc
pip install ccxt pandas numpy
import ccxt
import pandas as pd
import time

# --- CONFIGURATION ---
API_KEY = 'YOUR_MEXC_API_KEY'
API_SECRET = 'YOUR_MEXC_API_SECRET'
SYMBOL = 'BTC/USDT'
TIMEFRAME = '15m'
TRADE_SIZE = 0.001 # Amount to trade

# Initialize MEXC via CCXT
exchange = ccxt.mexc({
    'apiKey': API_KEY,
    'secret': API_SECRET,
    'enableRateLimit': True,
})

def fetch_data(symbol, timeframe, limit=1000):
    """Fetch historical OHLCV data."""
    bars = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    df = pd.DataFrame(bars, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    return df

def get_state(pct_change):
    """Categorize percentage returns into discrete market states."""
    if pct_change < -0.002: return 'STRONG_DOWN'
    elif pct_change < 0: return 'WEAK_DOWN'
    elif pct_change > 0.002: return 'STRONG_UP'
    else: return 'WEAK_UP'

def build_markov_model(df):
    """Build the transition probability matrix."""
    df['pct_change'] = df['close'].pct_change()
    df = df.dropna().copy()
    df['state'] = df['pct_change'].apply(get_state)
    
    # Calculate historical transition probabilities
    transitions = pd.crosstab(df['state'].shift(), df['state'], normalize='index')
    current_state = df['state'].iloc[-1]
    return transitions, current_state

def execute_trade(prediction):
    """Execute trades based on the Markov prediction."""
    print(f"Prediction for next candle: {prediction}")
    
    try:
        if prediction == 'STRONG_UP':
            print(f"Action: BUY {TRADE_SIZE} {SYMBOL}")
            # Real trade execution (Uncomment with caution):
            # exchange.create_market_buy_order(SYMBOL, TRADE_SIZE)
        elif prediction == 'STRONG_DOWN':
            print(f"Action: SELL {TRADE_SIZE} {SYMBOL}")
            # Real trade execution (Uncomment with caution):
            # exchange.create_market_sell_order(SYMBOL, TRADE_SIZE)
        else:
            print("Action: HOLD (No strong directional conviction)")
    except Exception as e:
        print(f"Trade execution failed: {e}")

def run_bot():
    print(f"Starting Markov Chain Bot on {SYMBOL} ({TIMEFRAME})...")
    while True:
        try:
            # 1. Fetch latest market data
            df = fetch_data(SYMBOL, TIMEFRAME)
            
            # 2. Build model and get current state
            matrix, current_state = build_markov_model(df)
            print(f"\nCurrent Market State: {current_state}")
            
            # 3. Predict next state
            if current_state in matrix.index:
                probabilities = matrix.loc[current_state]
                prediction = probabilities.idxmax()
                
                # 4. Execute Trade
                execute_trade(prediction)
            
            # Sleep until next candle
            time.sleep(15 * 60)
            
        except Exception as e:
            print(f"Error in main loop: {e}")
            time.sleep(60)

if __name__ == '__main__':
    # run_bot()
    print("Test run complete. Add API keys and uncomment run_bot() to begin trading.")
