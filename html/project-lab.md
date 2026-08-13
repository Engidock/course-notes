# HTML Project Mastery

> **👋 Hey Fresher — Read This First!**

> - HTML is the **skeleton of every web page**. Before CSS adds style and JavaScript adds behaviour, HTML defines what is on the page — headings, paragraphs, images, links, forms, tables.
> - Every website you have ever visited — Google, YouTube, Flipkart, Zomato — is built on HTML at its core.
> - This document uses **short code blocks** — each one covers exactly one concept — with a browser preview and bullet-point explanation for every single block.
> - **Company in this project:** BrightPath Academy — an ed-tech startup in Chennai building an online learning platform. You just joined as a Junior Frontend Developer. Your lead is Kavitha. You will build BrightPath's complete web pages from scratch using proper HTML.

#### What You Will Build in This Project

You will build **BrightPath Academy's complete website structure** — the Home page, Course Listing page, Contact page, and Enrolment Form — learning every important HTML concept along the way.

Document Structure, Text Elements, Links & Images, Lists & Tables, Forms, Semantic HTML, Accessibility, SEO Basics

> **🏗️ Phase 1 — Document Structure**

> DOCTYPE, html, head, body — understand the skeleton every web page is built on, and the tags inside <head> that matter for SEO.

> Headings, paragraphs, bold, italic, links, images — build BrightPath's home page content with proper text structure.

> Build BrightPath's course list with ordered and unordered lists, and a course comparison table with proper table structure.

> Build the student enrolment form — text inputs, dropdowns, radio buttons, checkboxes, date pickers, and submit button.

> Use header, nav, main, article, section, footer and accessibility attributes to make pages meaningful for screen readers and search engines.

**Scene 1 — BrightPath Academy Office, Chennai | First Task**

> **Kavitha** _Senior Frontend Developer — BrightPath Academy_
> 
> Welcome, Deepak. Our designer gave us the wireframes for the BrightPath website. Your job is to convert those wireframes into actual HTML pages. HTML is not about how things look — CSS handles that. HTML is about structure and meaning. A heading is a heading, a navigation menu is a navigation menu, a form is a form. You write HTML so that browsers, search engines, and screen readers all understand exactly what each part of the page IS.

> **Deepak (You)** _Junior Frontend Developer — Day 1 at BrightPath_
> 
> I know the basic tags like <p> and <h1> from tutorials. But I want to understand why we use each tag and how a real web page is actually structured — not just copying from examples.

> **Kavitha** _Senior Frontend Developer_
> 
> That's exactly the right question. Most freshers know the tags but not the WHY. We'll go through every tag with a reason. When you're done with this project, you'll be able to open any website, view its source code, and understand exactly what every element is doing and why it's there. That knowledge makes you a professional, not just a copy-paster.

> **Prasanna** _Tech Lead — BrightPath Academy_
> 
> And remember — HTML is about semantics, not just syntax. Google doesn't rank pages based on beautiful code. It ranks pages based on meaningful HTML. Using the right tags in the right places is what makes BrightPath appear in search results. A <div> where a <header> should be is not just sloppy code — it hurts our SEO ranking.

### 1. Phase 1 — HTML Document Structure

Every HTML page in the world starts with the same basic structure. Understanding this structure is step one before writing a single visible element.

> **The Big Picture — What HTML Actually Is**

> - HTML stands for **HyperText Markup Language** — it uses tags to mark up (label) content so browsers know what to display and how.
> - HTML describes **structure and meaning**. CSS describes appearance. JavaScript describes behaviour. These three are always separate concerns.
> - An HTML **element** consists of: an opening tag, content, and a closing tag: `<p>This is content</p>`
> - Some elements are **self-closing** (void elements) — they have no content or closing tag: `<img>`, `<br>`, `<input>`
> - **Attributes** go inside the opening tag and give extra information: `<a href="...">` — `href` is the attribute.

#### 1.1 The Complete HTML Boilerplate

```
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport"
content="width=device-width,
                 initial-scale=1.0">
<title>BrightPath Academy</title>
</head>
<body>
<!-- All visible content goes here -->
</body>
</html>
```

**📖 Every Line Explained**

- **<!DOCTYPE html>** — tells the browser "this is an HTML5 document." Must be on line 1. Without it, browsers enter "quirks mode" and render things differently.
- **<html lang="en">** — root element that wraps everything. `lang="en"` tells screen readers and search engines the page is in English.
- **<head>** — contains metadata (info about the page) — NOT visible on screen. Holds title, charset, viewport, CSS links, SEO meta tags.
- **charset="UTF-8"** — supports all characters including Indian languages, emojis, and special symbols.
- **viewport meta** — makes the page responsive on mobile. Without this, your page looks zoomed out on phones.
- **<title>** — text shown in the browser tab and in Google search results. Every page must have a unique, descriptive title.
- **<body>** — everything visible on the page goes here.

