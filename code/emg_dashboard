"""
EMG Rehabilitation Monitoring Dashboard
Run with:  streamlit run emg_dashboard.py
"""

import streamlit as st
import pandas as pd
import plotly.graph_objects as go

# ─── Page config ──────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="EMG Rehab Monitor",
    page_icon="🫀",
    layout="wide",
    initial_sidebar_state="collapsed",
)

# ─── Global CSS ───────────────────────────────────────────────────────────────
st.markdown("""
<style>
  /* ── Base ── */
  [data-testid="stAppViewContainer"] { background:#0f1923; color:#e2eaf2; }
  [data-testid="stHeader"]           { background:transparent; }
  [data-testid="stToolbar"]          { display:none; }
  [data-testid="stDecoration"]       { display:none; }

  /* ── Hide default uploader label & drag text, keep only the click zone ── */
  [data-testid="stFileUploaderDropzone"] {
    background: transparent !important;
    border: none !important;
    padding: 0 !important;
  }
  [data-testid="stFileUploaderDropzoneInstructions"] { display:none !important; }
  [data-testid="stFileUploaderDropzone"] > div { display:none !important; }
  /* Show only the Browse button */
  [data-testid="stFileUploaderDropzone"] button {
    width: 100% !important;
    height: 58px !important;
    background: #00c9b1 !important;
    color: #0f1923 !important;
    font-size: 1.05rem !important;
    font-weight: 700 !important;
    border: none !important;
    border-radius: 10px !important;
    cursor: pointer !important;
    letter-spacing: 0.5px !important;
  }
  [data-testid="stFileUploaderDropzone"] button:hover {
    background: #00b09c !important;
  }
  /* Uploaded file pill */
  [data-testid="stFileUploaderFile"] {
    background: #162232 !important;
    border: 1px solid #1e3248 !important;
    border-radius: 8px !important;
    padding: 8px 14px !important;
    margin-top: 12px !important;
  }

  /* ── Landing page ── */
  .landing-wrap {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 80vh;
    padding: 20px;
  }
  .landing-card {
    background: #121f2e;
    border: 1px solid #1e3248;
    border-radius: 20px;
    padding: 52px 56px 44px;
    max-width: 500px;
    width: 100%;
    text-align: center;
    box-shadow: 0 8px 40px rgba(0,0,0,0.4);
  }
  .landing-icon  { font-size: 3.6rem; margin-bottom: 18px; }
  .landing-title {
    font-size: 1.65rem; font-weight: 800; color: #e2eaf2;
    letter-spacing: -0.5px; margin-bottom: 8px;
  }
  .landing-desc {
    font-size: 0.92rem; color: #4a6a8a;
    line-height: 1.6; margin-bottom: 32px;
  }
  .landing-hint {
    font-size: 0.75rem; color: #2a4a6a;
    margin-top: 18px; letter-spacing: 0.3px;
  }
  .landing-steps {
    display: flex; justify-content: center; gap: 28px;
    margin-top: 36px; padding-top: 28px;
    border-top: 1px solid #1a2d40;
  }
  .step {
    display: flex; flex-direction: column; align-items: center; gap: 6px;
  }
  .step-num {
    width: 28px; height: 28px; border-radius: 50%;
    background: #162232; border: 1px solid #1e3248;
    display: flex; align-items: center; justify-content: center;
    font-size: 0.75rem; font-weight: 700; color: #00c9b1;
  }
  .step-lbl { font-size: 0.72rem; color: #3a5a7a; text-align: center; max-width: 80px; }

  /* ── Dashboard ── */
  .dash-title { font-size:1.7rem; font-weight:800; color:#e2eaf2; margin:0; }
  .dash-sub   { font-size:0.86rem; color:#3a6080; margin-top:3px; }
  .file-pill  {
    display:inline-flex; align-items:center; gap:6px;
    background:#162232; border:1px solid #1e3248; border-radius:20px;
    padding:4px 14px; font-size:0.78rem; color:#4a7a9a; margin-top:6px;
  }
  .sec {
    font-size:0.68rem; text-transform:uppercase; letter-spacing:2px;
    color:#2a4a6a; border-bottom:1px solid #1a2d40;
    padding-bottom:5px; margin-bottom:14px; margin-top:4px;
  }
  .kcard {
    background:#162232; border:1px solid #1e3248;
    border-radius:10px; padding:16px 18px 12px;
  }
  .kcard-lbl {
    font-size:0.66rem; text-transform:uppercase;
    letter-spacing:1.2px; color:#2a4a6a; margin-bottom:5px;
  }
  .kcard-val  { font-size:1.7rem; font-weight:700; color:#00c9b1; line-height:1; }
  .kcard-unit { font-size:0.8rem; color:#2a4a6a; margin-left:2px; }
  .aitem {
    background:#162232; border-left:3px solid #00c9b1;
    padding:9px 14px; border-radius:0 8px 8px 0;
    margin-bottom:7px; font-size:0.88rem; color:#c8daea;
  }
  .aitem.neg { border-left-color:#e05c6b; }
  .aitem.neu { border-left-color:#2a4a6a; }
  hr { border-color:#1a2d40; margin:22px 0; }
</style>
""", unsafe_allow_html=True)

