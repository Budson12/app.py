import streamlit as st
import numpy as np
import pandas as pd
import requests
import itertools
import os
import re
from fractions import Fraction

# Set up responsive mobile viewport configuration
st.set_page_config(page_title="Benter African Syndicate Engine", layout="centered")

# ==========================================
# 🎨 PREMIUM VISUAL COLOR ACCENT HIGHLIGHTS
# ==========================================
st.markdown("""
    <style>
    .stApp { background-color: #05070F; color: #E2E8F0; }
    .streamlit-expanderHeader { background-color: #111524 !important; border-radius: 8px !important; color: #FFD700 !important; font-weight: bold !important; border: 1px solid #FFD700; }
    div[data-testid="stMetricValue"] { color: #00E676 !important; font-weight: bold; }
    div.stButton > button:first-child { background-color: #FFD700 !important; color: #05070F !important; font-weight: bold !important; border-radius: 8px !important; width: 100%; border: none !important; box-shadow: 0px 0px 10px rgba(255, 215, 0, 0.4); }
    div.stButton > button:first-child:hover { background-color: #00E676 !important; color: #05070F !important; }
    div.stButton > button[data-testid="baseButton-primary"] { background: linear-gradient(135deg, #00E676 0%, #00B0FF 100%) !important; color: #FFFFFF !important; font-size: 16px !important; border: none !important; font-weight: bold !important; box-shadow: 0px 0px 15px rgba(0, 230, 118, 0.4); }
    </style>
""", unsafe_allow_html=True)

st.title("🥇 Benter African Syndicate Engine")
st.caption("Enterprise-Grade PMU Exploitation Station: Couplé, 2sur4, Tiercé, Quarté, Quinté+ (XAF)")

# Permanent file path to log transaction ledger metrics securely
LEDGER_FILE = "benter_pmu_ledger.csv"
if not os.path.exists(LEDGER_FILE):
    pd.DataFrame(columns=['Timestamp', 'Discipline', 'Bet Type', 'Selection', 'Stake (F CFA)', 'Odds Taken', 'Result', 'Net Profit/Loss (F CFA)']).to_csv(LEDGER_FILE, index=False)

# ==========================================
# 🛠️ UTILITY MATRICES FUNCTIONS
# ==========================================
def decimal_to_fractional_string(decimal_odds: float) -> str:
    if decimal_odds <= 1.0: return "0/1"
    frac = Fraction(decimal_odds - 1.0).limit_denominator(20)
    return "1/1" if frac.numerator == frac.denominator else f"{frac.numerator}/{frac.denominator}"

