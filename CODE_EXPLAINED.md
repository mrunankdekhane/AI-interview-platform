# 📖 CODE_EXPLAINED.md — Deep Dive: Every Line, Every Route, Every AI Call

---

## 1. BACKEND — Every File Explained Line by Line

---

### `Backend/server.js`

```js
require("dotenv").config()
```
Reads the `.env` file and injects all variables into `process.env`.
Must be the FIRST line so every other file that reads `process.env` gets the values.

```js
const app = require("./src/app")
const connectToDB = require("./src/config/database")
connectToDB()
```
Imports the Express app and the DB connector, then immediately connects to MongoDB.

```js
app.listen(3000, () => { console.log("Server is running on port 3000") })
```
Binds the Express app to TCP port 3000. Every incoming HTTP request comes here.

---

### `Backend/src/app.js`

```js
const express = require("express")
const app = express()
```
Creates the Express application instance.

```js
app.use(express.json())
```
Middleware that parses `Content-Type: application/json` request bodies.
Without this, `req.body` would be `undefined` for JSON requests.

```js
app.use(cookieParser())
```
Parses the `Cookie` HTTP header. Without this `req.cookies` would be `undefined`.
This is needed so the auth middleware can read `req.cookies.token`.

```js
app.use(cors({ origin: "http://localhost:5173", credentials: true }))
```
- `origin` — only requests from the Vite frontend are allowed.
- `credentials: true` — allows cookies to be sent with cross-origin requests.
Without `credentials: true`, the browser would strip the JWT cookie from every request.

```js
app.use("/api/auth", authRouter)
app.use("/api/interview", interviewRouter)
```
Mounts the two routers. Any request starting with `/api/auth` goes to `auth.routes.js`, and `/api/interview` goes to `interview.routes.js`.

---

### `Backend/src/config/database.js`

```js
await mongoose.connect(process.env.MONGO_URI)
```
Uses Mongoose to open a connection pool to MongoDB.
`MONGO_URI` from `.env` is e.g. `mongodb://localhost:27017/interview-ai`.
The database `interview-ai` is created automatically if it doesn't exist.

---

### `Backend/src/models/user.model.js`

```js
const userSchema = new mongoose.Schema({
    username: { type: String, unique: true, required: true },
    email:    { type: String, unique: true, required: true },
    password: { type: String, required: true }
})
```
Defines the shape of every user document in MongoDB.
- `unique: true` creates a MongoDB index that rejects duplicates.
- Passwords are stored as bcrypt hashes, never plain text.

```js
const userModel = mongoose.model("users", userSchema)
```
Creates a Model named `"users"` — Mongoose will use the collection `users` in MongoDB.

---

### `Backend/src/models/blacklist.model.js`

```js
const blacklistTokenSchema = new mongoose.Schema({ token: String }, { timestamps: true })
```
Every time a user logs out, their JWT is stored here.
`timestamps: true` adds `createdAt` automatically.

**Why needed?** JWTs are stateless — once issued they are valid until expiry.
To "invalidate" a token on logout, we store it here and check every request against this list.

---

### `Backend/src/models/interviewReport.model.js`

This model stores the complete AI output. Key design decisions:

```js
{ _id: false }   // on sub-schemas
```
`_id: false` on `technicalQuestionSchema` and `behavioralQuestionSchema` prevents Mongoose from adding a separate `_id` to each question object — keeps the array lean.

```js
skillGaps: [{ skill: String, severity: { enum: ["low","medium","high"] } }]
```
`enum` enforces that severity can only be one of three values.

```js
user: { type: mongoose.Schema.Types.ObjectId, ref: "users" }
```
A reference (foreign key) to the `users` collection. Allows Mongoose `.populate()` later.

```js
{ timestamps: true }
```
Adds `createdAt` and `updatedAt` to every report automatically.

---

### `Backend/src/middlewares/auth.middleware.js`

```js
const token = req.cookies.token
if (!token) return res.status(401).json({ message: "Token not provided." })
```
Every protected route requires a token. If missing → 401.

