% Gitlab Hackday
% Patrick Kleindienst, Benjamin Ißler, Fabian , Michael Ihls, Benjamin Binder
% 10.3.2018

## What we did

- Installed Gitlab natively on AWS
- Created a sample [Spring Boot](https://projects.spring.io/spring-boot/) project
- Built a CI/CD pipeline using several stages:
	- a build stage
	- a deployment stage
	- a test stage


## Setting up AWS

We installed an M4 large instance with the latest Ubuntu. To prevent crazy russians from hacking our machine we restricted SSH to the external subnet of our organization.


## Install Gitlab

First get a [free trial license](https://about.gitlab.com/free-trial/) for Gitlab EE.
For the Gitlab installation we followed [this tutorial](https://about.gitlab.com/installation/#ubuntu).


## Configure Gitlab

All configuration for your CI/CD can be accumulated in a central `.gitlab-ci.yml` file in the root directory of your project.

### Configure Gitlab-Runner

GitLab Runner runs your jobs and sends the results back to GitLab. It is coordinated by Gitlab.

You can run Gitlab Runner on a dedicated host like we did.
You have to register the runner at your Gitlab installation using a token shown in the Runners panel of Gitlab.
The runner can be configured to run jobs as Docker containers, or as shell commands. We chose the later path.

## Build CI/CD pipeline

The Gitlab CI/CD workflow is [pipeline oriented](https://docs.gitlab.com/ee/ci/pipelines.html). You can add pipelines in your project's Pipeline tab.
To add stages to your pipeline, add them to the `stage` section of your config:

``` yaml
stages:
  - build-stage
  - deploy-stage
  - test-stage
```

Now you can create jobs and add them to your stages.

### First stage: Unit tests

First you would create jobs running unit tests. Because we don't make mistakes when coding, we can skip this step.

### Second stage: Build stage

The next step would be to build your application.
Create a job for this by adding a job-section to the yml-file.

In the `script-section` you can add shell commands to be executed. In our case we built the application.

To save yourself some typing you can configure project-global variables (key-value-pairs) under settings > CI/CD. In your config you can access them using `${KEY}`.

Pass the result of the job to the next using an artifact. The path can be used in the next job.

```yaml
build-job:
  stage: build-stage
  environment:
    name: dev
    url: http://sample-app-chatty-duiker.cfapps.io/
  tags:
    - shell
  script:
    - ./mvnw install -D skipTests
  artifacts:
    name: "${CI_JOB_NAME}_${CI_COMMIT_REF_NAME}_${CI_JOB_ID}"
    when: always
    expire_in: 7d
    paths:
      - "target/spring-boot-sample-app-*-SNAPSHOT.jar"

```

### Deploy

After we have successfully built our application, we can push it to an integration testing platform.
We deployed our app to Cloud Foundry.

To install the Cloud Foundry CLI tool, follow [this tutorial](https://github.com/cloudfoundry/cli#installing-using-a-package-manager).

Then you can configure your job:

Login to your PaaS, in our case using `cf login -a HOSTNAME -u USERNAME -p PASSWORD`.
Push your application to the platform using `cf push APPNAME -p target/APPNAME`. If you are too lazy to configure a route for your app, simply add `--random-route`.

Your job should now look something like this:

```yaml
deploy-job:
  stage: deploy-stage
  tags:
    - shell
  script:
    - cd spring-boot-sample-app
    - "cf login -a ${CF_API} -u ${CF_USER} -p \"${CF_PASS}\""
    - cf push go-sample-app -p go-sample-app --random-route
```

### Smoke test

Although, thanks to our skill, unnecessary, we still configured a smoke test.

```yaml
test-job:
  stage: smoke-test
  tags:
    - shell
  script:
    - curl --silent http://sample-app-chatty-duiker.cfapps.io/actuator/health | jq '.status' | grep -o UP
```



<style>
	html { width: 100%; background-color: #fafafa; }
	body { max-width: 700px; background-color: white; margin: 0 auto; padding: 5em; margin-top: 2.5em; box-shadow: 0px 2px 5px 0px rgba(0,0,0,0.75); }
	body > section { border-bottom: 1px solid #e5e5e5; }
</style>
