import streamlit as st
import pandas as pd
import plotly.express as px
import sqlite3
import bcrypt
import os
import openai

# ================= CONFIG =================
st.set_page_config(page_title="DataPilot AI", layout="wide")

# ================= GLOBAL CSS =================
st.markdown("""
<style>
/* App background */
.stApp {background: #0f172a; color: #e5e7eb;}

/* Cards */
.card {
  background: #111827;
  padding: 20px;
  border-radius: 16px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.4);
}

/* Titles */
h1, h2, h3 {color: #f9fafb;}

/* Buttons */
.stButton>button {
  border-radius: 12px;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: white;
  border: none;
}

/* Sidebar */
section[data-testid="stSidebar"] {
  background: #020617;
}
</style>
""", unsafe_allow_html=True)

# ================= PERFORMANCE =================
@st.cache_data
def load_data(file_path):
    return pd.read_csv(file_path)

# ================= API =================
openai.api_key = st.secrets.get("OPENAI_API_KEY", "")

# ================= DATABASE =================
conn = sqlite3.connect("app.db", check_same_thread=False)
c = conn.cursor()

c.execute("CREATE TABLE IF NOT EXISTS users (username TEXT PRIMARY KEY, password BLOB)")
conn.commit()

# ================= AUTH =================
def hash_password(password):
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt())

def verify_password(password, hashed):
    return bcrypt.checkpw(password.encode(), hashed)

def signup(username, password):
    c.execute("SELECT * FROM users WHERE username=?", (username,))
    if c.fetchone(): return False
    c.execute("INSERT INTO users VALUES (?,?)", (username, hash_password(password)))
    conn.commit(); return True

def login(username, password):
    c.execute("SELECT password FROM users WHERE username=?", (username,))
    row = c.fetchone()
    return row and verify_password(password, row[0])

# ================= SESSION =================
if "auth" not in st.session_state:
    st.session_state.auth = False

# ================= LOGIN =================
if not st.session_state.auth:
    st.markdown("""
    <div style='text-align:center;margin-top:100px;'>
        <h1>🚀 DataPilot AI</h1>
        <p style='color:gray;'>AI-powered analytics platform</p>
    </div>
    """, unsafe_allow_html=True)

    col1, col2, col3 = st.columns([1,2,1])
    with col2:
        choice = st.radio("", ["Login", "Signup"])
        user = st.text_input("Username")
        pwd = st.text_input("Password", type="password")

        if choice == "Signup":
            if st.button("Create Account"):
                st.success("Created") if signup(user, pwd) else st.error("Exists")

        if choice == "Login":
            if st.button("Login"):
                if login(user, pwd):
                    st.session_state.auth = True
                    st.session_state.user = user
                    st.rerun()
                else:
                    st.error("Wrong credentials")

    st.stop()

# ================= LOGOUT =================
if st.sidebar.button("Logout"):
    st.session_state.auth = False
    st.rerun()

user = st.session_state.user

# ================= FILE =================
st.sidebar.markdown("## 📂 Data")
uploaded = st.sidebar.file_uploader("Upload", type=["csv","xlsx"])

os.makedirs("data", exist_ok=True)
file_path = f"data/{user}.csv"

if uploaded:
    df = pd.read_csv(uploaded) if uploaded.name.endswith("csv") else pd.read_excel(uploaded)
    df.to_csv(file_path, index=False)
elif os.path.exists(file_path):
    df = load_data(file_path)
else:
    st.warning("Upload data first"); st.stop()

# ================= AUTO DETECT =================
def detect(df):
    cols = df.columns
    d = next((c for c in cols if "date" in c.lower()), cols[0])
    r = next((c for c in cols if "revenue" in c.lower() or "sales" in c.lower()), cols[0])
    c = next((c for c in cols if "category" in c.lower()), cols[0])
    return d,r,c

date_col, revenue_col, category_col = detect(df)

df[date_col] = pd.to_datetime(df[date_col], errors='coerce')
df = df.dropna(subset=[date_col])

# ================= NAV =================
tab1, tab2, tab3 = st.tabs(["📊 Dashboard", "🤖 AI", "💬 Chat"])

# ================= DASHBOARD =================
with tab1:
    st.markdown("## Overview")

    revenue = df[revenue_col].sum()
    orders = len(df)
    aov = revenue/orders if orders else 0

    c1, c2, c3 = st.columns(3)
    c1.markdown(f"<div class='card'><h3>Revenue</h3><h2>{revenue:,.0f}</h2></div>", unsafe_allow_html=True)
    c2.markdown(f"<div class='card'><h3>Orders</h3><h2>{orders}</h2></div>", unsafe_allow_html=True)
    c3.markdown(f"<div class='card'><h3>AOV</h3><h2>{aov:,.0f}</h2></div>", unsafe_allow_html=True)

    cat = df.groupby(category_col)[revenue_col].sum().reset_index()
    fig1 = px.bar(cat, x=category_col, y=revenue_col)

    monthly = df.groupby(df[date_col].dt.to_period("M"))[revenue_col].sum().reset_index()
    monthly[date_col] = monthly[date_col].astype(str)
    fig2 = px.line(monthly, x=date_col, y=revenue_col)

    col1, col2 = st.columns(2)
    col1.plotly_chart(fig1, use_container_width=True)
    col2.plotly_chart(fig2, use_container_width=True)

# ================= AI =================
with tab2:
    st.markdown("## AI Insights")

    if st.button("Generate Insights"):
        sample = df.head(50).to_string()
        prompt = f"Analyze dataset:\n{sample}"

        if openai.api_key:
            res = openai.ChatCompletion.create(model="gpt-4o-mini", messages=[{"role":"user","content":prompt}])
            st.markdown(f"<div class='card'>{res.choices[0].message.content}</div>", unsafe_allow_html=True)
        else:
            st.warning("Add API key")

# ================= CHAT =================
with tab3:
    st.markdown("## Chat with Data")

    q = st.text_input("Ask")

    if st.button("Ask AI"):
        sample = df.head(50).to_string()
        prompt = f"Dataset:\n{sample}\nQuestion:{q}"

        if openai.api_key:
            res = openai.ChatCompletion.create(model="gpt-4o-mini", messages=[{"role":"user","content":prompt}])
            st.markdown(f"<div class='card'>{res.choices[0].message.content}</div>", unsafe_allow_html=True)

# ================= EXPORT =================
st.sidebar.markdown("---")
st.sidebar.download_button("Download CSV", df.to_csv(index=False).encode('utf-8'), "data.csv")
