# Lesson: Continuous Integration with CircleCI

---

## Learning Objectives

By the end of this lesson, learners will be able to:

1. **Explain** what Continuous Integration is and identify its key benefits in modern software development teams
2. **Configure** a CircleCI pipeline with build, test, and publish jobs for a Spring Boot application
3. **Implement** automated testing in a CI pipeline that triggers on every code commit
4. **Evaluate** CI pipeline results and troubleshoot common build failures

---

## Preparation

Before starting this lesson, ensure you have:

- Java 21 (already installed)
- Maven (already installed)
- Docker (already installed)
- [CircleCI Account](https://circleci.com/) - Create one using GitHub
- [Docker Hub Account](https://hub.docker.com) - Already created in Lesson 3
- GitHub account
- Your **devops-demo** Spring Boot project from previous lessons

---

## Lesson Overview

In this lesson, you will learn to automate your build, test, and deployment process using Continuous Integration. You will configure CircleCI to automatically build your Spring Boot application, run tests, and publish Docker images to Docker Hub whenever you push code to GitHub.

---

## Part 1 - Intro to Continuous Integration

### What is Continuous Integration (CI)

> Continuous Integration is a software development practice where members of a team integrate their work frequently, usually each person integrates at least daily - leading to multiple integrations per day. Each integration is verified by an automated build (including tests) to detect integration errors as quickly as possible. Many teams find that this approach leads to significantly reduced integration problems and allows a team to develop cohesive software more rapidly. - *Excerpt from [Martin Fowler](https://martinfowler.com/articles/continuousIntegration.html)*

### Why is Continuous Integration Needed? 

In the past, developers on a team might work in isolation for an extended period of time and only merge their changes to the main branch once their work was completed. This made merging code changes difficult and time-consuming, and also resulted in bugs accumulating for a long time without correction. These factors made it harder to deliver updates to customers quickly.

### The Benefits of Continuous Integration

- Improved code quality: CI enables developers to catch errors early in the development process, leading to better code quality and fewer bugs.
- Faster feedback: CI provides rapid feedback to developers, allowing them to quickly identify and fix issues.
- Faster time to market: CI enables teams to deliver high-quality software faster, reducing the time to market for new features.
- Improved collaboration: CI encourages collaboration between developers, enabling them to work together more effectively

### How does Continuous Integration Work? 

With continuous integration, developers frequently commit to a shared repository using a version control system such as Git. Prior to each commit, developers may choose to run local unit tests on their code as an extra verification layer before integrating. A continuous integration service automatically builds and runs unit tests on the new code changes to immediately surface any errors.

### CI Tools
For this module, we will be using CircleCI as our CI/CD workflow tool but there are other tools available online:
1. [GitHub Actions](https://github.com/features/actions)
2. [GitLab CI/CD](https://docs.gitlab.com/ee/ci/)
3. [Bamboo](https://www.atlassian.com/software/bamboo)
4. [Jenkins](https://www.jenkins.io/)

Each of these tools have their own advantages and disadvantages so it is up to the team to determine which CI/CD workflow tool to use.

### CI Pipeline

To achieve Continuous Integration, a CI Pipeline is created that would contain the list of jobs and workflows needed. 

A pipeline is a list of workflows that would be done in sequence. The workflow has a series of jobs that are defined by the developer. The pipeline can be automated such that any code changes done will trigger the pipeline and run the jobs.

There are typically three jobs in the CI Pipeline (can be more based on the organization's needs):
1. Build / compile
2. Run tests
3. Upon tests passed, publish container image to registry and proceed to the CD Pipeline

---

## Part 2 - Preparing Your DevOps Demo Project for CI

Before setting up CircleCI, we need to prepare your Spring Boot project with tests and verify all dependencies.

### Step 1: Verify Your pom.xml Has Test Dependencies

Open your `pom.xml` file and check if you have the `spring-boot-starter-test` dependency. It should look like this:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Test dependency - REQUIRED for JUnit tests -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**If you don't see `spring-boot-starter-test`, add it now.** This dependency includes JUnit and testing utilities.

### Step 2: Fix .dockerignore for CI

From Lesson 2, you may have a `.dockerignore` file that blocks all of `target/`. This will cause problems in CI because Docker needs to copy the JAR file.

**Update your `.dockerignore` to this:**

```
# Build output (but allow JAR files)
target/classes/
target/test-classes/
target/generated-sources/
target/maven-status/

# IDE files
.idea/
*.iml
.vscode/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Git
.git
.gitignore

# Logs
*.log
```

**Important:** Notice we exclude `target/classes/` but NOT `target/` itself. This allows `target/*.jar` to be copied by Docker.

### Step 3: Create Test Configuration

Create a file: `src/main/resources/application-test.properties`

Add this content:

```properties
# Test configuration - disables database requirement for web layer tests
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

This prevents database connection errors when running simple web layer tests.

### Step 4: Add a Simple Test to Your Project

Create this directory structure if it doesn't exist:
```
src/test/java/com/example/devopsdemo/controller/
```

Create a new file: `DemoControllerTest.java`

```java
package com.example.devopsdemo.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

/**
 * Test class for DemoController
 * This test verifies that the /hello endpoint returns the expected message
 */
@WebMvcTest(DemoController.class)
@TestPropertySource(locations = "classpath:application-test.properties")
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

**Key Points:**
- `@WebMvcTest` - Only tests the web layer (controllers)
- `@TestPropertySource` - Uses test configuration (no database needed)
- `mockMvc` - Simulates HTTP requests without starting full server

### Understanding the Test

Let's break down what this test does:

**1. Annotations:**
- `@WebMvcTest(DemoController.class)` - Tells Spring Boot to only load the web layer for testing, specifically the DemoController. This is much faster than loading the entire application context.
- `@TestPropertySource(locations = "classpath:application-test.properties")` - Loads test-specific configuration that disables database requirements
- `@Autowired` - Injects MockMvc, which simulates HTTP requests
- `@Test` - Marks this method as a test case that JUnit will run

**2. MockMvc:**
- `mockMvc` simulates HTTP requests without starting a full server
- This is faster than integration tests
- Perfect for testing controllers in isolation
- Allows you to verify HTTP status codes, response bodies, headers, etc.

**3. Test Logic:**
- `mockMvc.perform(get("/hello"))` - Simulates a GET request to the /hello endpoint
- `.andExpect(status().isOk())` - Verifies HTTP status is 200 OK
- `.andExpect(content().string("DevOps demo application is running!"))` - Verifies the response body matches the expected string exactly

### Step 5: Run the Test Locally

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
- Check `application-test.properties` exists
- Check test file is in correct package

### Step 6: Verify Dockerfile Exists

Make sure you have a `Dockerfile` in your project root from Lesson 2:

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

---

## Part 3 - Push Your Project to GitHub

CircleCI needs your code in a GitHub repository to run CI pipelines.

### Step 1: Create a New GitHub Repository

1. Go to [GitHub](https://github.com) and sign in
2. Click the **+** icon in the top-right corner
3. Click **New repository**
4. Repository name: `devops-demo`
5. Description: "DevOps demo Spring Boot application for CI/CD"
6. Choose **Public** (CircleCI free tier works best with public repos)
7. **Do NOT** initialize with README, .gitignore, or license
8. Click **Create repository**

### Step 2: Initialize Git in Your Project

Open Terminal in your **devops-demo** project directory.

**Initialize Git:**

```bash
git init
```

**Create .gitignore file:**

Create a file named `.gitignore` in your project root with this content:

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

## Part 4 - CircleCI Setup

### What is CircleCI?

CircleCI is a CI tool that simplifies parts of DevOps processes, letting engineering teams get to building products by allowing teams to build fully-automated pipelines, from testing to deployment.

### CircleCI Account Preparation and Setup

Step 1: Go to [circleci.com](https://circleci.com/)

Step 2: Click **Sign Up** and choose **Sign up with GitHub**

Step 3: Authorize CircleCI to access your GitHub account

Step 4: After signing in, you'll see your GitHub repositories

Step 5: Find your `devops-demo` repository and click **Set Up Project**

Step 6: CircleCI will ask if you want to use an existing config file or create a new one

Step 7: Select **"Use Existing Config"** (we'll create the config file next)

### Optional - CircleCI CLI

Before committing changes to Git, it is good to run the command `circleci config validate` to ensure the `config.yml` file has a valid configuration. Install CircleCI CLI [here](https://circleci.com/docs/local-cli/#installation).

---

## Part 5 - CI Configuration

### CircleCI's config.yml

The `config.yml` is a configuration file used to define and configure the pipeline for the project. It is stored in the `.circleci` directory of the project's repository and provides instructions to CircleCI on how to build, test, and deploy your code.

The `config.yml` consists of several parts that work together to define the pipeline. Here are some that will be encountered in the next few lessons.

1. **Jobs**: They describe specific tasks or steps in your CI/CD pipeline.
```yml
build:
  docker:
    - image: cimg/openjdk:21.0
  steps:
    - checkout
    - run: mvn clean install -DskipTests
```

2. **Workflows**: They define the sequence and dependencies of jobs within workflows. You can create complex workflows with multiple jobs and conditional logic.
```yml
workflows:
  build_test_workflow:
    jobs:
      - build
      - test:
          requires:
            - build
```

3. **Executors**: Specify the execution environment for jobs.
```yml
executors:
  java-executor:
    docker:
      - image: cimg/openjdk:21.0
```

4. **Environment variables**: Used to store sensitive data or configuration options

More can be found in the [documentation](https://circleci.com/docs/) of CircleCI.

### Step 1: Create the CircleCI Configuration File

In your **devops-demo** project, create this directory structure:

```
.circleci/
```

Inside `.circleci/`, create a file named `config.yml`

Your project structure should now look like:

```
devops-demo/
├── .circleci/
│   └── config.yml
├── src/
├── pom.xml
├── Dockerfile
└── docker-compose.yml
```

### Step 2: Start with Basic Configuration

Open `.circleci/config.yml` and add:

```yml
version: 2.1
```

This specifies the CircleCI configuration version.

---

## Part 6 - Building the CI Pipeline

We will create three jobs for our CI pipeline:
1. **Build** - Compile and package the application
2. **Test** - Run automated tests
3. **Publish** - Build and push Docker image to Docker Hub

Let's build these step by step.

---

### Job 1: The Build Job

In this job, we will:
1. Use a Java 21 Docker image
2. Check out code from GitHub
3. Install Maven dependencies
4. Build the application (without running tests yet)

Add this to your `config.yml`:

```yml
version: 2.1

jobs:
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
```

### Understanding the Build Job

Let's break down each part of this job in detail:

**1. Docker Image:**
```yml
docker:
  - image: cimg/openjdk:21.0
```
- `cimg/openjdk:21.0` is CircleCI's optimized Java 21 image
- Contains Java 21 JDK and Maven pre-installed
- Optimized for CircleCI with faster startup times
- This is the environment where all our build commands will run

**2. Checkout:**
```yml
- checkout
```
- Downloads your code from GitHub into the CircleCI environment
- This is the first step in almost every CircleCI job
- Without this, CircleCI wouldn't have your code to build!

**3. Cache Restoration:**
```yml
- restore_cache:
    keys:
      - maven-deps-{{ checksum "pom.xml" }}
      - maven-deps-
```
- Restores previously downloaded Maven dependencies from cache
- Speeds up builds significantly (avoids re-downloading dependencies every time)
- `{{ checksum "pom.xml" }}` creates a unique key based on pom.xml content
- If pom.xml changes, cache is invalidated and dependencies are re-downloaded
- If exact match not found, falls back to `maven-deps-` (partial match)
- **Why this matters:** Without caching, Maven would download all dependencies every time, making builds 5-10x slower!

**4. Build Command:**
```yml
- run:
    name: Install dependencies and build
    command: |
      echo "Building the application..."
      mvn clean install -DskipTests
```
- `mvn clean` - Removes old build artifacts from target/ directory
- `mvn install` - Downloads dependencies, compiles code, packages JAR file
- `-DskipTests` - Skips tests in this job (we'll run them separately in the test job)
- This separation allows us to see if the build fails vs if tests fail
- **Why separate build and test?** If build fails, we know it's a compilation error. If test fails, we know the code compiles but has bugs.

**5. Save Cache:**
```yml
- save_cache:
    paths:
      - ~/.m2
    key: maven-deps-{{ checksum "pom.xml" }}
```
- Saves downloaded Maven dependencies for future builds
- `~/.m2` is where Maven stores downloaded dependencies on Linux
- Next build will restore from this cache (much faster!)
- Cache is only updated if pom.xml changes

**6. Persist to Workspace:**
```yml
- persist_to_workspace:
    root: .
    paths:
      - target/*.jar
```
- Saves the built JAR file to share with other jobs
- The test and publish jobs can use this JAR without rebuilding
- This is different from caching - workspaces pass data between jobs in the same workflow
- **Key difference:** Cache persists across workflows; workspace only exists within one workflow run

---

### Job 2: The Test Job

In this job, we will:
1. Use the same Java 21 Docker image
2. Check out code
3. Restore the built JAR from the build job
4. Run automated tests

Add this job to your `config.yml`:

```yml
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
```

### Understanding the Test Job

Let's break down what makes this job important:

**1. Test Command:**
```yml
- run:
    name: Run tests
    command: |
      echo "Running tests..."
      mvn test
```
- `mvn test` runs all JUnit tests in the project
- Maven looks for test files in `src/test/java/`
- If any test fails, the job fails and the pipeline stops
- **Why this is critical:** This prevents broken code from reaching production

**2. Store Test Results:**
```yml
- store_test_results:
    path: target/surefire-reports
```
- CircleCI collects test results and displays them in the UI
- You can see which tests passed/failed without digging through logs
- Maven Surefire plugin generates these reports in XML format
- CircleCI parses these reports and shows nice summaries

**3. Store Artifacts:**
```yml
- store_artifacts:
    path: target/surefire-reports
```
- Stores test reports as downloadable files
- Useful for debugging test failures
- Artifacts persist even after the job finishes
- **Difference from test results:** Artifacts are raw files you can download; test results are parsed and displayed in UI

---

### Job 3: The Publish Job

In this job, we will:
1. Use a Docker-in-Docker image (to build Docker images)
2. Check out code
3. Attach the built JAR from the build job
4. Build a Docker image
5. Push the image to Docker Hub

Add this job to your `config.yml`:

```yml
  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker:
          version: 20.10.14
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
```

### Understanding the Publish Job

This job packages and publishes your application as a Docker image:

**1. Base Image:**
```yml
docker:
  - image: cimg/base:stable
```
- Uses a minimal base image (doesn't need Java since we're just building Docker images)
- We'll set up Docker separately using `setup_remote_docker`
- Keeps the job lightweight and fast

**2. Attach Workspace:**
```yml
- attach_workspace:
    at: .
```
- Retrieves the JAR file saved by the build job
- Puts it in `target/*.jar` so Dockerfile can copy it
- Without this step, the JAR wouldn't exist and Docker build would fail
- **This is why we persisted the workspace in the build job!**

**3. Setup Remote Docker:**
```yml
- setup_remote_docker:
    version: 20.10.14
```
- Enables Docker commands in CircleCI
- Required to build Docker images in CircleCI's environment
- Creates a separate Docker engine for security
- **Without this:** You'd get "Cannot connect to Docker daemon" errors

**4. Build Docker Image:**
```yml
- run:
    name: Build Docker image
    command: |
      docker build -t $DOCKER_USERNAME/devops-demo:latest .
```
- Uses your Dockerfile from Lesson 2
- Tags image as `USERNAME/devops-demo:latest`
- `$DOCKER_USERNAME` is an environment variable (we'll set this in the next section)
- The `.` means "use Dockerfile in current directory"

**5. Push to Docker Hub:**
```yml
- run:
    name: Push to Docker Hub
    command: |
      echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      docker push $DOCKER_USERNAME/devops-demo:latest
```
- Logs in to Docker Hub using credentials stored as environment variables
- `--password-stdin` reads password from standard input (more secure than command line argument)
- Pushes the image to your Docker Hub repository
- **Security note:** Never hardcode passwords! Always use environment variables
- `$DOCKER_PASSWORD` is an environment variable (we'll set this in the next section)
- Pushes the image to your Docker Hub repository
- `$DOCKER_PASSWORD` is an environment variable (we'll set this next)

---

### Step 3: Define the Workflow

Now tie all three jobs together using a workflow.

Add this at the end of your `config.yml`:

```yml
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
      - build              # Step 1: Build job runs first
      - test:              # Step 2: Test job runs after build
          requires:
            - build        # Only runs if build succeeds
      - publish:           # Step 3: Publish job runs after test
          requires:
            - test         # Only runs if test succeeds
```

**This creates a pipeline:**
```
Build → Test → Publish
```

If any job fails, the pipeline stops. This means:
- If build fails (compilation error), tests and publish never run
- If tests fail (bugs in code), publish never runs  
- Only if both build and tests succeed does the image get published

**Why this order matters:**
1. No point testing if code doesn't compile
2. No point publishing if tests fail
3. Only working, tested code reaches Docker Hub

---

### Complete config.yml File

Here's your complete `.circleci/config.yml`:

```yml
version: 2.1

jobs:
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

  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker:
          version: 20.10.14
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

## Part 7 - Setting Up Docker Hub Credentials in CircleCI

The publish job needs your Docker Hub username and password.

**Important:** Never hardcode passwords! Use environment variables.

### Add Environment Variables

1. Go to [CircleCI Dashboard](https://app.circleci.com/)
2. Click on your **devops-demo** project
3. Click **Project Settings** (top-right)
4. Click **Environment Variables** (left menu)
5. Click **Add Environment Variable**

**Add these two variables:**

| Name | Value |
|------|-------|
| `DOCKER_USERNAME` | Your Docker Hub username |
| `DOCKER_PASSWORD` | Your Docker Hub password |

CircleCI encrypts these values for security.

---

## Part 8 - Push Config and Trigger Pipeline

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
- ✅ Build job (compiles code)
- ✅ Test job (runs tests)
- ✅ Publish job (pushes to Docker Hub)

### Step 3: Verify on Docker Hub

1. Go to [Docker Hub](https://hub.docker.com/)
2. Check your `devops-demo` repository
3. You should see a new push with tag `latest`

**Success!** Your CI pipeline is working! 🎉

---

## Part 9 - Understanding the Complete Flow

Let's review what happens when you push code to GitHub:

```
1. Developer pushes code to GitHub
   └─> git push origin main

2. GitHub notifies CircleCI
   └─> "New commit detected!"

3. CircleCI starts the pipeline
   └─> Reads .circleci/config.yml

4. Build Job Runs
   ├─> Checks out code from GitHub
   ├─> Restores Maven cache (if available)
   ├─> Runs: mvn clean install -DskipTests
   ├─> Saves Maven cache for future builds
   └─> Saves JAR file to workspace
   
5. Test Job Runs (if build succeeds)
   ├─> Checks out code from GitHub
   ├─> Restores Maven cache
   ├─> Runs: mvn test
   ├─> Stores test results in CircleCI UI
   └─> Stores test reports as downloadable artifacts
   
6. Publish Job Runs (if test succeeds)
   ├─> Checks out code from GitHub
   ├─> Retrieves JAR from workspace
   ├─> Builds Docker image using Dockerfile
   ├─> Logs in to Docker Hub
   └─> Pushes image to Docker Hub

7. Image Available on Docker Hub
   └─> Anyone can: docker pull YOUR_USERNAME/devops-demo:latest
```

**Key Points:**
- Each job starts fresh in its own container
- Jobs share data through workspaces
- Cache speeds up repeated builds
- Pipeline stops at first failure

---

## Part 10 - Testing Your CI Pipeline

### Activity: Make a Change and Watch Pipeline

**Step 1:** Modify `DemoController.java`:

```java
@GetMapping("/hello")
public String hello() {
    return "DevOps demo - CI/CD is working!";
}
```

**Step 2:** Update the test:

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

Open `http://localhost:8080/hello` - you should see the new message!

---

## Troubleshooting Common Issues (Optional)

**Note to instructor:** This section can be used as reference material. Students can refer to it when they encounter issues.

### Issue 1: Test Fails Locally

**Error:** `ClassNotFoundException: org.postgresql.Driver`

**Solution:**
1. Check `spring-boot-starter-test` is in pom.xml
2. Check `application-test.properties` exists
3. Run: `mvn clean test`

---

### Issue 2: Docker Login Failed in CircleCI

**Error:** `Error response from daemon: Get https://registry-1.docker.io/v2/: unauthorized`

**Solution:**
1. Go to CircleCI → Project Settings → Environment Variables
2. Verify `DOCKER_USERNAME` and `DOCKER_PASSWORD` are correct
3. Make sure there are no extra spaces in values
4. Delete and re-add `DOCKER_PASSWORD` if needed
5. Retrigger pipeline (push a new commit)

---

### Issue 3: JAR File Not Found in Docker Build

**Error:** `COPY failed: file not found in build context`

**Cause:**
- Build job didn't persist the JAR
- Or publish job didn't attach workspace
- Or .dockerignore is blocking target/*.jar

**Solution:**
1. Check `.dockerignore` allows `target/*.jar`
2. Verify build job has:
```yml
- persist_to_workspace:
    root: .
    paths:
      - target/*.jar
```
3. Verify publish job has:
```yml
- attach_workspace:
    at: .
```

---

### Issue 4: "Cannot connect to Docker daemon"

**Error:** `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`

**Cause:**
Missing `setup_remote_docker` step in publish job.

**Solution:**
Verify your publish job has:
```yml
- setup_remote_docker:
    version: 20.10.14
```

---

### Issue 5: Pipeline doesn't trigger automatically

**Symptom:**
Pushing to GitHub doesn't start the pipeline.

**Solution:**
1. Go to CircleCI → Project Settings → Advanced
2. Check "Enable GitHub Checks" is enabled
3. Go to GitHub → Repository → Settings → Webhooks
4. Verify CircleCI webhook exists and shows recent deliveries
5. Try pushing a new commit to trigger the pipeline

---

## Activity - Group Discussion (10 minutes)

**Instructor:** Lead a brief discussion with these questions:

1. **Why do we have separate build and test jobs?**
   - Hint: What happens if the build fails vs if tests fail?
   - Answer: Helps us identify if it's a compilation error (build) or a logic error (test)

2. **What is the purpose of caching Maven dependencies?**
   - Hint: How long would builds take without caching?
   - Answer: Dramatically speeds up builds (5-10x faster) by not re-downloading dependencies

3. **Why do we use environment variables for Docker credentials?**
   - Hint: What would happen if we hardcoded the password in config.yml?
   - Answer: Security! Passwords in code can be leaked. Environment variables are encrypted.

4. **What happens if the publish job fails?**
   - Hint: Does the old image stay on Docker Hub?
   - Answer: Yes, old image remains. No broken images reach Docker Hub.

---

## Best Practices for CI Pipelines (Self Study)

### 1. Keep Tests Fast
- Fast tests = faster feedback to developers
- Aim for test job to complete in under 2 minutes
- Use unit tests in CI; save integration tests for CD pipeline
- Slow tests discourage frequent commits

### 2. Use Caching Wisely
- Cache Maven dependencies to speed up builds
- Cache Docker layers when possible
- Can reduce build time from 10 minutes to 2 minutes
- Cache invalidates when pom.xml changes

### 3. Fail Fast
- Run tests early in the pipeline
- Don't waste time building Docker images if tests fail
- Faster feedback = happier developers

### 4. Use Environment Variables for Secrets
- Never hardcode credentials in config files
- Environment variables are encrypted by CircleCI
- Makes config reusable across projects
- Passwords in Git history are permanent security risks

### 5. Monitor Your Pipelines
- Set up notifications (email, Slack) for build failures
- Review failed builds promptly
- Keep pipeline "green" (passing)
- A red pipeline becomes invisible - developers ignore it

### 6. Version Your Docker Images
- Don't rely only on `latest` tag
- Use commit SHA or version numbers
- Enables rollback to specific versions
- Better traceability in production

---

## Summary

You have successfully:
- ✅ Created automated CI pipeline with CircleCI
- ✅ Configured build, test, and publish jobs
- ✅ Added automated JUnit tests to Spring Boot application
- ✅ Set up automatic Docker image publishing to Docker Hub
- ✅ Tested the complete workflow from code commit to Docker Hub

**Key Takeaway:** Every code push now automatically builds, tests, and publishes your application - this is Continuous Integration in action!

---

## Assignment

Modify your CI pipeline to tag Docker images with commit SHA.

**Task:** Update the publish job to use `$CIRCLE_SHA1` instead of just `latest`.

**Hint:** Replace this line:
```bash
docker build -t $DOCKER_USERNAME/devops-demo:latest .
```

With:
```bash
docker build -t $DOCKER_USERNAME/devops-demo:$CIRCLE_SHA1 .
docker push $DOCKER_USERNAME/devops-demo:$CIRCLE_SHA1
```

**Test:** Push your changes and check Docker Hub for the new tag (will be a long SHA like `abc123def456...`).

---

END