```js
const isTokenBlacklisted = await tokenBlacklistModel.findOne({ token })
if (isTokenBlacklisted) return res.status(401).json({ message: "token is invalid" })
```
Even if the token is cryptographically valid, if the user already logged out it's rejected.

```js
const decoded = jwt.verify(token, process.env.JWT_SECRET)
req.user = decoded
next()
```
`jwt.verify` checks the signature and expiry. If valid, it returns the payload `{ id, username }`.
This gets attached to `req.user` so every controller knows who made the request.

---

### `Backend/src/middlewares/file.middleware.js`

```js
const upload = multer({
    storage: multer.memoryStorage(),
    limits: { fileSize: 3 * 1024 * 1024 }
})
```
- `memoryStorage()` — stores the file in RAM as a `Buffer` on `req.file.buffer`. No disk I/O.
- `fileSize: 3MB` — rejects files larger than 3 MB before they fully upload.

---

### `Backend/src/routes/auth.routes.js` — All Routes

| Route | Handler | What happens |
|---|---|---|
| `POST /api/auth/register` | `registerUserController` | Validates body → checks duplicate → hashes password → creates user → signs JWT → sets cookie → 201 |
| `POST /api/auth/login` | `loginUserController` | Finds user by email → compares password hash → signs JWT → sets cookie → 200 |
| `GET /api/auth/logout` | `logoutUserController` | Reads cookie token → saves to blacklist → clears cookie → 200 |
| `GET /api/auth/get-me` | `authUser` → `getMeController` | Verifies JWT → fetches user from DB by `req.user.id` → returns user data |

---

### `Backend/src/routes/interview.routes.js` — All Routes

| Route | Middlewares | Handler | What happens |
|---|---|---|---|
| `POST /api/interview/` | `authUser`, `upload.single("resume")` | `generateInterViewReportController` | Parse PDF → call AI → save to DB → 201 |
| `GET /api/interview/` | `authUser` | `getAllInterviewReportsController` | Fetch all reports for user (lightweight) → 200 |
| `GET /api/interview/report/:interviewId` | `authUser` | `getInterviewReportByIdController` | Fetch one full report by ID → 200 |
| `POST /api/interview/resume/pdf/:interviewReportId` | `authUser` | `generateResumePdfController` | Fetch report → call AI for HTML → Puppeteer → stream PDF → 200 |

---

### `Backend/src/controllers/auth.controller.js` — Line by Line

**`registerUserController`**
```js
const { username, email, password } = req.body
if (!username || !email || !password) return res.status(400)...
```
Manual validation. If any field is missing → reject with 400.

```js
const isUserAlreadyExists = await userModel.findOne({ $or: [{ username }, { email }] })
```
`$or` query checks if EITHER username OR email already exists in one DB query.

```js
const hash = await bcrypt.hash(password, 10)
```
`10` is the salt rounds. More rounds = slower hash = harder to brute-force. 10 is the standard.

```js
const token = jwt.sign({ id: user._id, username: user.username }, process.env.JWT_SECRET, { expiresIn: "1d" })
res.cookie("token", token)
```
Signs a token with the user's id and username as payload. Expires in 1 day.
Sets it as an HTTP cookie — browser stores and automatically sends it with every subsequent request.

**`logoutUserController`**
```js
const token = req.cookies.token
if (token) await tokenBlacklistModel.create({ token })
res.clearCookie("token")
```
Two-step logout:
1. Add the token to the blacklist so it can never be used again.
2. Delete the cookie from the browser.

---

### `Backend/src/controllers/interview.controller.js` — Line by Line

**`generateInterViewReportController`**
```js
const resumeContent = await (new pdfParse.PDFParse(Uint8Array.from(req.file.buffer))).getText()
```
`req.file.buffer` is the raw binary of the uploaded PDF.
`Uint8Array.from()` converts the Node Buffer to a typed array that `pdf-parse` expects.
`.getText()` extracts all plain text from the PDF pages.

```js
const interViewReportByAi = await generateInterviewReport({ resume: resumeContent.text, selfDescription, jobDescription })
```
Passes the three inputs to the AI service. Gets back a structured JS object.

```js
const interviewReport = await interviewReportModel.create({ user: req.user.id, ...interViewReportByAi, ... })
```
Spreads the AI output directly into the Mongoose `create()` call. `req.user.id` comes from the auth middleware.

**`getAllInterviewReportsController`**
```js
interviewReportModel.find({ user: req.user.id })
    .sort({ createdAt: -1 })
    .select("-resume -selfDescription -jobDescription -__v -technicalQuestions -behavioralQuestions -skillGaps -preparationPlan")
```
- `.find({ user: req.user.id })` — security: only return this user's reports.
- `.sort({ createdAt: -1 })` — newest first.
- `.select("-field")` — exclude heavy fields (minus prefix = exclude). Returns only `_id`, `title`, `matchScore`, `createdAt` for the list page. This avoids sending megabytes of question data just for the list.

**`generateResumePdfController`**
```js
res.set({ "Content-Type": "application/pdf", "Content-Disposition": `attachment; filename=resume_${id}.pdf` })
res.send(pdfBuffer)
```
Sets the correct MIME type so the browser treats the response as a PDF.
`attachment` tells the browser to download it rather than display it.

---

## 2. GenAI (Google Gemini) — Deep Dive

---

### What Model is Used?

```js
model: "gemini-3-flash-preview"
```
Gemini Flash is Google's fastest/cheapest model — optimized for structured output tasks where speed matters more than depth.

---

### How Structured Output Works (Zod → JSON Schema → Gemini)

This is the most important pattern in the codebase:

**Step 1 — Define shape with Zod:**
```js
const interviewReportSchema = z.object({
    matchScore: z.number().describe("A score 0-100 how well candidate matches the job"),
    technicalQuestions: z.array(z.object({
        question: z.string(),
        intention: z.string(),
        answer: z.string()
    })),
    behavioralQuestions: z.array(z.object({ ... })),
    skillGaps: z.array(z.object({
        skill: z.string(),
        severity: z.enum(["low", "medium", "high"])
    })),
    preparationPlan: z.array(z.object({
        day: z.number(),
        focus: z.string(),
        tasks: z.array(z.string())
    })),
    title: z.string()
})
```

**Step 2 — Convert to JSON Schema:**
```js
const jsonSchema = zodToJsonSchema(interviewReportSchema)
```
Converts the Zod definition into a JSON Schema object that the Gemini API understands.

**Step 3 — Pass to Gemini as a constraint:**
```js
const response = await ai.models.generateContent({
    model: "gemini-3-flash-preview",
    contents: prompt,
    config: {
        responseMimeType: "application/json",   // Force JSON output
        responseSchema: jsonSchema,              // Force this exact shape
    }
})
```
- `responseMimeType: "application/json"` — tells Gemini to output raw JSON, not markdown.
- `responseSchema` — Gemini's **constrained decoding** feature. The model is forced to produce output that strictly matches the schema. Fields like `severity` are constrained to only `"low"`, `"medium"`, or `"high"` — Gemini literally cannot output anything else.

**Step 4 — Parse:**
```js
return JSON.parse(response.text)
```
The response is always valid JSON matching the schema, so this never fails.

---

### GenAI Call 1 — Interview Report Generation

**Prompt:**
```
Generate an interview report for a candidate with the following details:
    Resume: [extracted PDF text]
    Self Description: [user's text]
    Job Description: [pasted job description]
```

**What Gemini does internally:**
1. Reads the resume and understands the candidate's skills, experience, and background.
2. Reads the job description and understands required skills, responsibilities, and seniority.
3. Compares both to calculate the match score.
4. Generates technical questions that would be asked for this specific role.
5. Generates behavioral questions relevant to the role level.
6. Identifies skill gaps — things the job requires that the candidate lacks.
7. Creates a realistic day-by-day plan to fill those gaps before the interview.
8. Extracts the job title from the description.
9. Returns everything in the constrained JSON schema.

