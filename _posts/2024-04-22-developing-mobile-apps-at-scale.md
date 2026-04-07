---
layout: post
title: "Developing Mobile Apps at Scale: 20 Rules for Large Engineering Orgs and Startups"
date: 2024-04-22
tags: [android, ios, architecture, process, scale]
---

Recently, without any specific reason, I was asked how to build an app or a software system at scale. My initial response was to suggest using lambdas and autoscaling groups to accommodate the exponential growth of customers. However, I soon realized that the question was about building a system for large organizations, where hundreds of engineers work together in cohesion without interfering with each other's work. Having worked in a small startup for the past few years, I almost forgot about this challenge. It was only after the conversation ended that I recalled all the lessons learned from working in a big company with a great engineering culture.

So, I decided to write down the rules and best practices that I remember and turn my writing into this article to use it as a reference for myself and, hopefully, help others as well.

This article will mainly focus on Android app development at scale, with some references to iOS app development. However, most of the points discussed here are applicable to software development in general.

In short, the development of any app or system at scale typically relies on four main pillars:

- **Scalability:** The ability of the system to handle increasing amounts of work or growth in its size without sacrificing performance.
- **Maintainability:** The ease with which a system can be maintained and updated over time — clean, modular, well-documented code, clear processes for version control, code reviews, and documentation.
- **Reliability:** Ensuring the system operates correctly and consistently under various conditions — robust error handling, monitoring, and testing practices.
- **Flexibility:** The ability of the system to adapt and evolve in response to changing requirements, technologies, and market conditions.

When you think about these pillars, it's clear that picking the right mix of strategies, processes, architectures, and tools is critical for creating a system that can grow, adapt, and keep running smoothly.

Here are the options, grouped into two categories: must-haves and nice-to-haves.

---

## Must-Haves

### 1. Version Control

Use version control and choose what best suits your organization: multiple repositories or a monorepo approach. Define a branching strategy to manage features, releases, and maintenance (hot fixes).

![Git Flow Branches](/assets/images/developing-mobile-apps-at-scale/git-flow-branches.webp)

For a typical development cycle, you should have at least a dedicated release branch for each deployed/published version, a dev branch where all developers can check out and merge their feature or task branches, and the main branch that contains all releases marked by tags.

### 2. Code Style

Ensure your codebase looks consistent by setting the same default style for all engineers in their IDEs. A consistent code style is the first line of defense, followed by tools like Lint, SonarQube, and Qodana for Android, as well as SwiftLint for iOS. These tools perform static code analysis, with the main objective of detecting and resolving potential problems before the code is compiled or executed.

![Code style: OK vs Wrong](/assets/images/developing-mobile-apps-at-scale/code-style.webp)

Including a common local code style schema into your Android and iOS projects brings additional benefits.

![Import/Export Schema in Android Studio](/assets/images/developing-mobile-apps-at-scale/import-export-schema.webp) It helps enforce uniformity across the team, making the codebase easier to read and maintain. This shared standard reduces cognitive overhead during code reviews and makes it easier for engineers to switch between projects without adapting to different coding conventions.

### 3. Best Practices

Promote best practices across the organization by hosting workshops or masterclasses, tech talks, and demos. Make onboard documentation short but maintained and up to date — probably the best way to update it is by newly arrived teammates if they find that a process or best practice has changed.

### 4. Modules

Break down the application into smaller, manageable components or modules. One way is to split your app into as many feature-dedicated modules as possible. This enables different teams to work on their features without affecting or being affected by other feature teams. Experiment with the best way to slice your app, but remember there will always be tradeoffs. ![One of the ways to break your app into modules, via Google Android Documentation](/assets/images/developing-mobile-apps-at-scale/modularization.webp)
*One of the ways to break your app into modules, via Google Android Documentation.*

The Shim interface design pattern can help untangle navigation dependencies between different modules.

### 5. Adopt Clean Architecture

Ensure the code is maintainable, scalable, and testable by adopting Clean Architecture. Choose one of the architectural patterns like MVP, MVVM, MVI, VIPER, etc.

### 6. Code Ownership

Designate code owners for each module and code/feature scope, requiring their review and approval for any changes in their code. No code should be merged without review by several other developers, preferably those familiar with the codebase.

Example `.github/CODEOWNERS` file:

```
# Teams can be specified as code owners.
*.go docs@example.com
*.txt @octo-org/octocats
/apps/ @octocat
/apps/github @doctocat
```

### 7. Code Reviews

Create a Pull Request Template so that each PR will look similar and contain a summary of information about the task, references to tickets, stakeholders, changes, and screenshots. Enforce code reviews before merging code changes. This not only improves the quality of code but also spreads knowledge among team members.

Example `.github/pull_request_template.md`:

```markdown
## Summary
<put the summary here>

## Change Log
<describe what you changed here>

## Remaining Tasks
- [ ] PM Approval
- [ ] QA Approval
- [ ] Design Approval

## Screenshots/Video
<links to screenshots, e.g.: before and after change>

## External Links
<links to tickets, design, documentation, etc.>
```

