# Chapter 20: Deployment & Cloud Native Architectures

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Differentiate between Client-Side Rendering (CSR) hosting and Server-Side Rendering (SSR) hosting requirements.
* Containerize an Angular application using Docker.
* Create a multi-stage Dockerfile using NGINX for static file serving.
* Understand the principles of a CI/CD Pipeline (GitHub Actions / Azure DevOps).
* Implement Environment Variable injection via Docker instead of hardcoding `environment.ts`.

---

## 2. Introduction: The Final Mile

Writing the application is only half the battle. In an enterprise environment, your code must be tested, compiled, optimized, and deployed automatically to a highly available, scalable cloud infrastructure. 

How you deploy Angular depends entirely on whether your application uses **Client-Side Rendering (CSR)** or **Server-Side Rendering (SSR)**.

---

## 3. Deployment Paradigms

### 1. CSR (Static Hosting)
If your Angular application is purely Client-Side Rendered (an SPA), the build output is just static files: `index.html`, `main.js`, `styles.css`, and some assets. 
* **Requirement:** A static file server.
* **Services:** Azure Blob Storage (Static Websites), AWS S3, Firebase Hosting, Cloudflare Pages, or a simple NGINX server.
* **Cost:** Practically free. Highly scalable via CDNs.

### 2. SSR (Node.js Hosting)
If your application uses Angular Universal / Angular SSR (as discussed in Chapter 15), the build output includes a Node.js server that dynamically generates HTML on the fly.
* **Requirement:** A compute engine capable of running Node.js.
* **Services:** Azure App Service, AWS Fargate, Google Cloud Run, or Kubernetes.
* **Cost:** Higher. Requires active CPU and Memory to handle incoming requests and render the pages.

---

## 4. Containerizing Angular (Docker)

In modern enterprise architectures, applications are deployed as Docker containers. This ensures the application runs exactly the same way on the developer's laptop as it does in the Production cluster.

### The CSR Dockerfile (Multi-Stage Build)
For a static SPA, we don't need Node.js in the final container. We use a **Multi-Stage Build**: Node.js builds the app, and NGINX serves it.

**`Dockerfile`**
```dockerfile
# STAGE 1: Build the Angular App
# Use a heavy Node image to install dependencies and compile the code
FROM node:20-alpine AS build
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm ci

# Copy the source code and build the production bundle
COPY . .
RUN npm run build -- --configuration=production

# STAGE 2: Serve the App using NGINX
# Use a lightweight NGINX image for the final container
FROM nginx:alpine
# Copy the compiled static files from Stage 1 into the NGINX web directory
COPY --from=build /app/dist/ev-platform/browser /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# Start NGINX
CMD ["nginx", "-g", "daemon off;"]
```

### The NGINX Fallback Routing Rule
When hosting an Angular SPA on NGINX, if a user refreshes the page at `/chargers/123`, NGINX will look for a folder named `chargers` and return a 404 error. You must configure NGINX to route all missing files back to `index.html` so the Angular Router can take over.

**`nginx.conf`**
```nginx
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        # THE MAGIC LINE: Fallback to index.html
        try_files $uri $uri/ /index.html; 
    }
}
```

---

## 5. Environment Variables: The Cloud Native Way

Historically, Angular developers hardcoded API URLs into `environment.prod.ts` and `environment.dev.ts`.

**The Problem:** If you build a Docker image for Staging, and then promote that *exact same image* to Production, the image still contains the Staging API URL hardcoded into the `main.js` bundle. You would have to rebuild the image entirely for Production, which violates the core Docker philosophy: **Build Once, Run Anywhere**.

### The Solution: Runtime Configuration Injection
Instead of using `environment.ts`, the Angular application should fetch a `config.json` file at runtime before the application boots up.

**`assets/config.json`**
```json
{
  "apiUrl": "http://localhost:5000"
}
```

**`app.config.ts`**
```typescript
import { APP_INITIALIZER } from '@angular/core';

export function loadConfig(http: HttpClient, configService: ConfigService) {
  return () => http.get('/assets/config.json').pipe(
    tap(config => configService.setConfig(config))
  );
}

export const appConfig = {
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: loadConfig,
      deps: [HttpClient, ConfigService],
      multi: true
    }
  ]
};
```

**The CI/CD Magic:**
Now, you build your Docker image *once*. When Kubernetes starts the container in the Staging environment, a Kubernetes ConfigMap silently overwrites the `/assets/config.json` file with the Staging URLs. When it starts in Production, it overwrites it with Production URLs. The JavaScript bundle is never rebuilt.

---

## 6. Continuous Integration / Continuous Deployment (CI/CD)

A CI/CD Pipeline automates the entire lifecycle of your application.

### The Pipeline Steps
1. **Linting:** Runs `ng lint` to enforce code style.
2. **Testing:** Runs `ng test` to ensure no unit tests are broken.
3. **Build:** Runs `ng build` to compile the AOT production bundle.
4. **Containerize:** Runs `docker build` and pushes the image to Azure Container Registry (ACR).
5. **Deploy:** Tells Azure App Service or Kubernetes to pull the new image and restart the pods.

### Example GitHub Actions Pipeline
```yaml
name: Production Deployment
on:
  push:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
          
      - name: Install Dependencies
        run: npm ci
        
      - name: Run Tests
        run: npm run test -- --watch=false
        
      - name: Build Application
        run: npm run build -- --configuration=production
        
      - name: Build & Push Docker Image
        run: |
          docker build -t myacr.azurecr.io/ev-platform:${{ github.sha }} .
          docker push myacr.azurecr.io/ev-platform:${{ github.sha }}
```

---

## 7. Common Mistakes

### Beginner Mistakes
* **Pushing `node_modules` to the server:** Never manually copy files to a server via FTP. The server (or Docker container) should always run `npm ci` (Clean Install) to ensure the exact dependency tree specified in `package-lock.json` is used.
* **Forgetting the NGINX fallback:** Resulting in 404 errors every time a user refreshes the page on any route other than the root `/`.

### Architect Mistakes
* **Leaking Secrets:** Storing Stripe Secret Keys or database passwords in `environment.ts` or `config.json`. **Any file served by Angular is 100% visible to the user.** The frontend should only ever hold *public* keys (like the Stripe Publishable Key). Secrets belong strictly on the Backend API or inside an Azure Key Vault.

---

## 8. Final Words from the Author

Congratulations! You have completed **Mastering Angular: From Fundamentals to Enterprise SaaS Architecture**.

We have journeyed from the basics of Components and Dependency Injection, through the Reactive revolution of RxJS and Signals, into the depths of Enterprise State Management and Advanced Routing, and finally conquered the complexities of Performance Optimization, Security, and Cloud Native Deployments.

You are no longer just writing code. You are architecting enterprise systems.

**Now, go build something incredible.**
