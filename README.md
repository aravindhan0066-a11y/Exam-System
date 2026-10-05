# 📚 Online Exam System

> A fully functional online exam and assignment management system built with **Google Apps Script**, **Google Sheets**, and **Google Drive** — no server required.

![Google Apps Script](https://img.shields.io/badge/Google%20Apps%20Script-4285F4?style=for-the-badge&logo=google&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google%20Sheets-34A853?style=for-the-badge&logo=google-sheets&logoColor=white)
![Google Drive](https://img.shields.io/badge/Google%20Drive-4285F4?style=for-the-badge&logo=google-drive&logoColor=white)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)

---

## ✨ Features

### 👨‍🏫 Admin / Professor
- ✅ Secure email-based login
- ✅ Create **MCQ**, **Descriptive**, and **Mixed** exams
- ✅ Create and manage **Assignments**
- ✅ Set start/end date & time, duration
- ✅ **Auto-grading** for MCQ exams
- ✅ Manual grading with feedback for descriptive answers
- ✅ Bulk import students via CSV
- ✅ View all submissions with grading interface
- ✅ Generate **Student / Exam / Batch** reports
- ✅ Upload study materials to Google Drive
- ✅ Email notifications to students

### 🎓 Student
- ✅ Login with Register Number or Email
- ✅ View assigned exams and assignments
- ✅ Take live exams with **countdown timer**
- ✅ **Auto-submit** when timer expires
- ✅ Submit assignments with text or file upload
- ✅ View submission status and scores

### 🔐 Master Admin (`aravindhan0066@gmail.com`)
- ✅ Full system control
- ✅ Add / Remove Co-Admins (Professors)
- ✅ View complete audit log
- ✅ Initialize system (create all Sheets & Drive folders)

---

## 📁 Project Structure

```
online-exam-system/
│
├── Code.gs              # Backend — Auth, DB, APIs, Business Logic
├── Index.html           # Frontend — Complete UI (Login + Admin + Student)
├── appsscript.json      # Google Apps Script manifest & OAuth scopes
├── .clasp.json          # CLASP config (for push/pull via CLI)
├── .gitignore
└── README.md
```

---

## 🚀 Quick Deployment (5 Steps)

### Step 1 — Open Google Apps Script
Go to 👉 [script.google.com](https://script.google.com) and click **New Project**

### Step 2 — Add the Files

**Code.gs** (default script file):
- Delete the default `myFunction()` code
- Copy everything from `Code.gs` and paste it
- Press `Ctrl+S` to save

**Index.html** (HTML file):
- Click **+** → **HTML** → Name it exactly `Index`
- Delete existing content, paste everything from `Index.html`
- Press `Ctrl+S` to save

**appsscript.json** (manifest):
- Go to **Project Settings** → tick **Show "appsscript.json"**
- Click `appsscript.json` in the file list
- Replace content with the `appsscript.json` from this repo

### Step 3 — Initialize the System
- In the Apps Script editor, click **Run** → select `initializeSystem`
- Authorize all permissions when prompted
- This creates all Google Sheets and Drive folders automatically

### Step 4 — Deploy as Web App
1. Click **Deploy** → **New Deployment**
2. Type: **Web App**
3. Execute as: **Me**
4. Who has access: **Anyone**
5. Click **Deploy** → copy the **Web App URL**

### Step 5 — First Login
- Open the Web App URL
- Select **Admin / Professor** tab
- Email: `aravindhan0066@gmail.com`
- Password: *(choose any — this becomes your master password on first login)*

---

## 🗄️ Database Structure (Google Sheets)

All data is stored in a Google Spreadsheet named `OnlineExamSystem_DB`:

| Sheet | Purpose |
|-------|---------|
| `StudentsDB` | Student records, hashed passwords |
| `AdminsDB` | Admin and co-admin accounts |
| `ExamsDB` | Exam definitions with questions (JSON) |
| `AssignmentsDB` | Assignment records |
| `SubmissionsDB` | All student submissions and scores |
| `AuditLog` | Every action logged with timestamp |
| `Sessions` | Active login sessions (8-hour expiry) |

---

## 📂 Google Drive Folders

```
OnlineExamSystem/
├── StudentsData/
├── Assignments/
├── Exams/
├── Submissions/       ← Student file uploads stored here
├── Reports/
└── StudyMaterials/    ← Professor uploaded materials
```

---

## 👥 User Roles

```
Master Admin  (aravindhan0066@gmail.com)
│  └── Full control: manage co-admins, audit log, system init
│
├── Co-Admin / Admin (Professor)
│   └── Create exams, assignments, grade, view students
│
└── Student
    └── Take exams, submit assignments, view results
```

---

## 📥 Bulk Import Students — CSV Format

```
Name, Batch, Class, RegisterNumber, Email
Aravindhan S, 2024-2025, 1st M.Sc, 24MSC001, aravi@college.edu
Priya R, 2024-2025, 1st M.Sc, 24MSC002, priya@college.edu
```

> **Default student password = Register Number** (e.g. `24MSC001`)  
> Students can change password after first login.

---

## 🔒 Security

- Passwords hashed with **SHA-256**
- Sessions expire after **8 hours**
- Role-based access enforced on **every API call**
- Students cannot see other students' data or answer keys
- All actions are **audit logged**

---

## ⚙️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Google Apps Script (V8 Runtime) |
| Database | Google Sheets |
| Storage | Google Drive |
| Frontend | HTML5, CSS3, Vanilla JavaScript |
| Auth | SHA-256 Password Hashing + Session Tokens |
| Email | Gmail API (via Apps Script) |

---

## 📋 Exam Types

| Type | Grading |
|------|---------|
| MCQ | ✅ Auto-graded instantly on submission |
| Descriptive | ✏️ Manual grading by professor |
| Mixed | MCQ auto-graded + Descriptive manual |

---

## 🛠️ Optional: Deploy via CLASP (Command Line)

```bash
# Install CLASP
npm install -g @google/clasp

# Login
clasp login

# Clone your existing project (get Script ID from Apps Script editor URL)
clasp clone YOUR_SCRIPT_ID

# Or push local files to Apps Script
clasp push

# Open in browser
clasp open
```

---

## 📸 Screenshots

> Login Page · Admin Dashboard · Exam Builder · Student Exam View · Reports

*(Add screenshots of your deployed system here)*

---

## 🤝 Contributing

1. Fork this repository
2. Create your feature branch (`git checkout -b feature/new-feature`)
3. Commit your changes (`git commit -m 'Add new feature'`)
4. Push to the branch (`git push origin feature/new-feature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the **MIT License** — free to use, modify, and distribute.

---

## 👨‍💻 Author

**Aravindhan**  
📧 aravindhan0066@gmail.com  
🏫 Built for university-level exam management

---

*Built with ❤️ using Google Apps Script — No server, no hosting cost, just Google.*
