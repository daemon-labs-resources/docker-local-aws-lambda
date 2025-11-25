# Learn to build and develop AWS Lambda functions locally with Docker

@todo Workshop description.

---

## 🛑 Prerequisites

Before beginning this workshop, please ensure your environment is correctly set up by following the instructions in our prerequisites documentation:

➡️ **[Prerequisites guide](https://github.com/daemon-labs-resources/prerequisites)**

Run the following command:

```shell
docker pull ...
```

> [!TIP]
> Pulling the Docker images isn't a requirement, but it helps to have it pre-downloaded so we don't wait for everyone to do it at the same time.

---

### Create project folder

Create a new folder for your project in a sensible location, for example:

```shell
mkdir -p ~/Documents/daemon-labs/docker-aws-lambda
```

> [!NOTE]
> You can either create this via a terminal window or your file explorer.

### Open the new folder in your code editor

> [!TIP]
> If you are using VSCode, we can now do everything from within the code editor.

### Create `./nodejs` folder

### Create `Dockerfile`

Add the following content:

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:22
```

### Create `docker-compose.yaml`

Add the following content to define your service:

```yaml
---
services:
  lambda:
    build: .
```

### Initial image check

Run the following command:

```shell
docker compose build
```

> [!NOTE]
> If you now run `docker images`, you'll see a newly created image which should be around 436MB in size.

Run the following command:

```shell
docker compose run -it --rm --entrypoint /bin/sh -v ./nodejs:/var/task lambda
```

> [!NOTE]
> This command opens an interactive session with the container.

Run the following command:

```shell
node --version
```

> [!NOTE]
> The output should start with `v22` followed by the latest minor and patch version.

### Initialise project and install dev dependencies

Run the following command:

```shell
npm init -y
```

> [!NOTE]
> Notice how the `nodejs` directory is automatically created on your host machine due to the volume mount.

Run the following command:

```shell
npm add --save-dev @types/node@22 @types/aws-lambda @tsconfig/recommended typescript
```

> [!NOTE]
> Notice this automatically creates a `package-lock.json` file.

### Exit the container

Run the following command:

```shell
exit
```

> [!NOTE]
> We are now done with the interactive container at this stage and no longer need it.

### Create `./nodejs/tsconfig.json`

Create `./nodejs/tsconfig.json` and add the following content to configure the TypeScript compiler:

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

### Create source file and scripts

Create `./nodejs/src/index.ts` with the following:

```typescript
import { Handler } from "aws-lambda";

export const handler: Handler = (event, context) => {
    console.log("Hello world!");
    console.log({ event, context });
};

```

Add the following to the `scripts` section in your `package.json`:

```json
"build": "tsc",
```

---

Update the `Dockerfile`

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:22

HEALTHCHECK --interval=1s --timeout=1s --retries=30 \
    CMD [ "curl", "-I", "http://localhost:8080" ]

COPY ./nodejs ${LAMBDA_TASK_ROOT}

RUN npm ci && npm run build

CMD [ "build/index.handler" ]
```

Run `docker compose up --build`

> [!WARNING]
> This Lambda starts but nothing happens.  
> **Exit your container by pressing `Ctrl+C`** on your keyboard.

Add a new service to the `docker-compose.yaml` file

```yaml
curl:
  image: curlimages/curl
  depends_on:
    lambda:
      condition: service_healthy
  command:
    - -s
    - -d {}
    - http://lambda:8080/2015-03-31/functions/function/invocations
```

Run `docker compose up --build`

> [!WARNING]
> The Lambda and cURL containers start and execute, the cURL container responded with an exit code of 0 but is still hanging.  
> **Exit your container by pressing `Ctrl+C`** on your keyboard.

Run `docker compose up --build --abort-on-container-exit`

> [!NOTE]
> The Lambda and cURL containers start and execute, the cURL container responded with an exit code of 0 and both containers shut down.

Add the following environment variables to the `Dockerfile`

```Dockerfile
ENV AWS_LAMBDA_FUNCTION_MEMORY_SIZE=128
ENV AWS_LAMBDA_FUNCTION_TIMEOUT=3
```

> [!TIP]
> This replicates the default settings of an AWS Lambda.  
> Without these the Docker image defaults to a memory size of 3008MB and potentially an infinite timeout.

Add the following environment variable to the `Dockerfile`

```Dockerfile
ENV AWS_LAMBDA_LOG_FORMAT=JSON
```

> [!TIP]
> When we executed our Lambda you might have noticed our two logs were printed as plain text.  
> In fact, the `console.log({ event, context });` didn't actually print out anything useful at all.

### Set up an event

Create an `events` directory

Create a `test.json` file, add the following and save:

```json
{
  "test": "test"
}
```

### Update the command in `docker-compose.yaml`

Update the command for the cURL container:

```yaml
command:
  - -s
  - -d 
  - ${LAMBDA_INPUT:-{}}
  - http://lambda:8080/2015-03-31/functions/function/invocations
```

> [!NOTE]
> By defining the command like this, by default it will pass in `{}` as the event still.

Add the `events` directory as a volume to the cURL container:

```yaml
volumes:
  - ./events:/events:ro
```

Run `LAMBDA_INPUT=@/events/test.json docker compose up --build --abort-on-container-exit`

> [!TIP]
> The `@` tells the cURL command that it should include the contents of a file rather than passing as a string.

### Update the `Dockerfile`

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:22

ENV AWS_LAMBDA_FUNCTION_MEMORY_SIZE=128
ENV AWS_LAMBDA_FUNCTION_TIMEOUT=3
ENV AWS_LAMBDA_LOG_FORMAT=JSON

HEALTHCHECK --interval=1s --timeout=1s --retries=30 \
    CMD [ "curl", "-I", "http://localhost:8080" ]

COPY ./nodejs/package*.json ${LAMBDA_TASK_ROOT}

RUN npm ci

COPY ./nodejs ${LAMBDA_TASK_ROOT}

RUN npm run build

CMD [ "build/index.handler" ]
```

### Update the `Dockerfile`

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:22 AS base

ENV AWS_LAMBDA_FUNCTION_MEMORY_SIZE=128
ENV AWS_LAMBDA_FUNCTION_TIMEOUT=3
ENV AWS_LAMBDA_LOG_FORMAT=JSON

HEALTHCHECK --interval=1s --timeout=1s --retries=30 \
    CMD [ "curl", "-I", "http://localhost:8080" ]

FROM base AS build

COPY ./nodejs/package*.json ${LAMBDA_TASK_ROOT}

RUN npm ci

COPY ./nodejs ${LAMBDA_TASK_ROOT}

RUN npm run build

FROM base

COPY --from=build ${LAMBDA_TASK_ROOT}/package*.json ${LAMBDA_TASK_ROOT}
COPY --from=build ${LAMBDA_TASK_ROOT}/build ${LAMBDA_TASK_ROOT}/build

RUN npm ci --only=production

CMD [ "build/index.handler" ]
```

### Create a `python` directory

### Create a `./python/Dockerfile`

### Create the `./python/requirements.txt`

### Create the handler file at `./python/app.py`

### Update `docker-compose.yaml`

### Run the Lambda

- SAM

- cleanup

---

## 🎉 Congratulations

You've just learnt to build and develop AWS Lambda functions locally with Docker.

### Recap of what you built
