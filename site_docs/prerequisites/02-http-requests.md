# HTTP Requests — How the Web Works

Learn how browsers and servers communicate. This is fundamental to everything in backend engineering.

---

## 1. What is HTTP?

**HTTP** = HyperText Transfer Protocol

It's the language that browsers and web servers use to talk to each other.

Think of it like mail:
- **You** = Browser (client)
- **Postal Service** = Internet
- **Recipient** = Web Server
- **Letter** = HTTP Request
- **Reply** = HTTP Response

You send a request ("I want the home page"), the server sends back a response ("Here's the home page").

---

## 2. The Request/Response Model

### Request

The browser **asks** the server for something:

```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
```

This means: "Please give me the file index.html from example.com"

### Response

The server **answers** with data:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<!DOCTYPE html>
<html>
    <h1>Welcome!</h1>
</html>
```

This means: "Here's your file. It's HTML. It's 1234 bytes long. Everything is OK (status 200)."

---

## 3. HTTP Methods

Different requests do different things.

### GET — Retrieve Data

**Use case**: Get a web page, get data from an API

```
GET /api/users HTTP/1.1
Host: api.example.com
```

**Response**: The server sends back the data (a list of users)

**In a browser**: When you visit `https://example.com`, your browser sends a GET request.

### POST — Send Data

**Use case**: Submit a form, create new data

```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "Alice",
    "email": "alice@example.com"
}
```

**Response**: The server creates a new user and tells you it was successful (status 201)

**In a browser**: When you fill out a form and click "Submit", it sends a POST request.

### PUT — Update Data

**Use case**: Update existing data

```
PUT /api/users/42 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "Alice Smith",
    "email": "alice.smith@example.com"
}
```

**Response**: The server updates user 42 and confirms (status 200)

### DELETE — Remove Data

**Use case**: Delete a user, delete a file

```
DELETE /api/users/42 HTTP/1.1
Host: api.example.com
```

**Response**: The server deletes user 42 and confirms (status 204 No Content)

### PATCH — Partial Update

**Use case**: Update just one field (not the whole object)

```
PATCH /api/users/42 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "email": "alice.new@example.com"
}
```

**Response**: The server updates just the email and confirms (status 200)

---

## 4. HTTP Status Codes

The status code tells you the result of your request.

### 2xx — Success ✅

| Code | Meaning | Example |
|------|---------|---------|
| **200** | OK | Request successful, here's the data |
| **201** | Created | New resource created successfully |
| **204** | No Content | Success, but no data to return |

### 3xx — Redirect 🔄

| Code | Meaning | Example |
|------|---------|---------|
| **301** | Moved Permanently | Resource moved to a new location |
| **302** | Found | Temporarily at a different location |
| **304** | Not Modified | Your cached version is still good |

### 4xx — Client Error ❌

| Code | Meaning | Example |
|------|---------|---------|
| **400** | Bad Request | You sent invalid data |
| **401** | Unauthorized | You need to log in first |
| **403** | Forbidden | You don't have permission |
| **404** | Not Found | Resource doesn't exist |
| **429** | Too Many Requests | You're requesting too fast (rate limited) |

### 5xx — Server Error 💥

| Code | Meaning | Example |
|------|---------|---------|
| **500** | Internal Server Error | Server crashed or encountered an error |
| **502** | Bad Gateway | Server is temporarily unavailable |
| **503** | Service Unavailable | Server is overloaded or down |

---

## 5. HTTP Headers

Headers are metadata about the request or response.

### Common Request Headers

```
GET /api/users HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer YOUR_TOKEN_HERE
Content-Type: application/json
```

| Header | Meaning |
|--------|---------|
| **Host** | Which server are you talking to? |
| **User-Agent** | What browser/app is making the request? |
| **Accept** | What format do you want? (JSON, HTML, etc.) |
| **Authorization** | Your login credentials or API key |
| **Content-Type** | What format is the data you're sending? |