# ─── Chart defaults ───────────────────────────────────────────────────────────
CL = dict(
    paper_bgcolor="rgba(0,0,0,0)", plot_bgcolor="#0d1b2a",
    font=dict(family="Segoe UI,sans-serif", color="#8aaac8", size=12),
    margin=dict(l=10, r=10, t=34, b=10),
    xaxis=dict(gridcolor="#1a2d40", linecolor="#1e3248",
               tickcolor="#1e3248", title_font=dict(size=11, color="#3a6080")),
    yaxis=dict(gridcolor="#1a2d40", linecolor="#1e3248",
               tickcolor="#1e3248", title_font=dict(size=11, color="#3a6080")),
    hoverlabel=dict(bgcolor="#162232", bordercolor="#1e3248",
                    font=dict(color="#e2eaf2", size=13)),
    legend=dict(bgcolor="rgba(0,0,0,0)"),
)
TEAL, AMBER, ROSE, SLATE, PURPLE = "#00c9b1","#f0a04b","#e05c6b","#6b8aad","#9b7fe8"

# ─── Helpers ──────────────────────────────────────────────────────────────────
def kcard(label, value, unit=""):
    return (f'<div class="kcard"><div class="kcard-lbl">{label}</div>'
            f'<div class="kcard-val">{value}'
            f'<span class="kcard-unit">{unit}</span></div></div>')

def safe(v, fmt=".0f"):
    try:    return f"{v:{fmt}}" if pd.notna(v) else "—"
    except: return "—"

def line_fig(x, y, name, color, ytitle, fill=False):
    fig = go.Figure(go.Scatter(
        x=x, y=y, mode="lines+markers", name=name,
        line=dict(color=color, width=2.5),
        marker=dict(size=7, color=color, line=dict(color="#0f1923", width=1.5)),
        fill="tozeroy" if fill else "none",
        hovertemplate=f"Session %{{x}}<br>{name}: %{{y}}<extra></extra>",
    ))
    fig.update_layout(**CL, xaxis_title="Session #", yaxis_title=ytitle)
    return fig

def bar_fig(x, y, name, color, ytitle):
    fig = go.Figure(go.Bar(
        x=x, y=y, name=name,
        marker=dict(color=color, opacity=0.85),
        hovertemplate=f"Session %{{x}}<br>{name}: %{{y}}<extra></extra>",
    ))
    fig.update_layout(**CL, xaxis_title="Session #", yaxis_title=ytitle, bargap=0.35)
    return fig

def rec_status(pct):
    if   pct <  90: return "Weak Recovery",     "#e05c6b", "#2a1520"
    elif pct < 120: return "Improving",          "#f0a04b", "#2a1f10"
    elif pct < 150: return "Good Progress",      "#00c9b1", "#0d2520"
    else:           return "Excellent Recovery", "#9b7fe8", "#1a1530"

