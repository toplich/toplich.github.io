# Build a Modern Hugo Website with GitHub, Cloudflare, and Resend


## 1. Introduction

:::info
This guide shows how to build and deploy a modern static website using a fully open-source and serverless stack.
:::

We will use the following tools:

- **Hugo** – fast and flexible static site generator  
- **GitHub Pages** – free web hosting with Git-based deployment  
- **Cloudflare** – DNS management, CDN, and HTTPS  
- **Resend** – transactional email API for contact forms

### ✅ Who is this for?

This setup is ideal for:

- Community and non-profit projects  
- Personal websites and portfolios  
- Lightweight, multilingual static sites

### 💡 Why this stack?

- Free and easy to maintain  
- Fast performance with global CDN  
- No backend or database needed  
- Clean separation between content and infrastructure

### 📦 What will you learn?

This guide covers:

1. Setting up a Hugo site  
2. Hosting it on GitHub Pages  
3. Securing it with Cloudflare (DNS + HTTPS)  
4. Adding a contact form with Resend via Cloudflare Worker

Let’s get started.

---

## 2. Setup

### 🛠️ Requirements

Make sure the following tools are installed:

- [Go](https://golang.org/dl/) (v1.20 or later)  
- [Git](https://git-scm.com/)  
- [Hugo Extended](https://gohugo.io/getting-started/installing/) (v0.111 or later)  
- [Node.js](https://nodejs.org/) (optional, for advanced features)

---

### 🧱 Create a new Hugo site

```bash
hugo new site mysite
cd mysite
```

---

### 🎨 Install Hugo Blox theme

Add the Hugo Blox theme as a Git submodule:

```bash
git init
git submodule add https://github.com/HugoBlox/hugo-blox-builder.git themes/hugo-blox-builder
```

Copy the example site content:

```bash
cp -r themes/hugo-blox-builder/exampleSite/* .
```

Install dependencies (optional):

```bash
npm install
```

---

### ⚙️ Basic configuration

Edit the following files:

- `config/_default/config.yaml` – site language, base URL  
- `config/_default/params.yaml` – branding, theme settings  
- `config/_default/menus.yaml` – navigation menu  
- `assets/media/` – upload your logo and favicon

Example `baseURL` (for GitHub Pages):

```yaml
baseURL: "https://yourusername.github.io/mysite/"
```

---

### 🚀 Run local server

```bash
hugo server
```

Visit [http://localhost:1313](http://localhost:1313) to preview your site.

---

:::tip
Use GitHub Desktop or CLI to version control all changes.
:::

---

## 3. Content Management

Managing content in Hugo is simple and efficient. All pages and posts are written in **Markdown** and organized in folders under the `content/` directory.

---

### 🗂️ Content structure

Each language has its own folder:

```
content/
├── en/
│   ├── _index.md
│   └── about.md
├── de/
│   └── about.md
└── fr/
    └── about.md
```

Each Markdown file represents a page and includes a **front matter** block at the top.

Example `about.md`:

```markdown
---
title: "About Us"
date: 2025-08-01
type: page
layout: page
---
```

---

### ✍️ Creating a new page

Use the `hugo new` command:

```bash
hugo new en/about.md
```

This will create a file in `content/en/about.md` with a pre-filled front matter.

---

### 📄 Page types

There are two common types of content:

- **Page** (`type: page`) – standalone pages like “About” or “Contact”
- **Post** (`type: post`) – blog/news entries shown in lists

Use different layouts if needed: `page`, `post`, or custom layouts.

---

### 🧭 Navigation menu

Define menus in `config/_default/menus.yaml`:

```yaml
main:
  - name: About
    url: /about/
    weight: 10
```

To add multilingual labels, use Hugo’s built-in `i18n` system:

```
i18n/
├── en.yaml
├── de.yaml
└── fr.yaml
```

Example `en.yaml`:

```yaml
- id: about
  translation: "About Us"
```

---

### 🌍 Multilingual setup

In `config/_default/config.yaml`, define the site languages:

```yaml
defaultContentLanguage: en
languages:
  en:
    languageName: English
    weight: 1
  de:
    languageName: Deutsch
    weight: 2
```

Use `relLangURL` in templates to link between languages.

---

### 👁️ Preview your content

Run the local server:

```bash
hugo server
```

Check the site in your browser at [http://localhost:1313](http://localhost:1313).

Use `hugo server --navigateToChanged` to auto-scroll to the last edited page.

---

:::tip
Use clear folder structure and consistent naming. Avoid spaces and uppercase letters in filenames.
:::

---

## 4. GitHub Deployment

You can deploy your Hugo site automatically to **GitHub Pages** using GitHub Actions. This approach works for both personal and project sites.

---

### 🧑‍💻 1. Create a GitHub repository

Create a new repository on [GitHub](https://github.com/new).  
Do **not** initialize it with a README or `.gitignore`.

Connect your local project:

```bash
git remote add origin https://github.com/yourusername/yourrepo.git
git branch -M main
git push -u origin main
```

---

### ⚙️ 2. Add GitHub Actions workflow

Create the file `.github/workflows/deploy.yml`:

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v3

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build site
        run: hugo --minify

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

---

### 🗂️ 3. Configure `config.yaml` for deployment

In `config/_default/config.yaml`, set the correct `baseURL`:

```yaml
baseURL: "https://yourusername.github.io/yourrepo/"
```

Make sure this path matches your actual repository name.

---

### 🛡️ 4. Enable GitHub Pages

Go to your repository → **Settings → Pages**  
Set **Source** to `Deploy from a branch` → choose `gh-pages` branch and `/ (root)` directory.

GitHub will host your site at:

```
https://yourusername.github.io/yourrepo/
```

---

### 🧪 5. Test the deployment

Every time you push to `main`, GitHub will:

- Build your site
- Deploy it to the `gh-pages` branch
- Make it available via GitHub Pages

Check the **Actions** tab for logs and status.

---

:::tip
If you're using a custom domain, configure it later via Cloudflare and set it in `config.yaml` under `baseURL`.
:::

---

## 5. Cloudflare Setup

Cloudflare acts as a global CDN, DNS provider, and HTTPS enabler for your static website. It improves performance, reliability, and security.

---

### 🌐 1. Add your domain to Cloudflare

1. Go to [https://dash.cloudflare.com](https://dash.cloudflare.com)  
2. Click **“Add a Site”**  
3. Enter your custom domain (e.g. `example.com`)  
4. Select the **Free Plan**

---

### 🧾 2. Update your domain’s nameservers

Cloudflare will show two nameservers (e.g. `sue.ns.cloudflare.com`, `tim.ns.cloudflare.com`).  
Update your domain registrar (e.g. Swizzonic, GoDaddy) to use these nameservers.

⚠️ DNS changes may take a few hours to propagate.

---

### 📁 3. Set DNS records

Go to **DNS → Records** and add:

- **Type:** `CNAME`  
- **Name:** `www`  
- **Target:** `yourusername.github.io.`  
- **Proxy status:** `Proxied`

Optional: add a redirect from root domain to `www` using **Page Rules**.

---

### 🔒 4. Configure SSL/TLS

Go to **SSL/TLS → Overview** and set:

- **SSL Mode:** `Full` or `Full (Strict)`  
- Auto HTTPS rewrites: ✅ enabled  
- Always Use HTTPS: ✅ enabled

> Use `Full (Strict)` only if GitHub Pages presents a valid certificate for your domain.

---

### 📄 5. Set custom domain in Hugo

In `config/_default/config.yaml`, update:

```yaml
baseURL: "https://www.example.com/"
```

If using multilingual setup, set `baseURL` under each language.

---

### ⚙️ 6. Optional: Configure additional features

Cloudflare offers many extras:

- **Caching rules**  
- **WAF and security level**  
- **Custom error pages**  
- **Analytics and bot protection**

You can manage these via the Cloudflare dashboard as needed.

---

:::tip
If you're using `www.example.com`, GitHub Pages should serve from `gh-pages` branch, and Cloudflare will handle HTTPS and DNS.
:::

---

## 6. Resend Contact Form Integration

You can send emails from a static site by using a contact form that submits to a Cloudflare Worker, which then forwards the data via the **Resend API**.

---

### 📬 1. Create a Resend account and API key

1. Go to [https://resend.com](https://resend.com)  
2. Sign up and verify your email  
3. Add your domain (e.g. `allegra-march.ch`)  
4. Set up DNS records (SPF, DKIM, DMARC)  
5. Create an API key under **API → Create API Key**

---

### 🧰 2. Create a Cloudflare Worker

Install Wrangler (CLI for Cloudflare Workers):

```bash
npm install -g wrangler
```

Login to your Cloudflare account:

```bash
wrangler login
```

Generate a new Worker:

```bash
wrangler init contact-worker --no-git
cd contact-worker
```

Edit `src/index.ts` (or `index.js` if using JS) with the following:

```ts
export default {
  async fetch(request: Request): Promise<Response> {
    if (request.method !== "POST") {
      return new Response("Only POST allowed", { status: 405 });
    }

    const data = await request.json();

    const res = await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: {
        "Authorization": "Bearer YOUR_RESEND_API_KEY",
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        from: "Form <form@yourdomain.com>",
        to: "your@email.com",
        subject: "New Contact Form Submission",
        text: `Name: ${data.name}\nEmail: ${data.email}\nMessage: ${data.message}`,
      }),
    });

    return new Response("Email sent", { status: 200 });
  },
};
```

Replace `YOUR_RESEND_API_KEY` and email addresses.

---

### 🔐 3. Set your Resend key as a secret

```bash
wrangler secret put RESEND_API_KEY
```

In your code, replace the key with:

```ts
"Authorization": `Bearer ${RESEND_API_KEY}`
```

---

### 🚀 4. Deploy the Worker

```bash
wrangler publish
```

Note the deployed URL (e.g. `https://contact-worker.yourname.workers.dev`)

---

### 🖊 5. Add the contact form to your Hugo site

```html
<form method="POST" action="https://contact-worker.yourname.workers.dev">
  <input type="text" name="name" required placeholder="Your Name">
  <input type="email" name="email" required placeholder="Your Email">
  <textarea name="message" required placeholder="Your Message"></textarea>
  <button type="submit">Send</button>
</form>
```

Style with CSS as needed.

---

### 🧪 6. Test your form

- Fill out the form on your site  
- Check your inbox  
- Monitor delivery in the Resend dashboard

---

:::tip
To prevent abuse, consider adding a simple honeypot field or reCAPTCHA. You can also validate requests in the Worker.
:::

---

## 7. Security & Spam Protection

To prevent abuse of your contact form, it's important to implement basic security and anti-spam measures. Static sites have no backend, so the protection must happen inside the **Cloudflare Worker**.

---

### 🔒 1. Enforce request method and content type

Make sure your Worker only accepts `POST` requests with JSON payload:

```ts
if (request.method !== "POST") {
  return new Response("Method Not Allowed", { status: 405 });
}

if (!request.headers.get("Content-Type")?.includes("application/json")) {
  return new Response("Invalid content type", { status: 400 });
}
```

---

### 🧪 2. Validate user input

Always sanitize and validate user input in the Worker:

```ts
const { name, email, message } = await request.json();

if (!name || !email || !message) {
  return new Response("Missing fields", { status: 400 });
}

if (!email.includes("@")) {
  return new Response("Invalid email", { status: 400 });
}
```

---

### 🧱 3. Honeypot field (invisible trap)

Add a hidden field in your HTML form:

```html
<input type="text" name="nickname" style="display:none">
```

In the Worker, check if it's filled:

```ts
if (data.nickname) {
  return new Response("Spam detected", { status: 403 });
}
```

Bots will likely fill hidden fields; real users won’t.

---

### ⏱ 4. Rate limiting (basic)

Cloudflare Workers don’t provide built-in rate limiting, but you can add logic using IP address + KV:

- Track timestamp of last submission per IP
- Block repeated requests within N seconds
- Example: use Cloudflare KV or durable objects (advanced)

---

### 🧩 5. reCAPTCHA (optional)

You can integrate Google reCAPTCHA v2/v3:

1. Add a reCAPTCHA widget to the form
2. Get the token on submit
3. Send the token to the Worker
4. Verify token via Google API inside the Worker

Requires server-side verification and is more complex to implement.

---

### ✉️ 6. Email protections via Resend

In your Resend dashboard:

- Enable SPF, DKIM, and DMARC on your domain  
- Monitor bounce/spam reports  
- Use a verified `from:` address to avoid spam filters

---

:::tip
Start simple with a honeypot and basic validation. Add rate-limiting or CAPTCHA only if spam becomes a problem.
:::

---

## 8. Conclusion & Resources

By following this guide, you’ve built a modern, fast, and secure static website using:

- **Hugo** for content generation  
- **GitHub Pages** for hosting  
- **Cloudflare** for DNS, CDN, and HTTPS  
- **Resend** for handling contact form emails via API

This stack is fully serverless, free to use, and easy to maintain. It’s suitable for personal projects, non-profits, and multilingual community websites.

---

### ✅ Summary of key benefits

- Fast and optimized delivery via CDN  
- No backend or database required  
- Secure contact form with spam protection  
- Easy to version, update, and collaborate via Git  
- Zero hosting cost (if using free tiers)

---

### 📚 Useful resources

- Hugo Documentation: [https://gohugo.io/documentation/](https://gohugo.io/documentation/)  
- Hugo Blox Theme: [https://docs.hugoblox.com/](https://docs.hugoblox.com/)  
- GitHub Pages: [https://pages.github.com/](https://pages.github.com/)  
- Cloudflare Docs: [https://developers.cloudflare.com/](https://developers.cloudflare.com/)  
- Resend API: [https://resend.com/docs](https://resend.com/docs)  
- Wrangler CLI: [https://developers.cloudflare.com/workers/wrangler/](https://developers.cloudflare.com/workers/wrangler/)

---

### 🙌 Final note

You now have a powerful foundation for your website — one that is fast, scalable, and privacy-respecting.

Extend it further with:

- Hugo shortcodes, partials, and custom layouts  
- Cloudflare KV for storing form submissions  
- Resend Webhooks for tracking delivery status  
- GitHub Actions for content automation

---

:::info
This guide is maintained by [Your Name] and used in real-world non-profit and volunteer projects.
:::


