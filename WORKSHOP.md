# Workshop title

Workshop description.

---

## 1. Project setup and basic build

1. **Create project folder**
   Create a new folder for your project in a sensible location, for example:

   ```shell
   mkdir -p ~/Documents/daemon-labs/docker-aws-lambda
   ```

   > [!NOTE]  
   > You can either create this via a terminal window or your file explorer.

3. **Open the new folder in your code editor**

   > If you are using VSCode, we can now do everything from within the code editor.

4. **Create `Dockerfile`**  
   Add the following content:

   ```Dockerfile
   FROM public.ecr.aws/lambda/nodejs:22
   ```

5. **Create `docker-compose.yaml`**  
   Add the following content to define your service:

   ```yaml
   ---
   services:
     lambda:
       build: .
   ```

   > You'll also notice that we're mounting a volume, this is to ensure any generated files are saved back to your local host folder.

6. **Initial image check**
   - Run the following command

     ```shell
     docker compose build
     ```

     > If you now run `docker images`, you'll see a newly created image which should be around 226MB in size.

   - Run the following command

     ```shell
     docker compose run -it --rm --entrypoint /bin/sh -v ./app:/var/task lambda
     ```

     > This command opens an interactive session with the container.

   - Run the following command

     ```shell
     node --version
     ```

     > The output should start with `v22` followed by the latest minor and patch version.

---

## 2. Dependency management and TypeScript config

1. **Initialise project and install dev dependencies**
   - Run the following command

   ```shell
    npm init -y
   ```

   > Notice how the `lambda` directory is automatically created on your host machine due to the volume mount.
   - Run the following command

     ```shell
     npm add --save-dev @types/node@22 @types/aws-lambda @tsconfig/recommended typescript
     ```

     > Notice this automatically creates a `package-lock.json` file.
     > Even though dependencies have been installed, if you run `docker images` again, you'll see the image size hasn't changed because the `node_modules` were written to your local volume, not the image layer.

2. **Exit the container**
   - Run the following command

   ```shell
   exit
   ```

   > We are now done with the interactive container at this stage and no longer need it.

3. **Create `tsconfig.json`**  
   Create `tsconfig.json` and add the following content to configure the TypeScript compiler:

   ```json
   {
     "extends": "@tsconfig/recommended/tsconfig.json",
     "compilerOptions": {
       "outDir": "./build/dist"
     }
   }
   ```

   > ℹ️ While you could auto-generate this file, our manual configuration using a recommended preset keeps the file minimal and clean.

4. **Create source file and scripts**
   - Create `./src/index.ts` with the following:

     ```typescript
     import { Handler } from "aws-lambda";

     export const handler: Handler = (event, context) => {
       console.log("Hello world!");
     };
     ```

   - Add the following to the `scripts` section in your `package.json`:

     ```json
     "build": "npm run build:tsc && npm run build:dependencies",
     "build:tsc": "rm -rf ./build/dist && tsc",
     "build:dependencies": "rm -rf ./build/dependencies && mkdir -p ./build/dependencies/nodejs && cp ./package*.json ./build/dependencies/nodejs && npm ci --omit dev --prefix ./build/dependencies/nodejs",
     ```

Update the `Dockerfile`

```Dockerfile
FROM public.ecr.aws/lambda/nodejs:22

COPY ./app /var/task

RUN npm ci && npm run build

HEALTHCHECK --interval=1s --timeout=1s --retries=30 \
    CMD [ "curl", "-I", "http://localhost:8080" ]

CMD [ "build/dist/index.handler" ]
```

Run `docker compose up --build`

> This Lambda starts but nothing happens

Kill the container `Ctrl+C`

Add a new service to the `docker-compos.yaml` file

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

> The Lambda and cURL containers start and execute, the cURL container responded with an exite code of 0 but is still hanging

Kill the container `Ctrl+C`

Run `docker compose up --build --abort-on-container-exit`

> The Lambda and cURL containers start and execute, the cURL container responded with an exite code of 0 and both containers shut down