#### 1.2 SEO Meta Tags — Help Google Find Your Page

```
<head>
<title>Online Courses | BrightPath Academy</title>
<meta name="description"
content="Learn coding, design
        and data science with expert mentors.">
<meta name="keywords"
content="online courses, coding,
        web development, Chennai">
</head>
```

**📖 Meta Tags for SEO**

- **description** — the text Google shows under your page title in search results. Keep it 150–160 characters. This directly affects click-through rate.
- **keywords** — less important for Google today but still used by some search engines and tools.
- These tags are **invisible to the user** but read by search engine bots.
- Every page should have a unique description — not the same one copy-pasted across all pages.

#### 1.3 Linking CSS and JavaScript

```
<head>
<!-- Link external CSS file -->
<link rel="stylesheet"
href="styles.css">
</head>
<body>
<!-- Link JavaScript at END of body -->
<script src="app.js"></script>
</body>
```

**📖 Linking Files**

- **<link rel="stylesheet">** — links an external CSS file. Goes inside <head>. The browser downloads and applies the CSS before rendering the page.
- **<script src="...">** — links a JavaScript file. Place it **at the end of <body>**, not in <head>.
- Why end of body for JS? — If placed in <head>, JS runs before the HTML is parsed and can't find DOM elements. At end of body, all HTML elements exist before JS runs.
- Modern alternative: `<script src="app.js" defer>` in <head> — `defer` makes the script run after HTML is fully parsed.

### 2. Phase 2 — Text, Links, and Images

**Business Problem:** BrightPath's home page needs a hero section with a heading, description text, a call-to-action link, and a promotional image. These are the most fundamental HTML elements — and the most misused by freshers.

**Scene 2 — Design Review, BrightPath | "Build the Home Page"**

> **Kavitha** _Senior Frontend Developer_
> 
> Deepak, the most common mistake I see from freshers is using headings for styling instead of structure. They write <h3> because "it looks the right size" — not because it's the third level of heading in the content hierarchy. Google reads headings to understand what your page is about. Use them for structure, not for font size. That's what CSS is for.

#### 2.1 Headings — Structure, Not Style

```
<h1>Learn to Code with BrightPath</h1>
<h2>Our Most Popular Courses</h2>
<h3>Web Development Track</h3>
<h4>Module 1: HTML & CSS</h4>
```

**📖 Heading Rules**

