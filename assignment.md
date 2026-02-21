# Assignment (Optional)

## Brief

Set up a complete CI/CD pipeline using CircleCI for your Spring Boot application with automated builds, tests, and Docker publishing.

1. **Create CircleCI Pipeline with Build and Test Jobs**
   - Use your Spring Boot project (devops-demo) with the `/hello` endpoint
   - Create a test file `DemoControllerTest.java` in `src/test/java/.../controller/`:
     - Use `@WebMvcTest` annotation
     - Test the `/hello` endpoint
     - Verify it returns status 200 OK
     - Verify it returns the expected message
   - Run the test locally with `mvn test` to ensure it passes
   - Push your project to a GitHub repository in your **personal account** (not organization)
   - Create `.circleci/config.yml` in your project with two jobs:
     - **build job**: 
       - Use `cimg/openjdk:21.0` image
       - Checkout code
       - Restore cache for Maven dependencies
       - Run `mvn clean install -DskipTests`
       - Save cache for Maven dependencies
       - Persist JAR file to workspace
     - **test job**:
       - Use `cimg/openjdk:21.0` image
       - Checkout code
       - Restore cache for Maven dependencies
       - Run `mvn test`
       - Store test results and artifacts
   - Create a workflow that runs build first, then test (only if build succeeds)
   - Sign up for CircleCI using your GitHub account
   - Connect CircleCI to your repository
   - Verify the pipeline runs successfully
   - Take screenshots showing both jobs passing

2. **Add Docker Publish Job to Pipeline**
   - Add a **publish job** to your `.circleci/config.yml`:
     - Use `cimg/base:stable` image
     - Checkout code
     - Attach workspace to get the JAR file
     - Setup remote Docker
     - Build Docker image: `docker build -t $DOCKER_USERNAME/devops-demo:latest .`
     - Login to Docker Hub and push the image
   - Update your workflow to run the publish job after test succeeds
   - In CircleCI, go to Project Settings → Environment Variables
   - Add two environment variables:
     - `DOCKER_USERNAME` (your Docker Hub username)
     - `DOCKER_PASSWORD` (your Docker Hub password)
   - Commit and push your updated config.yml
   - Verify the complete pipeline runs: build → test → publish
   - Check Docker Hub to confirm your image was pushed
   - Write a brief explanation (4-5 sentences) describing:
     - What each job in your pipeline does
     - Why tests run before publishing
     - How environment variables keep credentials secure
     - What would happen if the test job failed

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Java: https://docs.oracle.com/javase/
- Spring Boot: https://docs.spring.io/spring-boot/docs/current/reference/html/
- PostgreSQL: https://www.postgresql.org/docs/
- OWASP: https://cheatsheetseries.owasp.org/