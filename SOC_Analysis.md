# SOC Analysis — SQL Injection Traffic Evidence
### OWASP Juice Shop | Day 2 | Learning Journal

> 📝 **A quick note before reading:**
> I'm currently learning cybersecurity from scratch. This writeup is part of my learning journey — not a professional report. I'm documenting what I did, what I saw, and what I understood. If something looks basic, that's because I'm still figuring it out.

---

## What I Was Trying to Do

In Day 2, I exploited SQL Injection on OWASP Juice Shop to bypass the admin login without knowing the password.

After completing the attack, I wanted to understand what that attack would look like from a **defender's perspective** — specifically, what a SOC (Security Operations Centre) analyst would see if this happened on a real system.

So I opened **Wireshark** while running the attack to capture the live network traffic.

---

## My Lab Setup

| Thing | What it is |
|---|---|
| Kali Linux | My attacker machine (where I ran the attack from) |
| OWASP Juice Shop | The vulnerable web app I was attacking (running in Docker) |
| Wireshark | Tool I used to capture and read the network traffic |
| Docker | Used to run Juice Shop as a container on my machine |

---

## Something I Noticed — Two Different IP Addresses

When I opened Wireshark, I saw two different sets of IP addresses and got confused at first:

```
127.0.0.1  →  127.0.0.1       (one set)
172.17.0.1 →  172.17.0.2      (another set)
```

**Why does this happen?**

It's because of how Docker works. When Juice Shop runs inside a Docker container, it creates its own small internal network. So my traffic actually travels through two paths:

```
My browser on Kali
       ↓
127.0.0.1 (my machine talking to itself — called loopback)
       ↓
Docker takes over and routes it through its own network
       ↓
172.17.0.1  ──────►  172.17.0.2
(Docker gateway)      (Juice Shop container — the actual target)
```

So the `172.17.0.1 → 172.17.0.2` traffic is the most useful one to look at because:
- `172.17.0.1` = my Kali machine (the attacker)
- `172.17.0.2` = Juice Shop container (the victim)

This actually looks similar to how real network traffic looks in a company — where you'd see an external attacker IP connecting to an internal application server IP. My Docker setup accidentally mimicked that real-world pattern, which I thought was interesting.

---

## The 3 Screenshots and What They Mean

I captured 3 specific things in Wireshark. Here is what each one shows and why it matters.

---

### Screenshot 1 — The Attack Happening

<img width="1440" height="662" alt="Screenshot 2026-06-22 at 1 53 41 PM" src="https://github.com/user-attachments/assets/52867acb-b3a7-4837-824d-2f5eb58560b7" />


This is the **packet list view** in Wireshark — it shows all the network traffic in order.

The important row is the one that says:
```
POST /rest/user/login    172.17.0.1 → 172.17.0.2
```

This tells us:
- Someone sent a **POST request** (a request that submits data, like a login form)
- It went to the `/rest/user/login` endpoint (the login API of Juice Shop)
- It came from `172.17.0.1` (me — the attacker) going to `172.17.0.2` (Juice Shop)

At this point, a SOC analyst would think:
> *"Someone just attempted to log in. Let me look inside that request to see what they actually sent."*

---

### Screenshot 2 — The Malicious Payload

<img width="1440" height="662" alt="Screenshot 2026-06-22 at 1 54 09 PM" src="https://github.com/user-attachments/assets/ac0e65dc-15e6-4ffb-a431-75727c736259" />


This is what was **inside** the login request. When I clicked on the POST packet and expanded the HTTP section, I could see the actual data that was sent:

```json
{
  "email": "admin' OR 1=1--",
  "password": "anything"
}
```

This is the SQL Injection payload sitting right there in **plaintext**.

No real user would ever type `admin' OR 1=1--` as their email address. This is a clear sign of an attack.

**Why does this payload work?**

The backend of Juice Shop builds its database query by directly inserting whatever the user types. So my input turned the query into:

```sql
-- What the app normally asks the database:
SELECT * FROM Users WHERE email = 'user@email.com' AND password = '...'

-- What it asked after my injection:
SELECT * FROM Users WHERE email = 'admin' OR 1=1--' AND password = '...'
```

- `OR 1=1` is always true — so the email check passes no matter what
- `--` comments out the rest — so the password check is completely ignored
- The database returns the first user it finds — which is the admin account

A SOC analyst seeing this payload in logs would immediately know:
> *"This is not a normal login attempt. This is a SQL Injection attack. Now I need to check if it actually worked."*

---

### Screenshot 3 — The Attack Succeeded

<img width="1440" height="662" alt="Screenshot 2026-06-22 at 1 54 49 PM" src="https://github.com/user-attachments/assets/167febe3-d726-42fb-9690-b53fd5da782b" />


This is the **server's response** to the malicious login request.

It returned:
```
HTTP/1.1 200 OK
```

`200 OK` means the server accepted the request and the login was successful.

This is the most critical screenshot because it **confirms the breach**. The attacker got in.

If the response was `401 Unauthorized` it would mean the attack failed. But `200 OK` after a malicious payload means it worked.

**These 3 screenshots together tell the full story:**

```
Screenshot 1  →  Someone tried to log in
Screenshot 2  →  The login attempt contained a SQLi attack
Screenshot 3  →  The attack succeeded and they got in
```

In a real SOC investigation, an analyst always needs all three pieces:
the attempt, the evidence of malice, and the outcome.

---

## What is MITRE ATT&CK and Why Does It Matter Here?

When I was researching how to document this properly, I came across something called the **MITRE ATT&CK framework**.

Think of it as a giant reference library that documents every known way attackers break into systems, move around inside them, and cause damage. Every technique has been given a unique code number so that security teams worldwide can talk about the same thing without confusion.

The framework is organised into **Tactics** (the goal of the attacker) and **Techniques** (how they achieve that goal):

```
Tactic    =  What is the attacker trying to do?
Technique =  How are they doing it?
```

**My SQL Injection attack maps to:**

| Field | Value | What it means |
|---|---|---|
| Tactic | Initial Access | The attacker was trying to get INTO the system |
| Technique | T1190 | Exploit Public Facing Application |
| Method | SQL Injection | The specific way the exploit was done |

**T1190** is the code MITRE assigned to any attack where someone gains access by exploiting a vulnerability in a web application or other public-facing service. My SQLi attack fits exactly here because I exploited the login form of a web application.

Why does this matter to a SOC analyst? Because when an alert fires, tagging it with a MITRE technique immediately tells the analyst:
- What kind of attack it is
- Where in the attack chain the attacker currently is
- What they might try to do next

For example, after T1190 (getting in via SQLi), an attacker might next try:
- **T1078** — using the stolen admin credentials elsewhere
- **T1083** — exploring the application to find more data
- **T1005** — collecting user data from the database

---

## Writing a Detection Rule for This Attack

Now that I understand what the attack looks like in traffic, I tried writing a basic detection rule that would catch it.

The idea is simple: if a POST request goes to the login endpoint AND the body contains SQL injection patterns AND the server returns 200 OK — that should fire an alert.

**In Elastic SIEM (KQL):**

```kql
http.request.method: "POST"
and url.path: "/rest/user/login"
and http.request.body.content: "*OR 1=1*"
```

**In Microsoft Sentinel (KQL):**

```kql
CommonSecurityLog
| where RequestMethod == "POST"
| where RequestURL contains "/rest/user/login"
| where RequestContext contains "OR 1=1"
| project TimeGenerated, SourceIP, RequestURL, RequestContext
```

I haven't tested these rules in a live SIEM yet — that's the next step in my learning. But writing them helped me understand how detection rules connect to real attack patterns.

---

## Where Would This Be Caught in a Real Company?

A real SOC environment has multiple layers of detection. This attack could potentially be caught at any of these:

```
Layer 1 — WAF (Web Application Firewall)
          Detects the ' OR 1=1-- pattern in the HTTP request body
          Either blocks it or logs an alert
               ↓
Layer 2 — Network IDS (Intrusion Detection System)
          Tools like Snort or Suricata inspect packet contents
          Would flag the SQLi signature in the payload
               ↓
Layer 3 — SIEM (Elastic or Microsoft Sentinel)
          Correlates the malicious POST with the 200 OK response
          Fires an alert: "SQLi attempt resulted in successful login"
               ↓
Layer 4 — SOC Analyst
          Reviews exactly the kind of evidence I captured above
          Confirms it is a real breach and escalates to the IR team
```

The important thing I learned here is that **no single tool catches everything**. Defence in depth means having multiple layers so that if one misses something, another catches it.

---

## Key Things I Took Away From This

**1. The payload travels in plaintext**
Because Juice Shop uses HTTP (not HTTPS), the SQLi payload was completely visible in Wireshark without any decryption. In a real application using HTTPS you would need to look at application logs or WAF logs instead.

**2. Three pieces of evidence tell the complete story**
The POST request, the malicious payload inside it, and the 200 OK response together give you the full picture of what happened. Any one of them alone is incomplete.

**3. Docker networking accidentally mimics real architecture**
The two IP ranges I saw (127.0.0.1 and 172.17.0.x) happen because Docker creates its own internal network. The 172.17.0.x traffic is actually the more realistic view — it shows attacker IP to target IP, which is what real SOC traffic analysis looks like.

**4. Understanding the attack makes detection rules better**
Because I know exactly why `OR 1=1` works and what it does to the SQL query, I can write a detection rule that targets the right thing. Most people learning SOC work only see the defensive side — doing the attack first gave me context I would not have had otherwise.

**5. MITRE ATT&CK gives a common language**
Tagging this as T1190 means any security professional anywhere in the world immediately understands what kind of attack happened, where in the kill chain it sits, and what might come next.

---

## What's Next

- Complete DOM XSS challenge on Juice Shop (Day 3)
- Capture the XSS traffic in Wireshark using the same approach
- Set up Elastic SIEM and test the detection rules I wrote above
- Map each Juice Shop challenge to its MITRE ATT&CK technique

---


*This is part of my ongoing learning journal documenting my path through OWASP Top 10, network analysis, and SOC fundamentals. Everything here was done in a legal, controlled lab environment.*
