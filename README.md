// ============================================================
// ONLINE EXAM SYSTEM — Code.gs (Complete Backend)
// Master Admin: aravindhan0066@gmail.com
// Paste this entire file into your Google Apps Script project
// ============================================================

const CONFIG = {
  MASTER_ADMIN: "aravindhan0066@gmail.com",
  SHEETS: {
    STUDENTS:    "StudentsDB",
    ADMINS:      "AdminsDB",
    EXAMS:       "ExamsDB",
    ASSIGNMENTS: "AssignmentsDB",
    SUBMISSIONS: "SubmissionsDB",
    AUDIT_LOG:   "AuditLog",
    SESSIONS:    "Sessions"
  },
  FOLDERS: {
    ROOT:        "OnlineExamSystem",
    STUDENTS:    "StudentsData",
    ASSIGNMENTS: "Assignments",
    EXAMS:       "Exams",
    SUBMISSIONS: "Submissions",
    REPORTS:     "Reports",
    MATERIALS:   "StudyMaterials"
  }
};

function doGet(e) {
  const tmpl = HtmlService.createTemplateFromFile("Index");
  return tmpl.evaluate()
    .setTitle("Online Exam System")
    .addMetaTag("viewport", "width=device-width, initial-scale=1")
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

// ─── SYSTEM INIT ──────────────────────────────────────────────
function initializeSystem() {
  const ss = getOrCreateSpreadsheet();
  ensureSheets(ss);
  ensureDriveFolders();
  return { success: true, message: "System initialized successfully! All sheets and folders created." };
}

function getOrCreateSpreadsheet() {
  const files = DriveApp.getFilesByName("OnlineExamSystem_DB");
  if (files.hasNext()) return SpreadsheetApp.open(files.next());
  const ss = SpreadsheetApp.create("OnlineExamSystem_DB");
  DriveApp.getFileById(ss.getId()).moveTo(getFolder(CONFIG.FOLDERS.ROOT));
  return ss;
}

function getSpreadsheet() {
  const files = DriveApp.getFilesByName("OnlineExamSystem_DB");
  if (files.hasNext()) return SpreadsheetApp.open(files.next());
  initializeSystem();
  return getSpreadsheet();
}

function ensureSheets(ss) {
  const headers = {
    StudentsDB:    ["ID","Batch","Name","Class","RegisterNumber","Email","PasswordHash","CreatedAt","Active"],
    AdminsDB:      ["ID","Name","Email","Role","PasswordHash","AddedBy","CreatedAt","Active"],
    ExamsDB:       ["ID","Title","Type","Subject","Batch","Class","Instructions","Questions","StartTime","EndTime","Duration","AllowMultiple","ShuffleQ","CreatedBy","CreatedAt","Status"],
    AssignmentsDB: ["ID","Title","Subject","Batch","Class","Description","Attachments","StartTime","DueDate","MaxScore","CreatedBy","CreatedAt","Status"],
    SubmissionsDB: ["ID","ExamOrAssignID","Type","StudentID","StudentName","RegisterNumber","Batch","AnswerData","FileURL","SubmittedAt","StartedAt","Score","Status","Graded","GradedBy"],
    AuditLog:      ["ID","Timestamp","UserEmail","UserRole","Action","Details","IP"],
    Sessions:      ["Token","Email","Role","CreatedAt","ExpiresAt","Data"]
  };
  Object.keys(headers).forEach(name => {
    let sh = ss.getSheetByName(name);
    if (!sh) { sh = ss.insertSheet(name); sh.appendRow(headers[name]); sh.setFrozenRows(1); }
  });
  const blank = ss.getSheetByName("Sheet1");
  if (blank && ss.getSheets().length > 1) ss.deleteSheet(blank);
}

function ensureDriveFolders() {
  const root = getFolder(CONFIG.FOLDERS.ROOT);
  Object.values(CONFIG.FOLDERS).forEach(name => { if (name !== CONFIG.FOLDERS.ROOT) getSubFolder(root, name); });
}

function getFolder(name) {
  const f = DriveApp.getFoldersByName(name);
  return f.hasNext() ? f.next() : DriveApp.createFolder(name);
}

function getSubFolder(parent, name) {
  const f = parent.getFoldersByName(name);
  return f.hasNext() ? f.next() : parent.createFolder(name);
}

// ─── SESSIONS ────────────────────────────────────────────────
function createSession(email, role, extra) {
  const token = Utilities.getUuid();
  const now = new Date();
  const exp = new Date(now.getTime() + 8 * 3600000);
  getSpreadsheet().getSheetByName(CONFIG.SHEETS.SESSIONS)
    .appendRow([token, email, role, now.toISOString(), exp.toISOString(), JSON.stringify(extra || {})]);
  return token;
}

function validateSession(token) {
  if (!token) return null;
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.SESSIONS).getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === token && new Date(data[i][4]) > new Date())
      return { email: data[i][1], role: data[i][2], extra: JSON.parse(data[i][5] || "{}") };
  }
  return null;
}

