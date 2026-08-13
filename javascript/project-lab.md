# JavaScript Project Mastery

> **👋 Hey Fresher — Read This First!**

> - JavaScript is the **only language that runs inside a browser** — it's what makes web pages interactive. Every button click, form submission, dropdown menu, and live search you've ever used was powered by JavaScript.
> - This document uses **short code blocks** — each one covers exactly one concept — with bullet-point explanations right beside it so every line is clear.
> - You don't need to know everything before starting. We go from variables to async APIs step by step.
> - **Company in this project:** QuickHire — a job portal startup in Bengaluru. They need interactive features on their website: live job search, form validation, API calls to fetch listings, and a candidate dashboard. You just joined as a Junior Frontend Developer. Your lead is Samira. Let's build it.

#### What You Will Build in This Project

You will build **5 real JavaScript features** for QuickHire's job portal — each teaching you a new set of JS concepts through actual business problems.

Variables & Functions, DOM Manipulation, Events, Arrays & Objects, Fetch & APIs, Async/Await, ES6+, Error Handling

> **🧱 Phase 1 — JS Foundations**

> Variables, data types, functions, conditionals, and loops — explained through QuickHire's job data.

> Read and change web page content with JavaScript. Respond to button clicks, form inputs, and keyboard events.

> Store and process job listings using arrays and objects. Master map, filter, reduce and modern arrow functions.

> Call a real REST API to fetch job listings, handle loading states, and display errors gracefully.

> Build QuickHire's candidate dashboard: live search, filters, API integration, and localStorage persistence.

**Scene 1 — QuickHire Office, Bengaluru | First Day, Real Problems**

> **Samira** _Senior Frontend Developer — QuickHire_
> 
> Yash, our job listings page currently refreshes the entire page every time someone types in the search box. That's terrible UX — it should filter results instantly as you type, without any page reload. That's your first task. Pure JavaScript. No frameworks, no libraries. You'll learn more from doing this in vanilla JS than copying a React tutorial.

> **Yash (You)** _Junior Frontend Developer — Day 1 at QuickHire_
> 
> I have studied JavaScript basics — variables, if-else, for loops. But I'm not sure how to make the page change dynamically without a reload. How does the JS connect to the HTML on the page?

> **Samira** _Senior Frontend Developer_
> 
> Through the DOM — Document Object Model. Every HTML element on the page is a JavaScript object you can read and modify. You grab the search input element, listen for when its value changes, filter your jobs array, and update the page content — all without any reload. By end of today you'll have it working. Let's start with the foundations.

### 1. Phase 1 — JavaScript Foundations

Before building any features, understand the core building blocks: variables, data types, functions, and control flow. Every JS program is built from these.

> **The Big Picture — What JavaScript Actually Does**

> - JavaScript runs **inside the browser** — when a page loads, the browser downloads your .js files and executes them.
> - JS can **read and change any HTML element** on the page in real time — text, styles, attributes, visibility.
> - JS responds to **user actions** — clicks, typing, scrolling, mouse movement — without reloading the page.
> - JS can **call APIs** to fetch data from servers and display it dynamically.
> - Everything you see "happen" on a web page without a refresh — that's JavaScript.

#### 1.1 Variables — let, const, and var

```
// Three ways to declare a variable
let   jobTitle = "Frontend Developer";
const maxSalary = 1200000;
var   oldStyle  = "avoid this";

// let can be changed
jobTitle = "Full Stack Developer";

// const cannot be changed
maxSalary = 999; // ❌ Error!
```

**📖 Which Variable to Use**

- **const** — use for values that should never change (API URLs, fixed prices, config). This is your default choice.
- **let** — use when the value will change (loop counters, search query, selected item).
- **var** — old style, has confusing scoping bugs. Never use it in modern JS.
- Rule of thumb: **start with const, switch to let only if you need to reassign.**

#### 1.2 Data Types

```
// JavaScript has 7 primitive types
const title    = "DevOps Engineer";  // string
const salary   = 950000;          // number
const isActive = true;            // boolean
const manager  = null;            // null (intentionally empty)
let   startDate;                    // undefined (not set yet)
// Check the type
console.log(typeof salary);  // "number"
```

**📖 Data Types Explained**

- **string** — text in quotes: `"hello"`, `'world'`, or backtick template literals ``Hello ${name}``
- **number** — integers and decimals: `42`, `3.14`, `-100`
- **boolean** — only `true` or `false`
- **null** — you explicitly set a value to "nothing"
- **undefined** — variable was declared but never given a value
- **typeof** — operator that tells you the type of any value

#### 1.3 Functions — Reusable Blocks of Code