- **One <h1> per page** — it's the main title. Google reads this first to understand the page topic.
- **<h2>** — major sections of the page (like chapter titles).
- **<h3>** — subsections under an h2. Never skip levels (don't go h1 → h3, skipping h2).
- The hierarchy goes h1 → h2 → h3 → h4 → h5 → h6. Think of it as an **outline of your page content**.
- Never choose heading level based on visual size — that's CSS's job.

brightpath.in

## Learn to Code with BrightPath

### Our Most Popular Courses

#### Web Development Track

##### Module 1: HTML & CSS

#### 2.2 Paragraphs and Text Formatting

```
<p>BrightPath Academy offers expert-led
online courses for working professionals.</p>
<p>Enrol today and get
  <strong>50% off your first course</strong>.
  Limited seats available — act
  <em>now</em>!
</p>
<p>Use code <mark>BRIGHT50</mark> at checkout.</p>
<p>Course fee: ₹4,999 <del>₹9,999</del></p>
```

**📖 Text Tags**

- **<p>** — paragraph. Browser adds spacing before and after each one. Every block of text should be wrapped in <p>.
- **<strong>** — important text, rendered bold. Also tells screen readers to *stress* this word.
- **<em>** — emphasised text, rendered italic. Conveys emphasis in meaning, not just style.
- **<mark>** — highlighted text (yellow background by default).
- **<del>** — deleted/crossed-out text. Used for original price in discounts.
- Use **b** and **i** only for purely visual bold/italic with no semantic meaning.

brightpath.in

BrightPath Academy offers expert-led online courses for working professionals.

Enrol today and get **50% off your first course**. Limited seats available — act *now*!

Use code BRIGHT50 at checkout.

Course fee: ₹4,999 ₹9,999

#### 2.3 Links — Anchor Tags

```
<!-- Link to another page on same site -->
<a href="courses.html">Browse Courses</a>
<!-- Link to external website -->
<a href="https://youtube.com"
target="_blank"
rel="noopener noreferrer">
  Watch Free Preview
</a>
<!-- Jump to section on same page -->
<a href="#courses">Go to Courses</a>
```

**📖 Anchor Tag Attributes**

- **href** — the destination URL. Required. Without it, the link does nothing useful.
- **Relative path** (courses.html) — links to a file in the same folder as the current page.
- **Absolute URL** (https://...) — links to an external website.
- **target="_blank"** — opens link in a new browser tab.
- **rel="noopener noreferrer"** — always add this with target="_blank". Without it, the new tab can access your page's window object — a security vulnerability.
- **href="#id"** — jumps to an element with that ID on the same page (anchor link).

#### 2.4 Images

```
<!-- Basic image -->
<img
src="images/hero-banner.jpg"
alt="Students learning to code online"
width="800"
height="400"
>
<!-- Image with caption using figure -->
<figure>
<img src="mentor.jpg"
alt="Priya Sharma, Python mentor">
<figcaption>Priya Sharma — Python Lead</figcaption>
</figure>
```

**📖 Image Tag Rules**

- **alt attribute is mandatory** — it's the text shown if the image fails to load, and what screen readers read aloud to visually impaired users. Describe what the image shows.
- For decorative images that carry no meaning: `alt=""` (empty string) — tells screen readers to skip it.
- **width and height** — always specify these. They prevent "layout shift" (page jumping as images load), which hurts Google's Core Web Vitals score.
- **<figure> + <figcaption>** — semantically groups an image with its caption. More meaningful than a <div> + <p>.

### 3. Phase 3 — Lists and Tables

**Business Problem:** BrightPath's course listing page needs an unordered list of course features, an ordered list of learning steps, and a comparison table showing course details side by side.

#### 3.1 Unordered and Ordered Lists

```
<!-- Unordered list — bullet points -->
<ul>
<li>Live mentor sessions every week</li>
<li>Lifetime access to recordings</li>
<li>Certificate on completion</li>
</ul>
<!-- Ordered list — numbered steps -->
<ol>
<li>Enrol in your chosen course</li>
<li>Complete all video modules</li>
<li>Submit your final project</li>
<li>Receive your certificate</li>
</ol>
```

**📖 Lists Explained**

- **<ul>** (unordered list) — items where order doesn't matter. Rendered with bullet points. Use for features, options, navigation items.
- **<ol>** (ordered list) — items where order matters. Rendered with numbers. Use for steps, instructions, rankings.
- **<li>** (list item) — every item in any list must be wrapped in <li>. Don't put text directly inside <ul> or <ol>.
- Lists can be nested — put a <ul> inside an <li> to create a sub-list.

brightpath.in/courses

What you get:

- Live mentor sessions every week
- Lifetime access to recordings
- Certificate on completion

How it works:

1. Enrol in your chosen course
2. Complete all video modules
3. Submit your final project
4. Receive your certificate

#### 3.2 Tables — Comparison Data

```
<table>
<thead>
<tr>
<th>Course</th>
<th>Duration</th>
<th>Fee</th>
</tr>
</thead>
<tbody>
<tr>
<td>Web Development</td>
<td>3 months</td>
<td>₹4,999</td>
</tr>
<tr>
<td>Data Science</td>
<td>4 months</td>
<td>₹6,999</td>
</tr>
</tbody>
</table>
```

**📖 Table Structure**

- **<table>** — the outer wrapper for the entire table.
- **<thead>** — groups the header rows. Screen readers and browsers treat this differently from data rows.
- **<tbody>** — groups the data rows. Required for proper structure.
- **<tr>** — table row. All cells in a row go inside one <tr>.
- **<th>** — table header cell. Bold + centred by default. Use for column/row labels.
- **<td>** — table data cell. Regular text. The actual data content.
- Tables are for **tabular data only** — never use tables for page layout. That's CSS Grid/Flexbox's job.

brightpath.in/courses

Course

Duration

Fee

Web Development

3 months

₹4,999

Data Science

4 months

₹6,999

UI/UX Design

2 months

₹3,499

### 4. Phase 4 — Forms

**Business Problem:** BrightPath needs a student enrolment form — name, email, phone, course selection, experience level, preferred schedule, and a message. Forms are one of the most important and most incorrectly written parts of HTML. Every input must have a label. Every form needs proper accessibility.

**Scene 3 — BrightPath Accessibility Review | "Forms Without Labels"**

> **Prasanna** _Tech Lead — BrightPath Academy_
> 
> Deepak, I reviewed the enrolment form you built. The inputs have no labels — just placeholder text. A screen reader user cannot fill in this form. A visually impaired student cannot enrol in our courses. That is not just bad code — it is a legal accessibility issue in some countries and it hurts our SEO. Every input must have a proper <label> element connected to it.

#### 4.1 The Form Element

```
<form
action="/enrol"
method="POST"
>
<!-- All form fields go here -->
</form>
```

**📖 Form Attributes**

- **action** — the URL where form data is sent when submitted. In modern apps, JavaScript intercepts the submit event before it reaches the server.
- **method="POST"** — sends data in the request body (invisible in the URL). Use for forms with passwords, personal data, or large amounts of data.
- **method="GET"** — sends data as URL parameters (visible in the address bar). Use for search forms where results should be bookmarkable.

#### 4.2 Text Inputs with Labels

```
<!-- Label + Input connected by id/for -->
<label for="student-name">
  Full Name <span aria-hidden="true">*</span>
</label>
<input
type="text"
id="student-name"
name="studentName"
placeholder="Riya Sharma"
required
>
```

**📖 Label-Input Connection**

- **<label for="id">** — the `for` attribute matches the input's `id`. This connects them: clicking the label moves focus to the input.
- **id** — unique identifier for this input on the page. Used by the label's `for` attribute.
- **name** — the key used when form data is sent to the server. Required for all form fields.
- **placeholder** — hint text inside the input. *Not* a replacement for a label — it disappears when the user starts typing.
- **required** — HTML5 built-in validation. Browser refuses to submit if this field is empty.

#### 4.3 Input Types — More Than Just Text

```
<!-- Email (validates @ symbol) -->
<input type="email" id="email"
name="email" required>
<!-- Phone number -->
<input type="tel" id="phone"
name="phone"
pattern="[0-9]{10}">
<!-- Date picker -->
<input type="date" id="start-date"
name="startDate"
min="2026-01-01">
<!-- Password (hides characters) -->
<input type="password"
minlength="8">
```

**📖 Input Types — Use the Right One**

- **type="email"** — browser validates format (must contain @). Mobile keyboards show @ symbol.
- **type="tel"** — mobile keyboards show number pad. `pattern` attribute adds regex validation.
- **type="date"** — shows a date picker popup. `min/max` restrict the selectable range.
- **type="password"** — hides characters with dots. `minlength` enforces minimum length.
- **type="number"** — only allows numbers, shows spinner arrows.
- **type="checkbox"** — tick box (yes/no). **type="radio"** — one option from a group.
- Using the right input type gives you free validation and better mobile experience.

#### 4.4 Select Dropdown, Radio, and Checkbox

```
<!-- Dropdown select -->
<label for="course">Choose Course</label>
<select id="course" name="course">
<option value="">-- Select --</option>
<option value="webdev">Web Development</option>
<option value="data">Data Science</option>
</select>
```

**📖 Select Dropdown**

- **<select>** — creates a dropdown menu. The `name` attribute is sent to the server.
- **<option value="...">** — each selectable option. The `value` is what gets sent; the text between tags is what the user sees.
- First option with empty value is a placeholder ("-- Select --") — include this so the user can't accidentally submit without choosing.
- Add `required` to <select> to make selection mandatory.

```
<!-- Radio buttons — pick ONE from a group -->
<fieldset>
<legend>Experience Level</legend>
<input type="radio"
name="level"
id="beginner"
value="beginner">
<label for="beginner">Beginner</label>
<input type="radio"
name="level"
id="advanced"
value="advanced">
<label for="advanced">Advanced</label>
</fieldset>
```

**📖 Radio Buttons & Fieldset**

- **Radio buttons** — all options in a group must share the same `name`. This makes them mutually exclusive — selecting one deselects the others.
- Each radio button needs its own unique `id` and a corresponding `<label for="id">`.
- **<fieldset>** — groups related form controls. Adds a visual border by default.
- **<legend>** — labels the fieldset. Screen readers read the legend before each radio button's label, giving context.
- Always use fieldset + legend for radio button groups and checkbox groups.

```
<!-- Checkbox — can tick multiple -->
<input type="checkbox"
id="terms"
name="terms"
required>
<label for="terms">
  I agree to the
  <a href="/terms">Terms & Conditions</a>
</label>
<!-- Submit button -->
<button type="submit">
  Enrol Now
</button>
```

**📖 Checkbox & Submit**

- **Checkbox** — independent yes/no toggle. Multiple checkboxes with different names can all be checked simultaneously.
- When checked and form is submitted, the checkbox sends its `value`. When unchecked, nothing is sent for that field.
- **required** on a checkbox means "user must tick this before submitting" — used for mandatory terms agreement.
- **<button type="submit">** — submits the form when clicked. Always specify type="submit" — without it, buttons inside a form default to submit, which can cause unintended submits.
- **<button type="button">** — does nothing on its own; used when you want JavaScript to handle the click.

brightpath.in/enrol

### 5. Phase 5 — Semantic HTML and Accessibility

**Business Problem:** BrightPath's pages work but every element is a <div>. Google's crawler can't tell the difference between the navigation, the main content, a sidebar, and the footer — they all look the same. Screen reader users can't jump to the main content or skip the navigation. Semantic HTML fixes both problems.

**Scene 4 — SEO Review, BrightPath | "Google Can't Understand Our Site"**

> **Prasanna** _Tech Lead — BrightPath Academy_
> 
> Our pages are full of divs and spans with class names like "nav-container" and "main-wrapper". But HTML5 gives us tags that describe what a section IS — not just how to style it. <header>, <nav>, <main>, <article>, <section>, <aside>, <footer>. These semantic tags tell Google and screen readers the structure of the page. Using them is not optional — it directly affects our search ranking and accessibility compliance.

#### 5.1 Semantic Page Layout

```
<header>
<img src="logo.png" alt="BrightPath">
<nav>
<a href="/">Home</a>
<a href="/courses">Courses</a>
</nav>
</header>
<main>
<!-- Primary page content -->
</main>
<footer>
<p>© 2026 BrightPath Academy</p>
</footer>
```

**📖 Semantic Layout Tags**

- **<header>** — top section of the page (or a section). Typically contains logo and navigation. Not the same as <head>.
- **<nav>** — navigation links. Screen readers announce "navigation landmark" — users can skip directly to it.
- **<main>** — the primary unique content of the page. **One per page only.** Screen reader users press a shortcut to jump straight here, skipping the header/nav.
- **<footer>** — bottom of the page. Typically holds copyright, links, contact info.

#### 5.2 article, section, and aside

```
<main>
<section>
<h2>Featured Courses</h2>
<article>
<h3>Web Development</h3>
<p>Build real websites...</p>
</article>
<article>
<h3>Data Science</h3>
<p>Analyse data with Python...</p>
</article>
</section>
<aside>
<h3>Why BrightPath?</h3>
</aside>
</main>
```

**📖 Content Structure Tags**

- **<section>** — a thematic grouping of content with a heading. Use when content is related but not self-contained. Think of it as a chapter in a book.
- **<article>** — self-contained, independently distributable content. Each course card, each blog post, each news item is an article. Could be taken out of the page and still make sense on its own.
- **<aside>** — content tangentially related to the main content. Sidebars, callout boxes, related links, advertisements.
- Key question: "If I took this out of the page, would it still make sense alone?" Yes → article. No → section.

#### 5.3 ARIA and Accessibility

```
<!-- aria-label for icon buttons -->
<button aria-label="Close menu">
  ✕
</button>
<!-- aria-required for custom form fields -->
<input type="text"
aria-required="true"
aria-describedby="name-hint">
<span id="name-hint">Enter your full name</span>
<!-- Skip link for keyboard users -->
<a href="#main"
class="skip-link">
  Skip to main content
</a>
```

**📖 ARIA Accessibility Attributes**

- **aria-label** — provides a text description for screen readers when the visible text isn't descriptive enough. An "✕" button needs an aria-label of "Close menu" so screen readers say that instead of just "X".
- **aria-required="true"** — tells screen readers this field is required. Use alongside HTML `required` attribute.
- **aria-describedby** — points to another element's ID that provides additional description. Screen reader reads the input label then the description.
- **Skip link** — a hidden link that appears on Tab press, letting keyboard users jump directly to <main> without tabbing through every nav item. Required for accessibility compliance.

brightpath.in

🎓 BrightPath Academy

## Learn Skills That Get You Hired

Expert-led courses in web development, data science, and design.

### Featured Courses

#### Web Development

Build real websites with HTML, CSS & JavaScript.

#### Data Science

Analyse data with Python and visualise insights.

© 2026 BrightPath Academy | Chennai, India

> **💡 Fresher Tip — div and span — Use Them As Last Resort**

> - **<div>** — a generic block-level container with no semantic meaning. Use it only when no semantic element fits, or for CSS layout purposes.
> - **<span>** — a generic inline container with no semantic meaning. Use it only to apply CSS or JavaScript to a piece of text when no semantic tag fits.
> - Before writing <div>, ask: "Is this a header? A nav? An article? A section? A figure?" If yes, use that semantic tag instead.
> - A page full of <div>s works visually but is a machine-readable disaster — Google, screen readers, and developer tools can't understand the structure.

### 6. HTML Tags Quick Reference

Tag

What It Does

Notes

<!DOCTYPE html>

Declares HTML5 document

Must be first line, no closing tag

<html lang="en">

Root element of the page

Always set lang attribute

<head>

Contains metadata, title, CSS links

Not visible on page

<meta charset="UTF-8">

Character encoding

Supports all languages and emojis

<meta name="viewport">

Mobile responsive scaling

Required for mobile-friendly pages

<title>

Tab title and Google search title

Unique per page, 50–60 chars

<link rel="stylesheet">

Links external CSS file

Goes in <head>

<script src>

Links JavaScript file

Put at end of <body> or use defer

<h1> to <h6>

Headings, h1 = most important

One h1 per page, don't skip levels

<p>

Paragraph of text

Block element, adds space above/below

<strong>

Important text (bold)

Has semantic meaning — use for importance

<em>

Emphasised text (italic)

Has semantic meaning — use for emphasis

<a href>

Hyperlink

Add rel="noopener" for external links in new tab

<img src alt>

Image

alt is mandatory. Always set width+height.

<ul> / <ol>

Unordered / ordered list

All children must be <li>

<li>

List item

Only valid inside ul, ol, or menu

<table>

Table of data

For tabular data only — never for layout

<thead> / <tbody>

Table head / body groups

Use both for proper structure

<th> / <td>

Header cell / data cell

th gets bold+centred by default

<form>

Form container

action + method attributes needed

<label for>

Label for a form input

for= must match input's id=

<input type>

Form input field

Many types: text email password date tel number

<select> + <option>

Dropdown menu

name on select, value on each option

<textarea>

Multi-line text input

Use rows and cols attributes

<button type>

Clickable button

Always specify type: submit or button

<header>

Page or section header

Semantic — not the same as <head>

<nav>

Navigation links

Screen readers announce as navigation

<main>

Primary page content

Only one per page

<section>

Thematic group with heading

Content that belongs together

<article>

Self-contained content

Blog posts, news, course cards

<aside>

Related but secondary content

Sidebars, callouts, related links

<footer>

Bottom of page or section

Copyright, links, contact

<figure> + <figcaption>

Image with caption

Semantic grouping of media + description

<blockquote>

Long quotation from another source

Indented by default

<br>

Line break

Self-closing. Use sparingly — CSS margin is better

<hr>

Horizontal rule (thematic break)

Self-closing. For topic separations

### 7. Interview Questions — HTML

##### Interview Q&A — Fresher Level (0–1 Year HTML Experience)

**Q: Q1. What is the difference between HTML elements and attributes?**

A: **Element** — an HTML component consisting of a start tag, optional content, and an end tag: `<p>Hello</p>`. Some elements are void (self-closing): `<img>`, `<br>`, `<input>`.
**Attribute** — additional information placed inside the opening tag: `<a href="url" target="_blank">`. Attributes have a name and a value.
Some attributes are **boolean** — their presence alone means true: `<input required>` (no value needed).
Global attributes work on any element: `id`, `class`, `style`, `data-*`, `hidden`, `tabindex`.

**Q: Q2. What is semantic HTML and why does it matter?**

A: Semantic HTML means using tags that convey the **meaning and purpose** of the content, not just its appearance.
Examples: `<nav>` instead of `<div class="nav">`; `<article>` instead of `<div class="post">`; `<strong>` instead of `<b>`.
**SEO benefit** — search engines understand your page structure better, improving ranking.
**Accessibility benefit** — screen readers use semantic tags to navigate. Users can jump to <main>, <nav>, or <header> directly.
**Maintainability** — code is easier to read and understand for other developers.

**Q: Q3. Why is the alt attribute on images so important?**

A: **Accessibility** — screen readers read the alt text aloud to visually impaired users. Without alt, they hear "image" — completely uninformative.
**Broken image fallback** — if the image fails to load, the alt text is displayed instead of a broken image icon.
**SEO** — Google cannot "see" images; it reads alt text to understand what the image shows. Good alt text helps images appear in Google Image Search.
Alt text should **describe the content and purpose** of the image, not just its file name. "hero-banner.jpg" is useless; "Students working together on laptops in a modern classroom" is descriptive.
For decorative images with no information: use `alt=""` to tell screen readers to skip it.

**Q: Q4. What is the difference between <div> and semantic elements like <section> and <article>?**

A: **<div>** — a generic container with no semantic meaning. It tells the browser nothing about what the content is. Used purely for grouping elements for CSS or JavaScript purposes.
**<section>** — represents a thematic grouping of related content. Should have a heading. Tells the browser "this is a distinct topic within the page."
**<article>** — self-contained, independently publishable content. Could be removed from the page and still make sense (blog post, course card, news item).
Rule: before writing <div>, ask if there's a semantic tag that fits. If yes, use it. Fall back to <div> only when no semantic element is appropriate.

**Q: Q5. Why must every <input> have a <label> in a form?**

A: **Accessibility** — screen readers read the label when focus moves to the input. Without it, users hear "Edit text" with no context of what to type.
**Usability** — clicking the label moves focus to the input. On mobile, this doubles the tap target size — easier to tap a label than a tiny input.
**Placeholder is not a substitute** — placeholder text disappears when typing starts. If users forget what a field is for after typing, there's no label to refer back to.
The `for` attribute on <label> must match the `id` on the input to programmatically connect them.
Alternative: wrap the input inside the label (`<label>Name <input type="text"></label>`) — the connection is implicit in this case.

**Q: Q6. What is the difference between <script src="..."> at the top vs bottom of the page? What does "defer" do?**

A: **Script in <head>** — browser pauses HTML parsing to download and execute the script. If the script tries to access DOM elements, they don't exist yet → errors.
**Script at end of <body>** — all HTML is parsed first. DOM elements exist. Script runs after page is built. Old but reliable solution.
**defer attribute** — `<script src="app.js" defer>` in <head>. Browser downloads the script in parallel (non-blocking) and executes it after HTML is fully parsed. Same result as end-of-body but script downloads earlier.
**async attribute** — script downloads in parallel and executes as soon as it's downloaded, interrupting HTML parsing. Use only for independent scripts (analytics, ads) that don't depend on DOM or other scripts.
Modern best practice: use `defer` in <head> for all your scripts.

**Quiz: Quiz 1 — A developer wants to create a navigation menu. Which is the correct semantic HTML?**

- A) <div class="navigation"><a href="/">Home</a></div>
- B) <nav><a href="/">Home</a></nav>
- C) <menu><a href="/">Home</a></menu>
- D) <ul class="nav"><a href="/">Home</a></ul>

