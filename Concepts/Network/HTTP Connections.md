---
tags:
  - hol-blocking
  - http
  - http2
  - http3
  - multiplexing
  - network
  - tcp
---

## 1. The Old Way: HTTP/1.0 (One Request Per Connection)

In the early days of the web, a connection could _not_ handle multiple requests.

- Your browser would open a TCP connection, ask for `index.html`, get the file, and the connection would immediately close.
    
- If that HTML file had 10 images, your browser had to open and close **10 separate TCP connections** to get them. This was incredibly slow and wasteful.
    

## 2. The Step Forward: HTTP/1.1 (Keep-Alive / Persistent Connections)

To fix this, HTTP/1.1 introduced **Persistent Connections** (often called `Keep-Alive`).

- Instead of closing the connection after one request, the browser keeps it open.
    
- It sends a request for an image, waits for the response, and then uses that **same connection** to send the next request.
    

> ⚠️ **The Catch:** While it uses the same connection, it has to do so **in a line (serially)**. Request B cannot start until Request A is completely finished. If Request A gets stuck, everything behind it is blocked. This is known as **Head-of-Line (HoL) Blocking**.

## 3. The Modern Way: HTTP/2 and HTTP/3 (Multiplexing)

This is where connections became truly powerful. Modern web protocols use a technique called **Multiplexing**.

Instead of waiting in line, **multiple requests and responses are broken down into tiny pieces (frames) and sent down the exact same connection at the same time.**

- **How it works:** Your browser can request `image1.png`, `style.css`, and `script.js` all at once over a single connection. The server interleaves the pieces of these files together and sends them back. Your browser then reassembles them on the fly.
    
- **The Benefit:** No more waiting in line. One single TCP (or UDP in HTTP/3) connection can download dozens of files simultaneously.
    

## How the OS Tells Requests Apart

Going back to your previous question about ports: even if a single connection handles 50 requests at once via HTTP/2, your computer still only uses **one ephemeral port** for that entire connection.

The protocol inside the connection attaches a unique **Stream ID** to each request and response. Your browser looks at the Stream ID to know exactly which image or script a piece of data belongs to.

---

← Part of [[TCP & Connection Lifecycle]]
