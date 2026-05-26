# NGINX Notes — Beginner to Practical Understanding
Structured notes based on lecture content + clarified doubts + practical production understanding.

## 1. What is NGINX?
NGINX is a high-performance web server, reverse proxy, load balancer, and traffic manager. It is commonly used in production to serve frontend files, route API requests, manage HTTPS, protect backend services, and distribute traffic.

## 2. Development vs Production
Development flow:
- Browser → Vite (5173) → React frontend
- Browser → Express (5000) → Backend API → Database
In development, Vite acts as a temporary frontend web server and sometimes a proxy. Express handles backend logic.
Production flow:
- Browser → NGINX → Frontend / Backend
In production, NGINX replaces Vite for serving frontend files and routes requests to Express.

## 3. How NGINX fits into a Food Delivery App
Flow:
1. User opens website → NGINX serves React static files.
1. User adds items to cart → Frontend only, backend not involved.
1. User places order → Frontend sends /api/order request.
1. NGINX forwards /api/* requests to backend.
1. Backend stores order in DB.
1. Admin dashboard polls server → request again passes through NGINX.
1. Admin accepts order → backend updates DB → frontend gets updated status.

## 4. What NGINX Actually Does
- Serves static frontend files (HTML, CSS, JS, images)
- Reverse proxying (routes /api to backend)
- HTTPS/SSL handling
- Load balancing across multiple backend servers
- Rate limiting and protection from abuse
- Caching for faster responses

## 5. Common Misconceptions Corrected
❌ “One NGINX serving frontend + backend means monolith”
Correction: No. Monolith vs microservices depends on code/service coupling, not on whether requests enter through one NGINX. NGINX is just the traffic gateway.
❌ “default_server means localhost only”
Correction: default_server means fallback server block when no server_name matches. localhost is configured using server_name localhost; they are different concepts.
❌ “NGINX becomes bottleneck at high traffic”
Correction: NGINX is designed for high concurrency and is usually not the bottleneck. Database and backend become bottlenecks earlier. Large systems scale NGINX horizontally.

## 6. Virtual Hosting (Multiple Domains)
One NGINX server can host multiple domains by checking the Host header.
Example:
- foodapp.com → customer frontend
- admin.foodapp.com → admin frontend
- api.foodapp.com → backend API
This is configured using server_name.

## 7. Important NGINX Directories

## 8. sites-available vs sites-enabled
- You can create many site configs in sites-available.
- Only symlinked configs in sites-enabled become active.
- Usually one domain per app, but one NGINX can host many apps.

## 9. Example Static Website
Command:
echo "<h1>Custom Hello from NGINX Web Server</h1>" | sudo tee /var/www/html/index.html
This replaces the default homepage content served by NGINX.
After changes:
sudo systemctl reload nginx
Reload applies config/content updates without fully stopping NGINX.

## 10. Basic Server Block Example
server {
    listen 80;
    server_name foodapp.com;

    root /var/www/html;
    index index.html;

    location /api {
        proxy_pass http://localhost:5000;
    }
}

## 11. Docker vs Ubuntu NGINX
Ubuntu (apt install nginx): sites-available + sites-enabled present.
Docker official image: mostly conf.d/ instead of sites-available.

## 12. Learning Order Recommendation
1. Serve static HTML using NGINX
1. Understand server blocks and server_name
1. Learn root, location, index, try_files
1. Learn reverse proxying (/api → backend)
1. Learn HTTPS and SSL
1. Learn load balancing
1. Learn Docker + NGINX deployment

## 13. Quick Revision Mental Models
NGINX = Restaurant manager / traffic controller
Backend = Chef (business logic)
Database = Storage

## 14. Dynamic Websites + NGINX
A dynamic website does NOT mean frontend files constantly change. Frontend code (HTML/CSS/JS bundle) is usually static after build, while backend data changes dynamically.
- Static part:
- Navbar, buttons, forms, layout, components
- Dynamic part:
- Orders, user data, products, notifications, prices
Important idea:
You rebuild frontend only when frontend code changes, not when app data changes.
Example food delivery flow:
1. NGINX serves React build files.
1. Frontend loads in browser.
1. Frontend fetches fresh order/product data from backend APIs.
1. React rerenders UI using latest data.

## 15. Polling vs WebSockets
Does polling count as requests and billing?
Yes. Polling requests count as normal API requests. Depending on hosting provider, you may be billed based on compute, bandwidth, serverless invocations, or database usage.
Why polling can become expensive:
- Repeated unnecessary requests even when nothing changed
- Backend CPU usage
- Database reads
- Bandwidth consumption
Example:
Polling every 5 sec = 17,280 requests/day per user.
Why people still use polling:
- Very simple to implement
- Good enough for MVPs and small apps
- Many apps do not require instant updates
- WebSockets add complexity and scaling challenges
- HTTP polling works everywhere
When to use Polling:
- Admin dashboards
- Email refresh
- Analytics dashboards
- Simple status checks
When to use WebSockets / Socket.IO:
- Chat apps
- Food order tracking
- Notifications
- Live dashboards
- Multiplayer apps
Recommended engineering progression:
1. Version 1 → Polling
1. Version 2 → WebSockets
1. Version 3 → Redis Pub/Sub + scaling

## 16. Key Engineering Principle
Choose the simplest thing that works for the current scale. Do not overengineer for small projects.

## 17. Forward Proxy vs Reverse Proxy
- Proxy = middleman between client and server.
- Forward Proxy → works on behalf of client/user (VPN, company proxy).
- Reverse Proxy → works on behalf of server (NGINX).
- Forward proxy hides client identity.
- Reverse proxy hides backend infrastructure.
- Memory trick: Forward proxy protects client, reverse proxy protects server.

## 18. HTTPS and SSL Fundamentals
- HTTP transfers data in readable/plain form.
- HTTPS encrypts communication between browser and server.
- HTTPS provides two things:
- 1. Encryption of data in transit
- 2. Trust/identity verification using certificates
- Self-signed SSL = encrypted but not trusted.
- Production SSL = encrypted and trusted.

## 19. Self-Signed vs Trusted SSL
- Self-signed certificates are commonly used for localhost/development/testing.
- Browser warning does NOT mean encryption failed.
- Warning only means browser does not trust the issuer.
- Production usually uses trusted certificates such as Let’s Encrypt.
- Most modern platforms (Netlify/Vercel/Cloudflare) provide automatic SSL.

## 20. SSL Termination
- Browser → HTTPS → NGINX → HTTP → Backend
- NGINX decrypts HTTPS traffic and forwards internally using HTTP.
- This is called SSL termination.
- listen 443 ssl controls browser ↔ NGINX communication.
- proxy_pass http:// controls NGINX ↔ backend communication.

## 21. Why proxy_pass http:// still works with HTTPS
- HTTPS between browser and NGINX is separate from NGINX to backend.
- NGINX can receive HTTPS and internally forward plain HTTP.
- Ports alone do not determine HTTPS.
- 'ssl' directive enables HTTPS behavior.

## 22. NGINX + CDN + Netlify/Vercel
- If frontend is deployed on Netlify/Vercel, CDN serves static frontend directly.
- Flow: User → CDN → Static Frontend.
- API requests may still go through NGINX reverse proxy.
- NGINX may not be needed for frontend in managed hosting.

## 23. Caching in NGINX
- Static assets are naturally cached by browser/CDN.
- API caching in NGINX is manual, not automatic.
- Good cache candidates: public content, product catalog, restaurant list.
- Avoid caching user-specific data like cart/orders/profile.

## 24. Common Error: EADDRINUSE
- Meaning: address/port already in use.
- Occurs when another process already occupies same port.
- Useful commands:
- sudo lsof -i :3001
- ps aux | grep node
- kill -9 <PID>

## 25. HTTP Vulnerability Understanding
- HTTP traffic is readable/plain text during transmission.
- HTTPS encrypts traffic so intermediaries cannot easily inspect sensitive data.
- Sensitive information like passwords/session tokens should never travel over plain HTTP.
- HTTPS also protects integrity and server authenticity.