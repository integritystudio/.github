Here’s a simple, non-technical step-by-step guide you can give to someone:

---

# 🛑 How to Stop any of our domains (except integritystudio.ai and alephatx.info) from Showing a Website

*(Porkbun + GitHub Pages)*

This guide helps you take a website offline when:

* The domains are all managed in **[Porkbun](https://porkbun.com/)**
* The site is hosted on **[GitHub Pages]()**

---

## Option 1 (Fastest): Disconnect the Domain in GitHub

This is usually the easiest and safest method.

### Steps:

1. Go to your **GitHub repository**
2. Click **Settings**
3. Scroll down to **Pages** (left sidebar or main section)
4. Find the **Custom domain** field
5. **Delete the domain name**
6. Click **Save**

Result:
GitHub will stop serving your site at that domain.

---

## Option 2: Remove DNS Settings in Porkbun

This prevents the domain from pointing to GitHub at all.

### Steps:

1. Log into **Porkbun**
2. Go to **Domain Management**
3. Click your domain
4. Open **DNS Records**
5. Look for records like:

   * **A records** pointing to GitHub IPs (e.g. `185.199.x.x`)
   * **CNAME** pointing to `username.github.io`
6. **Delete those records**
7. Save changes

Result:
Your domain no longer connects to GitHub Pages.

---

## Option 3 (Temporary “Kill Switch”): Point Domain Somewhere Else

If you want a quick temporary stop:

### Steps:

1. In Porkbun DNS settings
2. Edit the domain to:

   * Remove GitHub records OR
   * Add a dummy record (e.g. point to `0.0.0.0`)
3. Save

Result:
Visitors will see an error or nothing at all.

---

## ⚠️ Important Notes

* Changes can take **a few minutes to a few hours** to fully apply
* Removing it in GitHub = cleaner
* Removing DNS = more permanent disconnect

---

## 👍 Recommended Approach

If you’re unsure, do this:

1. Remove the domain in **GitHub Pages**
2. Then clean up DNS in **Porkbun**

---
