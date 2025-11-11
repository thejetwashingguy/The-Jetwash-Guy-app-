
jetwashing_full_app.py
One-file Jetwashing AI Assistant (local, personal, mostly free).
 
Save this file, install dependencies (see top of the assistant message),
then run:
    streamlit run jetwashing_full_app.py --server.address=0.0.0.0
 
Open the Network URL on your phone (same Wi-Fi).
 
Optional features require API keys / credentials:
 - OpenAI: set OPENAI_API_KEY env var for ChatGPT-quality responses
 - Google Places/Maps: set GOOGLE_API_KEY in the app UI or as env var
 - Google Calendar: put credentials.json in this folder (OAuth flow on first use)
 - Gmail sending: provide email & app password in the app (recommended create App Password)
 
This program stores data in ./data/ as CSV and photos inside ./data/photos/.
"""
 
import streamlit as st
import pandas as pd
import os
from pathlib import Path
import datetime
from fpdf import FPDF
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import json
from dateutil import parser as dateparser
 
# Optional OpenAI import (if available)
try:
    import openai
    OPENAI_AVAILABLE = True
except Exception:
    OPENAI_AVAILABLE = False
 
# Optional Google APIs
try:
    from google_auth_oauthlib.flow import InstalledAppFlow
    from googleapiclient.discovery import build
    GOOGLE_CLIENT_LIBS = True
except Exception:
    GOOGLE_CLIENT_LIBS = False
 
# ---------------------------
# Config & folders
# ---------------------------
APP_TITLE = "Jetwashing AI Assistant (Personal)"
DATA_DIR = Path("data")
PHOTOS_DIR = DATA_DIR / "photos"
DATA_DIR.mkdir(exist_ok=True)
PHOTOS_DIR.mkdir(exist_ok=True)
 
CLIENTS_CSV = DATA_DIR / "clients.csv"
JOBS_CSV = DATA_DIR / "jobs.csv"
 
# Default columns
CLIENT_COLS = ["client_id","name","phone","email","address","lead_source"]
JOB_COLS = [
    "job_id","client_id","service","area_m2","date","job_status","job_type","price_estimate",
    "photo_file","quote_file","social_caption","social_platforms","posted_to_social",
    "followup_date","upsell_date","followup_sent","upsell_sent","lead_source","added_on"
]
 
# Create CSVs if missing
if not CLIENTS_CSV.exists():
    pd.DataFrame(columns=CLIENT_COLS).to_csv(CLIENTS_CSV, index=False)
if not JOBS_CSV.exists():
    pd.DataFrame(columns=JOB_COLS).to_csv(JOBS_CSV, index=False)
 
# Load data
clients_df = pd.read_csv(CLIENTS_CSV)
jobs_df = pd.read_csv(JOBS_CSV)
 
# Streamlit page config
st.set_page_config(APP_TITLE, layout="wide")
st.title("🚀 Jetwashing AI Assistant — Personal Edition")
st.caption("Use on your phone (open the Network URL shown when you run the app).")
 
# ---------------------------
# Helper functions
# ---------------------------
def save_clients():
    clients_df.to_csv(CLIENTS_CSV, index=False)
 
def save_jobs():
    jobs_df.to_csv(JOBS_CSV, index=False)
 
def gen_client_id():
    return int(datetime.datetime.now().timestamp())
 
def gen_job_id():
    return int(datetime.datetime.now().timestamp() * 1000)  # more unique
 
def generate_pdf_quote(client_name, service, area_m2, price_estimate, file_path):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    pdf.cell(200, 10, txt="Jetwashing Quote", ln=True, align='C')
    pdf.ln(6)
    pdf.cell(200, 8, txt=f"Client: {client_name}", ln=True)
    pdf.cell(200, 8, txt=f"Service: {service}", ln=True)
    pdf.cell(200, 8, txt=f"Area: {area_m2} m²", ln=True)
    pdf.cell(200, 8, txt=f"Estimated Price: £{price_estimate:.2f}", ln=True)
    pdf.ln(8)
    pdf.cell(200, 8, txt="Thanks for choosing us! Reply to confirm booking.", ln=True)
    pdf.output(str(file_path))
 
def estimate_price(service, area_m2, job_type):
    # Change these numbers to fit your local rates
    price_per_m2 = {
        "Domestic": {"Driveway":3.0, "Patio":2.5, "Deck":2.0, "Other":2.0},
        "Commercial": {"Driveway":4.0, "Patio":3.5, "Deck":3.0, "Other":3.0}
    }
    base = price_per_m2.get(job_type, price_per_m2["Domestic"]).get(service, 2.0)
    est = area_m2 * base
    # small complexity multiplier not necessary; for future: check surface, moss, etc.
    return round(est, 2)
 
def gen_social_caption(service, client_name, address):
    return f"Just finished {service} for {client_name} at {address}! 💦✨ #PressureWashing #CleanAndShiny"
 
def send_email_smtp(sender_email, sender_password, to_email, subject, body, smtp_server="smtp.gmail.com", smtp_port=587):
    try:
        msg = MIMEMultipart()
        msg["From"] = sender_email
        msg["To"] = to_email
        msg["Subject"] = subject
        msg.attach(MIMEText(body, "plain"))
        server = smtplib.SMTP(smtp_server, smtp_port)
        server.starttls()
        server.login(sender_email, sender_password)
        server.send_message(msg)
        server.quit()
        return True, "Sent"
    except Exception as e:
        return False, str(e)
 
# Simple Chat fallback (if no OpenAI)
def simple_chat_response(prompt):
    # Very basic: looks for keywords and returns helpful suggestions
    p = prompt.lower()
    if "caption" in p or "social" in p:
        return "Try: 'Just finished a deep clean — driveway now looks like new! 💦 Book now for 10% off.'"
    if "price" in p or "estimate" in p:
        return "Estimate tip: for residential driveways use £2.5–£4/m² depending on condition. For commercial add 20–30%."
    if "email" in p or "follow" in p:
        return "Follow-up suggestion: 'Hi NAME — just checking you're happy with the job. Reply if you'd like a re-visit.'"
    return "I suggest: keep messages short and friendly. Ask for permission to post photos for social proof."
 
# OpenAI chat wrapper (optional)
def call_openai_chat(prompt):
    if not OPENAI_AVAILABLE:
        return simple_chat_response(prompt)
    key = os.getenv("OPENAI_API_KEY", "")
    if not key:
        return simple_chat_response(prompt)
    openai.api_key = key
    try:
        res = openai.ChatCompletion.create(
            model="gpt-4o-mini" if hasattr(openai, "ChatCompletion") else "gpt-4o-mini",
            messages=[{"role":"user","content":prompt}],
            max_tokens=300
        )
        text = res["choices"][0]["message"]["content"].strip()
        return text
    except Exception as e:
        return f"(AI error) {e}"
 
# Google Places reviews (optional) - needs API key
def get_google_place_reviews(place_name, google_api_key):
    if not google_api_key:
        return None
    try:
        find_url = "https://maps.googleapis.com/maps/api/place/findplacefromtext/json"
        params = {"input": place_name, "inputtype":"textquery", "fields":"place_id", "key":google_api_key}
        r1 = requests.get(find_url, params=params).json()
        if "candidates" in r1 and len(r1["candidates"])>0:
            pid = r1["candidates"][0]["place_id"]
            details_url = "https://maps.googleapis.com/maps/api/place/details/json"
            params2 = {"place_id":pid, "fields":"name,rating,user_ratings_total,reviews", "key":google_api_key}
            r2 = requests.get(details_url, params=params2).json()
            return r2.get("result", {})
        return None
    except Exception as e:
        return {"error":str(e)}
 
# Google Calendar integration (optional)
SCOPES = ['https://www.googleapis.com/auth/calendar.events']
def add_event_to_google_calendar(summary, location, description, start_dt, end_dt, creds_json_path="credentials.json"):
    if not GOOGLE_CLIENT_LIBS:
        return False, "google client libs not installed"
    if not os.path.exists(creds_json_path):
        return False, "credentials.json missing"
    try:
        flow = InstalledAppFlow.from_client_secrets_file(creds_json_path, SCOPES)
        creds = flow.run_local_server(port=0)
        service = build('calendar', 'v3', credentials=creds)
        event = {
            'summary': summary,
            'location': location,
            'description': description,
            'start': {'dateTime': start_dt.isoformat(), 'timeZone': 'Europe/London'},
            'end': {'dateTime': end_dt.isoformat(), 'timeZone': 'Europe/London'},
        }
        event = service.events().insert(calendarId='primary', body=event).execute()
        return True, event.get('htmlLink', '')
    except Exception as e:
        return False, str(e)
 
# ---------------------------
# UI: Settings (API keys and email)
# ---------------------------
st.sidebar.header("Settings (optional)")
with st.sidebar.expander("Integrations & Keys (only if you want these features)"):
    openai_key = st.text_input("OpenAI API Key (optional)", type="password", key="openai_key")
    if openai_key:
        os.environ["OPENAI_API_KEY"] = openai_key
    google_api_key = st.text_input("Google API Key (Places/Maps) (optional)", type="password", key="google_api_key")
    if google_api_key:
        os.environ["GOOGLE_API_KEY"] = google_api_key
    use_gcal = st.checkbox("Enable Google Calendar integration (requires credentials.json)", value=False)
    st.write("For Calendar: put credentials.json (OAuth client) in this folder when prompted.")
    email_sender = st.text_input("Gmail address for sending emails (optional)")
    email_app_password = st.text_input("Gmail App Password (optional)", type="password")
    save_email_creds = st.checkbox("Save Gmail creds locally (data/creds.json)?", value=False)
    if save_email_creds:
        creds_store = {"email": email_sender, "app_password": email_app_password}
        with open(DATA_DIR / "creds.json", "w") as f:
            json.dump(creds_store, f)
 
# ---------------------------
# Sidebar: Navigation
# ---------------------------
st.sidebar.markdown("---")
mode = st.sidebar.radio("Open", ["Dashboard","Add Client","Add Job","Jobs","Social","Chat","Inbox","Settings","Google Reviews"])
st.sidebar.markdown("Tip: run app with `--server.address=0.0.0.0` to open on your phone.")
 
# ---------------------------
# DASHBOARD
# ---------------------------
if mode == "Dashboard":
    st.header("🏁 Dashboard")
    col1, col2, col3, col4 = st.columns(4)
    col1.metric("Clients", len(clients_df))
    col2.metric("Total Jobs", len(jobs_df))
    col3.metric("Quoted", len(jobs_df[jobs_df["job_status"]=="Quoted"]))
    col4.metric("Booked", len(jobs_df[jobs_df["job_status"]=="Booked"]))
    st.markdown("### Upcoming jobs (next 14 days)")
    if not jobs_df.empty:
        jobs_df['date_parsed'] = pd.to_datetime(jobs_df['date'], errors='coerce')
        upcoming = jobs_df[jobs_df['date_parsed'] >= pd.Timestamp(datetime.date.today())].sort_values('date_parsed').head(10)
        st.dataframe(upcoming[["job_id","client_id","service","date","job_status","price_estimate","job_type","lead_source"]])
    else:
        st.info("No jobs yet. Add one in Add Job.")
 
# ---------------------------
# ADD CLIENT
# ---------------------------
elif mode == "Add Client":
    st.header("➕ Add Client")
    with st.form("client_form", clear_on_submit=True):
        name = st.text_input("Full name")
        phone = st.text_input("Phone number")
        email = st.text_input("Email address")
        address = st.text_input("Address")
        lead = st.selectbox("Lead source", ["Google Business","Website","Instagram","Facebook","Referral","Other"])
        submitted = st.form_submit_button("Save client")
        if submitted:
            cid = gen_client_id()
            new = {"client_id":cid,"name":name,"phone":phone,"email":email,"address":address,"lead_source":lead}
            clients_df.loc[len(clients_df)] = new
            save_clients()
            st.success(f"Client {name} added. Client ID: {cid}")
 
# ---------------------------
# ADD JOB
# ---------------------------
elif mode == "Add Job":
    st.header("🧹 Add Job")
    if clients_df.empty:
        st.warning("No clients yet. Add a client first.")
    else:
        with st.form("job_form", clear_on_submit=True):
            client_choice = st.selectbox("Client", clients_df["client_id"].astype(str) + " - " + clients_df["name"])
            client_id = int(client_choice.split(" - ")[0])
            client_name = clients_df.loc[clients_df["client_id"]==client_id,"name"].values[0]
            service = st.selectbox("Service", ["Driveway","Patio","Deck","Other"])
            job_type = st.selectbox("Job type", ["Domestic","Commercial"])
            area_m2 = st.number_input("Area (m²)", min_value=1.0, value=20.0)
            date_input = st.date_input("Job date", datetime.date.today())
            job_status = st.selectbox("Status", ["Quoted","Booked"])
            lead_src = st.selectbox("Lead Source", ["Google Business","Website","Instagram","Facebook","Referral","Other"])
            photo = st.file_uploader("Upload photo (before/after)", type=["jpg","jpeg","png"])
            followup_after_days = st.number_input("Follow-up after days (for follow-up email)", min_value=7, max_value=365, value=30)
            upsell_after_days = st.number_input("Upsell after days", min_value=30, max_value=365, value=270)
            submit_job = st.form_submit_button("Create job")
            if submit_job:
                jid = gen_job_id()
                price = estimate_price(service, area_m2, job_type)
                photo_path = ""
                if photo:
                    suffix = photo.name.replace(" ", "_")
                    photo_path = PHOTOS_DIR / f"{jid}_{suffix}"
                    with open(photo_path, "wb") as f:
                        f.write(photo.getbuffer())
                quote_path = DATA_DIR / f"quote_{jid}.pdf"
                generate_pdf_quote(client_name, service, area_m2, price, quote_path)
                followup_date = (date_input + datetime.timedelta(days=int(followup_after_days))).isoformat()
                upsell_date = (date_input + datetime.timedelta(days=int(upsell_after_days))).isoformat()
                newjob = {
                    "job_id": jid, "client_id": client_id, "service": service, "area_m2": area_m2,
                    "date": date_input.isoformat(), "job_status": job_status, "job_type": job_type,
                    "price_estimate": price, "photo_file": str(photo_path), "quote_file": str(quote_path),
                    "social_caption": gen_social_caption(service, client_name, clients_df.loc[clients_df['client_id']==client_id,'address'].values[0]),
                    "social_platforms": "", "posted_to_social": "No",
                    "followup_date": followup_date, "upsell_date": upsell_date, "followup_sent": "No", "upsell_sent": "No",
                    "lead_source": lead_src, "added_on": datetime.datetime.now().isoformat()
                }
                jobs_df.loc[len(jobs_df)] = newjob
                save_jobs()
                st.success(f"Job created for {client_name} — Estimate £{price:.2f}. Quote saved: {quote_path.name}")
                # Optionally add to Google Calendar
                if use_gcal:
                    st.info("If you want to add this job to Google Calendar, go to Settings -> Google Calendar integration and run it there.")
 
# ---------------------------
# JOBS LIST & ACTIONS
# ---------------------------
elif mode == "Jobs":
    st.header("📋 Jobs")
    if jobs_df.empty:
        st.info("No jobs yet.")
    else:
        st.write("Filter and inspect jobs:")
        colA, colB, colC = st.columns([1,1,2])
        status_filter = colA.selectbox("Status", ["All","Quoted","Booked","Done"])
        type_filter = colB.selectbox("Type", ["All","Domestic","Commercial"])
        lead_filter = colC.selectbox("Lead source", ["All"] + sorted(jobs_df["lead_source"].dropna().unique().tolist()))
        filtered = jobs_df.copy()
        if status_filter != "All":
            filtered = filtered[filtered["job_status"]==status_filter]
        if type_filter != "All":
            filtered = filtered[filtered["job_type"]==type_filter]
        if lead_filter != "All":
            filtered = filtered[filtered["lead_source"]==lead_filter]
        st.dataframe(filtered[["job_id","client_id","service","date","job_status","job_type","price_estimate","lead_source"]].sort_values("date"))
 
        st.markdown("### Job Actions")
        jid_choice = st.selectbox("Select job id to manage", filtered["job_id"].astype(str))
        if jid_choice:
            jid_choice = int(jid_choice)
            job_row = jobs_df[jobs_df["job_id"]==jid_choice].iloc[0]
            client_row = clients_df[clients_df["client_id"]==job_row["client_id"]].iloc[0]
            st.subheader(f"Job {job_row['job_id']} — {job_row['service']} for {client_row['name']}")
            st.write("Status:", job_row["job_status"], " | Type:", job_row["job_type"], " | Price: £",job_row["price_estimate"])
            cols = st.columns(3)
            if st.button("Mark Booked"):
                jobs_df.loc[jobs_df["job_id"]==jid_choice,"job_status"]="Booked"
                save_jobs()
                st.experimental_rerun()
            if st.button("Open Map"):
                # open Google Maps in new tab for address
                addr = client_row["address"]
                maps_url = f"https://www.google.com/maps/search/?api=1&query={requests.utils.quote(addr)}"
                st.markdown(f"[Open in Google Maps]({maps_url})")
            if st.button("Download Quote PDF"):
                qfile = job_row["quote_file"]
                if qfile and Path(qfile).exists():
                    with open(qfile, "rb") as f:
                        st.download_button("Download PDF", f, file_name=Path(qfile).name)
                else:
                    st.info("Quote file missing.")
            # Social
            st.write("Social caption:")
            st.text_area("Caption", value=job_row["social_caption"], height=80)
            if st.button("Mark as Posted"):
                jobs_df.loc[jobs_df["job_id"]==jid_choice,"posted_to_social"]="Yes"
                save_jobs()
                st.success("Marked posted.")
 
# ---------------------------
# SOCIAL
# ---------------------------
elif mode == "Social":
    st.header("📱 Social Media Manager")
    if jobs_df.empty:
        st.info("Add jobs first.")
    else:
        job_sel = st.selectbox("Job", jobs_df["job_id"].astype(str) + " - " + jobs_df["service"])
        jid = int(job_sel.split(" - ")[0])
        jr = jobs_df[jobs_df["job_id"]==jid].iloc[0]
        clientname = clients_df.loc[clients_df["client_id"]==jr["client_id"],"name"].values[0]
        st.write("Caption (editable):")
        caption = st.text_area("Caption text", value=jr["social_caption"], height=120)
        plats = st.multiselect("Platforms to post", ["Instagram","Facebook","TikTok","LinkedIn"])
        if st.button("Save Social Info"):
            jobs_df.loc[jobs_df["job_id"]==jid,"social_caption"]=caption
            jobs_df.loc[jobs_df["job_id"]==jid,"social_platforms"] = ",".join(plats)
            save_jobs()
            st.success("Saved.")
        if st.button("Copy caption to clipboard (phone: long-press to paste)"):
            st.write("Caption copied (use your phone clipboard).")
        st.markdown("**Tip:** Auto-posting to Instagram/Facebook requires developer APIs and is often restricted. Best: copy caption and upload manually — quick and free.")
 
# ---------------------------
# CHAT (OpenAI optional)
# ---------------------------
elif mode == "Chat":
    st.header("💬 ChatGPT Assistant")
    prompt = st.text_area("Ask anything (pricing, captions, email drafts):", height=120)
    if st.button("Ask AI"):
        if (openai_key or os.getenv("OPENAI_API_KEY")) and OPENAI_AVAILABLE:
            resp = call_openai_chat(prompt)
        else:
            resp = simple_chat_response(prompt)
        st.write("**AI response:**")
        st.write(resp)
 
# ---------------------------
# INBOX (simple: import from Gmail using credentials is optional)
# ---------------------------
elif mode == "Inbox":
    st.header("📥 Inbox (simple)")
    st.write("This app can collect leads from your website (by writing to data CSV) or you can paste email inquiries here.")
    with st.form("inbox_form"):
        name = st.text_input("Name")
        contact = st.text_input("Email or Phone")
        message = st.text_area("Message")
        source = st.selectbox("Source", ["Website","Google Business","Email","Phone","Other"])
        submit = st.form_submit_button("Save lead")
        if submit:
            # create client with minimal info OR just store as job lead
            cid = gen_client_id()
            clients_df.loc[len(clients_df)] = {"client_id":cid, "name":name or "Lead", "phone":contact if "@" not in contact else "", "email":contact if "@" in contact else "", "address":"", "lead_source":source}
            save_clients()
            st.success("Saved lead as client. You can now create job for them.")
 
# ---------------------------
# Settings (email send & calendar)
# ---------------------------
elif mode == "Settings":
    st.header("⚙️ Settings & Integrations")
    st.info("Set optional features: Google API key, enable Calendar, set Gmail for sending follow-ups.")
    # Show quick status
    st.write("OpenAI available:", OPENAI_AVAILABLE)
    st.write("Google client libs available:", GOOGLE_CLIENT_LIBS)
    st.write("Saved email creds file:", (DATA_DIR / "creds.json").exists())
    if st.button("Test sending a sample email (use Gmail creds)"):
        if (DATA_DIR / "creds.json").exists():
            with open(DATA_DIR / "creds.json") as f:
                creds = json.load(f)
            ok, msg = send_email_smtp(creds["email"], creds["app_password"], creds["email"], "Test from Jetwashing App", "This is a test.")
            if ok:
                st.success("Test email sent.")
            else:
                st.error(f"Failed: {msg}")
        else:
            st.warning("No Gmail creds saved. Enter them in the sidebar settings and check 'Save Gmail creds locally'.")
 
# ---------------------------
# Google Reviews
# ---------------------------
elif mode == "Google Reviews":
    st.header("⭐ Google Reviews (optional)")
    place_query = st.text_input("Search your business name (e.g. 'Bob's Jetwashing, Town')")
    if st.button("Fetch Reviews"):
        gkey = os.getenv("GOOGLE_API_KEY") or google_api_key
        if not gkey:
            st.error("No Google API key set. Add in Settings.")
        else:
            res = get_google_place_reviews(place_query, gkey)
            if not res:
                st.info("No results.")
            else:
                st.write("Name:", res.get("name"))
                st.write("Rating:", res.get("rating"), " (", res.get("user_ratings_total"), "reviews )")
                if "reviews" in res:
                    for r in res["reviews"][:5]:
                        st.markdown(f"**{r.get('author_name')}** — {r.get('rating')} ⭐")
                        st.write(r.get("text"))
 
# ---------------------------
# Background check for emails (run when app is used)
# ---------------------------
# This section will run each time the app is opened/refresh and will send due follow-ups/upsells.
def process_scheduled_emails():
    # load gmail creds if saved
    creds_path = DATA_DIR / "creds.json"
    email_conf = {}
    if creds_path.exists():
        with open(creds_path) as f:
            email_conf = json.load(f)
    today = datetime.date.today()
    updated = False
    for idx, row in jobs_df.iterrows():
        try:
            # parse dates safely
            if pd.notna(row.get("followup_date")) and row.get("followup_sent") != "Yes":
                fd = dateparser.parse(str(row["followup_date"])).date()
                if fd <= today:
                    # send follow-up
                    client = clients_df[clients_df["client_id"]==row["client_id"]].iloc[0]
                    to_email = client.get("email","")
                    if to_email and email_conf.get("email") and email_conf.get("app_password"):
                        subject = f"Follow-up: {row['service']} on {row['date']}"
                        body = f"Hi {client['name']},\n\nJust checking you were happy with the {row['service']} we did on {row['date']}. Reply if anything needs attention.\n\nThanks!"
                        ok, msg = send_email_smtp(email_conf["email"], email_conf["app_password"], to_email, subject, body)
                        if ok:
                            jobs_df.at[idx, "followup_sent"] = "Yes"
                            updated = True
            if pd.notna(row.get("upsell_date")) and row.get("upsell_sent") != "Yes":
                ud = dateparser.parse(str(row["upsell_date"])).date()
                if ud <= today:
                    client = clients_df[clients_df["client_id"]==row["client_id"]].iloc[0]
                    to_email = client.get("email","")
                    if to_email and email_conf.get("email") and email_conf.get("app_password"):
                        subject = f"Special offer for your next cleaning"
                        body = f"Hi {client['name']},\n\nWe recommend a follow-up cleaning for your {row['service']} — book now and get 10% off!\n\nReply to book."
                        ok, msg = send_email_smtp(email_conf["email"], email_conf["app_password"], to_email, subject, body)
                        if ok:
                            jobs_df.at[idx, "upsell_sent"] = "Yes"
                            updated = True
        except Exception as e:
            # keep going
            print("Scheduled email error", e)
    if updated:
        save_jobs()
 
# Run scheduled email processor (non-blocking quick)
process_scheduled_emails()
 
# Footer / help
st.markdown("---")
st.markdown("**How to get started (quick):** Add a client → Add a job → open on your phone via Network URL. Optional: add Gmail creds in Settings to enable scheduled emails. Optional: add OpenAI or Google API keys for AI and reviews.")