> **Answer/explanation:** ✅ Answer: **B — <nav>**
**<nav>** is the semantic HTML5 element specifically designed for navigation links.
Screen readers announce a <nav> as a "navigation landmark" — users can skip to it or skip past it.
Option A uses a <div> — works visually but carries no semantic meaning.
Option D has invalid HTML — <a> elements must not be direct children of <ul>; only <li> elements can be.
A navigation typically contains a <ul> of <li> elements wrapping <a> links, all inside <nav>.

**Quiz: Quiz 2 — What is wrong with this form field? <input type="text" placeholder="Enter your name">**

- A) Nothing — placeholder is sufficient as a label
- B) The type attribute is wrong
- C) There is no <label> connected to this input — screen readers can't identify what this field is for, and placeholder disappears when typing starts
- D) input elements don't support placeholder

> **Answer/explanation:** ✅ Answer: **C — Missing label**
Placeholder text is *not* a label — it disappears when users start typing, leaving them without context.
Screen readers cannot identify the input's purpose from placeholder alone.
The correct code: add `id="name"` to the input and `<label for="name">Your Name</label>` above it.
This is tested in accessibility audits and fails WCAG 2.1 level A compliance — the minimum accessibility standard.

**Quiz: Quiz 3 — A page has these headings: h1, h3, h2, h4. What is wrong?**

