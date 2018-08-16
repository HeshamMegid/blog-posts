# How We Automate Our Ios Workflow at Instabug Using Circleci

At Instabug we build an SDK for mobile apps. Building an SDK often means that most tools created for automating tasks around development and deployment of iOS apps don't work for us, so we usually need to develop our own bespoke solutions.

To increase our productivity and keep thing moving fast, we like to automate things. One crucial component of automating many parts of our workflow is continuous integration.

This post goes through what our workflow is and how we automate it using CircleCI.

## Overview

With CircleCI 2.0, we can create multiple jobs under one workflow, with the ability to define dependencies between jobs and run some of them in parallel, which is great.

```yaml
workflows:
  version: 2
  build-test-and-generate-binary:
    jobs:
      - unit-tests
      - uitests-analyzer:
          filters:
            branches:
              only: 
                - master
                - /fix\/.*/
                - /release\/.*/
                - /Release\/.*/
      - hold:
          type: approval
          requires:
            - unit-tests
            - uitests-analyzer
      - generate-binary:
          requires:
            - hold
      - release:
          requires:
            - generate-binary
          filters:
            branches:
              only: 
                - master
```

Our workflow has 4 jobs:

1. Running unit tests, integration tests and Danger
2. Running UI tests and static analyzer
3. Generating binary
4. Releasing the SDK

Let's go through each one in details.

## 1. Running Unit Tests, Integration Tests and Danger

```yaml
unit-tests:
  macos:
    xcode: "9.3.1"
  working_directory: /Users/distiller/project
  environment:
    BUNDLE_PATH: vendor/bundle  # path to install gems and use for caching
    ARTIFACTS_DIRECTORY: /Users/distiller/project/artifacts
  steps:
    - checkout
    - restore_cache:
        keys:
        - gems-{{ checksum "Gemfile.lock" }}
        # Fall back to using the latest cache if no exact match is found.
        # - v1-gems-
    - run: 
        name: Create artifacts directory
        command: mkdir $ARTIFACTS_DIRECTORY
    # Install gems.
    - run:
        name: Bundle install
        command: bundle check || bundle install
        environment:
          BUNDLE_JOBS: 4
          BUNDLE_RETRY: 3
    - save_cache:
        key: gems-{{ checksum "Gemfile.lock" }}
        paths:
          - vendor/bundle
    - run:
        name: Pre-start simulator
        command: xcrun instruments -w "iPhone 8 (11.3) [" || true
    - run:
        name: Run Instabug unit tests
        command: set -o pipefail && xcodebuild -workspace Instabug/Instabug.xcworkspace -scheme AllUnitTests -sdk iphonesimulator -destination 'platform=iOS Simulator,OS=11.3,name=iPhone 8' -enableCodeCoverage YES test | xcpretty --color --report junit --output $ARTIFACTS_DIRECTORY/AllTests_unittest_results.xml
    - run:
        name: Run Danger
        command: bundle exec Danger
    - run: 
        name: Run integration tests
        command: set -o pipefail && xcodebuild -workspace Instabug/Instabug.xcworkspace -scheme InstabugIntegrationTests -sdk iphonesimulator -destination 'platform=iOS Simulator,OS=11.3,name=iPhone 8' -enableCodeCoverage YES test | xcpretty --color --report junit --output $ARTIFACTS_DIRECTORY/InstabugIntegrationTests_unittest_results.xml
```