function destroySession(token) {
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.SESSIONS);
  const data = sh.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) { if (data[i][0] === token) { sh.deleteRow(i+1); return; } }
}

// ─── AUTH ────────────────────────────────────────────────────
function loginUser(email, password, role) {
  try {
    email = email.trim().toLowerCase();
    const hash = hashPassword(password);
    if (role === "admin") {
      const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.ADMINS);
      const data = sh.getDataRange().getValues();
      // Auto-create master admin on very first login
      if (email === CONFIG.MASTER_ADMIN && data.length <= 1) {
        sh.appendRow([generateID(), "Master Admin", email, "masteradmin", hash, "system", new Date().toISOString(), 1]);
        const token = createSession(email, "masteradmin", { name: "Master Admin" });
        auditLog(email, "masteradmin", "FIRST_LOGIN", "Master admin account created");
        return { success: true, token, role: "masteradmin", name: "Master Admin" };
      }
      for (let i = 1; i < data.length; i++) {
        if (data[i][2].toLowerCase() === email && data[i][4] === hash && data[i][7] == 1) {
          const token = createSession(email, data[i][3], { name: data[i][1] });
          auditLog(email, data[i][3], "LOGIN", "Admin login");
          return { success: true, token, role: data[i][3], name: data[i][1] };
        }
      }
      return { success: false, message: "Invalid credentials" };
    }
    if (role === "student") {
      const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.STUDENTS).getDataRange().getValues();
      for (let i = 1; i < data.length; i++) {
        const match = data[i][4].toString().toLowerCase() === email || data[i][5].toLowerCase() === email;
        if (match && data[i][6] === hash && data[i][8] == 1) {
          const token = createSession(data[i][5], "student", { name: data[i][2], studentId: data[i][0], batch: data[i][1], class: data[i][3], registerNumber: data[i][4] });
          auditLog(data[i][5], "student", "LOGIN", "Student login");
          return { success: true, token, role: "student", name: data[i][2] };
        }
      }
      return { success: false, message: "Invalid credentials" };
    }
    return { success: false, message: "Unknown role" };
  } catch(err) { return { success: false, message: "Login error: " + err.message }; }
}

function logoutUser(token) {
  const s = validateSession(token);
  if (s) auditLog(s.email, s.role, "LOGOUT", "Logged out");
  destroySession(token);
  return { success: true };
}

function hashPassword(p) {
  return Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, p)
    .map(b => ('0'+(b&0xFF).toString(16)).slice(-2)).join('');
}

function generateID() {
  return Utilities.getUuid().replace(/-/g,'').substr(0,12).toUpperCase();
}

