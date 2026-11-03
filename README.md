# Piston Runner

## Runner for aceolearn code editor using open source piston

Perfect. You can run the Piston code execution engine directly inside GitHub Codespaces, using Docker-in-Docker.

Your existing configuration is a good starting point. However, there are a few things we should account for:

1. Piston needs Docker to launch isolated containers for executing submitted code.
2. The Piston API listens on port `2000`.
3. Piston does not automatically come with all language runtimes installed. You'll need to install the ones you want.
4. Runtime packages need persistent storage so they aren't lost when containers are recreated.

The official repository is [engineer-man/piston](https://github.com/engineer-man/piston?utm_source=chatgpt.com). Its installation instructions support Docker-based deployment.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



## 1. Update your devcontainer.json

I'd use this configuration:

```

{
    "name": "piston-test",
    "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
    "features": {
        "ghcr.io/devcontainers/features/docker-in-docker:2": {
            "version": "latest"
        }
    },
    "forwardPorts": [2000],
    "portsAttributes": {
        "2000": {
            "label": "Piston API",
            "onAutoForward": "notify"
        }
    },
    "postStartCommand": "docker --version && docker compose version",
    "hostRequirements": {
        "cpus": 4,
        "memory": "8gb"
    },
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-vscode.vscode-json"
            ]
        }
    }
}

```

The important changes are:

- Docker Compose version verification.
- Explicit Docker-in-Docker version.
- A label for the forwarded API port.

After saving, rebuild your Codespace so the updated container configuration takes effect.

## 2. Install Piston

Open the Codespaces terminal and execute:

```
git clone https://github.com/engineer-man/piston.git

cd piston
```

Start the API:

```
docker-compose up -d api
```

Install the CLI dependencies:

```
cd cli
npm install
cd ..
```

This starts the Piston API, but it won't yet have language runtimes installed.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## 3. Install programming language runtimes

For example, to install Python, JavaScript (Node.js), PHP, C++, and Bash:

```
./cli/index.js ppman install python
./cli/index.js ppman install node
./cli/index.js ppman install php
./cli/index.js ppman install gcc
./cli/index.js ppman install bash
```

You can also inspect available packages:

```
./cli/index.js ppman list
```

Runtime installation can take some time because the packages need to be downloaded.

## 4. Test that Piston is working

Check whether the API is responding:

```
curl http://localhost:2000/api/v2/runtimes
```

You should receive a JSON array containing installed language runtimes.

Now test code execution:

```
curl -X POST http://localhost:2000/api/v2/execute \
-H "Content-Type: application/json" \
-d '{
    "language": "python",
    "version": "*",
    "files": [
        {
            "name": "main.py",
            "content": "print(\"Hello from Piston!\")"
        }
    ]
}'
```

You should receive a JSON response containing:

```
{
    "run": {
        "stdout": "Hello from Piston!\n",
        "stderr": "",
        "code": 0
    }
}
```

## 5. Access Piston externally

Once running:

- Open the PORTS tab in Codespaces.
- Find port `2000`.
- Copy its forwarded address.

Your API will be accessible through a URL similar to:

```
https://YOUR-CODESPACE-2000.app.github.dev
```

GitHub normally forwards ports privately. You can use the forwarded address with the appropriate GitHub authentication, or access `localhost:2000` from applications running inside the same Codespace.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs+1

## To stop / save quota (Codespaces)

- Run `gh codespace stop` to stop immedialtely, or just close tab (which auto-stops after 30 mins of idle time).
- Delete codespace when done: GitHub > Code > Codespaces > (click) ... > Delete



Important security consideration

Piston executes arbitrary code. Don't expose its API publicly without putting proper authentication, request limits and other security controls in front of it. Also, make sure its runtime package storage is persistent.