### Common Response Headers

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234
Cache-Control: max-age=3600
Server: nginx/1.19.0
Set-Cookie: session_id=abc123
```

| Header | Meaning |
|--------|---------|
| **Content-Type** | What format is the response data? |
| **Content-Length** | How big is this response? |
| **Cache-Control** | How long should the browser keep this? |
| **Server** | What server software is running? |
| **Set-Cookie** | Store this data in the browser for later |

---

## 6. Request and Response Bodies

The **body** is the actual data being sent.

### GET Request (usually NO body)

```
GET /api/users HTTP/1.1
Host: api.example.com
```

### POST Request (HAS a body)

```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "Alice",
    "email": "alice@example.com"
}
```

The body is the JSON object with user data.

### Response Body

```
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 42,
    "name": "Alice",
    "email": "alice@example.com",
    "created_at": "2026-03-12T10:30:00Z"
}
```

The body is the JSON with the user's data.

---

## 7. JSON (JavaScript Object Notation)

JSON is the standard format for sending data via HTTP.

### JSON Basics

```json
{
    "name": "Alice",
    "age": 25,
    "is_active": true,
    "email": null,
    "hobbies": ["reading", "coding", "hiking"],
    "address": {
        "street": "123 Main St",
        "city": "New York",
        "zip": "10001"
    }
}
```

**Rules:**
- Keys must be in double quotes: `"name"`
- Values can be: string, number, boolean, null, array, or object
- Use commas to separate key-value pairs
- No trailing comma on the last item

---

## 8. Real-World Example

Let's trace a complete request/response:

### Step 1: You Visit a Website

You click a link or type in the address bar:
```
https://api.github.com/users/octocat
```

### Step 2: Browser Sends a Request

```
GET /users/octocat HTTP/1.1
Host: api.github.com
User-Agent: Mozilla/5.0 (Windows NT 10.0)
Accept: application/json
```

### Step 3: Server Processes the Request

GitHub's server looks up user "octocat" in its database.

### Step 4: Server Sends a Response

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 2048

{
    "login": "octocat",
    "id": 1,
    "avatar_url": "https://avatars.githubusercontent.com/u/1?v=4",
    "name": "The Octocat",
    "company": "@github",
    "blog": "https://github.blog",
    "location": "San Francisco",
    ...
}
```

### Step 5: Browser Displays the Data

The browser shows you the user's profile.

---

## 9. API = Application Programming Interface

An **API** is a URL that gives you data instead of a web page.

### Web Pages (for humans)

```
GET https://www.github.com/octocat
```

Returns: An HTML page with styling, images, and interactive elements.

### APIs (for programs)

```
GET https://api.github.com/users/octocat
```

Returns: Clean JSON data that your program can parse and use.

### Making an API Request from Python

```python
import requests

response = requests.get('https://api.github.com/users/octocat')
print(response.status_code)  # 200
print(response.json())       # Dictionary with user data

# Access specific data
user = response.json()
print(user['name'])          # The Octocat
print(user['location'])      # San Francisco
```

### Making an API Request from JavaScript

```javascript
fetch('https://api.github.com/users/octocat')
    .then(response => response.json())
    .then(data => {
        console.log(data.name);      // The Octocat
        console.log(data.location);  // San Francisco
    });
```

---

## 10. Hands-On: Make Real API Requests

### Using Python

```python
import requests

# Get a random quote
response = requests.get('https://quotable.io/random')
quote = response.json()

print(f'"{quote["content"]}"')
print(f'— {quote["author"]}')
```

### Using curl in Terminal

```bash
# Get a random quote
curl https://quotable.io/random | python -m json.tool
```

### Using JavaScript in Browser Console

```javascript
fetch('https://quotable.io/random')
    .then(r => r.json())
    .then(quote => console.log(quote.content));
```

---

## 11. Cheat Sheet

| Concept | Example |
|---------|---------|
| **GET** | Retrieve data |
| **POST** | Create new data |
| **PUT** | Update all fields |
| **PATCH** | Update some fields |
| **DELETE** | Remove data |
| **200** | Success |
| **404** | Not found |
| **500** | Server error |
| **JSON** | `{"key": "value"}` |
| **Header** | `Content-Type: application/json` |
| **Body** | The actual data being sent |

---

## Next Steps

1. **Try it**: Make requests to public APIs (PokeAPI, OpenWeatherMap, etc.)
2. **Understand**: When you see status codes, you'll know what they mean
3. **Remember**: This is how your backend will communicate with browsers and mobile apps
4. **Continue**: Move to [Command Line Basics](03-command-line-basics.md)

---

## Resources

- **MDN HTTP Guide**: https://developer.mozilla.org/en-US/docs/Web/HTTP
- **HowDNS Works**: https://www.youtube.com/watch?v=x3c1ih2NJEg (video)
- **Free Public APIs**: https://github.com/public-apis/public-apis

→ **[Next: Command Line Basics](03-command-line-basics.md)**