// ─── ADMIN MANAGEMENT ────────────────────────────────────────
function addAdmin(token, d) {
  const s = validateSession(token);
  if (!s || s.role !== "masteradmin") return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.ADMINS);
  const data = sh.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (data[i][2].toLowerCase() === d.email.toLowerCase()) return { success: false, message: "Email already exists" };
  }
  const id = generateID();
  sh.appendRow([id, d.name, d.email.toLowerCase(), d.role||"admin", hashPassword(d.password), s.email, new Date().toISOString(), 1]);
  auditLog(s.email, s.role, "ADD_ADMIN", "Added: "+d.email);
  try { GmailApp.sendEmail(d.email, "Online Exam System — Access Granted", `Hello ${d.name},\n\nYou've been added as ${d.role||"admin"}.\nURL: ${ScriptApp.getService().getUrl()}\nEmail: ${d.email}\nPassword: ${d.password}\n\nChange your password after first login.\n\n— Online Exam System`); } catch(e){}
  return { success: true, id };
}

function removeAdmin(token, email) {
  const s = validateSession(token);
  if (!s || s.role !== "masteradmin") return { success: false, message: "Unauthorized" };
  if (email === CONFIG.MASTER_ADMIN) return { success: false, message: "Cannot remove master admin" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.ADMINS);
  const data = sh.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (data[i][2].toLowerCase() === email.toLowerCase()) {
      sh.getRange(i+1,8).setValue(0);
      auditLog(s.email, s.role, "REMOVE_ADMIN", "Deactivated: "+email);
      return { success: true };
    }
  }
  return { success: false, message: "Admin not found" };
}

function getAdmins(token) {
  const s = validateSession(token);
  if (!s || s.role !== "masteradmin") return { success: false, message: "Unauthorized" };
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.ADMINS).getDataRange().getValues();
  return { success: true, admins: data.slice(1).filter(r=>r[2]&&r[2]!==CONFIG.MASTER_ADMIN)
    .map(r=>({ id:r[0], name:r[1], email:r[2], role:r[3], addedBy:r[5], createdAt:r[6], active:r[7] })) };
}

// ─── STUDENT MANAGEMENT ──────────────────────────────────────
function addStudents(token, arr) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.STUDENTS);
  const existing = sh.getDataRange().getValues();
  const emails = new Set(existing.slice(1).map(r=>r[5].toLowerCase()));
  const regs = new Set(existing.slice(1).map(r=>r[4].toString().toLowerCase()));
  let added=0, skipped=0;
  arr.forEach(st => {
    const em=(st.email||"").toLowerCase(), rg=(st.registerNumber||"").toString().toLowerCase();
    if(!em||!rg||emails.has(em)||regs.has(rg)){skipped++;return;}
    sh.appendRow([generateID(), st.batch||"", st.name||"", st.class||"", st.registerNumber, em, hashPassword(rg), new Date().toISOString(), 1]);
    emails.add(em); regs.add(rg); added++;
  });
  auditLog(s.email, s.role, "BULK_ADD_STUDENTS", `Added ${added}, skipped ${skipped}`);
  return { success: true, added, skipped };
}

function getStudents(token, filters) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.STUDENTS).getDataRange().getValues();
  let students = data.slice(1).filter(r=>r[0]).map(r=>({ id:r[0], batch:r[1], name:r[2], class:r[3], registerNumber:r[4], email:r[5], createdAt:r[7], active:r[8] }));
  if (filters) {
    if (filters.batch) students = students.filter(st=>st.batch===filters.batch);
    if (filters.class) students = students.filter(st=>st.class===filters.class);
    if (filters.search) { const q=filters.search.toLowerCase(); students=students.filter(st=>st.name.toLowerCase().includes(q)||st.registerNumber.toString().includes(q)||st.email.includes(q)); }
  }
  return { success: true, students };
}

function deleteStudent(token, id) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.STUDENTS);
  const data = sh.getDataRange().getValues();
  for (let i=1;i<data.length;i++) { if(data[i][0]===id){sh.getRange(i+1,9).setValue(0);auditLog(s.email,s.role,"DELETE_STUDENT","Removed: "+id);return{success:true};} }
  return { success: false, message: "Not found" };
}

