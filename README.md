# SmartReply AI

SmartReply AI is an intelligent, stateless email response system that leverages SMTP, IMAP, Procmail, and OpenAI's ChatGPT to automatically process and respond to incoming emails. Built with Python, SmartReply AI ensures each email is handled independently, promoting scalability and efficiency.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Setting Up Procmail](#1-setting-up-procmail)
  - [2. Python Script to Process Emails](#2-python-script-to-process-emails)
  - [3. Making the Script Executable](#3-making-the-script-executable)
  - [4. Automating the Process](#4-automating-the-process)
- [Security Considerations](#security-considerations)
- [Handling Edge Cases and Enhancements](#handling-edge-cases-and-enhancements)
- [Scheduling the Script (Optional)](#scheduling-the-script-optional)
- [Conclusion](#conclusion)
- [Additional Resources](#additional-resources)
- [License](#license)

## Features

- **Automated Email Responses:** Automatically fetches and responds to incoming emails using AI-generated replies.
- **Stateless Processing:** Each email is processed independently without maintaining session states.
- **Customizable Filters:** Utilize Procmail to filter and route specific emails for processing.
- **Secure Credential Management:** Supports environment variables for storing sensitive information.
- **Robust Error Handling and Logging:** Comprehensive logging for monitoring and troubleshooting.
- **Scalable Architecture:** Easily adaptable for higher email volumes and extended functionalities.

## Prerequisites

Before setting up SmartReply AI, ensure you have the following:

- **Python 3.x** installed on your system.
- An **email account** that supports IMAP and SMTP.
- **Procmail** installed and configured on your mail server.
- An **OpenAI API Key** for accessing ChatGPT.
- **Git** for cloning the repository (optional but recommended).

## Installation

1. **Clone the Repository**

   ```bash
   git clone https://github.com/yourusername/SmartReply-AI.git
   cd SmartReply-AI
   ```

2. **Create a Virtual Environment**

   It's good practice to use a virtual environment to manage dependencies.

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Install Dependencies**

   ```bash
   pip install -r requirements.txt
   ```

   **`requirements.txt`**

   ```plaintext
   imaplib2
   email
   smtplib
   openai
   python-dotenv
   ```

## Configuration

### 1. Setting Up Procmail

Procmail filters incoming emails and directs them to the SmartReply AI processing script.

**Sample `.procmailrc` Configuration:**

```bash
# ~/.procmailrc

# Set default mailbox
DEFAULT=$HOME/Maildir/

# Log file
LOGFILE=$HOME/procmail.log
VERBOSE=on

# Rule to process specific emails
:0
* ^Subject:.*Process with AI
| /usr/bin/python3 /path/to/SmartReply-AI/process_email.py
```

**Explanation:**

- **DEFAULT:** Specifies the default mailbox location.
- **LOGFILE:** Path to the Procmail log file.
- **VERBOSE:** Enables verbose logging.
- **Rule:** Captures emails with subjects containing "Process with AI" and pipes them to the Python processing script.

### 2. Python Script to Process Emails

Create a Python script (`process_email.py`) that performs the following:

1. **Fetches incoming emails via IMAP.**
2. **Generates AI responses using ChatGPT.**
3. **Sends the responses via SMTP.**

**Sample `process_email.py`:**

```python
#!/usr/bin/env python3

import imaplib
import email
import smtplib
import openai
import os
import sys
import logging
from email.mime.text import MIMEText
from email.header import decode_header
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Setup logging
logging.basicConfig(
    filename='/path/to/your/logfile.log',
    level=logging.INFO,
    format='%(asctime)s %(levelname)s:%(message)s'
)

# Configuration
IMAP_SERVER = os.getenv('IMAP_SERVER')
IMAP_PORT = int(os.getenv('IMAP_PORT', 993))
SMTP_SERVER = os.getenv('SMTP_SERVER')
SMTP_PORT = int(os.getenv('SMTP_PORT', 587))
EMAIL_ACCOUNT = os.getenv('EMAIL_ACCOUNT')
EMAIL_PASSWORD = os.getenv('EMAIL_PASSWORD')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')

def decode_mime_words(s):
    try:
        return ''.join(
            word.decode(encoding or 'utf-8') if isinstance(word, bytes) else word
            for word, encoding in decode_header(s)
        )
    except Exception as e:
        logging.error(f"Error decoding header: {e}")
        return s

def fetch_emails():
    try:
        mail = imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT)
        mail.login(EMAIL_ACCOUNT, EMAIL_PASSWORD)
        mail.select('inbox')

        # Search for all unseen emails
        status, messages = mail.search(None, '(UNSEEN)')
        if status != 'OK':
            logging.error("Failed to search emails.")
            return []

        email_ids = messages[0].split()
        emails = []

        for eid in email_ids:
            status, msg_data = mail.fetch(eid, '(RFC822)')
            if status != 'OK':
                logging.error(f"Failed to fetch email ID {eid}.")
                continue

            for response_part in msg_data:
                if isinstance(response_part, tuple):
                    msg = email.message_from_bytes(response_part[1])
                    subject = decode_mime_words(msg['Subject'])
                    from_ = decode_mime_words(msg['From'])
                    # Get email body
                    if msg.is_multipart():
                        for part in msg.walk():
                            if part.get_content_type() == 'text/plain':
                                body = part.get_payload(decode=True).decode(errors='ignore')
                                break
                    else:
                        body = msg.get_payload(decode=True).decode(errors='ignore')

                    emails.append({
                        'id': eid,
                        'subject': subject,
                        'from': from_,
                        'body': body
                    })
        mail.logout()
        return emails
    except Exception as e:
        logging.error(f"Error fetching emails: {e}")
        return []

def generate_ai_response(prompt):
    try:
        openai.api_key = OPENAI_API_KEY
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": prompt}
            ],
            max_tokens=150
        )
        return response.choices[0].message['content'].strip()
    except Exception as e:
        logging.error(f"Error generating AI response: {e}")
        return "I'm sorry, I couldn't process your request at this time."

def send_email(to_address, subject, body):
    try:
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = EMAIL_ACCOUNT
        msg['To'] = to_address

        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(EMAIL_ACCOUNT, EMAIL_PASSWORD)
            server.send_message(msg)
        logging.info(f"Sent email to {to_address}")
    except Exception as e:
        logging.error(f"Error sending email to {to_address}: {e}")

def main():
    emails = fetch_emails()
    if not emails:
        logging.info("No new emails to process.")
        return

    for email_data in emails:
        try:
            user_query = email_data['body']
            ai_response = generate_ai_response(user_query)

            response_subject = f"Re: {email_data['subject']}"
            response_body = ai_response
            recipient = email_data['from']

            send_email(recipient, response_subject, response_body)
            logging.info(f"Responded to {recipient} regarding '{email_data['subject']}'")
        except Exception as e:
            logging.error(f"Error processing email ID {email_data['id']}: {e}")

if __name__ == "__main__":
    main()
```

**Explanation:**

1. **Environment Variables:** Utilizes the `python-dotenv` package to manage sensitive credentials securely.
2. **Logging:** Records activities and errors to a log file for monitoring and troubleshooting.
3. **Email Fetching:** Connects to the IMAP server, fetches unseen emails, and extracts relevant details.
4. **AI Response Generation:** Sends the email content to ChatGPT and retrieves the generated response.
5. **Email Sending:** Composes and sends the response back to the original sender via SMTP.

### 3. Making the Script Executable

Ensure your Python script has executable permissions.

```bash
chmod +x /path/to/SmartReply-AI/process_email.py
```

### 4. Automating the Process

Procmail will automatically invoke the Python script whenever an email matches the specified rule. Ensure Procmail is correctly installed and that your `.procmailrc` is properly configured.

## Security Considerations

- **Secure Storage of Credentials:**
  - Store sensitive information like email credentials and OpenAI API keys in environment variables or a `.env` file.
  - **Example `.env` File:**

    ```dotenv
    IMAP_SERVER=imap.yourmail.com
    IMAP_PORT=993
    SMTP_SERVER=smtp.yourmail.com
    SMTP_PORT=587
    EMAIL_ACCOUNT=your_email@yourmail.com
    EMAIL_PASSWORD=your_password
    OPENAI_API_KEY=your_openai_api_key
    ```

  - **Note:** Ensure the `.env` file is excluded from version control by adding it to `.gitignore`.

- **Protect Your OpenAI API Key:**
  - Keep your API key confidential.
  - Rotate the key periodically and monitor its usage.

- **File Permissions:**
  - Restrict access to scripts and log files to prevent unauthorized access.

## Handling Edge Cases and Enhancements

- **Error Handling:**
  - Implement try-except blocks to handle network errors, API failures, and other exceptions gracefully.

- **Logging:**
  - Maintain comprehensive logs to monitor system behavior and troubleshoot issues effectively.

- **Rate Limiting:**
  - Adhere to OpenAI's rate limits to prevent exceeding usage quotas. Implement retry mechanisms or backoff strategies as needed.

- **Email Formatting:**
  - Enhance email parsing to handle HTML content, attachments, and various email formats.

- **Filtering Specific Emails:**
  - Customize Procmail rules to process only relevant emails based on criteria like sender, subject, keywords, etc.

## Scheduling the Script (Optional)

If you prefer not to use Procmail to trigger the script, you can schedule the Python script to run at regular intervals using `cron`.

**Sample `crontab` Entry:**

```bash
# Run the script every 5 minutes
*/5 * * * * /usr/bin/python3 /path/to/SmartReply-AI/process_email.py >> /path/to/SmartReply-AI/cron.log 2>&1
```

**Note:** Using Procmail is generally more immediate and efficient for processing incoming emails as they arrive.

## Conclusion

SmartReply AI offers a robust solution for automating email responses using the power of AI. By ensuring stateless processing, it maintains scalability and efficiency, making it suitable for both personal and organizational use. The integration of SMTP, IMAP, Procmail, and ChatGPT creates a seamless workflow that can be customized to fit various email processing needs.

---

## Additional Resources

- [Procmail Documentation](http://www.procmail.org/)
- [OpenAI API Documentation](https://beta.openai.com/docs/)
- [Python `imaplib` Module](https://docs.python.org/3/library/imaplib.html)
- [Python `smtplib` Module](https://docs.python.org/3/library/smtplib.html)
- [Python `email` Module](https://docs.python.org/3/library/email.html)
- [python-dotenv Documentation](https://saurabh-kumar.com/python-dotenv/)

## License

This project is licensed under the [MIT License](LICENSE).

---

**Disclaimer:** Ensure that your use of SmartReply AI complies with all relevant data protection regulations and respects user privacy. Always inform your email recipients about automated responses and obtain necessary consents where required.
