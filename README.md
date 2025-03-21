import os
import re
import requests
import whois
import time
from urllib.parse import urlparse

# Function to check and install missing dependencies
def install_dependencies():
    try:
        import requests, whois, urllib3
    except ModuleNotFoundError:
        print("[INFO] Some dependencies are missing. Installing now...")
        os.system("pip3 install requests whois urllib3")

# Install missing dependencies before proceeding
install_dependencies()

# Function to check URL for phishing indicators
def check_url_suspicious(url):
    suspicious_patterns = [
        r'\d+\.\d+\.\d+\.\d+',  # IP address in URL
        r'@',  # '@' in URL (used for redirection attacks)
        r'https?://[a-zA-Z0-9.-]+-[a-zA-Z0-9.-]+/',  # Hyphenated domain names
        r'https?://(www\.)?[a-zA-Z0-9]+\.[a-zA-Z]{2,6}/[a-zA-Z0-9]{10,}',  # Randomized paths
    ]
    for pattern in suspicious_patterns:
        if re.search(pattern, url):
            return True
    return False

# Function to check domain registration details
def check_domain_info(url):
    domain = urlparse(url).netloc
    try:
        domain_info = whois.whois(domain)
        if isinstance(domain_info.creation_date, list):
            creation_date = domain_info.creation_date[0]
        else:
            creation_date = domain_info.creation_date

        return f"[INFO] Domain {domain} was created on {creation_date}"
    except Exception as e:
        return f"[WARNING] Unable to fetch WHOIS info: {e}"

# Function to check URL against VirusTotal API (replace 'YOUR_API_KEY' with a valid key)
def check_virustotal(url):
    API_KEY = "YOUR_API_KEY"  # Replace with your VirusTotal API key
    VT_URL = "https://www.virustotal.com/api/v3/urls"

    headers = {"x-apikey": API_KEY}
    data = {"url": url}

    try:
        response = requests.post(VT_URL, headers=headers, data=data)
        if response.status_code == 200:
            result = response.json()
            if result.get("data"):
                analysis_id = result["data"]["id"]
                analysis_url = f"https://www.virustotal.com/gui/url/{analysis_id}/detection"
                return f"[INFO] Check VirusTotal Report: {analysis_url}"
        return "[INFO] URL not found in VirusTotal database."
    except Exception as e:
        return f"[ERROR] VirusTotal API Error: {e}"

# Function to write report
def write_report(url, results):
    report_file = "phishing_report.txt"
    with open(report_file, "a") as file:
        file.write(f"\n--- Phishing Scan Report ({time.strftime('%Y-%m-%d %H:%M:%S')}) ---\n")
        file.write(f"Scanned URL: {url}\n")
        for line in results:
            file.write(line + "\n")
        file.write("--------------------------------------------------\n")
    print(f"[INFO] Report saved to {report_file}")

# Main function
def phishing_scanner():
    url = input("Enter the URL to scan: ")
    results = [f"Scanning URL: {url}"]

    # Check if URL has phishing indicators
    if check_url_suspicious(url):
        results.append("[ALERT] This URL has suspicious characteristics!")
    else:
        results.append("[SAFE] No immediate phishing indicators detected.")

    # Check domain registration details
    results.append(check_domain_info(url))

    # Check against VirusTotal
    vt_result = check_virustotal(url)
    results.append(vt_result)

    # Display results
    for result in results:
        print(result)

    # Write report
    write_report(url, results)

# Run the scanner
if __name__ == "__main__":
    phishing_scanner()
