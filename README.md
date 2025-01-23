image: node:22-alpine

default:
  interruptible: true

variables:
  GIT_DEPTH: 0
  LIBRARY_NAME: "enterprise-components"
  NPM_TOKEN: $NPM_TOKEN # Assumes the NPM token is stored as a CI/CD variable in GitLab
  NEXT_RELEASE: 0.0.0
  STORY_BOOK_ENABLED: false
  COMPODOC_ENABLED: false
before_script:
  - echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > ~/.npmrc

.branch-rules:
  rules:
     # Rule for merge requests from feature branches to branch like x.x.x
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^feature\// && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(\d+\.\d+\.\d+)$/
      variables:
        VERSION: "${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}-rc.$CI_COMMIT_SHORT_SHA"

    # Rule for merge requests from feature branches
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^feature\//
      variables:
        VERSION: "${NEXT_RELEASE}-rc.$CI_COMMIT_SHORT_SHA"

    # Rule for merging release branches into main
    - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main" && $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME =~ /^(\d+\.\d+\.\d+)$/
      variables:
        VERSION: "${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME}"
        TAG_NAME: "${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME}"

    # Rule for release branches commit
    - if: $CI_COMMIT_BRANCH =~ /^(\d+\.\d+\.\d+)$/
      variables:
        VERSION: "${CI_COMMIT_BRANCH}-rc.$CI_COMMIT_SHORT_SHA"
    
stages:
  - Validate
  - Install
  - Test
  - Build
  - Release
  - Post-Release

Check Variables:
  stage: Validate
  script:
    - echo $CI_COMMIT_BRANCH
    - echo $VERSION
    - echo $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
    - echo $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    - echo $CI_PIPELINE_SOURCE

# Validate whether Package Version Exists in artifactory
Verify Package Version Existence:
  stage: Validate
  extends: 
    - .branch-rules
  script:
    - echo "Validating version $VERSION"
    - PACKAGE_VERSION_EXISTS=$(npm show @$LIBRARY_NAME/$CI_PROJECT_NAME@$VERSION _id 2>/dev/null || true)
    - if [[ $? != 0 ]]; then echo "Could not fetch package information from NPM Repository"; elif [[ $PACKAGE_VERSION_EXISTS ]]; then echo "Package version $VERSION already exists in NPM Repository" && exit 1; else echo "Package version $VERSION does not exists in NPM Repository" && exit 0; fi

# Install dependencies
Install Dependencies:
  stage: Install
  extends: 
    - .branch-rules
  script:
    - echo $VERSION
    - npm install --prefer-offline --no-audit --legacy-peer-deps
  allow_failure: false
  cache:
    key: node_modules-$VERSION
    paths:
      - node_modules/
    when: 'on_success'
    policy: pull-push

# Perform Unit Tests
Unit Test:
  image: entangledcognition/node-chrome:latest
  stage: Test
  extends: 
    - .branch-rules
  coverage: '/Statements\s+:\s(\d{1,3}(\.\d{2})?)%/'
  script:
    - npx ng test @$LIBRARY_NAME/$CI_PROJECT_NAME --watch=false --source-map=false --code-coverage=true --browsers=ChromeHeadlessCI
    - ls -a
  allow_failure: false
  cache:
    key: node_modules-$VERSION
    paths:
      - node_modules/
    when: 'on_success'
    policy: pull
  artifacts:
    paths:
      - coverage/lcov.info
    reports:
      junit: coverage/TESTS-*.xml

# Build the library artifact
Build Library Artifact:
  stage: Build
  extends: 
    - .branch-rules
  script:
    - cd $CI_PROJECT_DIR/projects/$LIBRARY_NAME/$CI_PROJECT_NAME
    - echo $VERSION
    - npm version $VERSION --allow-same-version
    - cd $CI_PROJECT_DIR
    - export NODE_OPTIONS=--max_old_space_size=4096
    - npm run build
  cache:
    key: node_modules-$VERSION
    paths:
      - node_modules/
    when: 'on_success'
    policy: pull
  artifacts:
    name: "$VERSION"
    paths:
      - dist/

