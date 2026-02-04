# Lesson 4.7: Continuous Integration with CircleCI

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Explain** what Continuous Integration is and identify its key benefits in modern software development teams
2. **Configure** a CircleCI pipeline with build, test, and publish jobs for a Spring Boot application
3. **Implement** automated testing in a CI pipeline that triggers on every code commit
4. **Evaluate** CI pipeline results and troubleshoot common build failures

---

## Prerequisites

Before starting this lesson, ensure you have:

- Java 21 (already installed)
- Maven (already installed)
- Docker (already installed)
- [CircleCI Account](https://circleci.com/) - Create one using GitHub
- [Docker Hub Account](https://hub.docker.com) - Already created in Lesson 4.3
- GitHub account
- Your **devops-demo** Spring Boot project from previous lessons

---

## Introduction

In this lesson, you will learn to automate your build, test, and deployment process using Continuous Integration. You will configure CircleCI to automatically build your Spring Boot application, run tests, and publish Docker images to Docker Hub whenever you push code to GitHub.

---

## Part 1 - Understanding Continuous Integration (20 minutes)

### What is Continuous Integration (CI)?

> Continuous Integration is a software development practice where members of a team integrate their work frequently, usually each person integrates at least daily - leading to multiple integrations per day. Each integration is verified by an automated build (including tests) to detect integration errors as quickly as possible. Many teams find that this approach leads to significantly reduced integration problems and allows a team to develop cohesive software more rapidly. - *Excerpt from [Martin Fowler](https://martinfowler.com/articles/continuousIntegration.html)*

### Why is Continuous Integration Needed?

In the past, developers on a team might work in isolation for an extended period of time and only merge their changes to the main branch once their work was completed. This made merging code changes difficult and time-consuming, and also resulted in bugs accumulating for a long time without correction. These factors made it harder to deliver updates to customers quickly.

**The Problem:**
- Developers work on features for weeks/months
- When they merge, massive conflicts occur
- Integration becomes painful
- Bugs hide in code for long periods
- Deploying updates is risky and slow

**The Solution: Continuous Integration**
- Integrate code frequently (multiple times per day)
- Automate builds and tests
- Catch bugs immediately
- Small changes = easier to debug
- Always have working code ready to deploy

### The Benefits of Continuous Integration

**1. Improved Code Quality** - Catch errors early, automated tests run on every commit, fewer bugs reach production

**2. Faster Feedback** - Know within minutes if your code broke something, don't wait days to discover issues

**3. Faster Time to Market** - Deploy multiple times per day instead of monthly, competitive advantage

**4. Improved Collaboration** - Shared responsibility for code quality, everyone sees build status

### How Does Continuous Integration Work?

With continuous integration, developers frequently commit to a shared repository using a version control system such as Git. Prior to each commit, developers may choose to run local unit tests on their code as an extra verification layer before integrating. A continuous integration service automatically builds and runs unit tests on the new code changes to immediately surface any errors.

**The CI Workflow:**

```
1. Developer writes code
   ↓
2. Developer commits code to Git
   ↓
3. Developer pushes to GitHub
   ↓
4. GitHub notifies CI server (CircleCI)
   ↓
5. CI server pulls code
   ↓
6. CI server builds application
   ↓
7. CI server runs automated tests
   ↓
8. CI server publishes build artifacts (Docker image)
   ↓
9. Developer receives feedback (pass/fail)
```

### CI Tools

For this module, we will be using **CircleCI** but other popular CI tools include:
- GitHub Actions, GitLab CI/CD, Jenkins, Azure DevOps

Each tool has advantages and disadvantages. Teams choose based on their needs.

### CI Pipeline Components

To achieve Continuous Integration, a **CI Pipeline** is created containing jobs and workflows.

**Pipeline:** A sequence of jobs that run automatically when code changes

**Workflow:** Defines which jobs run and in what order

**Jobs:** Individual tasks (build, test, publish)

**A typical CI Pipeline has three jobs:**

1. **Build / Compile**
   - Download dependencies
   - Compile source code
   - Package application (create JAR file)

2. **Test**
   - Run unit tests
   - Run integration tests
   - Verify code quality

3. **Publish**
   - Build Docker image
   - Push to container registry (Docker Hub)
   - Make artifact available for deployment

**If any job fails, the pipeline stops.** This prevents broken code from reaching production.

---

## Part 2 - Preparing Your DevOps Demo Project for CI (20 minutes)

Before setting up CircleCI, we need to prepare your Spring Boot project with tests and verify all dependencies.

### Step 1: Verify Your pom.xml

Open your `pom.xml` file and check you have these dependencies:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>devops-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>devops-demo</name>
    <description>DevOps demo application</description>
    
    <properties>
        <java.version>21</java.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Web Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Test dependency - REQUIRED for JUnit tests -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**Key points:**
- ✅ Spring Boot 3.2.0
- ✅ Java 21
- ✅ spring-boot-starter-web (for REST endpoints)
- ✅ spring-boot-starter-test (for JUnit tests)

**That's it!** Simple app, simple dependencies.

### Step 2: Verify Your DemoController

Make sure you have `src/main/java/com/example/devopsdemo/controller/DemoController.java`:

```java
package com.example.devopsdemo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Simple REST controller for DevOps demo
 */
@RestController
public class DemoController {

    /**
     * Hello endpoint - returns a simple message
     * This endpoint will be tested by our CI pipeline
     */
    @GetMapping("/hello")
    public String hello() {
        return "DevOps demo application is running!";
    }
}
```

**Simple app with one endpoint: `/hello`**

### Step 3: Add a Test for Your Controller

Create this directory structure if it doesn't exist:
```
src/test/java/com/example/devopsdemo/controller/
```

Create file: `DemoControllerTest.java`

```java
package com.example.devopsdemo.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

/**
 * Test class for DemoController
 * This test verifies that the /hello endpoint returns the expected message
 */
@WebMvcTest(DemoController.class)
public class DemoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    /**
     * Test that the /hello endpoint returns status 200 OK
     * and the correct message
     */
    @Test
    public void testHelloEndpoint() throws Exception {
        // Perform GET request to /hello
        mockMvc.perform(get("/hello"))
                // Verify response status is 200 OK
                .andExpect(status().isOk())
                // Verify response body matches expected text
                .andExpect(content().string("DevOps demo application is running!"));
    }
}
```

### Understanding the Test

Let's break down what this test does:

**1. Annotations:**
- `@WebMvcTest(DemoController.class)` - Tells Spring Boot to only load the web layer for testing, specifically the DemoController. This is much faster than loading the entire application.
- `@Autowired` - Injects MockMvc for simulating HTTP requests
- `@Test` - Marks this method as a JUnit test case

**2. MockMvc:**
- `mockMvc` simulates HTTP requests without starting a full server
- This is faster than integration tests
- Perfect for testing controllers
- Allows verification of HTTP status codes, response bodies, headers, etc.

**3. Test Logic:**
- `mockMvc.perform(get("/hello"))` - Simulates a GET request to /hello
- `.andExpect(status().isOk())` - Verifies HTTP status is 200 OK
- `.andExpect(content().string("..."))` - Verifies response body matches exactly

**Why this test matters:**
- Ensures your endpoint works correctly
- CI will run this on every code push
- If someone breaks the endpoint, CI will catch it immediately

### Step 4: Run the Test Locally

Verify the test works before pushing to GitHub.

```bash
mvn test
```

**Expected output:**

```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.devopsdemo.controller.DemoControllerTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] BUILD SUCCESS
```

**If you see `BUILD SUCCESS`, your test is working!** ✅

**If you see errors:**
- Check `spring-boot-starter-test` is in pom.xml
- Check test file is in correct package: `com.example.devopsdemo.controller`
- Check class name matches: `DemoControllerTest.java`
- Run `mvn clean test` to clear old builds

### Step 5: Verify Your Dockerfile

Make sure you have a `Dockerfile` in your project root from Lesson 4.2:

```bash
ls -la Dockerfile
```

If missing, create it now:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
ENV PORT=8080
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

**This Dockerfile:**
- Uses Java 21 Alpine image (lightweight)
- Copies the JAR file built by Maven
- Exposes port 8080
- Runs the Spring Boot application

---

## Part 3 - Push Your Project to GitHub (15 minutes)

CircleCI needs your code in a GitHub repository to run CI pipelines.

### Step 1: Create a New GitHub Repository

1. Go to [GitHub](https://github.com) and sign in
2. Click the **+** icon in the top-right corner
3. Click **New repository**
4. Fill in details:
   - **Repository name:** `devops-demo`
   - **Description:** "DevOps demo Spring Boot application for CI/CD"
   - **Public** (CircleCI free tier works best with public repos)
   - **Do NOT** initialize with README, .gitignore, or license
5. Click **Create repository**

### Step 2: Initialize Git in Your Project

Open Terminal in your **devops-demo** project directory.

**Initialize Git:**

```bash
git init
```

**Create .gitignore file:**

Create a file named `.gitignore` in your project root:

```
# Maven
target/
!.mvn/wrapper/maven-wrapper.jar
!**/src/main/**/target/
!**/src/test/**/target/

# IDE
.idea/
*.iml
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

This tells Git to ignore build artifacts and IDE files.

### Step 3: Add and Commit Your Code

```bash
# Add all files
git add .

# Commit with a message
git commit -m "Initial commit: Spring Boot DevOps demo with test"
```

### Step 4: Push to GitHub

Copy the commands from your GitHub repository page (they look like this):

```bash
git remote add origin https://github.com/YOUR_USERNAME/devops-demo.git
git branch -M main
git push -u origin main
```

**Replace `YOUR_USERNAME` with your actual GitHub username.**

After running these commands, refresh your GitHub repository page. You should see your code!

---

## Part 4 - CircleCI Setup (15 minutes)

### What is CircleCI?

CircleCI is a cloud-based CI/CD platform that automates your build, test, and deployment processes. It integrates seamlessly with GitHub and runs your pipeline whenever you push code.

**Why CircleCI?**
- ✅ Easy setup (connects to GitHub in minutes)
- ✅ Free tier for learning
- ✅ Cloud-based (no server to maintain)
- ✅ Fast builds with parallelization
- ✅ Great documentation

### Create CircleCI Account

**Step 1:** Go to [circleci.com](https://circleci.com/)

**Step 2:** Click **Sign Up** and choose **Sign up with GitHub**

**Step 3:** Authorize CircleCI to access your GitHub account

**Step 4:** After signing in, you'll see your GitHub repositories

**Step 5:** Find your `devops-demo` repository and click **Set Up Project**

**Step 6:** CircleCI will ask if you want to use an existing config file or create a new one

**Step 7:** Select **"Use Existing Config"** (we'll create the config file next)

---

## Part 5 - Building the CI Pipeline (50 minutes)

We will create three jobs for our CI pipeline:
1. **Build** - Compile and package the application
2. **Test** - Run automated tests
3. **Publish** - Build and push Docker image to Docker Hub

### CircleCI's config.yml

The `config.yml` tells CircleCI what to do, when, and where. It has:
- **Jobs** - Specific tasks (build, test, publish)
- **Workflows** - Defines sequence and dependencies
- **Executors** - Execution environment (Docker images)

**Location:** `.circleci/config.yml` (must be in this exact location)

Let's build the pipeline step by step.

### Step 1: Create the CircleCI Configuration File (5 minutes)

In your **devops-demo** project, create this directory:

```bash
mkdir .circleci
```

Inside `.circleci/`, create a file named `config.yml`

Your project structure should now look like:

```
devops-demo/
├── .circleci/
│   └── config.yml
├── src/
│   ├── main/
│   └── test/
├── pom.xml
├── Dockerfile
└── .gitignore
```

### Step 2: Start with Version Declaration

Open `.circleci/config.yml` and add:

```yml
version: 2.1
```

This specifies the CircleCI configuration version (2.1 is the latest).

---

### Job 1: The Build Job (20 minutes)

In this job, we will:
1. Use a Java 21 Docker image
2. Check out code from GitHub
3. Cache Maven dependencies (speeds up builds)
4. Build the application
5. Save the JAR file for other jobs

Add this to your `config.yml`:

```yml
version: 2.1

jobs:
  # ==========================================
  # BUILD JOB - Compile and package
  # ==========================================
  build:
    # Use CircleCI's Java 21 Docker image
    docker:
      - image: cimg/openjdk:21.0
    
    steps:
      # Step 1: Download code from GitHub
      - checkout
      
      # Step 2: Restore Maven dependencies from cache (if available)
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      
      # Step 3: Build the application
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      
      # Step 4: Save Maven dependencies to cache for next build
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      
      # Step 5: Save JAR file for other jobs to use
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar
```

### Understanding the Build Job

**Docker Image:**
```yml
docker:
  - image: cimg/openjdk:21.0
```
- CircleCI's optimized Java 21 image
- Contains Java 21 JDK and Maven pre-installed
- All commands run inside this container

**Checkout:**
```yml
- checkout
```
- Downloads your code from GitHub
- First step in almost every CircleCI job

**Cache Restoration:**
```yml
- restore_cache:
    keys:
      - maven-deps-{{ checksum "pom.xml" }}
      - maven-deps-
```
- Restores previously downloaded Maven dependencies
- `{{ checksum "pom.xml" }}` creates unique key based on pom.xml content
- If pom.xml changes, cache is invalidated
- **Without caching:** Builds take 5-10x longer!

**Build Command:**
```yml
- run:
    name: Install dependencies and build
    command: |
      mvn clean install -DskipTests
```
- `mvn clean` - Removes old build artifacts
- `mvn install` - Downloads dependencies, compiles code, packages JAR
- `-DskipTests` - Skips tests (we run them in separate job)

**Save Cache:**
```yml
- save_cache:
    paths:
      - ~/.m2
    key: maven-deps-{{ checksum "pom.xml" }}
```
- Saves downloaded dependencies for future builds
- `~/.m2` is where Maven stores dependencies
- Next build restores from this cache (much faster!)

**Persist to Workspace:**
```yml
- persist_to_workspace:
    root: .
    paths:
      - target/*.jar
```
- Saves JAR file to share with other jobs
- Test and publish jobs can use this JAR without rebuilding

---

### Job 2: The Test Job (20 minutes)

In this job, we will:
1. Use the same Java 21 Docker image
2. Check out code
3. Restore cached dependencies
4. Run automated tests
5. Store test results

Add this job to your `config.yml`:

```yml
  # ==========================================
  # TEST JOB - Run automated tests
  # ==========================================
  test:
    # Use CircleCI's Java 21 Docker image
    docker:
      - image: cimg/openjdk:21.0
    
    steps:
      # Step 1: Download code from GitHub
      - checkout
      
      # Step 2: Restore Maven dependencies from cache
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      
      # Step 3: Run tests
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      
      # Step 4: Store test results for CircleCI UI
      - store_test_results:
          path: target/surefire-reports
      
      # Step 5: Store test reports as downloadable artifacts
      - store_artifacts:
          path: target/surefire-reports
```

### Understanding the Test Job

**Test Command:**
```yml
- run:
    name: Run tests
    command: |
      mvn test
```
- Runs all JUnit tests in `src/test/java/`
- If any test fails, job fails and pipeline stops
- **Critical:** Prevents broken code from reaching production

**Store Test Results:**
```yml
- store_test_results:
    path: target/surefire-reports
```
- CircleCI parses test results and displays in UI
- Shows which tests passed/failed
- Maven Surefire plugin generates these reports in XML format

**Store Artifacts:**
```yml
- store_artifacts:
    path: target/surefire-reports
```
- Stores test reports as downloadable files
- Useful for debugging test failures
- Persists even after job finishes

---

### Job 3: The Publish Job (20 minutes)

In this job, we will:
1. Use a Docker-capable image
2. Get the JAR file from build job
3. Build a Docker image
4. Push the image to Docker Hub

Add this job to your `config.yml`:

```yml
  # ==========================================
  # PUBLISH JOB - Build and push Docker image
  # ==========================================
  publish:
    # Use base image with Docker support
    docker:
      - image: cimg/base:stable
    
    steps:
      # Step 1: Download code from GitHub
      - checkout
      
      # Step 2: Get JAR file from build job
      - attach_workspace:
          at: .
      
      # Step 3: Setup Docker (required for building images)
      - setup_remote_docker:
          version: 20.10.14
      
      # Step 4: Build Docker image
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/devops-demo:latest .
      
      # Step 5: Login and push to Docker Hub
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/devops-demo:latest
```

### Understanding the Publish Job

**Base Image:**
```yml
docker:
  - image: cimg/base:stable
```
- Minimal base image (doesn't need Java)
- We'll set up Docker separately

**Attach Workspace:**
```yml
- attach_workspace:
    at: .
```
- Retrieves JAR file saved by build job
- Puts it in `target/*.jar` so Dockerfile can copy it
- Without this, Docker build would fail!

**Setup Remote Docker:**
```yml
- setup_remote_docker
    
```
- Enables Docker commands in CircleCI
- Required to build Docker images
- Creates separate Docker engine for security

**Build Docker Image:**
```yml
- run:
    name: Build Docker image
    command: |
      docker build -t $DOCKER_USERNAME/devops-demo:latest .
```
- Uses your Dockerfile
- Tags image as `USERNAME/devops-demo:latest`
- `$DOCKER_USERNAME` is environment variable (set in next section)

**Push to Docker Hub:**
```yml
- run:
    name: Push to Docker Hub
    command: |
      echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      docker push $DOCKER_USERNAME/devops-demo:latest
```
- Logs in to Docker Hub using environment variables
- `--password-stdin` is more secure than command line argument
- Pushes image to Docker Hub
- **Security:** Never hardcode passwords! Always use environment variables

---

### Step 3: Define the Workflow (10 minutes)

Now tie all three jobs together using a workflow.

Add this at the end of your `config.yml`:

```yml
# ==========================================
# WORKFLOW - Run jobs in sequence
# ==========================================
workflows:
  build_test_publish:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
```

### Understanding the Workflow

```yml
workflows:
  build_test_publish:    # Workflow name
    jobs:
      - build              # Step 1: Build runs first
      - test:              # Step 2: Test runs after build
          requires:
            - build        # Only if build succeeds
      - publish:           # Step 3: Publish runs after test
          requires:
            - test         # Only if test succeeds
```

**This creates a pipeline:**
```
Build → Test → Publish
```

**If any job fails, pipeline stops:**
- Build fails → Test and Publish don't run
- Test fails → Publish doesn't run
- Only if both succeed → Image gets published

---

### Complete config.yml File

Here's your complete `.circleci/config.yml`:

```yml
version: 2.1

jobs:
  # ==========================================
  # BUILD JOB - Compile and package
  # ==========================================
  build:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar

  # ==========================================
  # TEST JOB - Run automated tests
  # ==========================================
  test:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      - store_test_results:
          path: target/surefire-reports
      - store_artifacts:
          path: target/surefire-reports

  # ==========================================
  # PUBLISH JOB - Build and push Docker image
  # ==========================================
  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/devops-demo:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/devops-demo:latest

# ==========================================
# WORKFLOW - Run jobs in sequence
# ==========================================
workflows:
  build_test_publish:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
```

---

## Part 6 - Setting Up Docker Hub Credentials in CircleCI (10 minutes)

The publish job needs your Docker Hub username and password.

**Important:** Never hardcode passwords! Use environment variables.

### Add Environment Variables

1. Go to [CircleCI Dashboard](https://app.circleci.com/)
2. Click on your **devops-demo** project
3. Click **Project Settings** (top-right)
4. Click **Environment Variables** (left menu)
5. Click **Add Environment Variable**

**Add these two variables:**

**Variable 1:**
- **Name:** `DOCKER_USERNAME`
- **Value:** Your Docker Hub username (e.g., `john123`)
- Click **Add Environment Variable**

**Variable 2:**
- **Name:** `DOCKER_PASSWORD`
- **Value:** Your Docker Hub password
- Click **Add Environment Variable**

**Why environment variables?**
- ✅ Secure (CircleCI encrypts them)
- ✅ Not visible in code or logs
- ✅ Easy to update without changing code
- ✅ Same config works for different users

---

## Part 7 - Push Config and Trigger Pipeline (20 minutes)

### Step 1: Commit and Push

In your terminal:

```bash
# Check status
git status

# Add all files
git add .

# Commit
git commit -m "Add CircleCI configuration"

# Push to GitHub
git push origin main
```

### Step 2: Watch the Pipeline

1. Go to [CircleCI Dashboard](https://app.circleci.com/)
2. Your pipeline should start automatically!
3. Click on the workflow to see job progress

**You'll see:**
- ⏳ Build job (compiling...)
- ⏳ Test job (running tests...)
- ⏳ Publish job (pushing to Docker Hub...)

**Timeline:**
- Build: ~2-3 minutes (first time, with caching: ~1 minute)
- Test: ~1-2 minutes
- Publish: ~2-3 minutes

**Total: ~5-8 minutes**

### Step 3: Verify on Docker Hub

1. Go to [Docker Hub](https://hub.docker.com/)
2. Login
3. Check your `devops-demo` repository
4. You should see a new push with tag `latest`

**Success!** Your CI pipeline is working! 🎉

**What just happened?**
1. You pushed code → GitHub notified CircleCI
2. Build job: Compiled code, saved JAR
3. Test job: Ran tests (if build succeeded)
4. Publish job: Built Docker image, pushed to Docker Hub (if tests passed)

**Key takeaway:** Pipeline stops at first failure - only working, tested code reaches Docker Hub!

---

## Part 8 - Testing Your CI Pipeline (10 minutes)

### Make a Change and Watch Pipeline

**Step 1:** Modify `DemoController.java`:

```java
@GetMapping("/hello")
public String hello() {
    return "DevOps demo - CI/CD is working!";
}
```

**Step 2:** Update the test to match:

```java
.andExpect(content().string("DevOps demo - CI/CD is working!"));
```

**Step 3:** Test locally:

```bash
mvn test
```

**Step 4:** Push to GitHub:

```bash
git add .
git commit -m "Update hello message"
git push origin main
```

**Step 5:** Watch CircleCI run the pipeline again!

**Step 6:** Pull and test the new image:

```bash
docker pull YOUR_USERNAME/devops-demo:latest
docker run -p 8080:8080 YOUR_USERNAME/devops-demo:latest
```

Open `http://localhost:8080/hello` - you should see: "DevOps demo - CI/CD is working!"

---

## Troubleshooting Guide

### Issue 1: Test Fails Locally

**Error:** Tests don't pass when running `mvn test`

**Solution:**
1. Check `spring-boot-starter-test` is in pom.xml
2. Verify test file is in correct package
3. Run: `mvn clean test` to clear old builds
4. Check test matches controller message exactly

---

### Issue 2: Docker Login Failed in CircleCI

**Error:** `unauthorized: incorrect username or password`

**Solution:**
1. Go to CircleCI → Project Settings → Environment Variables
2. Verify `DOCKER_USERNAME` and `DOCKER_PASSWORD` are correct
3. Make sure there are no extra spaces
4. Delete and re-add `DOCKER_PASSWORD` if needed
5. Push a new commit to retrigger pipeline

---

### Issue 3: JAR File Not Found in Docker Build

**Error:** `COPY failed: file not found in build context`

**Cause:** Build job didn't persist JAR, or publish job didn't attach workspace

**Solution:**
1. Verify build job has:
```yml
- persist_to_workspace:
    root: .
    paths:
      - target/*.jar
```
2. Verify publish job has:
```yml
- attach_workspace:
    at: .
```

---

### Issue 4: "Cannot connect to Docker daemon"

**Error:** `Cannot connect to the Docker daemon`

**Cause:** Missing `setup_remote_docker` step

**Solution:**
Verify publish job has:
```yml
- setup_remote_docker:
    version: 20.10.14
```

---

### Issue 5: Pipeline doesn't trigger automatically

**Symptom:** Pushing to GitHub doesn't start pipeline

**Solution:**
1. Go to CircleCI → Project Settings → Advanced
2. Check "Only build pull requests" is disabled
3. Go to GitHub → Repository → Settings → Webhooks
4. Verify CircleCI webhook exists
5. Try pushing a new commit

---

## Summary

### What You Accomplished Today

1. ✅ Created automated CI pipeline with CircleCI
2. ✅ Configured build, test, and publish jobs
3. ✅ Added automated JUnit tests to Spring Boot application
4. ✅ Set up automatic Docker image publishing to Docker Hub
5. ✅ Tested the complete workflow from code commit to Docker Hub

### Key Takeaways

**1. CI Automates Everything:**
- Every code push triggers build, test, publish
- No manual steps
- Fast feedback (5-8 minutes)

**2. Tests Prevent Broken Code:**
- Tests must pass before publishing
- Broken code never reaches Docker Hub
- Catch bugs immediately

**3. Caching Speeds Up Builds:**
- Maven dependencies cached
- First build: 8 minutes
- Subsequent builds: 3-4 minutes

**4. Environment Variables = Security:**
- Never hardcode passwords
- CircleCI encrypts secrets
- Easy to update credentials

---

## Additional Resources

### CI/CD Concepts
- [Martin Fowler on Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)
- [What is CI/CD?](https://www.redhat.com/en/topics/devops/what-is-ci-cd)

### CircleCI Documentation
- [CircleCI Docs](https://circleci.com/docs/)
- [Java/Maven Guide](https://circleci.com/docs/language-java-maven/)
- [CircleCI Orbs](https://circleci.com/developer/orbs)

### Video Tutorials
- [CircleCI Tutorial for Beginners](https://www.youtube.com/results?search_query=circleci+tutorial)
- [CI/CD Explained](https://www.youtube.com/watch?v=scEDHsr3APg)

---

**End of Lesson 4.7**

**Congratulations!** You've successfully set up a complete CI pipeline. Every code push now automatically builds, tests, and publishes your application. This is the foundation of modern DevOps practices! 🎉

**Next Lesson:** Lesson 4.8 - Saturday Coaching (Apply Docker to simple-crm-lite)