// ─── EXAM MANAGEMENT ─────────────────────────────────────────
function createExam(token, d) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const id = generateID();
  getSpreadsheet().getSheetByName(CONFIG.SHEETS.EXAMS).appendRow([
    id, d.title, d.type, d.subject, d.batch, d.class, d.instructions||"",
    JSON.stringify(d.questions||[]), d.startTime, d.endTime, d.duration,
    d.allowMultiple?1:0, d.shuffleQ?1:0, s.email, new Date().toISOString(), "active"
  ]);
  auditLog(s.email, s.role, "CREATE_EXAM", `${d.title} (${id})`);
  sendNotifications(d.batch, d.class, "New Exam: "+d.title, `Exam "${d.title}" scheduled.\nStart: ${d.startTime}\nEnd: ${d.endTime}`);
  return { success: true, id };
}

function updateExam(token, examId, updates) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.EXAMS);
  const data = sh.getDataRange().getValues();
  const cols = {title:1,type:2,subject:3,batch:4,class:5,instructions:6,questions:7,startTime:8,endTime:9,duration:10,allowMultiple:11,shuffleQ:12,status:15};
  for (let i=1;i<data.length;i++) {
    if (data[i][0]===examId) {
      Object.keys(updates).forEach(k=>{if(cols[k]!==undefined)sh.getRange(i+1,cols[k]+1).setValue(k==="questions"?JSON.stringify(updates[k]):updates[k]);});
      auditLog(s.email,s.role,"UPDATE_EXAM",examId);
      return { success: true };
    }
  }
  return { success: false, message: "Not found" };
}

function deleteExam(token, id) { return updateExam(token, id, {status:"deleted"}); }

function getExams(token, filters) {
  const s = validateSession(token);
  if (!s) return { success: false, message: "Unauthorized" };
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.EXAMS).getDataRange().getValues();
  let exams = [];
  for (let i=1;i<data.length;i++) {
    if (!data[i][0]||data[i][15]==="deleted") continue;
    const e = { id:data[i][0],title:data[i][1],type:data[i][2],subject:data[i][3],batch:data[i][4],class:data[i][5],instructions:data[i][6],
      questions:data[i][7]?JSON.parse(data[i][7]):[],startTime:data[i][8],endTime:data[i][9],duration:data[i][10],
      allowMultiple:data[i][11]==1,shuffleQ:data[i][12]==1,createdBy:data[i][13],createdAt:data[i][14],status:data[i][15] };
    if (s.role==="student") {
      if (e.batch!==s.extra.batch&&e.batch!=="All") continue;
      if (e.class!==s.extra.class&&e.class!=="All") continue;
      e.questions=(e.questions||[]).map(q=>({id:q.id,text:q.text,type:q.type,options:q.options,marks:q.marks}));
    }
    if (filters) {
      if (filters.status&&e.status!==filters.status) continue;
      if (filters.batch&&e.batch!==filters.batch&&e.batch!=="All") continue;
    }
    exams.push(e);
  }
  return { success: true, exams };
}

// ─── ASSIGNMENT MANAGEMENT ───────────────────────────────────
function createAssignment(token, d) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const id = generateID();
  getSpreadsheet().getSheetByName(CONFIG.SHEETS.ASSIGNMENTS).appendRow([
    id, d.title, d.subject, d.batch, d.class, d.description||"", d.attachments||"",
    d.startTime||new Date().toISOString(), d.dueDate, d.maxScore||100, s.email, new Date().toISOString(), "active"
  ]);
  auditLog(s.email, s.role, "CREATE_ASSIGNMENT", `${d.title} (${id})`);
  sendNotifications(d.batch, d.class, "New Assignment: "+d.title, `Assignment "${d.title}" posted.\nDue: ${d.dueDate}`);
  return { success: true, id };
}

