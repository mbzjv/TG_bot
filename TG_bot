import requests
import time

TOKEN = "8601104005:AAFVAutybEBlSWGC_KR8VjMPdBLoVMFHUHI"
CHAT_ID = "398901069"

previous_oi = {}
sent_signals = set()


# --- TELEGRAM ---
def send_message(text):
    url = f"https://api.telegram.org/bot{TOKEN}/sendMessage"
    requests.post(url, data={"chat_id": CHAT_ID, "text": text})


# --- ВСЕ ТИКЕРЫ ---
def get_all_tickers():
    url = "https://fapi.binance.com/fapi/v1/ticker/24hr"
    return requests.get(url).json()


# --- ТОП МОНЕТЫ ---
def get_top_symbols(limit=30):
    tickers = get_all_tickers()
    usdt_pairs = [t for t in tickers if t["symbol"].endswith("USDT")]
    usdt_pairs.sort(key=lambda x: float(x["quoteVolume"]), reverse=True)
    return usdt_pairs[:limit]


# --- ФАНДИНГ ---
def get_funding(symbol):
    try:
        url = f"https://fapi.binance.com/fapi/v1/fundingRate?symbol={symbol}&limit=1"
        data = requests.get(url).json()
        return float(data[0]["fundingRate"])
    except:
        return 0


# --- ОИ ---
def get_open_interest(symbol):
    try:
        url = f"https://fapi.binance.com/fapi/v1/openInterest?symbol={symbol}"
        data = requests.get(url).json()
        return float(data["openInterest"])
    except:
        return 0


# --- СИЛА ОИ ---
def oi_strength(oi_change):
    if oi_change > 150000:
        return "STRONG 🔥"
    elif oi_change > 30000:
        return "MEDIUM"
    else:
        return "WEAK"


# --- ФАЗА ---
def get_phase(price_change, funding, oi_change):
    if price_change > 6 and funding < -0.0005:
        return "LATE"
    elif price_change > 2 and funding < -0.0001 and oi_change > 0:
        return "EARLY"
    else:
        return "MID"


# --- ЛИКВИДАЦИИ (proxy через объем + импульс) ---
def liquidation_signal(price_change, volume):
    if price_change > 4 and volume > 20_000_000:
        return "SHORTS LIQUIDATING 🔥"
    if price_change < -4 and volume > 20_000_000:
        return "LONGS LIQUIDATING 🔥"
    return "none"


# --- РЕЙТИНГ ---
def get_rating(price_change, funding, oi_change, volume):
    score = 0

    if abs(price_change) > 2:
        score += 1
    if abs(funding) > 0.0001:
        score += 1
    if abs(oi_change) > 30000:
        score += 1
    if volume > 10_000_000:
        score += 1

    if score == 4:
        return "A+ 🔥"
    elif score == 3:
        return "A"
    else:
        return "B"


# --- ПРОВЕРКА ---
def check_signal(symbol, price_change, volume):
    funding = get_funding(symbol)
    current_oi = get_open_interest(symbol)

    prev_oi = previous_oi.get(symbol, current_oi)
    previous_oi[symbol] = current_oi

    oi_change = current_oi - prev_oi

    # фильтр мусора
    if volume < 8_000_000:
        return None

    strength = oi_strength(abs(oi_change))
    phase = get_phase(price_change, funding, oi_change)
    liq = liquidation_signal(price_change, volume)
    rating = get_rating(price_change, funding, oi_change, volume)

    key = f"{symbol}"

    # --- ЛОНГ ---
    if (
        price_change > 2
        and funding < -0.0001
        and oi_change > 30000
    ):
        if key not in sent_signals:
            sent_signals.add(key)
            return (
                f"{symbol} 🚀 ЛОНГ | {rating}\n"
                f"Цена: {price_change:.2f}%\n"
                f"Фандинг: {funding:.6f}\n"
                f"ОИ: +{int(oi_change)} ({strength})\n"
                f"Фаза: {phase}\n"
                f"{liq}"
            )

    # --- ШОРТ ---
    if (
        price_change > 2
        and funding > 0.0001
        and oi_change < -30000
    ):
        if key not in sent_signals:
            sent_signals.add(key)
            return (
                f"{symbol} ⚠️ ШОРТ | {rating}\n"
                f"Цена: {price_change:.2f}%\n"
                f"Фандинг: {funding:.6f}\n"
                f"ОИ: {int(oi_change)} ({strength})\n"
                f"Фаза: DISTRIBUTION\n"
                f"{liq}"
            )

    return None


# --- СТАРТ ---
send_message("🔥 PRO Сканер запущен")

while True:
    try:
        top_symbols = get_top_symbols(30)

        for coin in top_symbols:
            symbol = coin["symbol"]
            price_change = float(coin["priceChangePercent"])
            volume = float(coin["quoteVolume"])

            signal = check_signal(symbol, price_change, volume)

            if signal:
                send_message(signal)
                print(signal)

        time.sleep(60)

    except Exception as e:
        print("Ошибка:", e)
        time.sleep(10)