- A) Nothing — headings can be in any order
- B) You can only use h1 and h2 in HTML5
- C) Heading levels are skipped — h1 jumps to h3, missing h2. Screen readers and Google expect headings to follow a logical outline without skipping levels.
- D) h3 must come before h2 alphabetically

> **Answer/explanation:** ✅ Answer: **C — Skipped heading levels**
Headings should follow a strict hierarchy: h1 → h2 → h3 (you can go deeper inside subsections).
Skipping from h1 to h3 (missing h2) breaks the document outline that screen readers use to navigate.
It's valid to go back up levels (h3 → h2 when a new section starts) but you should never skip forward (h2 → h4).
Think of headings like a table of contents — every level 3 item must belong inside a level 2 item.

> **HTML Project — Core Takeaways for Freshers**

> - HTML is about **structure and meaning**, not appearance. CSS is for appearance. Use the right tag for the right job — not the tag that "looks right."
> - The **viewport meta tag** and **charset meta tag** are not optional — they are required on every page for proper mobile display and character encoding.
> - **One <h1> per page**. Use headings in strict hierarchy (h1 → h2 → h3) without skipping levels. Google uses headings to understand your page's topic structure.
> - **Every <img> must have an alt attribute**. Always set width and height on images to prevent layout shift. Use <figure>+<figcaption> for images with captions.
> - **Every <input> must have a <label>** connected via matching `for`/`id` attributes. Placeholder text is not a replacement for a label.
> - Use semantic tags — **<header>, <nav>, <main>, <article>, <section>, <aside>, <footer>** — instead of meaningless <div>s wherever possible.
> - When linking to external sites with target="_blank", always add **rel="noopener noreferrer"** to prevent the linked page from accessing your window object.
> - Place <script> tags at the **end of <body>**, or use the `defer` attribute in <head>, so JavaScript runs after HTML is fully parsed.

