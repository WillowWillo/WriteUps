# Security Case Study: Cleartext Session Cookie Leakage via HTTPDNS in Mobile App

**Author:** Willow  
**Category:** Web & Mobile Application Security / Network Protocol Analysis  
**CWE:** [CWE-319: Cleartext Transmission of Sensitive Information](https://cwe.mitre.org/data/definitions/319.html), [CWE-614: Sensitive Cookie in HTTPS Session Without 'Secure' Attribute](https://cwe.mitre.org/data/definitions/614.html)  
**Status:** Disclosed Responsibly / Case Study  

---

## 📌 Executive Summary

During a passive network monitoring session conducted on a local personal environment, a critical security misconfiguration was observed in a targeted mobile application ecosystem.

When executing background auxiliary requests—specifically **HTTPDNS resolution** queries—the mobile application transmitted sensitive authentication cookies (`token`, `mt_c_token`, and `mtcp-token`) in plain text over unencrypted HTTP (port 80). 

This write-up covers the passive discovery methodology, technical root cause, potential security risks, and recommended mitigation practices.

---

## 🛠️ Context & Discovery Methodology

While analyzing local home network traffic using network packet capture tools, I observed recurring plain-text HTTP requests originating from a personal mobile device running the application.

Rather than conducting intrusive exploitation or aggressive penetration testing, the traffic was captured **passively** to observe protocol behavior. Upon identifying active JWTs/Session Cookies embedded in unencrypted headers, the inspection was halted, and a formal vulnerability disclosure report was authored to adhere to strict ethical and legal guidelines.

---

## 🔍 Technical Deep Dive & Vulnerability Details

### Root Cause
The vulnerability stems from two combined architectural flaws:
1. **Auxiliary Services over HTTP:** The application relies on HTTPDNS (`httpdnsvip.[REDACTED].com`) over standard unencrypted HTTP (Port `80/TCP`) to bypass traditional local DNS resolvers.
2. **Missing `Secure` Cookie Flag & Overly Broad Scope:** The session cookies (`token`, `mt_c_token`, `mtcp-token`) were issued without the `Secure` attribute and assigned to a high-level domain wildcard (e.g., `.[REDACTED].com`).

Because of the broad scope and missing `Secure` attribute, the client OS automatically appends active session cookies to **any** HTTP request made to any subdomain under `.[REDACTED].com`, including unencrypted HTTP endpoints.

---

## 🧪 Steps to Reproduce (Passive Observation)

1. **Environment Setup:** Connect the target mobile device to a wireless access point configured to capture local network interfaces (e.g., using `Wireshark` or `tcpdump`).
2. **App Execution:** Launch the application and maintain an active authenticated session.
3. **Trigger Background Logic:** Perform routine in-app navigation to trigger background HTTPDNS lookup routines.
4. **Traffic Filtering:** Apply the following display filter to isolate unencrypted HTTP traffic:
   ```query
   tcp.port == 80 && http

    Inspect Payload: Locate the GET /multifetch request directed to httpdnsvip.[REDACTED].com.

    Header Analysis: Examine the Cookie header inside the HTTP request.

Example Captured Request (Sanitized Header)
HTTP

GET /multifetch HTTP/1.1
Host: httpdnsvip.[REDACTED].com
User-Agent: MobileClient/2.x (Android)
Accept: */*
Cookie: token=[REDACTED_JWT_TOKEN]; mt_c_token=[REDACTED_TOKEN]; mtcp-token=[REDACTED_TOKEN]
Connection: close

⚠️ Threat & Impact Analysis

    Passive Eavesdropping: An attacker operating on the same physical or local network (e.g., untrusted Wi-Fi access points, compromised routers, or malicious ISP nodes) can extract active session tokens using simple packet sniffers.

    Session Hijacking / Account Takeover: Because these tokens represent primary authentication factors, acquiring them allows an adversary to impersonate the victim's account without credentials or multi-factor authentication.

    Zero TLS Overhead required for Attackers: Exploitation requires no SSL/TLS interception, certificate forging, or client-side trust installation—the data is exposed natively in cleartext.

🛡️ Remediation & Best Practices

To properly remediate this vulnerability, the following architectural controls should be enforced:

    Set the Secure Attribute on Cookies:

    Ensure that all sensitive authentication cookies explicitly include the ; Secure directive in their Set-Cookie headers. This prevents the browser/client network stack from transmitting the cookie over unencrypted HTTP connections.
    HTTP

    Set-Cookie: token=...; Secure; HttpOnly; SameSite=Strict; Domain=api.[REDACTED].com

    Restrict Cookie Scope:

    Avoid setting session cookies on top-level wildcard domains (.[REDACTED].com). Restrict cookie domains strictly to subdomains that handle stateful user authentication (e.g., api.[REDACTED].com).

    Enforce HTTPS for HTTPDNS / DoH:

    Migrate auxiliary background requests (such as DNS resolution) to secure channels using DNS over HTTPS (DoH) or standard HTTPS APIs (https://).

    Network Security Configuration (Mobile App):

    Configure mobile client network policies to explicitly prohibit cleartext traffic app-wide.

    Android (res/xml/network_security_config.xml):
    XML

    <network-security-config>
        <base-config cleartextTrafficPermitted="false" />
    </network-security-config>

⚖️ Responsible Disclosure & Disclaimer

    Disclaimer:

    This write-up is published strictly for educational, defensive, and research purposes. All observations were made passively on my own personal network environment without disrupting services or exploiting third-party users. The findings were responsibly communicated to the vendor prior to public discussion.
