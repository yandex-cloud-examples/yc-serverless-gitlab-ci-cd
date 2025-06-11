# Demonstration of using CI/CD to deploy serverless apps in Yandex Cloud

This example demonstrates a test app that is deployed in Yandex Cloud on serverless components. The entire pipeline of building, testing, and deploying the app is fully automated with GitLab CI/CD.

## Involved services and technologies

* [Gitlab Runner](https://docs.gitlab.com/runner/): To automate processes.
* [Docker](https://www.docker.com/): To containerize the app.
* [Yandex Container Registry](https://yandex.cloud/services/container-registry): To store the app Docker image.
* [Yandex Serverless Containers](https://yandex.cloud/services/serverless-containers): To deploy the app in serverless mode.
* [Yandex API Gateway](https://yandex.cloud/services/api-gateway): For the app to receive traffic.
* [Yandex Managed Service for YDB](https://yandex.cloud/services/ydb) (serverless Document API mode): To store the app data.
* [Yandex Lockbox](https://yandex.cloud/services/lockbox): To store and deliver the app secrets.

## How to deploy this demo app locally

To deploy this example app yourself, you will need a GitLab repository and a Yandex Cloud account.

Make sure to have the following apps and utilities installed locally:
* [Yandex Cloud CLI](https://yandex.cloud/docs/cli/)
* [`jq` JSON stream processor](https://stedolan.github.io/jq/)
* [`yq` YAML stream processor](https://github.com/mikefarah/yq)
* [Python 3.8 or higher](https://www.python.org/)
* Python libraries listed in [application/requirements.txt](application/requirements.txt)

Also, note that Yandex Lockbox is currently at the [Preview](https://yandex.cloud/docs/overview/concepts/launch-stages) stage. To use it in your cloud, you need to request access [here](https://yandex.cloud/services/lockbox#preview-form).

## Initial setup

1. Copy the contents of this directory to the root of your GitLab repository.
1. To deploy the basic infrastructure to the root of the repository, run this command:

   ```bash
   YC_CLOUD_ID=<your cloud ID> ./bootstrap.sh
   ```

Once completed, the script will output the values of the GitLab environment variables. Save them, as you will need them later.

## About the app

This demo project implements a simple [Django](https://www.djangoproject.com/) web app that simulates the product cart of an e-commerce service.

The database stores product descriptions and is populated with test data from the Bootstrap script. The service stores the product cart status in a user session.

The Django app is deployed in a serverless container with secrets securely delivered to the app using Yandex Lockbox. API Gateway receives user requests and redirects them to the app's container.

## Environments

In this example, we use these two environments to deploy our app:
* **prod**: Production environment that is available to users.
* **testing**: Test environment that is used for testing the app before its release to **prod**.

For each environment, the Bootstrap script creates a separate folder in Yandex Cloud, as well as dedicated static resources, such as a DB and service accounts. This isolates the environments from each other at the [Yandex Identity and Access Management](https://yandex.cloud/services/iam) settings level.

The **prod** and **testing** environments each contain one deployed test bench.

In addition, the project employs a common folder named `infra`. All built Docker images of the app are published into that folder. For the service accounts of the **prod** and **testing** environments, permissions in the `infra` folder are limited to pulling images from there. To publish the images into the `infra` folder, a separate service account named `builder` is used.

## Automated processes

After adding the CI script configuration file, `.gitlab-ci.yml`, the build script will run.

It runs as follows:
* Building the image from the repo branch and pushing it to the Container Registry.
* Deploying the app to the `testing` environment.
* Testing the app: by now, we only use `curl` if the app home page is available.
* Deleting the test app.
* Deploying the app to production. Additionally, an option with environments is used, to save your deployment and restore it if needed.

When you publish changes to the `main` branch (by PR branch merge or direct push to `main`), the build script will automatically run again. This way, PR changes are automatically rolled out to the deployed test bench.