function updateAssignment(token, assignId, updates) {
  const s = validateSession(token);
  if (!s || (s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.ASSIGNMENTS);
  const data = sh.getDataRange().getValues();
  const cols = {title:1,subject:2,batch:3,class:4,description:5,attachments:6,startTime:7,dueDate:8,maxScore:9,status:12};
  for (let i=1;i<data.length;i++) {
    if (data[i][0]===assignId) {
      Object.keys(updates).forEach(k=>{if(cols[k]!==undefined)sh.getRange(i+1,cols[k]+1).setValue(updates[k]);});
      return { success: true };
    }
  }
  return { success: false, message: "Not found" };
}

function deleteAssignment(token, id) { return updateAssignment(token, id, {status:"deleted"}); }

function getAssignments(token, filters) {
  const s = validateSession(token);
  if (!s) return { success: false, message: "Unauthorized" };
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.ASSIGNMENTS).getDataRange().getValues();
  let assignments = [];
  for (let i=1;i<data.length;i++) {
    if (!data[i][0]||data[i][12]==="deleted") continue;
    const a = {id:data[i][0],title:data[i][1],subject:data[i][2],batch:data[i][3],class:data[i][4],description:data[i][5],attachments:data[i][6],startTime:data[i][7],dueDate:data[i][8],maxScore:data[i][9],createdBy:data[i][10],createdAt:data[i][11],status:data[i][12]};
    if (s.role==="student") {
      if (a.batch!==s.extra.batch&&a.batch!=="All") continue;
      if (a.class!==s.extra.class&&a.class!=="All") continue;
    }
    if (filters) {
      if (filters.batch&&a.batch!==filters.batch&&a.batch!=="All") continue;
    }
    assignments.push(a);
  }
  return { success: true, assignments };
}

// ─── SUBMISSIONS ─────────────────────────────────────────────
function submitExam(token, sub) {
  const s = validateSession(token);
  if (!s) return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.SUBMISSIONS);
  if (!sub.allowMultiple) {
    const existing = sh.getDataRange().getValues();
    for (let i=1;i<existing.length;i++) {
      if (existing[i][1]===sub.examId&&existing[i][3]===s.extra.studentId&&existing[i][2]==="exam")
        return { success: false, message: "Already submitted. Multiple attempts not allowed." };
    }
  }
  const id = generateID();
  sh.appendRow([id, sub.examId, "exam", s.extra.studentId, s.extra.name, s.extra.registerNumber, s.extra.batch, JSON.stringify(sub.answers||[]), "", new Date().toISOString(), sub.startedAt||"", "", "submitted", 0, ""]);
  const exam = getExamForGrading(sub.examId);
  if (exam && exam.type !== "Descriptive") {
    const score = autoGrade(exam.questions, sub.answers);
    const rows = sh.getDataRange().getValues();
    for (let i=1;i<rows.length;i++) {
      if (rows[i][0]===id) { sh.getRange(i+1,12).setValue(score.total); sh.getRange(i+1,14).setValue(1); sh.getRange(i+1,13).setValue("graded"); break; }
    }
    auditLog(s.email,"student","SUBMIT_EXAM",`${sub.examId} score:${score.total}`);
    return { success: true, id, score };
  }
  auditLog(s.email,"student","SUBMIT_EXAM",sub.examId);
  return { success: true, id };
}

function submitAssignment(token, sub) {
  const s = validateSession(token);
  if (!s||s.role!=="student") return { success: false, message: "Unauthorized" };
  const id = generateID();
  getSpreadsheet().getSheetByName(CONFIG.SHEETS.SUBMISSIONS).appendRow([id, sub.assignmentId, "assignment", s.extra.studentId, s.extra.name, s.extra.registerNumber, s.extra.batch, sub.textAnswer||"", sub.fileUrl||"", new Date().toISOString(), "", "", "submitted", 0, ""]);
  auditLog(s.email,"student","SUBMIT_ASSIGNMENT",sub.assignmentId);
  return { success: true, id };
}

