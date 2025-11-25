# Learn to build and develop AWS Lambda functions locally with Docker

@todo Workshop description.

---

## 🛑 Prerequisites

Before beginning this workshop, please ensure your environment is correctly set up by following the instructions in our prerequisites documentation:

➡️ **[Prerequisites guide](https://github.com/daemon-labs-resources/prerequisites)**

### Retrieve Docker images

#### Load Docker images

> [!CAUTION]
> This only works when attending a workshop in person.  
> Due to having a number of people trying to retrieve Docker images at the same time, this allows for a more efficient way.
>
> If you are **NOT** in an in-person workshop, see [pull docker images](#pull-docker-images).

Once the facilitator has given you an IP address, open `http://<IP-ADDRESS>:8000` in your browser.

When you see the file listing, download the `workshop-images.tar` file.

> [!WARNING]
> Your browser may block the download initially, when prompted, allow it to download.

Run the following command:

```shell
docker load -i ~/Downloads/workshop-images.tar
```

#### Pull Docker images

> [!CAUTION]
> Only use this approach if you are running through this workshop on your own.
>
> If you are in an in-person workshop, see [load docker images](#load-docker-images).

Run the following command:

```shell
docker pull public.ecr.aws/lambda/nodejs:24
docker pull public.ecr.aws/lambda/python:3.14
docker pull curlimages/curl
docker pull localstack/localstack
```

### Validate Docker images

Run the following command:

```shell
docker images
```

> [!NOTE]
> You should now see four images listed.

---

## 1. The foundation

**Goal:** Get a working container environment running.

### Create project folder

Create a new folder for your project:

```shell
mkdir -p ~/Documents/daemon-labs/docker-aws-lambda
```

> [!NOTE]
> You can either create this via a terminal window or your file explorer.

### Open the new folder in your code editor

> [!TIP]
> If you are using VSCode, we can now do everything from within the code editor.  
> You can open the terminal pane via Terminal -> New Terminal.

### Create the code subdirectory

We keep our application code separate from infrastructure config.

```shell
mkdir nodejs
```

### Create the `Dockerfile`

Create the file at `./nodejs/Dockerfile` (inside the subdirectory).

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:24
```

### Create `docker-compose.yaml`

Create this file in the **root** of your project.

```yaml
services:
  lambda:
    build: ./nodejs
```

### Initialise the container

Run this command to start an interactive shell.

```shell
docker compose run -it --rm --entrypoint /bin/sh -v ./nodejs:/var/task lambda
```

---

## 2. The application

**Goal:** Initialise a TypeScript Node.js project.

### Initialise the project

Inside the container shell:

```shell
npm init -y
```

### Install dependencies

```shell
npm add --save-dev @types/node@24 @types/aws-lambda @tsconfig/recommended typescript
```

### Exit the container

```shell
exit
```

### Configure TypeScript

Create `./nodejs/tsconfig.json` locally:

```json
{
  "extends": "@tsconfig/recommended/tsconfig.json",
  "compilerOptions": {
    "outDir": "./build"
  }
}
```

### Create the handler

Create `./nodejs/src/index.ts`:

```typescript
import { Handler } from "aws-lambda";

export const handler: Handler = (event, context) => {
  console.log("Hello world!");
  console.log({ event, context });

  return {
    statusCode: 200,
    body: "Hello World!",
  };
};
```

### Add build script

Update `./nodejs/package.json` scripts:

```json
"scripts": {
  "build": "tsc"
},
```

---

## 3. The runtime

**Goal:** Make the container act like a real Lambda server.

### Add `.dockerignore`

Create `./nodejs/.dockerignore` (inside the subdirectory).  
This is critical because our build context is now that specific folder.

```plaintext
build
node_modules
```

### Update `Dockerfile`

Update `./nodejs/Dockerfile`.  
Notice that the `COPY` paths are cleaner now because they are relative to the `nodejs` folder.

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:24

HEALTHCHECK --interval=1s --timeout=1s --retries=30 \
    CMD [ "curl", "-I", "http://localhost:8080" ]

COPY ./package*.json ./

RUN npm ci

COPY ./ ./

RUN npm run build

CMD [ "build/index.handler" ]
```

### Add cURL service

Update `docker-compose.yaml` (in the root) to include a service that triggers our Lambda.

```yaml
services:
  curl:
    image: curlimages/curl
    depends_on:
      lambda:
        condition: service_healthy
    command:
      - -s
      - -d {}
      - http://lambda:8080/2015-03-31/functions/function/invocations
  lambda:
    build: ./nodejs
```

### Run the stack

```shell
docker compose up --build --abort-on-container-exit
```

---

## 4: Developer experience

**Goal:** Simulate real-world events and environments.

### Add environment variables

Update `./nodejs/Dockerfile`:

```Dockerfile
ENV AWS_LAMBDA_FUNCTION_MEMORY_SIZE=128
ENV AWS_LAMBDA_FUNCTION_TIMEOUT=3
ENV AWS_LAMBDA_LOG_FORMAT=JSON
```

### Create an event file

Create `./events/test.json` in the root (keep events outside the code folder):

```json
{
  "user": "Alice",
  "action": "login"
}
```

### Inject the event

Update `docker-compose.yaml`:

```yaml
curl:
  # ... existing config
  command:
    - -s
    - -d
    - ${LAMBDA_INPUT:-{}}
    - http://lambda:8080/2015-03-31/functions/function/invocations
  volumes:
    - ./events:/events:ro
```

### Test with data

```shell
LAMBDA_INPUT=@/events/test.json docker compose up --build --abort-on-container-exit
```

---