Publish StoryBook:
  stage: Release
  needs: ["Build Library Artifact"]
  rules:
    # First, check STORY_BOOK_ENABLED
    - if: $STORY_BOOK_ENABLED != "true"
      when: never
    - !reference [.branch-rules, rules]
  script:
    - |
      cd $CI_PROJECT_DIR/projects/$LIBRARY_NAME/$CI_PROJECT_NAME
      echo "Setting version to $VERSION"
      npm version $VERSION --allow-same-version
      cd $CI_PROJECT_DIR
      export NODE_OPTIONS=--max_old_space_size=4096
      npm run build-storybook
      mkdir -p public/$CI_COMMIT_REF_NAME
      mv $CI_PROJECT_DIR/dist/storybook/$LIBRARY_NAME/$CI_PROJECT_NAME public/$CI_COMMIT_REF_NAME
      echo "Storybook deployed to: https://$CI_PROJECT_ROOT_NAMESPACE.gitlab.io/-/$LIBRARY_NAME/$CI_PROJECT_NAME/-/jobs/$CI_JOB_ID/artifacts/public/$CI_COMMIT_REF_NAME/material/index.html"
  cache:
    key: node_modules-$VERSION
    paths:
      - node_modules/
    when: 'on_success'
    policy: pull-push
  artifacts:
    paths:
      - public


Publish Compdoc:
  stage: Release
  needs: ["Build Library Artifact"]
  rules:
    # First, check STORY_BOOK_ENABLED
    - if: $COMPODOC_ENABLED != "true"
      when: never
    - !reference [.branch-rules, rules] 
  script:
    - |
      cd $CI_PROJECT_DIR/projects/$LIBRARY_NAME/$CI_PROJECT_NAME
      echo "Setting version to $VERSION"
      npm version $VERSION --allow-same-version
      cd $CI_PROJECT_DIR
      export NODE_OPTIONS=--max_old_space_size=4096
      npm run compodoc:build
      mkdir -p public/compodoc
      mv $CI_PROJECT_DIR/compodoc public
      echo "Compodoc deployed to: https://$CI_PROJECT_ROOT_NAMESPACE.gitlab.io/-/$LIBRARY_NAME/$CI_PROJECT_NAME/-/jobs/$CI_JOB_ID/artifacts/public/compodoc/index.html"
  cache:
    key: node_modules-$VERSION
    paths:
      - node_modules/
    when: 'on_success'
    policy: pull-push
  artifacts:
    paths:
      - public

# Publish to NPM registry under @enterprise-components scope
Publish to Registry:
  stage: Release
  needs: ["Build Library Artifact"]
  extends: 
    - .branch-rules
  script:
    - echo "registry=https://registry.npmjs.org/" >> ~/.npmrc
    - echo "${NPM_TOKEN}"
    - echo "$NPM_TOKEN"
    - echo '$NPM_TOKEN'
    - echo $NPM_TOKEN
    - echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > ~/.npmrc
    - cat ~/.npmrc
    - npm whoami
    - npm publish $CI_PROJECT_DIR/dist/$LIBRARY_NAME/$CI_PROJECT_NAME

# Prepare for creating a release
Prepare Release:
  stage: Post-Release
  interruptible: false
  image:
    name: alpine/git:2.36.3
    entrypoint: [""]
  rules:
    - if: '$CI_COMMIT_REF_PROTECTED == "true" && $CI_COMMIT_TAG && $CI_COMMIT_MESSAGE =~ /Merge.+branch\s.(\d+\.)?(\d+\.)?(\*|\d+).\sinto(.*)/'
    - if: '$CI_COMMIT_REF_PROTECTED == "true" && $CI_COMMIT_TAG && $CI_COMMIT_MESSAGE =~ /Merge.+branch\s.renovate\/(.*).\sinto\s.(.*)/'

  script:
    - echo "RELEASE_DESCRIPTION=$(git show -s --format=%B $CI_COMMIT_SHA | grep \"See merge request\")" >> variables.env
  artifacts:
    reports:
      dotenv: variables.env

# Create release
Release:
  stage: Post-Release
  interruptible: false
  image: gitlab-org/release-cli:latest
  needs:
    - job: Prepare Release
      artifacts: true
  rules:
    - if: '$CI_COMMIT_REF_PROTECTED == "true" && $CI_COMMIT_TAG && $CI_COMMIT_MESSAGE =~ /Merge.+branch\s.(\d+\.)?(\d+\.)?(\*|\d+).\sinto(.*)/'
    - if: '$CI_COMMIT_REF_PROTECTED == "true" && $CI_COMMIT_TAG && $CI_COMMIT_MESSAGE =~ /Merge.+branch\s.renovate\/(.*).\sinto\s.(.*)/'

  script:
    - echo "Creating release for $CI_COMMIT_TAG"
  release:
    tag_name: $CI_COMMIT_TAG
    description: $RELEASE_DESCRIPTION