##### HTML Code Standards — BrightPath Engineering Rules

- Indent nested HTML with 2 spaces — consistent indentation shows the document hierarchy and makes nesting bugs immediately visible
- Write attribute values in double quotes always: `href="page.html"` not `href=page.html` — single-word values work without quotes but it is inconsistent and confusing
- Use lowercase for all tag names and attribute names: `<div class="...">` not `<DIV CLASS="...">` — HTML5 is case-insensitive but lowercase is universal convention
- Always validate your HTML at **validator.w3.org** before deploying — it catches unclosed tags, missing required attributes, and structural errors that cause inconsistent rendering across browsers
- Use HTML entities for special characters: `&amp;` for &, `&lt;` for <, `&gt;` for >, `&copy;` for © — putting raw special characters directly in HTML can break rendering
- Every page must have a unique, descriptive `<title>` of 50–60 characters — it appears in Google search results and browser tabs; duplicate titles confuse both users and search engines

##### 🏋️ Hands-On Exercises — Extend the BrightPath Website

1. **Build the full BrightPath home page:** Using all Phase 1–5 concepts, create `index.html` with: <header> containing logo and <nav> with 4 links; a <main> with a hero <section> (h1, p, <a> as call-to-action button), a courses <section> with 3 <article> course cards each with h3, p, and a price; a <footer> with copyright and social links. Validate at validator.w3.org.
2. **Add a complete FAQ section:** Use the <details> and <summary> elements to create an accordion FAQ — clicking a question expands the answer without any JavaScript. Add 5 FAQs about BrightPath's courses. This is a native HTML5 feature most freshers don't know about.
3. **Build the complete enrolment form:** Create `enrol.html` with a form containing: text input for full name (required), email input, phone input with pattern validation, select dropdown for course (5 options), radio buttons for experience level (grouped in fieldset+legend), date input for preferred start date, textarea for message (rows="4"), checkbox for terms agreement (required), and a submit button. All inputs must have proper labels.
4. **Add a data table with colspan and rowspan:** Create a course schedule table for BrightPath using <thead>, <tbody>, <tfoot>, and use `colspan` to span a cell across multiple columns (e.g. "Weekend Batch" spanning Saturday and Sunday columns) and `rowspan` to merge rows for a course that runs across multiple weeks.
5. **Audit your page for accessibility:** Install the "axe DevTools" browser extension and run it on your BrightPath home page. Fix every issue it flags. Common issues to look for: missing alt text, colour contrast failures, missing form labels, incorrect heading hierarchy, and buttons without accessible names.

