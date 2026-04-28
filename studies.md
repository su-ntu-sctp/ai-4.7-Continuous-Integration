# 4.7 Self Studies

**Estimated Preparation Time:** 70 minutes

---

> **📌 Note:** The videos in these self studies are selected to help you build familiarity with the tools and concepts before class. The approach, project structure, and commands used in the videos may differ from what we do in the lesson. Focus on understanding the **concepts and the why** — not on replicating the video step by step. The lesson content is what you should follow during class.

---

## Task 1 — CircleCI with GitHub (25 minutes)

Watch the following video on getting started with CircleCI:

- 📹 [CircleCI with GitHub — https://www.youtube.com/watch?v=Cev6oIG-t4o]

While watching, refer to **lesson.md Part 7 and Part 5** and pay attention to:
- How CircleCI connects to a GitHub repository
- How the `.circleci/config.yml` file is structured
- What jobs and workflows are and how they relate to each other
- How CircleCI triggers a pipeline automatically when code is pushed to GitHub

**Guiding Questions:**
1. What file does CircleCI look for in your repository to know how to run the pipeline?
2. How does CircleCI know when to trigger a new pipeline run?
3. What is the difference between a job and a workflow in CircleCI?

---

## Task 2 — Understanding CI Pipeline Jobs (20 minutes)

No video for this task. Refer to **lesson.md Part 1 and Part 5** and read through the three pipeline jobs (build, test, publish) carefully, then answer the following:

1. Why is the build job run before the test job — why not run them at the same time?
2. The publish job uses `cimg/base:stable` instead of `cimg/openjdk:21.0`. Why is that?
3. What does `persist_to_workspace` do in the build job, and why does the publish job need `attach_workspace`?
4. Why does the publish job use `echo $DOCKER_PASSWORD | docker login --password-stdin` instead of typing the password directly?

**Guiding Questions:**
1. What happens to the publish job if the test job fails?
2. Why is Maven dependency caching important in a CI environment?
3. What is the purpose of `store_test_results` in the test job?

---

## Task 3 — Prepare Your Project for CI (25 minutes)

No video for this task. Refer to **lesson.md Part 2 and Part 3** and complete the following steps on your local machine before class:

1. Remove `docker-compose.yml` from your devops-demo project
2. Remove PostgreSQL and JPA dependencies from `pom.xml`
3. Clean up `application.properties` — remove all database config
4. Verify your `DemoController.java` has no database dependencies
5. Create `DemoControllerTest.java` in the correct package
6. Run the test locally to confirm it passes:
```bash
mvn test
```

Come to class with a working, simplified project and a passing test. This will save significant time during the lesson.

**Guiding Questions:**
1. Why do we remove the database dependencies before setting up CI?
2. What does `@WebMvcTest(DemoController.class)` do differently from loading the full application context?
3. Why is it important to run tests locally before pushing to GitHub?

---

## Active Engagement Strategies

- Pause the video when the `.circleci/config.yml` is shown and compare its structure to the one in lesson.md
- After watching, try to explain the CI workflow (code push → pipeline trigger → jobs → feedback) in your own words without looking at the lesson
- For Task 3, complete the project cleanup and run the test before class — do not leave it for the lesson

---

## Additional Reading Material

- [What is CI/CD? — Red Hat](https://www.redhat.com/en/topics/devops/what-is-ci-cd)
- [Martin Fowler on Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)
- [CircleCI Getting Started — CircleCI Docs](https://circleci.com/docs/getting-started/)
- [CircleCI config.yml Reference](https://circleci.com/docs/configuration-reference/)