def delta_html(label, v0, v1, unit="", hib=True):
    if pd.isna(v0) or pd.isna(v1):
        return f'<div class="aitem neu">⬥ {label}: insufficient data</div>'
    ok  = (v1 > v0) if hib else (v1 < v0)
    arr = "▲" if v1 > v0 else ("▼" if v1 < v0 else "→")
    col = TEAL if ok else ROSE
    cls = "aitem" if ok else "aitem neg"
    return (f'<div class="{cls}"><span style="color:{col};font-weight:700">'
            f'{arr} {label}</span>: {v0:.1f}{unit} → <strong>{v1:.1f}{unit}</strong></div>')

# ─── Session state init ───────────────────────────────────────────────────────
if "df"       not in st.session_state: st.session_state.df       = None
if "filename" not in st.session_state: st.session_state.filename = ""

# ─────────────────────────────────────────────────────────────────────────────
#  PAGE 1 — UPLOAD LANDING
# ─────────────────────────────────────────────────────────────────────────────
if st.session_state.df is None:

    st.markdown('<div class="landing-wrap">', unsafe_allow_html=True)

    _, card_col, _ = st.columns([1, 1.6, 1])

    with card_col:
        # Card top (pure HTML — no interactivity needed)
        st.markdown("""
        <div class="landing-card">
          <div class="landing-icon">🫀</div>
          <div class="landing-title">EMG Rehab Monitor</div>
          <div class="landing-desc">
            Upload the session log exported from your ESP32 device
            to view your full rehabilitation progress dashboard.
          </div>
        </div>
        """, unsafe_allow_html=True)

        # Streamlit file uploader — styled above to look like a single button
        uploaded = st.file_uploader(
            "Browse file",
            type=["csv"],
            label_visibility="collapsed",   # hide label; button text is enough
        )

        # Steps row below the card
        st.markdown("""
        <div style="display:flex;justify-content:center;gap:32px;
                    margin-top:28px;padding-top:22px;border-top:1px solid #1a2d40;">
          <div class="step">
            <div class="step-num">1</div>
            <div class="step-lbl">Export CSV from SD card</div>
          </div>
          <div style="color:#1a2d40;font-size:1.4rem;margin-top:4px">→</div>
          <div class="step">
            <div class="step-num">2</div>
            <div class="step-lbl">Click Browse & select file</div>
          </div>
          <div style="color:#1a2d40;font-size:1.4rem;margin-top:4px">→</div>
          <div class="step">
            <div class="step-num">3</div>
            <div class="step-lbl">Dashboard loads instantly</div>
          </div>
        </div>
        <div class="landing-hint">emg_log.csv · columns: Date, Time, AvgActivation,
        BestMVC, StrengthScore, Fatigue, HoldTime, Reps, Duration, Recovery</div>
        """, unsafe_allow_html=True)

        # ── Process file once selected ──
        if uploaded is not None:
            with st.spinner("Loading session data…"):
                df = pd.read_csv(uploaded)
                df.columns = df.columns.str.strip()
                num_cols = ["AvgActivation","BestMVC","StrengthScore",
                            "Fatigue","HoldTime","Reps","Duration","Recovery"]
                for c in num_cols:
                    if c in df.columns:
                        df[c] = pd.to_numeric(df[c], errors="coerce")
                df.dropna(subset=[c for c in num_cols if c in df.columns],
                          how="all", inplace=True)
                df.reset_index(drop=True, inplace=True)
                df["Session"] = df.index + 1
                if "Date" in df.columns and "Time" in df.columns:
                    df["DateTime"] = pd.to_datetime(
                        df["Date"].astype(str)+" "+df["Time"].astype(str),
                        dayfirst=True, errors="coerce")

            if df.empty:
                st.error("⚠️ File is empty or columns don't match expected format.")
            else:
                st.session_state.df       = df
                st.session_state.filename = uploaded.name
                st.rerun()

    st.markdown('</div>', unsafe_allow_html=True)
    st.stop()

# ─────────────────────────────────────────────────────────────────────────────
#  PAGE 2 — FULL DASHBOARD
# ─────────────────────────────────────────────────────────────────────────────
df       = st.session_state.df
filename = st.session_state.filename
latest   = df.iloc[-1]
sessions = df["Session"]