### HTML Project Complete 🎉

You have built BrightPath Academy's complete website structure — proper document boilerplate with SEO meta tags, semantic page layout, text and image content, a course comparison table, a fully accessible enrolment form, and semantic HTML5 markup that works for search engines, screen readers, and developers alike.

> **Kavitha**
> 
> "Deepak, I ran the BrightPath home page through Google's Lighthouse tool. Accessibility score: 98 out of 100. Structure score: 100. SEO score: 95. Those numbers are achievable in HTML alone, before a single line of CSS or JavaScript. Proper HTML is the foundation everything else is built on. You got it right."

> **Prasanna**
> 
> "The enrolment form you wrote — I tested it with a screen reader. Every field had a proper label, the fieldsets had legends, the submit button had clear text. A visually impaired student can fill in that form completely independently. That's what accessible HTML enables. You built something that works for everyone."

> **Next: CSS — Style the BrightPath Pages You Just Built**

> - Selectors and specificity — target exactly the elements you want to style
> - Box model — margin, padding, border, content — how every element takes up space on the page
> - Flexbox — build navigation bars, card grids, and centred layouts in minutes
> - CSS Grid — two-dimensional layouts for the full page structure
> - Media queries — make pages look great on mobile, tablet, and desktop
> - CSS custom properties (variables) — define BrightPath's colour palette once, use everywhere
> - Transitions and animations — hover effects, smooth reveals, and loading spinners