```
// Traditional function declaration
function formatSalary(amount) {
  return "₹" + amount.toLocaleString("en-IN");
}

// Arrow function (ES6 — modern style)
const formatSalary = (amount) => {
  return "₹" + amount.toLocaleString("en-IN");
};

// Short arrow (one-liner, return implicit)
const double = n => n * 2;

console.log(formatSalary(950000)); // ₹9,50,000
```

**📖 Functions — Three Styles**

- **Function declaration** — classic style, can be called before it's defined (hoisted).
- **Arrow function** — modern ES6 style. More concise. `=>` means "returns".
- **Short arrow** — when the body is a single expression, you can omit `{}` and `return`.
- **Parameters** — variables that receive values when the function is called.
- **return** — sends a value back to whoever called the function. Without return, function gives `undefined`.

#### 1.4 Template Literals — The Modern Way to Build Strings

```
const company  = "Google";
const role     = "SDE-2";
const salary   = 2400000;

// Old way (confusing concatenation)
const old = "Job: " + role + " at " + company;

// Modern way — template literal with backticks
const msg = `Job: ${role} at ${company}
Salary: ₹${salary.toLocaleString("en-IN")}`;
```

**📖 Template Literals**

- Use **backticks** ( ` ) instead of quotes.
- **${expression}** — embed any JS expression directly inside the string.
- Can span **multiple lines** naturally — no \n needed.
- Much cleaner than joining strings with `+` — use template literals everywhere in modern code.

### 2. Phase 2 — DOM Manipulation and Events

**Business Problem:** QuickHire's "Apply Now" button does nothing when clicked — it needs to show a confirmation message. The job count at the top of the page is hardcoded — it needs to update dynamically. The page title needs to change when a user selects a job category. All of this is DOM manipulation.

**Scene 2 — QuickHire Dev Meeting | "The Page Is Static"**

> **Ronak** _Product Lead — QuickHire_
> 
> The product team wants the job count at the top to update as users filter results. And clicking "Apply" should immediately show a toast notification — "Application submitted!" — not redirect to a new page. This is all DOM work. You need to grab elements, read their content, change their text, show and hide elements — all with plain JavaScript. No React, no jQuery. Vanilla JS only.

#### 2.1 Selecting Elements from the Page

```
// Select ONE element
const searchBox = document
  .querySelector("#job-search");

const heading = document
  .querySelector(".page-title");

// Select MANY elements
const cards = document
  .querySelectorAll(".job-card");
```

**📖 querySelector Explained**

- **document.querySelector()** — finds the *first* element matching a CSS selector. Returns null if not found.
- **#job-search** — selects by ID (hash symbol = ID)
- **.page-title** — selects by class (dot = class)
- **document.querySelectorAll()** — finds *all* matching elements, returns a NodeList (array-like).
- Use the same selectors you know from CSS — that's intentional.

#### 2.2 Changing Page Content

```
const countEl  = document.querySelector("#job-count");
const container = document.querySelector("#jobs-list");

// Change text content (safe — no HTML injection)
countEl.textContent = "42 jobs found";

// Insert HTML (use carefully)
container.innerHTML = `<div class="card">
  <h3>DevOps Engineer</h3>
</div>`;

// Change a CSS style directly
countEl.style.color = "green";

// Add / remove a CSS class
countEl.classList.add("highlight");
countEl.classList.remove("hidden");
```

**📖 Changing Elements**

- **.textContent** — reads or sets the text inside an element. Safe: it treats everything as plain text, never interprets HTML tags.
- **.innerHTML** — reads or sets the HTML inside an element. Powerful but risky: never put user-typed text here (XSS attack risk).
- **.style.propertyName** — set a CSS property directly in JS (camelCase: `backgroundColor` not `background-color`).
- **.classList.add/remove/toggle** — the preferred way to change styling: add or remove CSS classes, keep styling in your CSS file.

#### 2.3 Event Listeners — Respond to User Actions

```
const applyBtn = document.querySelector("#apply-btn");

// Listen for a click event
applyBtn.addEventListener("click", function() {
  alert("Application submitted!");
});

// Same thing with arrow function (cleaner)
applyBtn.addEventListener("click", () => {
  alert("Application submitted!");
});
```

**📖 addEventListener**

- **addEventListener(eventType, function)** — "when this event happens on this element, run this function."
- **"click"** — fires when user clicks. Other events: `"input"`, `"submit"`, `"keydown"`, `"mouseover"`, `"change"`.
- The function passed in is called a **callback** — JavaScript calls it for you when the event fires.
- You can add multiple event listeners to the same element — they all fire independently.

#### 2.4 The Event Object — What Actually Happened

```
const searchInput = document.querySelector("#job-search");

searchInput.addEventListener("input", (event) => {
  // event.target = the element that was typed in
const query = event.target.value;
  console.log(`User typed: ${query}`);
  filterJobs(query);
});
```

**📖 The Event Object**

- The callback function automatically receives an **event object** — it contains information about what happened.
- **event.target** — the specific element that triggered the event (the input box in this case).
- **event.target.value** — what the user has typed into that input at this moment.
- **"input"** event — fires every time the content of an input changes (each keystroke). Great for live search.
- Other useful event properties: `event.key` (which key was pressed), `event.preventDefault()` (stop default browser behaviour).

> **💡 Fresher Tip — event.preventDefault() — When to Use It**

> - When a `<form>` is submitted, the browser's **default behaviour** is to reload the page and send data to a URL.
> - In modern web apps, you want to handle form data with JavaScript instead — `event.preventDefault()` stops the browser from doing its default action.
> - When a link is clicked, default is to navigate — `preventDefault()` stops navigation so you can handle it in JS.
> - Always call it as the **first line** inside your event handler: `event.preventDefault();`

### 3. Phase 3 — Arrays, Objects, and ES6 Methods

**Business Problem:** QuickHire has an array of 200 job listings. Users need to filter by title keyword, filter by minimum salary, and sort by date. Writing manual for-loops for all this is messy. Modern JavaScript's array methods (filter, map, reduce, sort) solve this cleanly in a few lines each.

**Scene 3 — QuickHire Sprint Planning | "The Filter Feature"**

> **Samira** _Senior Frontend Developer_
> 
> Yash, we have the job data in a JavaScript array. I need three things: filter by keyword, filter by salary range, and sort by posted date. A junior dev might write three nested for-loops. A professional uses filter(), filter() again, and sort(). Three lines of code instead of 30. Learn these array methods — they are used in every real project and tested in every JS interview.

#### 3.1 Objects — Structured Data

```
// An object representing one job listing
const job = {
  id:       101,
  title:    "DevOps Engineer",
  company:  "InfraCorp",
  salary:   1400000,
  location: "Hyderabad",
  isRemote: true,
  skills:   ["Docker", "Kubernetes", "AWS"]
};

// Access properties
console.log(job.title);         // "DevOps Engineer"
console.log(job["salary"]);    // 1400000
```

**📖 Objects in JavaScript**

- An **object** is a collection of **key: value** pairs, grouped in `{}`.
- Keys are called **properties**. Values can be any type — string, number, boolean, array, or even another object.
- **dot notation** (`job.title`) — use when you know the property name.
- **bracket notation** (`job["salary"]`) — use when the property name is stored in a variable.
- Objects are how you model real-world things: a user, a job, a product, an order.

#### 3.2 Arrays — Lists of Items

```
// Array of job objects
const jobs = [
  { title: "DevOps Engineer", salary: 1400000 },
  { title: "Frontend Dev",   salary: 900000  },
  { title: "Data Analyst",   salary: 800000  }
];

// Access by index (starts at 0!)
console.log(jobs[0].title); // "DevOps Engineer"
console.log(jobs.length);   // 3
// Add to end
jobs.push({ title: "Backend Dev", salary: 1100000 });
```

**📖 Arrays Explained**

- An **array** is an ordered list of items, stored in `[]`.
- Items are accessed by **index starting at 0** — first item is `array[0]`, not `array[1]`.
- **.length** — number of items in the array.
- **.push(item)** — add an item to the end.
- **.pop()** — remove the last item.
- Arrays of objects are the most common data structure in frontend JavaScript — API responses almost always return arrays of objects.

#### 3.3 filter() — Get a Subset of an Array

```
// Filter jobs by keyword in title
const results = jobs.filter(job =>
  job.title.toLowerCase()
     .includes(query.toLowerCase())
);

// Filter jobs above a salary threshold
const highPay = jobs.filter(job =>
  job.salary >= 1000000
);
```

**📖 filter() Method**

- **filter()** — returns a *new* array containing only items where the callback returns `true`.
- The original array is **not modified** — always creates a new array.
- The callback runs once per item — if it returns true, item is included; if false, item is excluded.
- **.toLowerCase().includes()** — case-insensitive search: convert both strings to lowercase before comparing.
- Used for: search features, filtering by category, filtering by price range.

#### 3.4 map() — Transform Every Item in an Array

```
// Create an HTML card for each job
const cards = jobs.map(job =>
`<div class="job-card">
    <h3>${job.title}</h3>
    <p>${job.company}</p>
    <span>₹${job.salary.toLocaleString("en-IN")}</span>
  </div>`
);

// cards is now an array of HTML strings
// Join them into one string and put on page
container.innerHTML = cards.join("");
```

**📖 map() Method**

- **map()** — returns a *new* array where every item has been transformed by the callback.
- The callback takes each item and *returns* what the new version should be.
- Original array is **not modified**.
- **.join("")** — joins an array of strings into one string with no separator. Used to combine multiple HTML strings into one block.
- Used for: converting raw data into HTML, extracting one property from each object, reformatting numbers.

#### 3.5 Destructuring — Clean Way to Unpack Values

```
// Object destructuring
const { title, company, salary } = job;
// Same as:
// const title   = job.title;
// const company = job.company;
// const salary  = job.salary;
// Array destructuring
const [first, second] = jobs;

// In a function parameter
const showJob = ({ title, salary }) => {
  console.log(`${title}: ₹${salary}`);
};
```

**📖 Destructuring**

- **Object destructuring** — extract multiple properties from an object into separate variables in one line.
- **Array destructuring** — extract items by position into named variables.
- The variable names **must match the property names** for object destructuring.
- In function parameters — you can destructure directly: the function receives the object but only extracts the properties it needs.
- Massively reduces repetition like `job.title, job.salary, job.company` everywhere.

#### 3.6 Spread Operator — Copy and Combine

```
// Spread array — copy without mutating
const allJobs    = [...jobs, newJob];

// Spread object — copy and override
const updatedJob = { ...job, salary: 1600000 };

// Combined: merge two arrays
const combined = [...frontendJobs, ...backendJobs];
```

**📖 Spread Operator (...)**

- **...** — "spread" = expand all items/properties of an array or object.
- **[...jobs, newJob]** — creates a new array with all existing jobs plus newJob. Original array unchanged.
- **{ ...job, salary: 1600000 }** — creates a new object with all job properties, but overrides salary. Original job unchanged.
- Critical for React and modern state management — you **never mutate** arrays or objects directly; you create new copies with spread.

### 4. Phase 4 — Fetch, Promises, and Async/Await

**Business Problem:** QuickHire's job listings are stored on a server — they need to be fetched from an API endpoint and displayed on the page. Fetching data over the network takes time. JavaScript must not freeze the page while waiting. That's why we need async programming.

**Scene 4 — QuickHire Backend Team Meeting | "Connect the Frontend to the API"**

> **Samira** _Senior Frontend Developer_
> 
> Yash, the backend team has built the REST API — it's live at api.quickhire.in/jobs. Your job: fetch the job listings from that URL and render them on the page. But understand this first: fetching data is asynchronous. The browser sends the request, but it doesn't wait there doing nothing. It continues running other code. When the data arrives, your callback function runs. That's the event loop. Understanding this is the difference between a junior and a solid JS developer.

> **Why Async? — The Simple Explanation**

> - JavaScript runs in one thread — it can only do **one thing at a time**.
> - Fetching data from a server can take **500ms to 3 seconds**. If JS waited (blocked), the entire page would freeze — no clicks, no scrolling, nothing.
> - Instead, JS **starts the fetch request and moves on** — the page stays responsive.
> - When the data arrives, JS runs your **callback function** to process it.
> - This "start now, finish later" pattern is called **asynchronous programming**. Promises and async/await are the modern tools to manage it cleanly.

#### 4.1 fetch() — Request Data from an API

```
// fetch returns a Promise
const promise = fetch("https://api.quickhire.in/jobs");

// .then() runs when data arrives
// .catch() runs if something goes wrong
fetch("https://api.quickhire.in/jobs")
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error(error));
```

**📖 fetch() and .then()**

- **fetch(url)** — sends an HTTP GET request. Returns a **Promise** — a placeholder that will hold the result when it arrives.
- **.then(callback)** — "when the data arrives, run this function." Returns a new Promise so you can chain.
- **response.json()** — the first .then gets the raw Response object. You must call .json() to parse the body into a JavaScript object.
- **.catch(callback)** — runs only if the request or parsing failed (network error, server down).

#### 4.2 async/await — The Cleaner Way

```
// async/await is cleaner than .then chains
async function loadJobs() {
  try {
    const response = await fetch(
      "https://api.quickhire.in/jobs"
    );
    const jobs = await response.json();
    renderJobs(jobs);
  } catch (error) {
    showError("Could not load jobs");
  }
}
```

**📖 async/await Explained**

- **async function** — marks a function as asynchronous. It can use the `await` keyword inside.
- **await** — "pause here and wait for this Promise to resolve, then give me the result." Makes async code look synchronous.
- The function does NOT block the browser — JS pauses only that function's execution, not the whole page.
- **try/catch** — the async/await way to handle errors. `try` contains the happy path; `catch` handles failures.
- This is the **modern standard** — prefer async/await over .then() chains.

#### 4.3 Show Loading State While Fetching

```
async function loadJobs() {
  const container = document.querySelector("#jobs");

  // Show spinner before fetch
  container.innerHTML = "<p>Loading...</p>";

  try {
    const res  = await fetch("/api/jobs");
    const jobs = await res.json();
    container.innerHTML = jobs
      .map(j => `<div>${j.title}</div>`)
      .join("");
  } catch {
    container.innerHTML = "<p>Error loading jobs.</p>";
  }
}
```

**📖 Loading States — Professional Pattern**

- Always show a **loading indicator** before the fetch — users must know data is coming.
- Replace the loading indicator with **actual data** in the try block after await.
- Replace with an **error message** in the catch block if it fails.
- This three-state pattern (Loading → Success or Error) is used in every real web app.
- In production you'd use a proper spinner component, but the pattern is identical.

### 5. Phase 5 — Complete Dashboard Project

**Business Problem:** QuickHire needs a candidate dashboard that: (1) fetches job listings from the API, (2) filters in real time as the user types, (3) saves the user's search preference so it persists after page refresh, and (4) shows a count of matching results. This is a complete, real feature combining everything you've learned.

**Scene 5 — Product Demo Day | "Build the Full Feature"**

> **Ronak** _Product Lead — QuickHire_
> 
> The investors are here tomorrow. We need the job dashboard working: search box, live filter, result count, and the search should remember what you typed last time you visited. Yash — this is your feature. Everything you've learned this week comes together here: fetch the data, filter with array methods, render with map, listen for input events, save to localStorage. You have until end of day today.

#### 5.1 localStorage — Remember Data Between Page Refreshes

```
// Save data to localStorage (persists forever)
localStorage.setItem("lastSearch", query);

// Read it back (even after page refresh)
const saved = localStorage.getItem("lastSearch");

// Save an object (must stringify/parse)
localStorage.setItem("filters",
  JSON.stringify({ role: "DevOps", min: 800000 })
);
const filters = JSON.parse(
  localStorage.getItem("filters") || "{}"
);
```

**📖 localStorage**

- **localStorage** — browser storage that persists even after the tab is closed and the page is refreshed.
- **setItem(key, value)** — save a string. Key and value must both be strings.
- **getItem(key)** — retrieve a string. Returns `null` if key doesn't exist.
- **JSON.stringify()** — convert an object to a JSON string (required because localStorage only stores strings).
- **JSON.parse()** — convert a JSON string back to a JS object.
- **|| "{}"** — fallback: if getItem returns null, use an empty object string to prevent JSON.parse crash.

#### 5.2 The Complete Dashboard — All Concepts Together

This is the full feature — short, focused blocks showing how each piece connects.

```
// dashboard.js — QuickHire Job Dashboard
const searchInput = document.querySelector("#search");
const jobsGrid    = document.querySelector("#jobs-grid");
const countLabel  = document.querySelector("#result-count");
let   allJobs     = [];  // holds all fetched jobs
```

> Select all the elements we need to interact with — done once at the top.
`allJobs` is declared with `let` (not const) because it will be assigned after the API call.
It starts as an empty array — we populate it after fetch completes.

```
// Fetch jobs from API on page load
async function init() {
  jobsGrid.innerHTML = "<p class='loading'>Loading jobs...</p>";
  try {
    const res  = await fetch("/api/jobs");
    allJobs = await res.json();
    const saved = localStorage.getItem("lastSearch") || "";
    searchInput.value = saved;
    renderJobs(saved);
  } catch {
    jobsGrid.innerHTML = "<p>Could not load jobs. Try again.</p>";
  }
}
```

> Show "Loading..." while waiting — never leave users staring at a blank screen.
After fetching, store jobs in `allJobs` — the master list never changes.
Load the saved search from localStorage and restore it — the search box remembers what the user typed last time.
Call `renderJobs(saved)` so if there was a saved search, results are already filtered on load.

```
// Filter and render jobs matching the query
function renderJobs(query = "") {
  const q = query.toLowerCase().trim();

  const filtered = q
    ? allJobs.filter(j =>
        j.title.toLowerCase().includes(q) ||
        j.skills.some(s => s.toLowerCase().includes(q))
      )
    : allJobs;

  countLabel.textContent = `${filtered.length} jobs found`;
  jobsGrid.innerHTML = filtered.map(j =>
`<div class="job-card">
      <h3>${j.title}</h3>
      <p>${j.company} — ${j.location}</p>
      <span>₹${j.salary.toLocaleString("en-IN")}</span>
    </div>`
  ).join("");
}
```

> **query = ""** — default parameter: if no argument passed, query is empty string (show all jobs).
**q = query.trim()** — remove leading/trailing spaces so " DevOps " behaves the same as "DevOps".
**q ? ... : allJobs** — ternary: if query is non-empty, filter; otherwise show all jobs.
**.some()** — returns true if *any* item in the array passes the test. Used here to check if any skill matches the query.
**filtered.map().join("")** — convert each job object into an HTML string, then join all strings into one HTML block.

```
// Listen for typing in the search box
searchInput.addEventListener("input", (e) => {
  const query = e.target.value;
  localStorage.setItem("lastSearch", query);
  renderJobs(query);
});

// Start the app
init();
```

> The **"input"** event fires on every keystroke — instant live filtering.
Save the current query to **localStorage** on every keystroke — it's always up to date.
Call **renderJobs(query)** — filter and re-render the list immediately.
Call **init()** at the bottom — this starts everything when the script loads.
The entire dashboard is ~40 lines of clean JavaScript. No framework. No library.

### 6. Common JavaScript Mistakes Freshers Make

**Mistakes and Fixes**

- **🔴 == vs ===** — 

- **🔴 Mutating Arrays/Objects** — 

- **🔴 Forgetting await** — 

- **🔴 querySelector on null** — 

- **🔴 var in loops** — 

- **🔴 innerHTML with user input** — 

### 7. Essential JavaScript Methods Reference

Method / Property

What It Does

Example

console.log()

Print value to browser console (F12)

console.log(jobs.length)

typeof value

Returns type as string

typeof 42 → "number"

String(n)

Convert number to string

String(42) → "42"

Number(s)

Convert string to number

Number("42") → 42

parseInt(s)

Parse integer from string

parseInt("42px") → 42

arr.filter(fn)

New array with items where fn returns true

jobs.filter(j => j.salary > 1000000)

arr.map(fn)

New array where each item is transformed

jobs.map(j => j.title)

arr.find(fn)

First item where fn returns true

jobs.find(j => j.id === 101)

arr.some(fn)

True if ANY item passes fn

jobs.some(j => j.isRemote)

arr.every(fn)

True if ALL items pass fn

jobs.every(j => j.salary > 0)

arr.reduce(fn, init)

Reduce array to single value

jobs.reduce((sum, j) => sum + j.salary, 0)

arr.sort(fn)

Sort array in place (mutates!)

[...jobs].sort((a,b) => b.salary - a.salary)

arr.includes(val)

True if array contains value

skills.includes("Docker")

arr.join(sep)

Join array into string

["a","b"].join(", ") → "a, b"

str.split(sep)

Split string into array

"a,b,c".split(",") → ["a","b","c"]

str.includes(sub)

True if string contains substring

"DevOps".includes("Dev") → true

str.trim()

Remove leading/trailing whitespace

" hello ".trim() → "hello"

JSON.stringify(obj)

Convert object to JSON string

JSON.stringify({a:1}) → '{"a":1}'

JSON.parse(str)

Convert JSON string to object

JSON.parse('{"a":1}') → {a:1}

localStorage.setItem(k,v)

Save string to browser storage

localStorage.setItem("key", "val")

setTimeout(fn, ms)

Run function after delay in milliseconds

setTimeout(() => alert("hi"), 2000)

### 8. Interview Questions — JavaScript

##### Interview Q&A — Fresher Level (0–1 Year JavaScript Experience)

**Q: Q1. What is the difference between let, const, and var?**

A: **var** — old, function-scoped (not block-scoped), hoisted to top of function. Causes bugs in loops and closures. Never use in modern JS.
**let** — block-scoped (lives only inside the `{}` it was declared in), can be reassigned. Use when value will change.
**const** — block-scoped, cannot be reassigned after declaration. Does not mean the value is immutable — you can still push to a const array or change object properties. Use as your default.
Rule: use const by default; switch to let only when you need to reassign.

**Q: Q2. What is the DOM and how does JavaScript interact with it?**

A: DOM stands for **Document Object Model** — when a browser loads an HTML page, it creates a tree of JavaScript objects, one per HTML element.
Every tag (`<div>`, `<p>`, `<button>`) becomes a JavaScript object with properties and methods.
**document.querySelector()** — finds an element by CSS selector and returns its JS object.
You then read or modify that object's properties: `.textContent`, `.innerHTML`, `.style`, `.classList` — and the browser immediately reflects the changes on screen.
This is why you can change a page's content without reloading — you're modifying the in-memory DOM tree, not the original HTML file.

**Q: Q3. What is the difference between == and === in JavaScript?**

A: **==** (loose equality) — compares values after type coercion. JS converts both values to the same type before comparing. `"5" == 5` is true. `0 == false` is true.
**===** (strict equality) — compares both value AND type. No coercion. `"5" === 5` is false. `0 === false` is false.
Always use **===**. The == coercion rules are confusing and cause subtle bugs. There is no legitimate reason to use == in modern JavaScript.

**Q: Q4. What is a Promise and how does async/await relate to it?**

A: A **Promise** is a JavaScript object representing a value that doesn't exist yet but will in the future — like a "receipt" for an operation that's in progress (e.g. a network request).
A Promise has three states: **pending** (still waiting), **fulfilled** (completed successfully), or **rejected** (failed).
**async/await** is syntactic sugar over Promises — it makes async code look and behave like synchronous code, but under the hood it's still Promises.
**await** pauses the async function's execution until the Promise resolves, then returns the result.
async/await + try/catch is the modern standard. Prefer it over raw .then()/.catch() chains.

**Q: Q5. What is the difference between map(), filter(), and reduce()?**

A: **map(fn)** — transforms every item. Returns a *new array* of the same length. Example: convert an array of job objects into an array of HTML strings.
**filter(fn)** — keeps only items where fn returns true. Returns a *new array* that is equal or shorter. Example: keep only jobs above ₹10 lakh.
**reduce(fn, init)** — reduces an entire array to a single value. The callback receives an accumulator and the current item. Example: sum all salaries: `jobs.reduce((sum, j) => sum + j.salary, 0)`.
All three return a new array/value — they do not modify the original array.
Can be chained: `jobs.filter(j => j.isRemote).map(j => j.title)` — filter remote jobs, then extract just their titles.

**Q: Q6. What is event bubbling and how does event.stopPropagation() work?**

A: When you click an element, the event fires on that element AND on every parent element up to `document`. This is called **event bubbling**.
Example: click a button inside a div — the click event fires on the button, then the div, then body, then document.
**event.stopPropagation()** — stops the event from bubbling up to parent elements. Useful when you have a click handler on a parent and don't want it to fire when a child is clicked.
**Event delegation** — instead of adding listeners to every child, add one listener to the parent and check `event.target` to see which child was clicked. More efficient for large lists.

**Quiz: Quiz 1 — What will this code print? const a = [1,2,3]; a.push(4); console.log(a);**

- A) Error — you can't modify a const array
- B) [1, 2, 3, 4]
- C) [4, 1, 2, 3]
- D) [1, 2, 3] — push doesn't work on const

> **Answer/explanation:** ✅ Answer: **B — [1, 2, 3, 4]**
**const prevents reassignment** — you can't do `a = [1,2,3,4]` (that's a new value).
But **const does NOT make the contents immutable** — you can still push, pop, or modify properties of the object/array.
Think of const as: "this variable name will always point to the same array object." The array's contents can still change.

**Quiz: Quiz 2 — You write: const data = fetch("/api/jobs"); console.log(data); — What does data contain?**

- A) The array of job objects from the API
- B) A Promise object (not the data yet)
- C) null (fetch is not defined without import)
- D) An error message

> **Answer/explanation:** ✅ Answer: **B — A Promise object**
fetch() returns a Promise *immediately* — the network request hasn't even finished yet.
You'll see `Promise { <pending> }` in the console — the data is not there yet.
To get the actual data you need: `const res = await fetch("/api/jobs"); const data = await res.json();`
Forgetting `await` is one of the most common mistakes when learning async JS.

**Quiz: Quiz 3 — What does this return? [10, 5, 25, 15].filter(n => n > 10).map(n => n * 2)**

- A) [20, 10, 50, 30]
- B) [50, 30]
- C) [10, 5, 25, 15] — arrays cannot be chained
- D) [25, 15] — filter result only

> **Answer/explanation:** ✅ Answer: **B — [50, 30]**
**Step 1 — filter(n > 10):** keeps 25 and 15 → `[25, 15]`
**Step 2 — map(n * 2):** doubles each → `[50, 30]`
Array methods are **chainable** because each one returns a new array.
This chaining pattern is used constantly in real projects to transform and filter data in one readable expression.

> **JavaScript Project — Core Takeaways for Freshers**

> - Always use **const** as default, **let** when you need to reassign, and **never use var** — it has scoping quirks that cause subtle bugs.
> - Always use **===** for comparisons, never ==. Loose equality with type coercion is a source of bugs in every codebase.
> - Master the four array methods: **filter** (select subset), **map** (transform), **reduce** (aggregate), and **find** (get first match). These replace most for-loops in modern JS.
> - Always use **async/await with try/catch** for API calls — never leave async code without error handling. Users must always see something useful (loading state → success or error message).
> - Use **textContent** for inserting user-typed text into the DOM. Only use innerHTML for controlled, sanitised content — innerHTML with user input is an XSS security vulnerability.
> - Use **event.target.value** inside event handlers to read the current value of an input — not a stored variable that might be stale.
> - Use **localStorage** to persist user preferences between sessions — search queries, selected filters, dark mode preference, authentication tokens.
> - Open the browser **DevTools Console (F12)** constantly while developing — console.log() and reading error messages there is how you debug JS. Get comfortable with it from day one.

##### JavaScript Code Standards — QuickHire Engineering Rules

- Separate concerns: keep JavaScript in `.js` files, not inside HTML `<script>` tags inline with markup — makes code reviewable and cacheable
- Use descriptive variable names: `filteredJobs` not `arr2`, `handleSearchInput` not `fn`
- Add `async` functions for any operation that involves a network request, timer, or file read — never block the main thread
- Always handle the error case in every fetch call — what does the user see if the API is down? That must be designed and coded, not ignored
- When sorting arrays, sort a **copy** (`[...arr].sort()`) not the original — sort() mutates the array in place and will break features that depend on the original order
- Use `console.error()` for errors (shows red in DevTools), `console.warn()` for warnings, `console.log()` for general debugging — and remove all console.log calls before pushing to production

##### 🏋️ Hands-On Exercises — Extend the Dashboard

1. **Add a salary filter:** Add a range input (`<input type="range">`) that filters jobs by minimum salary. Listen for its "input" event, read `event.target.value`, and add a second `.filter()` call in `renderJobs()` that keeps only jobs above the threshold. Display the current threshold value dynamically using `textContent`.
2. **Add sort by salary:** Add a `<select>` dropdown with options "Salary: High to Low" and "Salary: Low to High". Listen for the "change" event. In renderJobs(), after filtering, add a `[...filtered].sort()` call before the map. Use the selected value to determine sort direction.
3. **Add a "Save Job" feature with localStorage:** Add a bookmark icon to each job card. When clicked, toggle the job's ID in a `savedJobIds` array stored in localStorage. Jobs whose IDs are in the saved list should render with a highlighted bookmark. Add a "Saved Jobs" tab that shows only bookmarked jobs.
4. **Add debouncing to the search:** Currently, renderJobs() is called on every keystroke. For a real API call, that's too many requests. Implement debouncing: use setTimeout to delay the renderJobs call by 300ms after the last keystroke. If the user types again within 300ms, clear the previous timeout and restart. This is a real interview question.
5. **Handle HTTP errors properly:** Currently if the API returns a 404 or 500 error, the code treats it as success because fetch() only rejects on network failure. Add a check: `if (!res.ok) throw new Error("Server error: " + res.status)`. This is how professional code distinguishes "request reached the server but server returned an error" from "request never reached the server."

### JavaScript Project Complete 🎉

You have built QuickHire's complete live job dashboard — from raw JavaScript fundamentals to a real async application with DOM manipulation, array methods, API calls, error handling, and localStorage persistence. Everything a company actually needs from a junior frontend developer.

> **Samira**
> 
> "Yash, the live search you built filters 200 job listings as you type with zero lag. The investor demo went perfectly — the CEO typed "DevOps" and results appeared instantly. No page reload, no spinner, just instant filtering. That is what professional JavaScript looks like. And you wrote it in one day."

> **Ronak**
> 
> "And the saved search in localStorage — the lead investor typed a search, closed the tab, came back an hour later and the search was still there. He asked how you did it. I told him one line of code. He laughed. That's JavaScript done right."

> **Next: Advanced JavaScript — Closures, Prototypes, Modules & Testing**

> - Closures — functions that remember the variables from their outer scope; used in event handlers, factory functions, and module patterns
> - Prototype chain — how JavaScript objects inherit methods; understanding this explains how arrays have .map() and objects have .toString()
> - ES Modules — import/export syntax for splitting code into separate files; how real projects are structured
> - Debounce and throttle — performance patterns for search inputs and scroll events
> - JavaScript testing — write unit tests with Jest for your functions; test DOM interactions with Testing Library
> - Web APIs — Intersection Observer (lazy loading), Web Storage API, Fetch with headers and POST, FormData for file uploads
