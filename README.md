# SQL Injection: Authentication Bypass in OWASP Juice Shop

**OWASP Category:** A03:2025 – Injection  
**Difficulty:** Beginner  
**Lab:** Kali Linux + Docker + OWASP Juice Shop  
**Date:** June 19, 2026  
**Status:** ✅ Completed

---

## 🎬 Video Demo

https://github.com/user-attachments/assets/0bd87eac-d906-4a6c-b1bb-2c5515ee7e9f


*Watch the full exploit: Login page → SQL Injection payload → Successful login as admin → Dev Tools network tab*

---

## 🎯 What I Learned

Before this, I thought of web applications as just "pages loading." Now I understand:

- Web pages send **multiple API calls** (not just one request)
- Each API call processes **user input** (like email and password)
- If that API doesn't validate input properly, **vulnerabilities exist**
- SQL Injection happens when the API builds SQL queries with **user input directly**

The same `/api/login` endpoint I analyzed for network calls was vulnerable to SQL Injection.

---

## 📋 Prerequisites

- Kali Linux (or any Linux with Docker)
- Docker installed
- Git for cloning repositories

---

## 🏗️ Environment Setup

### Step 1: Install Docker

```bash
sudo apt update
sudo apt install docker.io
```

### Step 2: Run Juice Shop

```bash
sudo docker run --rm -p 3000:3000 bkimminich/juice-shop
```

### Step 3: Access Juice Shop

Open browser and go to: `http://localhost:3000`

---

## 🔍 Understanding the Vulnerability

### What Happens at the API Level

When you click "Login" on a website:

1. Your browser sends a **POST request** to `/api/login`
2. The API receives your **email and password**
3. The API builds a **SQL query** to check if you're valid
4. The API sends back a **response** (success or failure)

**The vulnerability:** The API builds that SQL query using **string concatenation** with your input.

```javascript
// VULNERABLE CODE:
const query = `SELECT * FROM Users WHERE email = '${email}' AND password = '${password}'`;
```

When you enter `admin' OR '1'='1'--`, that becomes part of the SQL query.

---

## 🎯 Exploitation Steps

### Step 1: Navigate to Login Page

<img width="1440" height="752" alt="JuiceShop" src="https://github.com/user-attachments/assets/2924dcc0-416b-411d-847a-38433a455c40" />

Click the **Login** button (top right corner).

### Step 2: Analyze the Request (Understanding the API)


1. Press `F12` to open Dev Tools
2. Click **Network** tab
3. Clear previous logs
4. Enter test credentials: `email=test@test.com`, `password=test`
5. Click Login
6. Click `/api/login` request
7. Check **Request** → See parameters: `email` and `password`


**What I learned:** The login form sends a POST request to `/api/login` with JSON containing email and password.

### Step 3: Inject SQL Payload

**Email field:**

admin' OR '1'='1'--

**Password field:**

test123


**Click Login**

<img width="1440" height="644" alt="SQL payload " src="https://github.com/user-attachments/assets/b93285c2-7292-4513-aeaf-7b2f0b75cdba" />


### Step 4: Success!

- Logged in as **admin** without knowing the password! ✅
- Check your profile (top right) → Shows "admin@juice-shop.com"

<img width="1440" height="407" alt="Screenshot 2026-06-20 at 2 06 15 PM" src="https://github.com/user-attachments/assets/85163c54-15c5-493c-b6bd-dd5581a28a65" />


### Step 5: Verify with Dev Tools

1. Open Network tab again (`F12`)
2. Click Login with payload
3. Click `/api/login` request
4. Check **Response** → Should show:

```json
{
  "email": "admin@juice-shop.com",
  "userId": 1,
  "token": "eyJhbGciOi..."
}
```

<img width="1439" height="171" alt="payload" src="https://github.com/user-attachments/assets/a6308208-55b8-472b-b058-eb7883a4e054" />

<img width="1440" height="470" alt="Screenshot 2026-06-22 at 8 12 50 AM" src="https://github.com/user-attachments/assets/1bd0a418-d6ca-43c9-9f90-014f4b5b7757" />

**This is the same API call I analyzed before, but now I understand what happens when it's vulnerable!**

---

## 💡 SQL Query Breakdown

### Normal Query (Before Injection):

```sql
SELECT * FROM Users WHERE email = 'john@example.com' AND password = 'secret123'
```

**Database evaluates:**
- `email = 'john@example.com'` → ❌ FALSE (wrong email)
- `AND password = 'secret123'` → ❌ FALSE (wrong password)
- **Result:** FALSE → Login fails ❌

### After SQL Injection:

```sql
SELECT * FROM Users WHERE email = 'admin' OR '1'='1'--' AND password = 'test123'
```

**After `--` comment (rest ignored):**

```sql
SELECT * FROM Users WHERE email = 'admin' OR '1'='1'
```

**Database evaluates:**
- `email = 'admin'` → ❌ FALSE
- `OR 1=1` → ✅ TRUE (1 always equals 1!)
- **Result:** TRUE → Login succeeds as admin! ✅

---

## 🔍 Why Each Part of the Payload Works

| Part | Purpose |
|------|---------|
| `admin` | Targets admin user specifically |
| `'` | Closes the email string (critical!) |
| `OR` | Logical OR operator |
| `1=1` | Always TRUE condition |
| `--` | SQL comment (ignores password check) |

---

## 🆚 SQL Injection vs Brute Force (Hydra)

| Aspect        | SQL Injection              | Hydra Brute Force             |
|-------------  |----------------------------|------------------------------ |
| **Method**    | Manipulate SQL query logic | Guess passwords               |
| **Attempts**  | 1 attempt (instant) ⚡     | 100-1000s attempts (slow) 🐢 |
| **Bypasses**  | Password check entirely    | Works if weak password exists |
| **Detection** | Harder (single request)    | Easy (many requests)          |

---

## 🛡️ How to Fix It

### 1. Use Parameterized Queries (Prepared Statements)

```javascript
// ❌ VULNERABLE:
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ SECURE:
const query = 'SELECT * FROM users WHERE email = ?';
db.execute(query, [email]);
```

**Why this works:** The `?` placeholder tells the database to treat input as data, not SQL code.

### 2. Input Validation
- Validate email format before processing
- Reject inputs containing SQL characters (`'`, `"`, `;`, `--`)

### 3. Use ORM Frameworks
- Sequelize, TypeORM, SQLAlchemy handle queries safely
- Automatically use parameterized queries

---

## 🔍 Finding the Vulnerable Code

Since Juice Shop is open-source, I cloned it to see the actual vulnerable code:

```bash
git clone https://github.com/juice-shop/juice-shop.git
cd juice-shop
grep -rn "SELECT.*FROM.*email.*password" .
```

**Found at:** `src/routes/login.ts` line 34

**Vulnerable Code:**

```javascript
models.sequelize.query(
  `SELECT * FROM Users WHERE email = '${email}' AND password = '${password}' AND deletedAt IS NULL`,
  { type: models.sequelize.QueryTypes.SELECT }
)
```

**Problem:** String concatenation (`${email}`) allows SQL injection!

---


## 🚀 Where I'm Going Next

This was Day 2 of learning OWASP Top 10. I'm building my understanding step by step:

- **Day 1:** Understanding web app network interactions (HTTP requests, API calls)
- **Day 2:** Exploiting SQL Injection in API calls
- **Next:** DOM XSS (Cross-Site Scripting) - also part of A03:2025 Injection

---

**Author:** Bhadra Menon  
**Learning Journey:** Documenting my path to cybersecurity  
**GitHub:** https://github.com/bhadramenon
**LinkedIn:** https://www.linkedin.com/in/bhadra-menon/

---

*If you're also learning cybersecurity, feel free to connect! I love sharing what I learn and hearing about others' journeys.*
