# lalitkumarthapa-dos-attack-
dos attack code
**import subprocess
import time
from collections import defaultdict
from datetime import datetime
import smtplib
from email.message import EmailMessage

# CONFIGURATION
LOG_FILE = "/var/log/apache2/access.log"  # Change this path to your server's log
THRESHOLD = 100     # max requests per time window
TIME_WINDOW = 10    # seconds
BLOCK_DURATION = 600  # seconds

EMAIL_ALERT = True
EMAIL_RECEIVER = "admin@example.com"
EMAIL_SENDER = "yourbot@example.com"
EMAIL_PASSWORD = "yourpassword"  # Use app password or secure store

blocked_ips = {}

def send_email_alert(ip):
    msg = EmailMessage()
    msg.set_content(f"DoS Attack Detected from IP: {ip}\nBlocked at: {datetime.now()}")
    msg["Subject"] = f"⚠ DoS Detected: {ip} Blocked"
    msg["From"] = EMAIL_SENDER
    msg["To"] = EMAIL_RECEIVER

    try:
        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
            smtp.login(EMAIL_SENDER, EMAIL_PASSWORD)
            smtp.send_message(msg)
        print(f"[ALERT] Email sent for IP: {ip}")
    except Exception as e:
        print(f"[ERROR] Email alert failed: {e}")

def block_ip(ip):
    subprocess.call(["iptables", "-A", "INPUT", "-s", ip, "-j", "DROP"])
    blocked_ips[ip] = time.time()
    print(f"[BLOCKED] {ip} blocked via iptables")
    if EMAIL_ALERT:
        send_email_alert(ip)

def unblock_expired_ips():
    current_time = time.time()
    for ip in list(blocked_ips):
        if current_time - blocked_ips[ip] > BLOCK_DURATION:
            subprocess.call(["iptables", "-D", "INPUT", "-s", ip, "-j", "DROP"])
            print(f"[UNBLOCKED] {ip} unblocked")
            del blocked_ips[ip]

def tail_log_file(filename):
    with open(filename, "r") as f:
        f.seek(0, 2)  # Go to end
        while True:
            line = f.readline()
            if not line:
                time.sleep(0.1)
                continue
            yield line

def monitor_requests():
    ip_tracker = defaultdict(list)
    for line in tail_log_file(LOG_FILE):
        try:
            ip = line.split()[0]
            now = time.time()
            ip_tracker[ip].append(now)

            # Keep only recent requests
            ip_tracker[ip] = [t for t in ip_tracker[ip] if now - t <= TIME_WINDOW]

            if ip not in blocked_ips and len(ip_tracker[ip]) > THRESHOLD:
                block_ip(ip)

            unblock_expired_ips()
        except Exception as e:
            print(f"[ERROR] Log processing failed: {e}")

if _name_ == "_main_":
    print("[*] DoS Protection Script Running...")
    monitor_requests()**
