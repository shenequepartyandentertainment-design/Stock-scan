
#!/usr/bin/env python3

import csv
import os
import time
from datetime import datetime, date
from pathlib import Path

import pandas as pd
import yfinance as yf

# ==================================
# FREE OPTIONS FLOW SCANNER PRO
# ==================================

TICKERS = ["SPY", "QQQ", "AAPL", "TSLA"]

MIN_VOLUME = 100
MIN_PREMIUM = 25000
MIN_VOL_OI = 1.0
MAX_DTE = 14

SCAN_INTERVAL = 180

OUTPUT = Path("output")
OUTPUT.mkdir(exist_ok=True)

CSV_FILE = OUTPUT / "options_flow.csv"


def number(value):
    try:
        if pd.isna(value):
            return 0
        return float(value)
    except:
        return 0


def estimate_side(last, bid, ask):
    if bid <= 0 or ask <= 0 or ask < bid:
        return "UNKNOWN"

    tolerance = max(0.01, (ask - bid) * 0.10)

    if last >= ask - tolerance:
        return "POSSIBLE BUYING AT ASK"

    if last <= bid + tolerance:
        return "POSSIBLE SELLING AT BID"

    return "MID / UNCLEAR"


def scan_stock(symbol):

    print(f"\nScanning {symbol}...")

    stock = yf.Ticker(symbol)

    results = []

    try:
        expirations = stock.options
        price = number(stock.fast_info.get("last_price"))
    except Exception as error:
        print("Data error:", error)
        return results

    if price <= 0:
        return results

    for expiration in expirations:

        exp_date = datetime.strptime(
            expiration, "%Y-%m-%d"
        ).date()

        dte = (exp_date - date.today()).days

        if dte < 0 or dte > MAX_DTE:
            continue

        try:
            chain = stock.option_chain(expiration)
        except Exception:
            continue

        for option_type, contracts in [
            ("CALL", chain.calls),
            ("PUT", chain.puts)
        ]:

            for _, contract in contracts.iterrows():

                volume = number(contract["volume"])
                oi = number(contract["openInterest"])

                last = number(contract["lastPrice"])
                bid = number(contract["bid"])
                ask = number(contract["ask"])

                strike = number(contract["strike"])

                if volume < MIN_VOLUME:
                    continue

                premium = last * volume * 100

                vol_oi = volume / oi if oi > 0 else 0

                side = estimate_side(last, bid, ask)

                reasons = []

                if volume >= MIN_VOLUME:
                    reasons.append("HIGH VOLUME")

                if vol_oi >= MIN_VOL_OI:
                    reasons.append("VOLUME/OI SPIKE")

                if premium >= MIN_PREMIUM:
                    reasons.append("LARGE PREMIUM ESTIMATE")

                if side == "POSSIBLE BUYING AT ASK":
                    reasons.append("POSSIBLE BUYING")

                strong = (
                    volume >= MIN_VOLUME
                    and premium >= MIN_PREMIUM
                    and vol_oi >= MIN_VOL_OI
                )

                if not reasons:
                    continue

                if strong and option_type == "CALL":
                    signal = "HEAVY CALL ACTIVITY"

                elif strong and option_type == "PUT":
                    signal = "HEAVY PUT ACTIVITY"

                else:
                    signal = "WATCH"

                result = {
                    "Time": datetime.now().isoformat(),
                    "Ticker": symbol,
                    "Stock Price": round(price, 2),
                    "Type": option_type,
                    "Expiration": expiration,
                    "DTE": dte,
                    "Strike": strike,
                    "Last": last,
                    "Bid": bid,
                    "Ask": ask,
                    "Volume": int(volume),
                    "Open Interest": int(oi),
                    "Volume/OI": round(vol_oi, 2),
                    "Estimated Premium": round(premium, 2),
                    "Buy Side Estimate": side,
                    "Signal": signal,
                    "Reason": ", ".join(reasons)
                }

                results.append(result)

    return results


def save_results(results):

    if not results:
        return

    file_exists = CSV_FILE.exists()

    with open(CSV_FILE, "a", newline="") as file:

        writer = csv.DictWriter(
            file,
            fieldnames=results[0].keys()
        )

        if not file_exists:
            writer.writeheader()

        writer.writerows(results)


def main():

    print("=" * 55)
    print("       OPTIONS FLOW SCANNER PRO")
    print("=" * 55)

    print("WATCHING:", TICKERS)

    while True:

        all_results = []

        for ticker in TICKERS:

            try:
                results = scan_stock(ticker)
                all_results.extend(results)

            except Exception as error:
                print(ticker, "ERROR:", error)

        if all_results:

            df = pd.DataFrame(all_results)

            df = df.sort_values(
                "Estimated Premium",
                ascending=False
            )

            print("\nOPTIONS FLOW RESULTS")

            print(df.to_string(index=False))

            save_results(all_results)

        else:

            print("\nNO UNUSUAL ACTIVITY DETECTED")

        print(f"\nNext scan in {SCAN_INTERVAL} seconds.")

        time.sleep(SCAN_INTERVAL)


if __name__ == "__main__":
    main()