**Example Output Shape:**
```json
{
  "matchScore": 74,
  "title": "Senior Frontend Engineer",
  "technicalQuestions": [
    {
      "question": "Explain how React's reconciliation algorithm works.",
      "intention": "Tests deep understanding of React internals, not just surface usage.",
      "answer": "Explain the Virtual DOM diffing process, the fiber architecture in React 18+..."
    }
  ],
  "behavioralQuestions": [
    {
      "question": "Tell me about a time you had a conflict with a teammate.",
      "intention": "Assesses conflict resolution and communication skills.",
      "answer": "Use the STAR method: Situation, Task, Action, Result. Focus on resolution..."
    }
  ],
  "skillGaps": [
    { "skill": "TypeScript", "severity": "high" },
    { "skill": "GraphQL", "severity": "medium" }
  ],
  "preparationPlan": [
    { "day": 1, "focus": "TypeScript Fundamentals", "tasks": ["Read TypeScript handbook ch1-3", "Convert a JS project to TS"] },
    { "day": 2, "focus": "System Design", "tasks": ["Watch Gaurav Sen system design playlist", "Practice designing a URL shortener"] }
  ]
}
```

---

### GenAI Call 2 — Resume PDF Generation

**Prompt:**
```
Generate resume for a candidate with the following details:
    Resume: [original resume text]
    Self Description: [user text]
    Job Description: [job description]

    The response should be a JSON object with a single field "html" which contains
    the HTML content of the resume...
    - Tailored for the given job description
    - ATS friendly
    - Not sound AI-generated
    - 1-2 pages when printed
    - Simple and professional design
```

**Schema:**
```js
const resumePdfSchema = z.object({
    html: z.string()
})
```

**What Gemini does:**
1. Reads the candidate's background from the resume/self-description.
2. Reads what the job is looking for.
3. Rewrites and restructures the resume to emphasize relevant skills and experience.
4. Generates clean, print-ready HTML with inline CSS (for Puppeteer compatibility).
5. Keeps it ATS-friendly (no images, tables with simple structure, semantic text).
6. Returns `{ "html": "<html>...</html>" }`.

**Then Puppeteer converts it to PDF:**
```js
const browser = await puppeteer.launch()
const page = await browser.newPage()
await page.setContent(htmlContent, { waitUntil: "networkidle0" })
// waitUntil: "networkidle0" = wait until no network requests for 500ms
// This ensures any fonts/styles referenced in the HTML are fully loaded

const pdfBuffer = await page.pdf({
    format: "A4",
    margin: { top: "20mm", bottom: "20mm", left: "15mm", right: "15mm" }
})
await browser.close()
return pdfBuffer
```

Puppeteer opens a real headless Chrome, renders the HTML exactly as a browser would, then uses Chrome's print-to-PDF engine to generate a pixel-perfect PDF. The buffer is sent directly to the client.

---

## 3. FRONTEND — Every File Explained

---

### `Frontend/src/main.jsx`
```jsx
ReactDOM.createRoot(document.getElementById('root')).render(<App />)
```
Mounts React into the `<div id="root">` in `index.html`. This is the single entry point.

---

### `Frontend/src/App.jsx`
```jsx
<AuthProvider>
  <InterviewProvider>
    <RouterProvider router={router} />
  </InterviewProvider>
</AuthProvider>
```
Context providers wrap the entire app. This means ANY component anywhere in the tree can access auth state or interview state via hooks — no prop drilling needed.

---

### `Frontend/src/app.routes.jsx`
```jsx
export const router = createBrowserRouter([
    { path: "/login",                  element: <Login /> },
    { path: "/register",               element: <Register /> },
    { path: "/",                       element: <Protected><Home /></Protected> },
    { path: "/interview/:interviewId", element: <Protected><Interview /></Protected> }
])
```
`createBrowserRouter` uses the browser's History API — URLs change without page reloads.
`:interviewId` is a dynamic segment; its value is read with `useParams()` inside `Interview.jsx`.

---

### `Frontend/src/features/auth/auth.context.jsx`
```jsx
export const AuthContext = createContext()

export const AuthProvider = ({ children }) => {
    const [user, setUser] = useState(null)    // null = not logged in
    const [loading, setLoading] = useState(true) // true initially = checking session

    return (
        <AuthContext.Provider value={{ user, setUser, loading, setLoading }}>
            {children}
        </AuthContext.Provider>
    )
}
```
`loading` starts as `true` because on first render we don't know if the user is logged in yet — we need to check the cookie with `getMe()` first.

