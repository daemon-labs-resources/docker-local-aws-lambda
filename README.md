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
>  
> ```shell
> $ docker images
> REPOSITORY                     TAG       IMAGE ID       CREATED        SIZE
> localstack/localstack          latest    de4d3256398a   25 hours ago   1.17GB
> public.ecr.aws/lambda/python   3.14      983ca119258a   3 days ago     584MB
> public.ecr.aws/lambda/nodejs   24        30d41baede74   3 days ago     449MB
> curlimages/curl                latest    26c487d15124   2 weeks ago    24.5MB
> ```
>  
> _Image IDs, created and sizes may vary._

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

### Run the initial build

Run the following command:

```shell
docker compose build
```

> [!NOTE]
> At this stage, if you loaded the Docker images as part of the prerequisites, if you run `docker images` you should see the `lambda/nodejs` image as well as a new image which is the same size.

### Initialise the container

Run this command to start an interactive shell:

```shell
docker compose run -it --rm --entrypoint /bin/sh -v ./nodejs:/var/task lambda
```

> [!WARNING]
> Due to AWS not creating multi-platform images we need to start an interactive shell rather than passing in commands.  
> For example, if we were to run the following command:
> ```shell
> docker compose run -it --rm --entrypoint /bin/sh -v ./nodejs:/var/task lambda node --version
> ```
> In some cases, we would receive the error `/var/lang/bin/node: /var/lang/bin/node: cannot execute binary file`.

### Image check

Run the following command:

```shell
node --version
```

> [!NOTE]
> The output should start with `v24` followed by the latest minor and patch version.

---

## 2. The application

**Goal:** Initialise a TypeScript Node.js project.

### Initialise the project

Inside the container shell:

```shell
npm init -y
```

> [!NOTE]
> Notice how the `nodejs/package.json` file is automatically created on your host machine due to the volume mount.

### Install dependencies

```shell
npm add --save-dev @types/node@24 @types/aws-lambda @tsconfig/recommended typescript
```

> [!NOTE]
> Notice this automatically creates a `nodejs/package-lock.json` file as well as the `nodejs/node_modules` directory.

### Exit the container

```shell
exit
```

> [!NOTE]
> At this stage, we no longer need the interactive shell and can return to the code editor.
> Even though dependencies have been installed, if you run `docker images` again, you'll see the image size hasn't changed because the `node_modules` were written to your local volume, not via an image layer.

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

> [!NOTE]
> While you could auto-generate this file, our manual configuration using a recommended preset keeps the file minimal and clean.

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

> [!NOTE]
> At this stage we have the main building blocks for the application, but our runtime doesn't know what to do with them.

---

## 3. The runtime

**Goal:** Make the container act like a real Lambda server.

### Add `.dockerignore`

Create `nodejs/.dockerignore` (inside the subdirectory):

```plaintext
build
node_modules
```

> [!NOTE]
> We're makeing sure that no matter where we're building the image it never loads in any built files or local `node_modules`.  
> That way, whenever we're building it is done in an identical way and reduces the possibility of "it worked on my machine".

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

> [!NOTE]
> As we're now doing the dependency install as part of the build, when you run `docker images` you'll notice our Docker image has increased in size.
>
> ```shell
> $ docker images       
> REPOSITORY                     TAG       IMAGE ID       CREATED         SIZE
> your-lambda                    latest    05b92630088f   3 seconds ago   483MB
> public.ecr.aws/lambda/nodejs   24        30d41baede74   3 days ago      449MB
> ```

> [!TIP]
> When running `docker images` you'll notice that we have got a dangling image that looks a bit like this:
> 
> ```shell
> $ docker images       
> REPOSITORY                     TAG       IMAGE ID       CREATED         SIZE
> your-lambda                    latest    05b92630088f   3 seconds ago   483MB
> public.ecr.aws/lambda/nodejs   24        30d41baede74   3 days ago      449MB
> <none>                         <none>    17e6c55f785f   3 days ago      449MB
> ```
>
> When you rebuilt the image, Docker moved the "nametag" to your new version, leaving the old version behind as a nameless orphan.
> 
> Any dangling images can be cleaned with the following command:
> 
> ```shell
> docker image prune
> ```

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
    interval: 1s
    timeout: 1s
    retries: 30
```

> [!TIP]
> The healthcheck allows Docker (and us) to know when a container is up and running as expected.  
> If you were to run `docker ps` in a different terminal window while our containers were starting up you might see the following:
>
> ```shell
> $ docker ps
> CONTAINER ID   IMAGE             COMMAND                  CREATED        STATUS                                     PORTS     NAMES
> bf2696aeaabf   solution-lambda   "/lambda-entrypoint.…"   1 second ago   Up Less than a second (health: starting)             your-lambda-1
> ```
>
> If you ran `docker ps` once the container was able to pass the healthcheck you would hopefully see the following:
>
> ```shell
> $ docker ps
> CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS                    PORTS     NAMES
> bf2696aeaabf   solution-lambda   "/lambda-entrypoint.…"   36 seconds ago   Up 35 seconds (healthy)             your-lambda-1
> ```
>
> _If the container wasn't able to pass the healthcheck then you would eventually see `unhealthy` instead._

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
  # ... existing config
```

> [!NOTE]
> As we have the healthceck in place, we can actually tell the `curl` container not to start until it gets that healthy response.

### Try running the stack

Run the following command:

```shell
docker compose up
```

> [!WARNING]
> The problem with this specific command is that the Lambda continues to run despite the cURL container running and exiting.
> **Exit your container by pressing Ctrl+C on your keyboard.**

### Run the stack

Run the following command:

```shell
docker compose up --abort-on-container-exit
```

> [!TIP]
> With this extra attribute, we've told Docker to terminate all other running containers when one exits.

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
  console.log("Hello world!");
  console.log({ event, context });

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

## 8. Cleanup

**Goal:** Remove containers and reclaim disk space.

Since we are done with the workshop, let's remove the resources we created.

Run the following command to stop all services, remove the containers/networks, and delete all images used by this project (including the Node/Python base images, LocalStack, and the custom image we built):

```shell
docker compose down --rmi all
```

> [!WARNING]
> If you followed the prerequisites to run `docker load` this command will not actually remove all images, the `lambda/node` and `lambda/python` images still exist.
>  
> To remove these you'll need to run the following:
> ```shell
> docker rmi public.ecr.aws/lambda/nodejs:24
> ```
> ```shell
> docker rmi public.ecr.aws/lambda/python:3.14
> ```


---

## 🎉 Congratulations

You have built a clean, modular, serverless development environment.