function getSubmissions(token, filters) {
  const s = validateSession(token);
  if (!s) return { success: false, message: "Unauthorized" };
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.SUBMISSIONS).getDataRange().getValues();
  let subs = [];
  for (let i=1;i<data.length;i++) {
    if (!data[i][0]) continue;
    if (s.role==="student"&&data[i][3]!==s.extra.studentId) continue;
    const sub = {id:data[i][0],examOrAssignId:data[i][1],type:data[i][2],studentId:data[i][3],studentName:data[i][4],registerNumber:data[i][5],batch:data[i][6],answerData:data[i][7],fileUrl:data[i][8],submittedAt:data[i][9],startedAt:data[i][10],score:data[i][11],status:data[i][12],graded:data[i][13],gradedBy:data[i][14]};
    if (filters) {
      if (filters.examId&&sub.examOrAssignId!==filters.examId) continue;
      if (filters.type&&sub.type!==filters.type) continue;
      if (filters.batch&&sub.batch!==filters.batch) continue;
    }
    subs.push(sub);
  }
  return { success: true, submissions: subs };
}

function gradeSubmission(token, subId, score, feedback) {
  const s = validateSession(token);
  if (!s||(s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const sh = getSpreadsheet().getSheetByName(CONFIG.SHEETS.SUBMISSIONS);
  const data = sh.getDataRange().getValues();
  for (let i=1;i<data.length;i++) {
    if (data[i][0]===subId) {
      sh.getRange(i+1,12).setValue(score); sh.getRange(i+1,13).setValue("graded"); sh.getRange(i+1,14).setValue(1); sh.getRange(i+1,15).setValue(s.email);
      auditLog(s.email,s.role,"GRADE",`${subId}:${score}`);
      try { GmailApp.sendEmail(data[i][4], "Your Submission Has Been Graded", `Hello ${data[i][4]},\n\nScore: ${score}\n${feedback?'Feedback: '+feedback:''}\n\n— Online Exam System`); } catch(e){}
      return { success: true };
    }
  }
  return { success: false, message: "Not found" };
}

function getExamForGrading(examId) {
  const data = getSpreadsheet().getSheetByName(CONFIG.SHEETS.EXAMS).getDataRange().getValues();
  for (let i=1;i<data.length;i++) { if(data[i][0]===examId) return {type:data[i][2],questions:data[i][7]?JSON.parse(data[i][7]):[]}; }
  return null;
}

function autoGrade(questions, answers) {
  let total=0, max=0;
  (questions||[]).forEach(q => {
    max += q.marks||1;
    if (q.type==="MCQ") {
      const a = (answers||[]).find(x=>x.questionId===q.id);
      if (a && String(a.answer)===String(q.correctAnswer)) total += q.marks||1;
    }
  });
  return { total, max };
}

// ─── DASHBOARD ───────────────────────────────────────────────
function getDashboardData(token) {
  const s = validateSession(token);
  if (!s) return { success: false, message: "Unauthorized" };
  const now = new Date();
  if (s.role==="student") {
    const examsRes=getExams(token,null), assignRes=getAssignments(token,null), subsRes=getSubmissions(token,null);
    const subs=subsRes.submissions||[];
    const subAssignIds=new Set(subs.filter(x=>x.type==="assignment").map(x=>x.examOrAssignId));
    return { success:true, role:"student", data:{
      upcomingExams:(examsRes.exams||[]).filter(e=>new Date(e.startTime)>now).length,
      activeExams:(examsRes.exams||[]).filter(e=>new Date(e.startTime)<=now&&new Date(e.endTime)>=now).length,
      pendingAssignments:(assignRes.assignments||[]).filter(a=>!subAssignIds.has(a.id)&&new Date(a.dueDate)>=now).length,
      totalSubmissions:subs.length, recentExams:examsRes.exams||[], recentAssignments:assignRes.assignments||[], submissions:subs
    }};
  }
  const ss=getSpreadsheet();
  const students=ss.getSheetByName(CONFIG.SHEETS.STUDENTS).getDataRange().getValues();
  const exams=ss.getSheetByName(CONFIG.SHEETS.EXAMS).getDataRange().getValues();
  const assigns=ss.getSheetByName(CONFIG.SHEETS.ASSIGNMENTS).getDataRange().getValues();
  const subs=ss.getSheetByName(CONFIG.SHEETS.SUBMISSIONS).getDataRange().getValues();
  return { success:true, role:s.role, data:{
    totalStudents:students.slice(1).filter(r=>r[8]==1).length,
    totalExams:exams.slice(1).filter(r=>r[15]!=="deleted").length,
    totalAssigns:assigns.slice(1).filter(r=>r[12]!=="deleted").length,
    totalSubs:subs.slice(1).filter(r=>r[0]).length,
    activeExams:exams.slice(1).filter(r=>r[15]==="active"&&new Date(r[8])<=now&&new Date(r[9])>=now).length,
    pendingGrade:subs.slice(1).filter(r=>r[13]==0&&r[12]==="submitted").length,
    upcomingDeadlines:assigns.slice(1).filter(r=>r[12]==="active"&&new Date(r[8])>now&&new Date(r[8])<new Date(now.getTime()+7*86400000)).length
  }};
}

// ─── REPORTS ────────────────────────────────────────────────
function generateReport(token, type, filters) {
  const s = validateSession(token);
  if (!s||(s.role!=="masteradmin"&&s.role!=="admin")) return { success: false, message: "Unauthorized" };
  const ss=getSpreadsheet();
  const subs=ss.getSheetByName(CONFIG.SHEETS.SUBMISSIONS).getDataRange().getValues().slice(1).filter(r=>r[0]);
  const students=ss.getSheetByName(CONFIG.SHEETS.STUDENTS).getDataRange().getValues().slice(1);
  const exams=ss.getSheetByName(CONFIG.SHEETS.EXAMS).getDataRange().getValues().slice(1);
  if (type==="student") {
    const st=students.find(x=>x[0]===filters.studentId);
    if (!st) return { success:false, message:"Student not found" };
    const stSubs=subs.filter(r=>r[3]===filters.studentId);
    return { success:true, report:{ student:{id:st[0],name:st[2],batch:st[1],class:st[3],registerNumber:st[4],email:st[5]},
      submissions:stSubs.map(x=>({examOrAssignId:x[1],type:x[2],submittedAt:x[9],score:x[11],graded:x[13]})),
      totalSubmissions:stSubs.length, gradedCount:stSubs.filter(x=>x[13]==1).length,
      averageScore:stSubs.filter(x=>x[11]).reduce((a,x)=>a+(parseFloat(x[11])||0),0)/(stSubs.filter(x=>x[11]).length||1) }};
  }
  if (type==="exam") {
    const exSubs=subs.filter(r=>r[1]===filters.examId&&r[2]==="exam");
    const ex=exams.find(x=>x[0]===filters.examId);
    return { success:true, report:{ exam:ex?{id:ex[0],title:ex[1],subject:ex[3],batch:ex[4]}:{},
      totalSubmissions:exSubs.length, gradedCount:exSubs.filter(x=>x[13]==1).length,
      averageScore:exSubs.filter(x=>x[11]).reduce((a,x)=>a+(parseFloat(x[11])||0),0)/(exSubs.filter(x=>x[11]).length||1),
      submissions:exSubs.map(x=>({studentName:x[4],registerNumber:x[5],batch:x[6],submittedAt:x[9],score:x[11],graded:x[13]})) }};
  }
  if (type==="batch") {
    const bSt=students.filter(x=>x[1]===filters.batch);
    const bSubs=subs.filter(x=>x[6]===filters.batch);
    return { success:true, report:{ batch:filters.batch, totalStudents:bSt.length, totalSubmissions:bSubs.length,
      students:bSt.map(st=>{const ss2=bSubs.filter(b=>b[3]===st[0]);return{name:st[2],registerNumber:st[4],submissions:ss2.length,averageScore:ss2.filter(x=>x[11]).reduce((a,x)=>a+(parseFloat(x[11])||0),0)/(ss2.filter(x=>x[11]).length||1)};}) }};
  }
  return { success:false, message:"Unknown type" };
}

// ─── NOTIFICATIONS ───────────────────────────────────────────
function sendNotifications(batch, cls, subject, body) {
  try {
    const students=getSpreadsheet().getSheetByName(CONFIG.SHEETS.STUDENTS).getDataRange().getValues().slice(1);
    students.forEach(st=>{
      if(st[8]!=1) return;
      if(batch!=="All"&&st[1]!==batch) return;
      if(cls!=="All"&&st[3]!==cls) return;
      try{GmailApp.sendEmail(st[5],subject,body);}catch(e){}
    });
  } catch(e){}
}

// ─── AUDIT LOG ───────────────────────────────────────────────
function auditLog(email, role, action, details) {
  try { getSpreadsheet().getSheetByName(CONFIG.SHEETS.AUDIT_LOG).appendRow([generateID(), new Date().toISOString(), email, role, action, details, ""]); } catch(e){}
}

function getAuditLogs(token, limit) {
  const s = validateSession(token);
  if (!s||s.role!=="masteradmin") return { success:false, message:"Unauthorized" };
  const data=getSpreadsheet().getSheetByName(CONFIG.SHEETS.AUDIT_LOG).getDataRange().getValues();
  return { success:true, logs:data.slice(1).reverse().slice(0,limit||100).map(r=>({id:r[0],timestamp:r[1],email:r[2],role:r[3],action:r[4],details:r[5]})) };
}

// ─── FILES ───────────────────────────────────────────────────
function uploadFile(token, b64, name, mime, folderType) {
  const s = validateSession(token);
  if (!s) return { success:false, message:"Unauthorized" };
  const folder=getSubFolder(getFolder(CONFIG.FOLDERS.ROOT), folderType||CONFIG.FOLDERS.SUBMISSIONS);
  const file=folder.createFile(Utilities.newBlob(Utilities.base64Decode(b64), mime, name));
  file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
  return { success:true, fileId:file.getId(), fileUrl:file.getUrl() };
}

function getStudyMaterials(token) {
  const s = validateSession(token);
  if (!s) return { success:false, message:"Unauthorized" };
  const folder=getSubFolder(getFolder(CONFIG.FOLDERS.ROOT), CONFIG.FOLDERS.MATERIALS);
  const files=folder.getFiles();
  const materials=[];
  while(files.hasNext()){const f=files.next();materials.push({id:f.getId(),name:f.getName(),url:f.getUrl(),mimeType:f.getMimeType(),createdDate:f.getDateCreated()});}
  return { success:true, materials };
}

// ─── CHANGE PASSWORD ─────────────────────────────────────────
function changePassword(token, oldPass, newPass) {
  const s = validateSession(token);
  if (!s) return { success:false, message:"Unauthorized" };
  const oldH=hashPassword(oldPass), newH=hashPassword(newPass);
  const shName = s.role==="student" ? CONFIG.SHEETS.STUDENTS : CONFIG.SHEETS.ADMINS;
  const col = s.role==="student" ? 7 : 5;
  const emailCol = s.role==="student" ? 6 : 3;
  const sh=getSpreadsheet().getSheetByName(shName);
  const data=sh.getDataRange().getValues();
  for(let i=1;i<data.length;i++){
    if(data[i][emailCol-1].toLowerCase()===s.email&&data[i][col-1]===oldH){sh.getRange(i+1,col).setValue(newH);return{success:true};}
  }
  return { success:false, message:"Incorrect current password" };
}