# ── Header ────────────────────────────────────────────────────────────────────
h1, h2 = st.columns([3, 1])
with h1:
    st.markdown(
        '<div class="dash-title">🫀 EMG Rehabilitation Monitor</div>'
        '<div class="dash-sub">Session-by-session electromyography · Physiotherapy Progress Tracker</div>',
        unsafe_allow_html=True)
with h2:
    st.markdown("<br>", unsafe_allow_html=True)
    if st.button("📂  Load new file", use_container_width=True):
        st.session_state.df       = None
        st.session_state.filename = ""
        st.rerun()
    st.markdown(f'<div class="file-pill">📄 {filename} &nbsp;·&nbsp; {len(df)} sessions</div>',
                unsafe_allow_html=True)

st.markdown("<hr>", unsafe_allow_html=True)

# ── KPI cards ─────────────────────────────────────────────────────────────────
st.markdown('<div class="sec">LATEST SESSION — KEY METRICS</div>', unsafe_allow_html=True)
cols = st.columns(8)
for col, (lbl, val, unit) in zip(cols, [
    ("Current MVC",    safe(latest.get("BestMVC")),         ""),
    ("Recovery",       safe(latest.get("Recovery")),         "%"),
    ("Avg Activation", safe(latest.get("AvgActivation")),    "%"),
    ("Strength Score", safe(latest.get("StrengthScore")),    "/10"),
    ("Fatigue",        safe(latest.get("Fatigue")),          "%"),
    ("Reps",           safe(latest.get("Reps")),             ""),
    ("Hold Time",      safe(latest.get("HoldTime"), ".1f"),  "s"),
    ("Duration",       safe(latest.get("Duration")),         "s"),
]):
    col.markdown(kcard(lbl, val, unit), unsafe_allow_html=True)

st.markdown("<br>", unsafe_allow_html=True)

# ── Status + Recovery ─────────────────────────────────────────────────────────
sc, rc = st.columns([1, 3])
with sc:
    st.markdown('<div class="sec">PATIENT STATUS</div>', unsafe_allow_html=True)
    rv = float(latest.get("Recovery", 0) or 0)
    slabel, scolor, sbg = rec_status(rv)
    icon = {"#e05c6b":"🔴","#f0a04b":"🟡","#00c9b1":"🟢","#9b7fe8":"🟣"}[scolor]
    st.markdown(f"""
    <div style="background:{sbg};border:1px solid {scolor}33;border-radius:12px;
                padding:26px 16px;text-align:center;margin-top:4px">
      <div style="font-size:2.8rem;margin-bottom:8px">{icon}</div>
      <div style="color:{scolor};font-size:1.2rem;font-weight:700">{slabel}</div>
      <div style="color:#2a4a6a;font-size:0.8rem;margin-top:5px">Recovery: {safe(rv)}%</div>
    </div>""", unsafe_allow_html=True)

with rc:
    st.markdown('<div class="sec">RECOVERY PROGRESS</div>', unsafe_allow_html=True)
    if "Recovery" in df.columns:
        fig = line_fig(sessions, df["Recovery"], "Recovery %", TEAL, "Recovery (%)", fill=True)
        fig.add_hline(y=100, line_dash="dot", line_color="#2a4a6a",
                      annotation_text="Baseline 100%", annotation_font_color="#2a4a6a")
        st.plotly_chart(fig, use_container_width=True)

st.markdown("<hr>", unsafe_allow_html=True)

# ── Muscle performance ────────────────────────────────────────────────────────
st.markdown('<div class="sec">MUSCLE PERFORMANCE TRENDS</div>', unsafe_allow_html=True)
c1, c2 = st.columns(2)
with c1:
    st.markdown("**Best MVC Trend**")
    if "BestMVC" in df.columns:
        st.plotly_chart(line_fig(sessions, df["BestMVC"], "Best MVC", AMBER, "MVC (RMS)"),
                        use_container_width=True)
with c2:
    st.markdown("**Fatigue Trend**")
    if "Fatigue" in df.columns:
        st.plotly_chart(line_fig(sessions, df["Fatigue"], "Fatigue %", ROSE, "Fatigue (%)"),
                        use_container_width=True)