First, we install all gems used by our project, like [Danger](http://danger.systems/) and [fastlane](https://fastlane.tools/) either by restoring the cache of a previous install, or by doing a fresh install.

We then run our unit tests. Due to how our project is structured, we have around 15 framework targets, with a separate test target for each. To run all our unit tests at once, we have an `AllUnitTests` scheme that runs all test targets.

After running our unit tests, we run Danger, which is great tool to automate some of the chores around code reviews. Here's what we use it for right now.

1. Enforce having has a description and a link to a Jira issue for all pull requests
2. Ensure that all pull requests that add new user-facing strings add localized versions of those strings.
3. Run [xcov](https://github.com/nakiostudio/xcov) to generate a report about our tests coverage, and post it as a comment on the pull request
4. Ensure that all pull requests that modify any UI file run UI tests before merging the pull request.

Last steps of this job is to run our integration tests, which is just a test target with a similar configuration to our unit test targets.

This job runs on every pull request, so we have to make sure it contains all the essential checks/tests, and that it also runs in a reasonable time. It currently runs in around 8 minutes, with the majority of the time going to making a clean build of the SDK for running tests.

## Running Ui Tests and Static Analyzer

```yaml
uitests-analyzer:
  macos:
    xcode: "9.3.1"
  working_directory: /Users/distiller/project
  environment:
    BUNDLE_PATH: vendor/bundle  # path to install gems and use for caching
    ARTIFACTS_DIRECTORY: /Users/distiller/project/artifacts
  steps:
    - checkout
    - restore_cache:
        keys:
        - gems-{{ checksum "Gemfile.lock" }}
        # Fall back to using the latest cache if no exact match is found.
        # - v1-gems-
    - run: 
        name: Create artifacts directory
        command: mkdir $ARTIFACTS_DIRECTORY
    # Install gems.
    - run:
        name: Bundle install
        command: bundle check || bundle install
        environment:
          BUNDLE_JOBS: 4
          BUNDLE_RETRY: 3
    - run:
        name: Pre-start simulator
        command: xcrun instruments -w "iPhone 8 (11.3) [" || true
    - run:
        name: Run UI tests
        command: |
          if [[ -n "${RUN_UI_TESTS}" || $CIRCLE_BRANCH = 'master' ]]; then
            set -o pipefail && xcodebuild -workspace Instabug/Instabug.xcworkspace -scheme InstabugDemoUITests -sdk iphonesimulator -destination 'platform=iOS Simulator,OS=11.3,name=iPhone 8' -only-testing:InstabugDemoUITests/AlertsUITests -only-testing:InstabugDemoUITests/BugReportingDisabledAttachmentsUITests -only-testing:InstabugDemoUITests/BugReportingUITests -only-testing:InstabugDemoUITests/IBGAnnotationUITests -only-testing:InstabugDemoUITests/IBGChatBasicUITests -only-testing:InstabugDemoUITests/IBGPromptVCUITests -only-testing:InstabugDemoUITests/IBGStatusBarTests test | xcpretty --color --report junit --output $ARTIFACTS_DIRECTORY/xcode/uitest_results_1.xml
            set -o pipefail && xcodebuild -workspace Instabug/Instabug.xcworkspace -scheme InstabugDemoUITests -sdk iphonesimulator -destination 'platform=iOS Simulator,OS=11.3,name=iPhone 8' -only-testing:InstabugDemoUITests/IBGSurveysFlowUITests -only-testing:InstabugDemoUITests/IBGSurveysUITests -only-testing:InstabugDemoUITests/InstabugUITests -only-testing:InstabugDemoUITests/IBGFeatureRequestUITests test | xcpretty --color --report junit --output $ARTIFACTS_DIRECTORY/xcode/uitest_results_2.xml
          else
            echo 'Skipping running UI Tests.'
          fi
        no_output_timeout: 15m

    - run:
        name: Run static analyzer on other frameworks
        command: sh ./Scripts/analyze_Instabug.sh
```

This job is very similar to what we do in the first job, but it instead runs a UI tests target and the static analyzer using `xcodebuild analyze`.

This job only runs on master, and fix/release branches, so we're okay with it taking a bit longer to run. It currently finishes in around 40 minutes and runs in parallel with the first job. It can also be run on demand on any pull request regardless of its branch by mentioning a simple GitHub bot we wrote that uses the CircleCI API to trigger a specific job.

## Generating Binary

```yaml
generate-binary: 
  macos:
    xcode: "9.3.1"
  working_directory: /Users/distiller/project
  environment:
    BUNDLE_PATH: vendor/bundle  # path to install gems and use for caching
    ARTIFACTS_DIRECTORY: /Users/distiller/project/artifacts
  steps:
    - checkout
    - restore_cache:
        keys:
        - gems-{{ checksum "Gemfile.lock" }}
        # Fall back to using the latest cache if no exact match is found.
        # - v1-gems-
    - run: 
        name: Create artifacts directory
        command: mkdir $ARTIFACTS_DIRECTORY
    # Install gems.
    - run:
        name: Bundle install
        command: bundle check || bundle install
        environment:
          BUNDLE_JOBS: 4
          BUNDLE_RETRY: 3
    - run:
        name: Install signing identity
        command: |
          bundle exec fastlane setup_signing
    - run:
        name: Increment version number
        command: |
          ./Scripts/IncrementSDKVersion.swift
          /usr/libexec/PlistBuddy -c "Set :CFBundleVersion $CIRCLE_BUILD_NUM" "Instabug/InstabugI/Info.plist"
    - run:
        name: Generate fat binary for Instabug static
        command: |
          xcodebuild -workspace Instabug/Instabug.xcworkspace -scheme Framework -sdk iphoneos -destination generic/platform=iOS clean build | xcpretty
    - run:
        name: Link Instabug static with dynamic project
        command: |
          ruby ./Instabug-dynamic/linkInstabug.rb
    - run:
        name: Generate fat binary for Instabug dynamic
        command: |
          xcodebuild -project Instabug-dynamic/Instabug.xcodeproj -scheme Framework -sdk iphoneos -destination generic/platform=iOS clean archive | xcpretty
    - run:
        name: Generate appledoc
        command: |
          sh ./Scripts/generate_appledoc.sh
    - run:
        name: Create framework archive
        command: |
          find ./Instabug/Instabug-SDK-Static -path '*/.*' -prune -o -type f -print | zip $ARTIFACTS_DIRECTORY/Instabug-static.zip -@
          find ./Instabug-dynamic/Instabug-SDK -path '*/.*' -prune -o -type f -print | zip $ARTIFACTS_DIRECTORY/Instabug.zip -@
          find ./Instabug-Docs -path '*/.*' -prune -o -type f -print | zip $ARTIFACTS_DIRECTORY/appledoc.zip -@
    - run:
        name: Test Fat Binaries are not Corrupted
        command: |
          xcodebuild -project InstabugProductionDemo/InstabugProductionDemo.xcodeproj -scheme InstabugProductionDemoUITests -sdk iphonesimulator -destination 'platform=iOS Simulator,OS=11.3,name=iPhone 8' test | xcpretty
    - store_artifacts:
        path: artifacts
    - persist_to_workspace:
        root: .
        paths:
          - .
```

After the first two jobs have finished, we can generate a binary of the SDK from any branch. This requires the approval of a hold from the CircleCI dashboard.

This job will install our code signing identity, which we share across the team using [fastlane](https://docs.fastlane.tools/actions/match/) . It then generates both static and dynamic variants of our framework. For more on building binary framework, check our previous [blog post](https://instabug.com/blog/ios-binary-framework/).

It then bundles up both frameworks in a zip archive and stores it in the artifacts directory to be available for download.

## Releasing the SDK

```yaml
release:
  macos:
    xcode: "9.3.1"
  working_directory: /Users/distiller/project
  environment:
    ARTIFACTS_DIRECTORY: /Users/distiller/project/artifacts
  steps:
    - attach_workspace:
        at: /Users/distiller/project
    - run:
        name: Release
        command: ./release
```

The last job releases the SDK to the public. It runs Unleash, our homegrown CLI app for releasing the SDK.

Unleash does the following:

1. Create and push a new tag to our private repository
2. Push the updated framework to https://github.com/Instabug/Instabug-iOS/ and create a GitHub release
3. Publish framework to CocoaPods
4. Update our Carthage [JSON file](https://github.com/Instabug/Instabug-iOS/blob/master/Instabug.json)
5. Upload the release to our own API for consumption on the Instabug dashboard and website.

## Conclusion

We're pretty happy with the workflow we have so far. Having a reliable CI process makes us ship with confidence, and automating our release process saves a ton of time since we release once a week. 

CircleCI has been a great tool to use for our continuous integration, specially with CircleCI 2.0.
