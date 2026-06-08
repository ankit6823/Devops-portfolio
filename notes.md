FROM node:20-bookworm-slim AS deps
-> This pull the official Image of node 20 from docker hub
-> I chose slim because it is less in Size
-> I have use Multistage build so I have alias as deps

WORKDIR /app
→ Sets the working directory inside the container
→ All subsequent commands run from /app
→ Creates the directory if it doesn't exist

COPY package*.json ./
-> COPY is used to copy the local file into docker Image
-> package* .json is used wildcard(*) because any file starting with package and ending with.json 
-> ./ files is copied in current directory inside the image (usually set by WORKDIR/app)

RUN npx playwright install --with-deps chromium
-> It install the playwright package with dependency of chromium

FROM node:20-bookworm-slim
-> This pull the official Image of node 20 from docker hub
-> I have choose bookworm-slim because we using multistage build it only used for running the file

RUN apt-get update && apt-get install
-> This will update all required system dependency to run application

COPY --from=deps /app/node_modules ./node_modules
-> This copy the node modules from builder stage from alias as deps

COPY . .
-> This copy the src code which is in same directory

RUN mkdir -p downloads logs
-> This create a folder called downloads logs -p it will publish the folder

ENV HEADLESS=true
-> ENV is environmental variable is stating that any browser should run in headless mode means without GUI

ENV NODE_ENV=production
-> Sets NODE_ENV to production. In Node.js applications, this typically enables production optimizations and disables development-only features.

ENV PLAYWRIGHT_BROWSERS_PATH=/root/.cache/ms-playwright
-> Sets the path where Playwright (a browser automation tool) will store its browser binaries.

ENV DISPLAY=:99
->Sets the DISPLAY environment variable to :99. This is commonly used in Linux environments to specify the X server display number, often for running GUI applications in a virtual display (useful for headless browser testing).

EXPOSE 3000
-> This tell that docker that container will listen on network port 3000

CMD ["node", "src/api/server.js"]
-> This will main filw we have to run