def calculate_box_cost(bet_type, horses_count):
    if horses_count < 2: return 0
    base_cost = 500
    if bet_type == "Couplé Box": return int(len(list(itertools.combinations(range(horses_count), 2))) * base_cost)
    elif bet_type == "Tiercé Box":
        if horses_count < 3: return 0
        return int(len(list(itertools.permutations(range(horses_count), 3))) // 6 * base_cost)
    elif bet_type == "Quarté Box":
        if horses_count < 4: return 0
        return int(len(list(itertools.permutations(range(horses_count), 4))) // 24 * base_cost)
    elif bet_type == "Quinté+ Box":
        if horses_count < 5: return 0
        return int(len(list(itertools.permutations(range(horses_count), 5))) // 120 * base_cost)
    return 0

# ==========================================
# SIDEBAR CONTROLS & MANAGEMENT BLOCKS
# ==========================================
with st.sidebar:
    st.markdown("<h2 style='color:#FFD700;'>🔑 Network Connections</h2>", unsafe_allow_html=True)
    api_key = st.text_input("TheOddsAPI Secret Token Key:", type="password", value="")
    data_mode = st.radio("System Data Ingestion Route:", ["Simulation Mode (Offline)", "Live API Data Feed (Online)"])
    cloud_sync = st.checkbox("Enable Remote Cloud Backup Sync", value=False)
    
    st.markdown("<h2 style='color:#FFD700;'>💰 Bankroll Management</h2>", unsafe_allow_html=True)
    bankroll = st.number_input("Total Bankroll (F CFA):", min_value=500, value=250000, step=5000)
    kelly_fraction = st.slider("Kelly Fraction Multiplier:", 0.05, 1.0, 0.10, step=0.05)
    
    st.markdown("<h2 style='color:#FFD700;'>📐 Market Bias Controls</h2>", unsafe_allow_html=True)
    pmu_takeout_pct = st.slider("Kiosk Takeout Margin (%)", 10.0, 30.0, 22.0, step=0.5, help="House profit cut subtracted automatically before overlay evaluations.")
    favorite_longshot_power = st.slider("Favorite-Longshot Power Law", 0.75, 1.00, 0.88, step=0.01)
    
    st.markdown("<h2 style='color:#FFD700;'>🛑 Safety Switch</h2>", unsafe_allow_html=True)
    stop_loss_limit = st.number_input("Stop-Loss Threshold (F CFA):", min_value=0, value=50000, step=5000)

with st.expander("⚙️ TRACK DISCIPLINE & ENVIRONMENT CONDITIONS", expanded=True):
    track_moisture = st.slider("Live Track Moisture Scale:", 0.0, 1.0, 0.40)
    race_type = st.selectbox("Select Race Discipline:", ["Flat Sprint (1000m-1400m)", "Flat Long (1600m+)", "Handicapping Race", "Hurdle / Jumps", "Trotting / Harness Race"])
    track_turn = st.selectbox("Select Track Configuration Bend:", ["Left Hand Bend", "Right Hand Bend", "Straight Track"])
    race_grade = st.selectbox("⚡ AUTOMATIC FIELD CLASS-PAR PRESET:", ["Medium Tier (Standard Track Average)", "Low Tier (Maiden / Local Stakes)", "Elite Tier (Group Cups / Grand Prix)"])

type_clean = "flat_sprint" if "Sprint" in race_type else ("flat_long" if "Long" in race_type else ("handicap" if "Handicap" in race_type else ("hurdle" if "Hurdle" in race_type else "trotting")))
turn_clean = "left" if "Left" in track_turn else ("right" if "Right" in track_turn else "straight")

if "Low" in race_grade: auto_speed, auto_class = 78.0, 72.0
elif "Elite" in race_grade: auto_speed, auto_class = 95.0, 92.0
else: auto_speed, auto_class = 88.0, 82.0

if type_clean == "flat_sprint": auto_distance = 1200
elif type_clean == "flat_long": auto_distance = 1600
elif type_clean == "handicap": auto_distance = 2000
elif type_clean == "hurdle": auto_distance = 3400
else: auto_distance = 2400

# ==========================================
# EXTRACTION DRAWER
# ==========================================
with st.expander("📋 LIVE EXTRACTION DRAWER (COPY-PASTE / HANDWRITING ENGINE)", expanded=False):
    st.write("Paste raw odds or transcribed entries below. The matrix will auto-align them:")
    raw_ingest_text = st.text_area("Scraper Entry Box:", value="")
    parsed_odds_list = []
    if raw_ingest_text:
        found_tokens = re.findall(r"[-+]?\d*\.\d+|\d+", raw_ingest_text)
        parsed_odds_list = [float(token) for token in found_tokens if float(token) > 1.0][:6]
        if parsed_odds_list: st.success(f"Successfully scraped {len(parsed_odds_list)} target numeric prices from entry block!")

# ==========================================
# SECTION 2: VARIABLES INPUT MATRIX
# ==========================================
st.markdown("<h3 style='color:#FFD700;'>1. Live Variables Input Matrix</h3>", unsafe_allow_html=True)
st.info("💡 Time-Saver Active: Baseline Speed, Class, and Race Distance are automatically configured based on your preset selectors above.")

base_odds = [2.80, 4.50, 4.00, 5.50, 9.00, 14.00]
if parsed_odds_list:
    for index, parsed_val in enumerate(parsed_odds_list):
        if index < len(base_odds): base_odds[index] = parsed_val

default_matrix = pd.DataFrame({
    'Horse Name': ['Horse 1', 'Horse 2', 'Horse 3', 'Horse 4', 'Horse 5', 'Horse 6'],
    'Live Decimal Odds': base_odds,
    'Baseline Speed': [auto_speed] * 6,
    'Class Rating': [auto_class] * 6,
    'Race Distance (m)': [auto_distance] * 6,
    'Jockey/Driver Wt (kg)': [54.5, 56.5, 53.0, 55.5, 58.0, 53.5],
    'Total Carried Wt (kg)': [510.0, 531.5, 493.0, 517.5, 568.0, 501.5],
    'Gait Consistency %': [98.0, 95.0, 99.0, 88.0, 94.0, 91.0],
    'Draw Gate Position': [2, 5, 1, 4, 3, 6],
    'Morning Decimal Odds': [b * 0.9 for b in base_odds]
})

if data_mode == "Live API Data Feed (Online)":
    if not api_key:
        st.warning("⚠️ Access Blocked: Provide API Key in sidebar.")
        working_df = default_matrix
    else:
        url = "https://the-odds-api.com"
        params = {'apiKey': api_key, 'regions': 'uk', 'markets': 'h2h', 'oddsFormat': 'decimal'}
        try:
            res = requests.get(url, params=params, timeout=10)
            if res.status_code == 200:
                payload = res.json()
                if payload and len(payload) > 0:
                    event = payload
                    st.success(f"🔗 Connected Online: {event['sport_title']}")
                    parsed_runners = []
                    for idx, outcome in enumerate(event.get('bookmakers', []).get('markets', []).get('outcomes', [])):
                        parsed_runners.append({
                            'Horse Name': f"Horse {idx+1}", 'Live Decimal Odds': float(outcome['price']), 'Baseline Speed': auto_speed, 'Class Rating': auto_class, 'Race Distance (m)': auto_distance, 'Draw Gate Position': idx + 1, 'Morning Decimal Odds': float(outcome['price']) * 0.9
                        })
                    working_df = pd.DataFrame(parsed_runners).drop_duplicates(subset=['Horse Name'])
                else: working_df = default_matrix
            else: working_df = default_matrix
        except Exception: working_df = default_matrix
else:
    working_df = default_matrix

working_df['Display Odds (Fractional)'] = working_df['Live Decimal Odds'].apply(lambda x: decimal_to_fractional_string(x))
edited_df = st.data_editor(working_df, num_rows="dynamic", use_container_width=True)

# 🛠️ AUTO-FIX SHIELD: Safe check logic to stop File Not Found error flags on first run
if os.path.exists(LEDGER_FILE) and os.path.getsize(LEDGER_FILE) > 50:
    df_ledger = pd.read_csv(LEDGER_FILE)
    has_history = True
else:
    df_ledger = pd.DataFrame()
    has_history = False

current_drawdown = 0
if has_history and not df_ledger.empty:
    total_pnl = df_ledger['Net Profit/Loss (F CFA)'].sum()
    if total_pnl < 0: current_drawdown = abs(total_pnl)

safety_multiplier = 1.0
if current_drawdown >= stop_loss_limit:
    safety_multiplier = 0.35
