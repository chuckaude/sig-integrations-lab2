# SIG Integration Lab 2
The goal of this lab is to provide hands on experience integrating a Polaris scan into a GitLab pipeline using the [Synopsys GitLab Template](https://gitlab.com/synopsys/synopsys-template) and demonstrating its post scan capabilities. As part of this lab, we will:
- execute a full scan, viewing the results in the Polaris UI
- break the build based on a policy defined in the Polaris UI
- review the exported SARIF report
- introduce a vulnerable code change that adds a comment to the Merge Request

This repository contains everything you need to complete the lab except for the two prerequisites listed below.

# Prerequisites

1. [signup](https://gitlab.com/users/sign_up) for a free GitLab Account
2. [create](https://polaris.synopsys.com/developer/default/polaris-documentation/t_make-token) a Polaris Access Token

# Clone repository

1. Clone this repository into your GitLab account via _GiLab → Projects → New Project → Import Project → Repository by URL_
   - repostiory url: https://github.com/chuckaude/sig-integrations-lab2.git
   - username & password not needed
   - optionally change project slug (repository name)
   - set visbility to _public_ (for debugging purposes)

 **Milestone 1** :heavy_check_mark:

# Setup pipeline

1. Create a project access token with Developer role and API scope via _GitLab → Project → Settings → Access Token → Add New Token_
2. Add the following variables via _GitLab → Project → Settings → CI/CD → Variables_. Be sure to select **masked** for the tokens and unset **protect** for all three.
   - POLARIS_SERVERURL
   - POLARIS_ACCESSTOKEN
   - GITLAB_USER_TOKEN
4. Add a coverity.yaml to the project repository via _GitLab → Project → New file (plus icon top middle)_

```
capture:
  build:
    clean-command: mvn -B clean
    build-command: mvn -B -DskipTests package
analyze:
  checkers:
    webapp-security:
      enabled: true
```

4. Create a new pipeline via _GitLab → Project → Build → Pipeline Editor → Configure Pipeline_. Replace the template with the following and **be sure to change the chuckaude prefix to your name** for the Polaris application name.

```
include:
  - project: synopsys/synopsys-template
    ref: v1.9.0
    file: templates/synopsys-template.yml

stages:
  - build
  - test
  - security
  - deploy

variables:
  SCAN_BRANCHES: "/^(main|master|develop|stage|release)$/"

cache:
  paths:
    - .m2/repository/
    - target/

image: maven:3-eclipse-temurin-17

build:
  stage: build
  script: mvn -B compile

test:
  stage: test
  script: mvn -B test

deploy:
  stage: deploy
  only:
    variables:
      - $CI_COMMIT_REF_NAME =~ $SCAN_BRANCHES
  script: mvn -B install

polaris:
  stage: security
  rules:
    - if: (($CI_COMMIT_BRANCH =~ $SCAN_BRANCHES && $CI_PIPELINE_SOURCE != 'merge_request_event') ||
        ($CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ $SCAN_BRANCHES && $CI_PIPELINE_SOURCE == 'merge_request_event'))
  variables:
    BRIDGE_POLARIS_SERVERURL: $POLARIS_SERVERURL
    BRIDGE_POLARIS_ACCESSTOKEN: $POLARIS_ACCESSTOKEN
    BRIDGE_POLARIS_ASSESSMENT_TYPES: 'SAST,SCA'
    BRIDGE_POLARIS_APPLICATION_NAME: chuckaude-$CI_PROJECT_NAME
    BRIDGE_POLARIS_PRCOMMENT_ENABLED: 'true'
    BRIDGE_POLARIS_REPORTS_SARIF_CREATE: 'true'
    BRIDGE_GITLAB_USER_TOKEN: $GITLAB_USER_TOKEN
    # INCLUDE_DIAGNOSTICS: 'true'
  before_script:
    - apt-get -qq update && apt-get install -y curl unzip
  extends: .run-synopsys-tools
  artifacts:
    name: "SARIF report"
    when: always
    paths:
      - .bridge/Polaris SARIF Generator/report.sarif.json
    expire_in: 30 days
```

 **Milestone 2** :heavy_check_mark:
 
# Full Scan

1. Monitor your pipeline run and wait for scan to complete via _GitLab → Project → Build → Pipelines_
   - Note that the scan completes and the pipeline passes. This is because the default policy is "notify on critical & high issues".
2. From the Polaris UI, [create a policy](https://polaris.synopsys.com/developer/default/polaris-documentation/t_post_scan_policies) that breaks the build and assign it to your project.
3. Run the pipeline again. Once it completes, select the failed polaris step to see policy enforcement and a failed pipeline.

**Milestone 3** :heavy_check_mark:

4. Download the SARIF Report via _GitLab → Project → Build → Download Artifacts (download icon right side)_

**Milestone 4** :heavy_check_mark:

# PR scan

1. Edit pom.xml _GitLab → Project → Code → Repository → pom.xml → Edit button upper right_
   - change log4j version from 2.14.1 to 2.15.0
3. Change target branch to main-patch1, leave _start merge request_ checked, then click on _Commit Changes_
4. Review changes and click on _Create Merge Request_
5. Monitor pipeline run _GitLab → Project → Build → Pipelines_
6. Once pipeline completes, navigate back to MR and see the MR comment via _GitLab → Project → Merge requests_

**Milestone 5** :heavy_check_mark:

# Congratulations

You have now configured a Polaris scan in a GitLab pipeline and demonstrated all the functionality of the [Synopsys GitLab Template](https://gitlab.com/synopsys/synopsys-template). :clap: :trophy:
