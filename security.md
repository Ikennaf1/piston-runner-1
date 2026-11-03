For your setup, I recommend putting a reverse proxy in front of Piston, with authentication and rate limiting.

The architecture should look like this:

Your web application / students

HTTP requests containing source code

Nginx reverse proxy

Authentication

Rate limiting

Request size limits

Timeouts

Piston API

Private Docker network — port 2000

Docker execution containers

Isolated language runtimes

The important principle is that students should never communicate directly with Piston. They should communicate with Nginx, which decides whether their requests are allowed.

## 1. Set up authentication

For a first implementation, we can use an API key.

Your application sends:

```
POST /api/v2/execute
Authorization: Bearer YOUR_SECRET_API_KEY
Content-Type: application/json
```

Nginx checks the key before forwarding the request to Piston.

## 2. Add rate limiting

For example, we can start with:

- Maximum 10 execution requests per minute per IP.
- Maximum 1 MB request body.
- Maximum 30 seconds for an API request.
- Reject requests without valid authentication.

These are starting values, not universal security limits. You can adjust them based on your students' usage.

## 3. Configure Nginx

Let's implement this with Nginx.

Create a directory in your project:

```
mkdir -p nginx
```

Create `nginx/default.conf`:

```

# Rate limit: maximum 10 requests per minute per IP
limit_req_zone $binary_remote_addr zone=piston_limit:10m rate=10r/m;

# Limit simultaneous connections per IP
limit_conn_zone $binary_remote_addr zone=piston_conn:10m;

server {
    listen 8080;

    server_name _;

    # Maximum submitted source-code payload
    client_max_body_size 256k;

    # Reject requests taking too long
    client_body_timeout 10s;
    send_timeout 15s;

    location /api/v2/ {

        # Authentication
        auth_basic "Piston API";
        auth_basic_user_file /etc/nginx/.htpasswd;

        # Rate limiting
        limit_req zone=piston_limit burst=5 nodelay;
        limit_req_status 429;

        # Connection limiting
        limit_conn piston_conn 5;

        # Forward requests to Piston
        proxy_pass http://piston:2000/api/v2/;

        proxy_http_version 1.1;

        proxy_connect_timeout 5s;
        proxy_send_timeout 15s;
        proxy_read_timeout 15s;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```

A few explanations:

- `auth_basic` requires a valid username and password.
- `limit_req` restricts how frequently an IP can submit requests.
- `limit_conn` limits simultaneous connections.
- `client_max_body_size` prevents someone from submitting excessively large source files.
- `proxy_pass` forwards accepted requests to Piston.

The rate limit is applied per IP, not per student. For a real educational platform, you'll eventually want per-user quotas as well.

Nginx's official documentation covers these rate and connection limiting directives: [Request limiting](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html?utm_source=chatgpt.com) and [Connection limiting](https://nginx.org/en/docs/http/ngx_http_limit_conn_module.html?utm_source=chatgpt.com).

## 4. Create an authentication password

Install the password utility:

```
sudo apt-get update
sudo apt-get install apache2-utils -y
```

Generate a username and password:

```
htpasswd -cB nginx/.htpasswd pistonuser
```

It will prompt you for a password.

This creates a password file that Nginx will use to authenticate requests.

Important: Never commit `.htpasswd` to GitHub. Add it to `.gitignore`.

```
nginx/.htpasswd
```

## 5. Put Nginx and Piston on the same Docker network

This is particularly important.

Piston's official Docker Compose setup starts its API service as `api`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



In your existing Piston `docker-compose.yml`, add an Nginx service:

```

services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:8080"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/.htpasswd:/etc/nginx/.htpasswd:ro
    depends_on:
      - api
    restart: unless-stopped

```

Keep the existing Piston `api` service.

One important modification: Remove its public port mapping if it currently contains:

```
ports:
  - "2000:2000"
```

Nginx and Piston will communicate through Docker's internal network. You don't need to publish port 2000 on the Codespace.

Also, change the upstream in your Nginx configuration to:

```
proxy_pass http://api:2000;
```

The hostname `api` is the Docker Compose service name.

Both services automatically share the default Compose network unless you have explicitly configured separate networks.

## 6. Start everything

From the Piston repository directory:

```
docker compose up -d
```

Check the containers:

```
docker compose ps
```

Your Codespace should now expose port `8080`, not port `2000`.

Update your `devcontainer.json`:

```
"forwardPorts": [8080]
```

In the Codespaces PORTS tab, keep port 8080 private while testing.

GitHub Codespaces supports private, organization-level and public port visibility. Public ports can be accessed by anyone who knows the URL, without GitHub authentication.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs+1



## 7. Test authentication

Test without credentials:

```
curl http://localhost:8080/api/v2/runtimes
```

You should get:

```
401 Unauthorized
```

Now test with credentials:

```
curl -u pistonuser \
http://localhost:8080/api/v2/runtimes
```

Curl will prompt you for the password.

You should now receive the available runtimes.

Test code execution through Nginx:

```
curl -u pistonuser \
-X POST http://localhost:8080/api/v2/execute \
-H "Content-Type: application/json" \
-d '{
  "language": "python",
  "version": "*",
  "files": [
    {
      "content": "print(2 + 3)"
    }
  ]
}'
```

If everything works, the response should contain `5`.

## 8. An important improvement for your actual student platform

There is something I want to emphasize before you deploy this.

Do not put the Nginx username and password inside your frontend JavaScript.

For example, don't do this:

```
fetch("https://your-piston-api.com/api/v2/execute", {
    headers: {
        Authorization: "Basic ..."
    }
});
```

Students can inspect browser requests and retrieve those credentials.

Instead, use this architecture:

Student's browser

Authenticated student account

Your application backend

Checks student login, permissions and usage quota

Nginx

Server-side authentication and rate limiting

Piston

Executes submitted code

Your backend should also enforce:

- A maximum number of executions per student per minute.
- A maximum source-code size.
- Allowed programming languages.
- Maximum execution time.
- Maximum compile and runtime memory.
- Maximum concurrent executions per student.

One subtle but important detail: Piston accepts execution time and memory limits in the submitted JSON. Your backend should set or clamp these values itself rather than trusting values supplied by students. The Piston API documents these controls.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



Also, Piston uses Isolate for sandboxing, including Linux namespaces, unprivileged users and resource controls. This is valuable protection, but it doesn't replace securing the surrounding Docker host.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## My recommendation

I'd build this in two stages:

1. Get Piston running behind Nginx in Codespaces, with the API port hidden.
2. Build a small Node.js or PHP authentication gateway that connects your student platform to Piston.

One question before we proceed:

What backend language does your student platform use?

Backend technology

PHP

Node.js

Python

Other / not decided

How will students access Piston?

Through my website after logging in

Directly through an API

Both

Continue

I can then give you a complete working implementation rather than just security recommendations.&#x20;

One final note: Codespaces is excellent for development and testing, but I would move the execution engine to a dedicated server before using it as a production service for students.