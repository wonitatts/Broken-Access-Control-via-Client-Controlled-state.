# Broken-Access-Control-via-Client-Controlled-state.


"""
# User Role Controlled by Request Parameter

**Lab:** PortSwigger Web Security Academy — Access Control
**Tools used:** Burp Suite (Proxy, Repeater)
**Vulnerability class:** Broken Access Control

---

## TL;DR

The website decided whether I was an administrator by reading a value my *own browser* sends back to it — a cookie that literally said `Admin=false`. Because a browser controls its own cookies, I simply changed that value to `Admin=true`. The site believed me, let me into the admin panel, and I was able to delete another user. The core mistake: the app trusted the client to be honest about its own permissions.

---

## Background: what's actually going on

When you use a website, your browser and the server are constantly passing notes back and forth. One kind of note is a **cookie** — a small piece of data the server hands your browser, which your browser then includes on every future request.

Cookies are normally used to remember *who you are* (a login session). The problem in this lab is that the site also used a cookie to remember *what you're allowed to do*. And anything your browser holds, you can edit.

The tool I used to see and edit these notes is **Burp Suite**, an intercepting proxy — it sits between the browser and the server so you can read and modify traffic before it's sent.

---

## Steps to reproduce

1. **Logged in** with the provided low-privilege account (`wiener`). Normal user, no admin access.

2. **Inspected the request** in Burp and noticed two cookies being sent:
   - a random `session` value (the legitimate login token), and
   - a second cookie: `Admin=false`.

   That second one is the tell. The server was reading my role straight from a value I control.

3. **Sent the request to Repeater** (Burp's tool for editing one request, firing it, and reading the response — over and over).

4. **Changed `Admin=false` to `Admin=true`** and resent it. The response now contained a link to `/admin` that wasn't there before — proof the server now treated me as an admin.

5. **Requested the admin panel directly** by changing the request path to `GET /admin` (keeping `Admin=true`). The response showed the admin dashboard, including a list of users each with a **Delete** link.

6. **Read the delete link's format** from the page: `/admin/delete?username=carlos`.

7. **Fired the delete request** for the target user:
   ```
   GET /admin/delete?username=carlos
   Cookie: Admin=true; session=...
   ```
   The server responded `302 Found` (a redirect back to the admin panel) — the confirmation that the action succeeded. Lab solved.

---

## Why it works

Authorization — the decision of *what a user is allowed to do* — was being made from data the user can change. The server asked, in effect, "Does this request claim to be admin?" instead of "Is this user actually an admin?"

Trusting client-supplied input for a security decision is the whole bug. No passwords were cracked and nothing was brute-forced. I just told the server I was an admin, and it took my word for it.

This is **Broken Access Control**, which sits at #1 on the [OWASP Top 10](https://owasp.org/www-project-top-ten/). It's usually not exotic — it's ordinary oversights exactly like this one.

---

## How to fix it

Make the authorization decision on the **server side**, based on data the client can't forge.

- The user's role should live in the server's own session record, looked up using that random `session` token — **never** in a separate cookie the browser can edit.
- The correct check is: "Does the session *on the server* belong to an admin?" — not "Did the request *say* it was admin?"

General principle: never trust input, and never let the client vote on its own permissions.

---

## Detection (the defender's angle)

If you're defending an app, here's how an attack like this shows up:

- **Identity mismatch in logs.** An authenticated session belonging to a *non-admin* user hitting `/admin` or `/admin/delete` is a loud anomaly. The signal is a privileged endpoint being reached by an identity your server-side records say isn't privileged.
- **Alert on privileged actions.** A `delete?username=` request from a session that should never have reached the admin panel is worth an immediate look.
- **Log both sides and diff them.** Record the server's notion of who the user is *alongside* what the request claims. When those two disagree, someone is likely tampering.

---

## Takeaways

- Client-side data (cookies, hidden form fields, URL parameters) is a *suggestion*, not a *fact*. Security decisions must be verified server-side.
- Reading responses carefully matters more than sending clever requests — every step here came from noticing what changed in the response.
- The exploit and the fix are two sides of the same idea: don't trust the client.



"""
