# Learn to build and develop AWS Lambda functions locally with Docker

This guide walks you through setting up a robust local Serverless development environment using **Docker**, **AWS Lambda**, **TypeScript**, and **LocalStack**.  
It focuses on emulating the cloud runtime entirely offline, optimising production images with multi-stage builds, and mocking external services like S3 to create a complete, cost-free development workflow.

---

## 🛑 Prerequisites

### General/global prerequisites

Before beginning this workshop, please ensure your environment is correctly set up by following the instructions in our prerequisites documentation:

➡️ **[Prerequisites guide](https://github.com/daemon-labs-resources/prerequisites)**

### Load Docker images

> [!CAUTION]
> This only works when attending a workshop in person.  
> Due to having a number of people trying to retrieve Docker images at the same time, this allows for a more efficient way.
>
> If you are **NOT** in an in-person workshop, continue to the [workshop](#1-the-foundation), Docker images will be pulled as needed.

Once the facilitator has given you an IP address, open `http://<IP-ADDRESS>:8000` in your browser.

When you see the file listing, download the `workshop-images.tar` file.

> [!WARNING]
> Your browser may block the download initially, when prompted, allow it to download.

Run the following command:

```shell
docker load -i ~/Downloads/workshop-images.tar
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
mkdir ./nodejs
```

### Create the `Dockerfile`

Create the file at `nodejs/Dockerfile` (inside the subdirectory).

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

Run this command to start an interactive shell:

```shell
docker compose run -it --rm --entrypoint /bin/sh -v ./nodejs:/var/task lambda
```

### Image check

Run the following command:

```shell
node --version
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

Create `nodejs/tsconfig.json` locally:

```json
{
  "extends": "@tsconfig/recommended/tsconfig.json",
  "compilerOptions": {
    "outDir": "./build"
  }
}
```

### Create the handler

Create `nodejs/src/index.ts`:

```typescript
import { Handler } from "aws-lambda";

export const handler: Handler = async (event, context) => {
  console.log("Hello world!");
  console.log({ event, context });

  return {
    statusCode: 200,
    body: "Hello World!",
  };
};
```

### Add build script

Update `nodejs/package.json` scripts:

```json
"build": "tsc"
```

---

## 3. The runtime

**Goal:** Make the container act like a real Lambda server.

### Add `.dockerignore`

Create `nodejs/.dockerignore` (inside the subdirectory).  
This is critical because our build context is now that specific folder.

```plaintext
build
node_modules
```

### Update `Dockerfile`

Update `nodejs/Dockerfile`:

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:24

COPY ./package*.json ${LAMBDA_TASK_ROOT}

RUN npm ci

COPY ./ ${LAMBDA_TASK_ROOT}

RUN npm run build

CMD [ "build/index.handler" ]
```

### Update Lambda healthcheck

Update `docker-compose.yaml`:

```yaml
lambda:
  build: ./nodejs
  healthcheck:
    test:
      - CMD
      - curl
      - -I
      - http://localhost:8080
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

### Run the stack

```shell
docker compose up --build --abort-on-container-exit
```

---

## 4: Developer experience

**Goal:** Simulate real-world events and environments.

### Add environment variables

Update `docker-compose.yaml`:

```yaml
services:
  # ... existing config
  lambda:
    # ... existing config
    environment:
      AWS_LAMBDA_FUNCTION_MEMORY_SIZE: 128
      AWS_LAMBDA_FUNCTION_TIMEOUT: 3
      AWS_LAMBDA_LOG_FORMAT: JSON
```

### Create the events subdirectory

Create the events subdirectory in the root (keep events outside the code folder):

```shell
mkdir ./events
```

### Create a custom event file

Create `events/custom.json`:

```json
{
  "user": "Alice",
  "action": "login"
}
```

### Create API Gateway event file

Create `events/api-gateway.json`:

```json
{
  "resource": "/",
  "path": "/",
  "httpMethod": "POST",
  "body": "{\"user\": \"Alice\"}",
  "isBase64Encoded": false
}
```

### Inject the event

Update `docker-compose.yaml`:

```yaml
services:
  curl:
    # ... existing config
    command:
      - -s
      - -d
      - ${LAMBDA_INPUT:-{}}
      - http://lambda:8080/2015-03-31/functions/function/invocations
volumes:
  - ./events:/events:ro
  # ... existing config
```

### Test with data

```shell
docker compose up --build --abort-on-container-exit
```

```shell
LAMBDA_INPUT=@/events/custom.json docker compose up --build --abort-on-container-exit
```

```shell
LAMBDA_INPUT=@/events/api-gateway.json docker compose up --build --abort-on-container-exit
```

---

## 5. Optimisation

**Goal:** Prepare for production with multi-stage builds.

### Multi-stage build

Replace `nodejs/Dockerfile` with this optimised version:

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:24 AS base

FROM base AS builder

COPY ./package*.json ${LAMBDA_TASK_ROOT}

RUN npm ci

COPY ./ ${LAMBDA_TASK_ROOT}

RUN npm run build

FROM base

COPY --from=builder ${LAMBDA_TASK_ROOT}/package*.json ${LAMBDA_TASK_ROOT}

RUN npm ci --only=production

COPY --from=builder ${LAMBDA_TASK_ROOT}/build ${LAMBDA_TASK_ROOT}/build

CMD [ "build/index.handler" ]
```

### Test the optimised build

Run the following command to ensure everything still works:

```shell
docker compose up --build --abort-on-container-exit
```

---

## 6: Advanced integration

**Goal:** Connect to LocalStack.

### Add LocalStack Service to `docker-compose.yaml`

```yaml
localstack:
  image: localstack/localstack
  healthcheck:
    test:
      - CMD
      - curl
      - -f
      - http://localhost:4566/_localstack/health
    interval: 1s
    timeout: 1s
    retries: 30
```

### Update Lambda config

Update `docker-compose.yaml`:

```yaml
depends_on:
  localstack:
    condition: service_healthy
environment:
  AWS_LAMBDA_FUNCTION_MEMORY_SIZE: 128
  AWS_LAMBDA_FUNCTION_TIMEOUT: 3
  AWS_LAMBDA_LOG_FORMAT: JSON
  AWS_ENDPOINT_URL: http://localstack:4566
  AWS_SECRET_ACCESS_KEY: test
  AWS_ACCESS_KEY_ID: test
  AWS_REGION: us-east-1
```

### Update code

Run this command to start an interactive shell:

```shell
docker compose run -it --rm --no-deps --entrypoint /bin/sh -v ./nodejs:/var/task lambda
```

Install the SDK:

```shell
npm install @aws-sdk/client-s3
```

Exit the container

```shell
exit
```

Next, update `nodejs/src/index.ts` with the S3 client logic:

```typescript
import { Handler } from "aws-lambda";
import { S3Client, ListBucketsCommand } from "@aws-sdk/client-s3";

const client = new S3Client({
  endpoint: process.env.AWS_ENDPOINT_URL, // Points to LocalStack
  forcePathStyle: true, // Required for local mocking
  region: process.env.AWS_REGION,
});

export const handler: Handler = async (event, context) => {
  try {
    const command = new ListBucketsCommand({});
    const response = await client.send(command);

    console.log("S3 Buckets:", response.Buckets);

    return {
      statusCode: 200,
      body: JSON.stringify(response.Buckets || []),
    };
  } catch (error) {
    console.error(error);
    return {
      statusCode: 500,
      body: "Error connecting to S3",
    };
  }
};
```

### Final run

Run the following command:

```shell
docker compose up --build --abort-on-container-exit
```

---

## 7. Bonus: Swapping runtimes

**Goal:** Demonstrate the versatility of Docker by swapping to Python.

One of the biggest advantages of developing Lambdas with Docker is that the infrastructure pattern remains exactly the same, regardless of the language you use.

### Create a Python `Dockerfile`

Create `python/Dockerfile` with the following content:

```Dockerfile
FROM public.ecr.aws/lambda/python:3.14

COPY ./ ${LAMBDA_TASK_ROOT}

CMD [ "app.handler" ]
```

### Create the Python handler

Create the handler file at `python/app.py`:

```python
def handler(event, context):
    return "Hello World!"
```

### Update `docker-compose.yaml`

Update the `lambda` service in `docker-compose.yaml` to point to the Python folder:

```yaml
services:
  lambda:
    build: ./python
```

### Run it

```shell
docker compose up --build --abort-on-container-exit
```

> [!NOTE]
> You will see the build process switch to pulling the Python base image, but the curl command and event injection work exactly the same way.

---

## 🎉 Congratulations

You have built a clean, modular, serverless development environment.