---

### `Frontend/src/features/auth/hooks/useAuth.js`
```jsx
useEffect(() => {
    const getAndSetUser = async () => {
        try {
            const data = await getMe()    // calls GET /api/auth/get-me
            setUser(data.user)            // if cookie valid → restore session
        } catch(err) {}
        finally { setLoading(false) }    // always stop loading
    }
    getAndSetUser()
}, [])  // runs once on app mount
```
This `useEffect` is the **session restoration** logic. When the app first loads, it silently checks if the user has a valid cookie. If yes → user is auto-logged-in. If no → `user` stays `null`.

```jsx
const handleLogin = async ({ email, password }) => {
    setLoading(true)
    try {
        const data = await login({ email, password })
        setUser(data.user)
    } catch(err) {}
    finally { setLoading(false) }
}
```
Same pattern for all three actions. `setLoading(true)` during the call, `setLoading(false)` in `finally` so the spinner always stops even on error.

---

### `Frontend/src/features/auth/components/Protected.jsx`
```jsx
const Protected = ({ children }) => {
    const { loading, user } = useAuth()

    if (loading) return <main><h1>Loading...</h1></main>
    if (!user)   return <Navigate to="/login" />
    return children
}
```
Three states:
1. **Loading** — session check in progress → show spinner (prevents flash of login redirect).
2. **No user** — not authenticated → redirect to `/login`.
3. **Has user** — render the protected page.

---

### `Frontend/src/features/auth/services/auth.api.js`
```js
const api = axios.create({
    baseURL: "http://localhost:3000",
    withCredentials: true   // ← sends cookies with every request
})
```
`withCredentials: true` is critical. Without it, the browser won't send the JWT cookie to the backend (cross-origin security policy).

---

### `Frontend/src/features/interview/interview.context.jsx`
```jsx
const [loading, setLoading] = useState(false)  // false = no active request
const [report, setReport] = useState(null)     // single report for Interview page
const [reports, setReports] = useState([])     // list of reports for Home page
```

---

### `Frontend/src/features/interview/hooks/useInterview.js`

```js
const { interviewId } = useParams()

useEffect(() => {
    if (interviewId) {
        getReportById(interviewId)   // on /interview/:id → fetch that report
    } else {
        getReports()                 // on / → fetch all reports list
    }
}, [interviewId])
```
Smart auto-fetch: the same hook is used on both Home and Interview pages. The presence of `interviewId` in the URL determines which data to fetch.

```js
const getResumePdf = async (interviewReportId) => {
    response = await generateResumePdf({ interviewReportId })  // blob response
    const url = window.URL.createObjectURL(new Blob([response], { type: "application/pdf" }))
    const link = document.createElement("a")
    link.href = url
    link.setAttribute("download", `resume_${interviewReportId}.pdf`)
    document.body.appendChild(link)
    link.click()
}
```
Programmatic download technique:
1. Receives binary PDF data as a Blob from Axios.
2. Creates a temporary browser object URL pointing to the blob.
3. Creates a hidden `<a>` tag with `download` attribute.
4. Simulates a click → browser downloads the file.
5. The element is appended/clicked but never visible to the user.

---

### `Frontend/src/features/interview/services/interview.api.js`

```js
export const generateInterviewReport = async ({ jobDescription, selfDescription, resumeFile }) => {
    const formData = new FormData()
    formData.append("jobDescription", jobDescription)
    formData.append("selfDescription", selfDescription)
    formData.append("resume", resumeFile)    // File object from input[type=file]

    const response = await api.post("/api/interview/", formData, {
        headers: { "Content-Type": "multipart/form-data" }
    })
    return response.data
}
```
`FormData` is needed because we're sending both text fields AND a binary file in one request. Multer on the backend reads this `multipart/form-data` format.

```js
export const generateResumePdf = async ({ interviewReportId }) => {
    const response = await api.post(`/api/interview/resume/pdf/${interviewReportId}`, null, {
        responseType: "blob"   // ← tells Axios to receive binary data, not parse as JSON
    })
    return response.data
}
```
`responseType: "blob"` is critical — without it Axios would try to parse the binary PDF as text/JSON and corrupt it.