c3, c4 = st.columns(2)
with c3:
    st.markdown("**Average Activation Trend**")
    if "AvgActivation" in df.columns:
        st.plotly_chart(line_fig(sessions, df["AvgActivation"],
                                 "Avg Activation %", PURPLE, "Activation (%)"),
                        use_container_width=True)
with c4:
    st.markdown("**Session Duration Trend**")
    if "Duration" in df.columns:
        st.plotly_chart(line_fig(sessions, df["Duration"],
                                 "Duration (s)", SLATE, "Duration (s)"),
                        use_container_width=True)

# ── Volume ────────────────────────────────────────────────────────────────────
st.markdown("<hr>", unsafe_allow_html=True)
st.markdown('<div class="sec">EXERCISE VOLUME</div>', unsafe_allow_html=True)
c5, c6 = st.columns(2)
with c5:
    st.markdown("**Repetitions per Session**")
    if "Reps" in df.columns:
        st.plotly_chart(bar_fig(sessions, df["Reps"], "Reps", TEAL, "Repetitions"),
                        use_container_width=True)
with c6:
    st.markdown("**Hold Time per Session**")
    if "HoldTime" in df.columns:
        st.plotly_chart(bar_fig(sessions, df["HoldTime"],
                                "Hold Time (s)", AMBER, "Hold Time (s)"),
                        use_container_width=True)

# ── Progress analysis ─────────────────────────────────────────────────────────
st.markdown("<hr>", unsafe_allow_html=True)
st.markdown('<div class="sec">PROGRESS ANALYSIS</div>', unsafe_allow_html=True)
if len(df) >= 2:
    f0   = df.iloc[0]
    html = ""
    for metric, lbl, unit, hib in [
        ("BestMVC",       "MVC",              "",    True),
        ("Recovery",      "Recovery",         "%",   True),
        ("Fatigue",       "Fatigue",          "%",   False),
        ("AvgActivation", "Avg Activation",   "%",   True),
        ("Reps",          "Repetition Count", "",    True),
        ("HoldTime",      "Hold Time",        "s",   True),
        ("Duration",      "Session Duration", "s",   True),
        ("StrengthScore", "Strength Score",   "/10", True),
    ]:
        if metric in df.columns:
            html += delta_html(lbl, f0.get(metric, float("nan")),
                               latest.get(metric, float("nan")), unit, hib)
    st.markdown(html, unsafe_allow_html=True)
    r0 = float(f0.get("Recovery", 0) or 0)
    r1 = float(latest.get("Recovery", 0) or 0)
    dr = r1 - r0
    st.markdown(
        f"<br><div style='color:#2a4a6a;font-size:0.86rem'>"
        f"Across <strong style='color:#8aaac8'>{len(df)}</strong> sessions, recovery has "
        f"{'improved' if dr>=0 else 'declined'} by "
        f"<strong style='color:{'#00c9b1' if dr>=0 else '#e05c6b'}'>{abs(dr):.1f}%</strong>"
        f" ({r0:.1f}% → {r1:.1f}%).</div>",
        unsafe_allow_html=True)
else:
    st.info("Upload at least 2 sessions to see progress comparisons.")

# ── Session history ───────────────────────────────────────────────────────────
st.markdown("<hr>", unsafe_allow_html=True)
st.markdown('<div class="sec">SESSION HISTORY</div>', unsafe_allow_html=True)
disp   = df[[c for c in df.columns if c != "DateTime"]].copy()
fc     = disp.select_dtypes(include="float").columns
disp[fc] = disp[fc].round(2)
st.dataframe(disp, use_container_width=True,
             height=min(420, 56 + len(disp)*36), hide_index=True)

# ── Footer ────────────────────────────────────────────────────────────────────
st.markdown(
    "<div style='text-align:center;color:#1a2d40;font-size:0.74rem;"
    "margin-top:36px;padding-top:14px;border-top:1px solid #1a2d40'>"
    "EMG Rehabilitation Monitor · ESP32 / SD Card · Streamlit + Plotly"
    "</div>", unsafe_allow_html=True)