### 8. Unit Tests and Coverage

Maintain high unit test coverage across all modules. Remember that 100% unit test coverage isn't the ultimate goal — the focus should be on delivering a high-quality app. Use JaCoCo for Android coverage reports and Xcode's built-in code coverage for iOS.

```groovy
android {
    buildTypes {
        debug {
            testCoverageEnabled true
        }
    }
}
```

Run the following Gradle command to generate the coverage report locally:

```bash
./gradlew connectedDebugAndroidTest
```

You can also make generating coverage reports part of your CI/CD pipeline:

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    steps:
      - uses: actions/checkout@v3
      - name: Run Unit Tests with Coverage
        run: ./gradlew testDebugUnitTestCoverageReport
      - uses: codecov/codecov-action@v3
```

### 9. UI Automation

In cases where unit testing isn't feasible, UI automation can be a valuable tool for ensuring quality. The basic tools for Android are Espresso and XCTest for iOS.

### 10. Integration Tests and OpenAPI Specification (Swagger)

Consider using Swagger to help you design, build, document, and consume REST APIs. Use Postman or write integration tests for backend components and endpoints to validate they function correctly within the larger system.

### 11. Continuous Integration and Deployment (CI/CD)

Use CI tools such as Jenkins, BuildKite, or GitHub Actions to automate building, testing, and reporting. Ensure every merge request runs through this CI pipeline to catch issues early. No merge request into a dev branch should be merged manually.

![CI/CD Pipeline](/assets/images/developing-mobile-apps-at-scale/cicd-pipeline.webp)

Here's a proven workflow: Write code, make it functional, execute unit tests, create a pull request, pass static analysis validations, request design review followed by code review. Incorporate manual or automated QA in this process. If your code fails at any step, fix the issue and repeat.

### 12. Monitor Google Play and App Store Reviews

Assign dedicated team members to monitor user reviews in Google Play and the Apple Store, promptly addressing relevant issues.

![Customer Review](/assets/images/developing-mobile-apps-at-scale/customer-review.webp)

### 13. Monitor Performance Metrics in Firebase

Consider integrating the Firebase Crashlytics SDK into your apps. It greatly helps investigate crashes and ANRs (Application Not Responding), identifying the responsible teams by examining the responsible code and its owners. Firebase works for both Android and iOS.

![Healthy release in Firebase](/assets/images/developing-mobile-apps-at-scale/healthy-release.webp)

### 14. Communicate Changes

Keep stakeholders and other developers informed about changes, ideally before implementation even begins. Announce timelines, consider who might be affected, and proactively reach out to them.

### 15. Architecture Decisions

Discuss architectural decisions with a broad team, including architects, graphic designers, QA, and project managers, to ensure everyone agrees on the changes and is aware of the timelines.

### 16. Developer Testing

Require developers to test their work — not only passing build and unit tests but also running the app and manually testing new features, as well as any potentially affected screen, page, or feature.

---

## Nice-to-Haves

### 17. Feature Toggles/Flags

Implement feature toggles to test new features and easily disable them if issues arise, without the need for a release update.

### 18. In-House A/B Testing

Implement in-house A/B testing for greater flexibility and control over experiments.

### 19. Bug Reporting Tools

Include a bug/issue reporting tool in your app, accessible to all employees and customers, allowing them to easily report issues or crashes.

### 20. Cross-Team Training and Collaboration

Encourage developers from different teams, skills, or platforms to work together temporarily for a few weeks to learn other technologies and better understand the entire codebase, process, flow, and product itself. This can improve overall quality and enhance engineering competence across the organization.

---

This is obviously not everything, and each of those 20 rules can be expanded into a dedicated article, but referencing them is a good starting point.

---

## References

**Version Control, Branching, PR, Code Owners**
- [Atlassian: Branching strategies](https://www.atlassian.com/agile/software-development/branching)
- [GitHub: Code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [GitHub: PR templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/creating-a-pull-request-template-for-your-repository)

**Code Style**
- [Android Kotlin style guide](https://developer.android.com/kotlin/style-guide)
- [Swift style guide](https://github.com/kodecocodes/swift-style-guide)

**Architecture & Modularization**
- [Android architecture guide](https://developer.android.com/topic/architecture)
- [Android modularization guide](https://developer.android.com/topic/modularization)
- [Now in Android modularization](https://github.com/android/nowinandroid/blob/main/docs/ModularizationLearningJourney.md)

**Source Code Analyzers**
- [Android Lint](https://developer.android.com/studio/write/lint)
- [SwiftLint](https://github.com/realm/SwiftLint)
- [SonarQube](https://www.sonarsource.com/products/sonarqube/)

**CI/CD Tools**
- [GitHub Actions](https://docs.github.com/en/actions)
- [BuildKite](https://buildkite.com)
- [Fastlane](https://fastlane.tools)

---

*Follow me on [Medium](https://medium.com/@sergey-nes) and [LinkedIn](https://www.linkedin.com/in/sergey-neskoromny/) for updates.*