---

### `Frontend/src/features/interview/pages/Home.jsx`

```jsx
const resumeInputRef = useRef()
```
`useRef` creates a reference to the hidden file input. When the user clicks the dropzone label, the browser opens the file picker. `resumeInputRef.current.files[0]` then reads the selected file.

```jsx
const handleGenerateReport = async () => {
    const resumeFile = resumeInputRef.current.files[0]
    const data = await generateReport({ jobDescription, selfDescription, resumeFile })
    navigate(`/interview/${data._id}`)  // redirect to the new report
}
```
After generation, the app immediately navigates to the interview page for the new report. The `_id` is returned from the backend.

---

### `Frontend/src/features/interview/pages/Interview.jsx`

```jsx
const NAV_ITEMS = [
    { id: 'technical',  label: 'Technical Questions', icon: <svg.../> },
    { id: 'behavioral', label: 'Behavioral Questions', icon: <svg.../> },
    { id: 'roadmap',    label: 'Road Map',             icon: <svg.../> },
]
```
Navigation config as a data array. Rendered with `.map()` — adding a new section only requires adding an item here.

```jsx
const QuestionCard = ({ item, index }) => {
    const [open, setOpen] = useState(false)
    return (
        <div className='q-card'>
            <div className='q-card__header' onClick={() => setOpen(o => !o)}>
                Q{index + 1}: {item.question}  [chevron icon]
            </div>
            {open && (
                <div className='q-card__body'>
                    <Intention> + <Model Answer>
                </div>
            )}
        </div>
    )
}
```
Each question is a self-contained collapsible card. `open` state is local to each card so opening one doesn't affect others.

```jsx
const scoreColor =
    report.matchScore >= 80 ? 'score--high' :
    report.matchScore >= 60 ? 'score--mid'  : 'score--low'
```
CSS class is computed dynamically from the score. The class maps to green/yellow/red styling in the SCSS.

```jsx
{report.skillGaps.map((gap, i) => (
    <span key={i} className={`skill-tag skill-tag--${gap.severity}`}>
        {gap.skill}
    </span>
))}
```
Each skill gap tag gets a class like `skill-tag--high`, `skill-tag--medium`, or `skill-tag--low` — styled differently in SCSS.

---

## 4. Complete Request-Response Reference

### POST `/api/auth/register`
**Request:**
```json
{ "username": "john", "email": "john@example.com", "password": "secret123" }
```
**Response (201):**
```json
{ "message": "User registered successfully", "user": { "id": "...", "username": "john", "email": "john@example.com" } }
```
**Side effect:** Sets `token` cookie on the client.

---

### POST `/api/auth/login`
**Request:** `{ "email": "john@example.com", "password": "secret123" }`
**Response (200):** Same shape as register.
**Side effect:** Sets `token` cookie.

---

### GET `/api/auth/logout`
**Response (200):** `{ "message": "User logged out successfully" }`
**Side effects:** Token added to blacklist collection. `token` cookie cleared.

---

### GET `/api/auth/get-me` 🔒
**Response (200):** `{ "message": "...", "user": { "id", "username", "email" } }`

---

### POST `/api/interview/` 🔒
**Request:** `multipart/form-data` with fields:
- `resume` (file) — PDF, max 3MB
- `jobDescription` (text)
- `selfDescription` (text)

**Response (201):** Full `interviewReport` object with all AI fields.

---

### GET `/api/interview/` 🔒
**Response (200):**
```json
{
  "interviewReports": [
    { "_id": "...", "title": "Senior Frontend Engineer", "matchScore": 74, "createdAt": "..." }
  ]
}
```
Only lightweight fields — no questions/resume text.

---

### GET `/api/interview/report/:interviewId` 🔒
**Response (200):** Full `interviewReport` with all fields including questions, skill gaps, plan.

---

### POST `/api/interview/resume/pdf/:interviewReportId` 🔒
**Response (200):** Binary PDF stream.
Headers:
```
Content-Type: application/pdf
Content-Disposition: attachment; filename=resume_<id>.pdf
```
