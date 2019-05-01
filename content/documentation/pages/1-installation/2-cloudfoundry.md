---
path: 'installation/cloudfoundry/'
title: 'Cloud Foundry'
description: 'Install Data Flow on your Cloud Foundry'
---

# Cloud Foundry Installation

This section covers how to install Spring Cloud Data Flow on Cloud Foundry.

## System Requirements

Spring Cloud Data Flow requires a few data services to perform streaming
and task or batch processing. You have two options when you provision
Spring Cloud Data Flow and related services on Cloud Foundry:

- The simplest (and automated) method is to use the [Spring Cloud Data
  Flow for PCF tile](https://network.pivotal.io/products/p-dataflow).
  This is an opinionated tile for Pivotal Cloud Foundry. It
  automatically provisions the server and the required data services,
  thus simplifying the overall getting-started experience. You can
  read more about the installation
  [here](https://docs.pivotal.io/scdf/).

- Alternatively, you can provision all the components manually.

The following section goes into the specifics of how to install manually.

### Provisioning a Rabbit Service Instance on Cloud Foundry

RabbitMQ is used as a messaging middleware between streaming apps and is available as a PCF tile.

You can use `cf marketplace` to discover which plans are available to you, depending on the details of your Cloud Foundry setup.
For example, you can use [Pivotal Web Services](https://run.pivotal.io/), as the following example shows:

    cf create-service cloudamqp lemur rabbit

### Provision a PostgreSQL Service Instance on Cloud Foundry

An RDBMS is used to persist Data Flow state, such as stream and task definitions, deployments, and executions.

You can use `cf marketplace` to discover which plans are available to you, depending on the details of your Cloud Foundry setup. For example, you can use [Pivotal Web Services](https://run.pivotal.io/), as the following example shows:

    cf create-service elephantsql panda my_postgres

[[tip | Database Connection Limits]]
| If you intend to create and run batch-jobs as Task pipelines in SCDF,
| you must ensure that the underlying database instance includes enough
| connections capacity so that the batch-jobs, Task, and SCDF can
| concurrently connect to the same database instance without running
| into connection limits. This usually means you can't use any free plans.

## Install on Cloud Foundry

To install Cloud Foundry:

1.  Download the Data Flow server and shell applications, by running the
    following example commands:

        wget https://repo.spring.io/{version-type-lowercase}/org/springframework/cloud/spring-cloud-dataflow-server/{project-version}/spring-cloud-dataflow-server-{project-version}.jar

        wget https://repo.spring.io/{version-type-lowercase}/org/springframework/cloud/spring-cloud-dataflow-shell/{project-version}/spring-cloud-dataflow-shell-{project-version}.jar

2.  Download [Skipper](https://cloud.spring.io/spring-cloud-skipper/),
    to which Data Flow delegates stream lifecycle operations, such as
    deployment, upgrading and rolling back. To do so, use the following
    command:

        wget https://repo.spring.io/{skipper-version-type-lowercase}/org/springframework/cloud/spring-cloud-skipper-server/{skipper-version}/spring-cloud-skipper-server-{skipper-version}.jar

3.  Push Skipper to Cloud Foundry

    Once you have installed Cloud Foundry, you can push Skipper to
    Cloud Foundry. To do so, you need to create a manifest for Skipper.

    You will use the `SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[pws]_DEPLOYMENT_SERVICES` setting in the `Skipper Server` configuration, which automatically binds RabbitMQ to the deployed streaming applications.

    The following example shows a typical manifest for Skipper:

        ---
        applications:
        - name: skipper-server
          host: skipper-server
          memory: 1G
          disk_quota: 1G
          instances: 1
          timeout: 180
          buildpack: java_buildpack
          path: <PATH TO THE DOWNLOADED SKIPPER SERVER UBER-JAR>
          env:
            SPRING_APPLICATION_NAME: skipper-server
            SPRING_PROFILES_ACTIVE: cloud
            JBP_CONFIG_SPRING_AUTO_RECONFIGURATION: '{enabled: false}'
            SPRING_CLOUD_SKIPPER_SERVER_STRATEGIES_HEALTHCHECK_TIMEOUTINMILLIS: 300000
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_URL: https://api.run.pivotal.io
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_ORG: <org>
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_SPACE: <space>
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_DOMAIN: cfapps.io
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_USERNAME: <email>
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_PASSWORD: <password>
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_SKIP_SSL_VALIDATION: false
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_DELETE_ROUTES: false
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_SERVICES: <serviceName>
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_STREAM_ENABLE_RANDOM_APP_NAME_PREFIX: false
            SPRING_CLOUD_SKIPPER_SERVER_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_MEMORY: 2048m
        services:
        - <services>

    You need to fill in `<org>`, `<space>`, `<email>`, `<password>`,
    `<middlewareServiceName>` (RabbitMQ or Apache Kafka) and
    `<services>` (such as PostgresSQL) before running these commands.
    Once you have the desired config values in `manifest.yml`, you can
    run the `cf push` command to provision the skipper-server.

    [[warning | SSL Validation]]
    | Set _Skip SSL Validation_ to `true` only if you run on a Cloud
    | Foundry instance by using self-signed certificates (for example,
    | in development). Do not use self-signed certificates
    | for production.

    [[tip | Buildpacks]]
    |
    | When specifying the `buildpack`, our examples typically specify
    | `java_buildpack` or `java_buildpack_offline`. Use the CF command
    | `cf buildpacks` to get a listing of available relevant buildpacks
    | for your environment.

4.  Configure and run the Data Flow Server.

One of the most important configuration details is providing credentials to the Cloud Foundry instance so that the server can itself spawn
applications.
You can use any Spring Boot-compatible configuration mechanism (passing program arguments, editing configuration files before
building the application, using [Spring Cloud Config](https://github.com/spring-cloud/spring-cloud-config), using environment variables, and others), although some may prove more practicable than others, depending on how you typically deploy applications to Cloud Foundry.

In later sections, we show how to deploy Data Flow by using [environment variables](#getting-started-cloudfoundry-deploying-using-env-vars) or a [Cloud Foundry manifest](#getting-started-cloudfoundry-deploying-using-manifest).
However, there are some general configuration details you should be aware of in either approach.

### General Configuration

This section covers some things to be aware of when you install into
Cloud Foundry.

#### Unique names

You must use a unique name for your application. An application with the same name in the same organization causes your deployment to fail.

#### Memory Settings

The recommended minimum memory setting for the server is 2G. Also, to
push apps to PCF and obtain application property metadata, the server
downloads applications to a Maven repository hosted on the local disk.
While you can specify up to 2G as a typical maximum value for disk
space on a PCF installation, you can increase this to 10G. Read the
[maximum disk quota](#getting-started-maximum-disk-quota-configuration) > [???](#getting-started-maximum-disk-quota-configuration) section for
information on how to configure this PCF property. Also, the Data Flow
server itself implements a Last-Recently-Used algorithm to free disk
space when it falls below a low-water-mark value.

#### Routing

If you push to a space with multiple users (for example, on PWS), the route you chose for your application name may already be taken.
You can use the `--random-route` option to avoid this when you push the server application.

#### Maven repositories

If you need to configure multiple Maven repositories, a proxy, or authorization for a private repository, see [Maven Configuration](http://docs.spring.io/spring-cloud-dataflow/docs//reference/htmlsingle/#getting-started-maven-configuration).

### Installing using a Manifest

As an alternative to setting environment variables with the `cf set-env` command, you can curate all the relevant environment variables in a
`manifest.yml` file and use the `cf push` command to provision the server.
The following example shows such a manifest file:

    ---
    applications:
    - name: data-flow-server
      host: data-flow-server
      memory: 2G
      disk_quota: 2G
      instances: 1
      path: {PATH TO SERVER UBER-JAR}
      env:
        SPRING_APPLICATION_NAME: data-flow-server
        SPRING_PROFILES_ACTIVE: cloud
        JBP_CONFIG_SPRING_AUTO_RECONFIGURATION: '{enabled: false}'
        MAVEN_REMOTEREPOSITORIES[REPO1]_URL: https://repo.spring.io/libs-snapshot
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_URL: https://api.huron.cf-app.com
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_ORG: sabby20
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_SPACE: sabby20
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_DOMAIN: apps.huron.cf-app.com
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_USERNAME: admin
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_PASSWORD: ***
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_SKIP_SSL_VALIDATION: true
        SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_SERVICES: postgreSQL
        SPRING_CLOUD_SKIPPER_CLIENT_SERVER_URI: https://<skipper-host-name>/api
    services:
    - postgreSQL

[[tip]]
| You must deploy Skipper first and then configure the URI location where the Skipper server runs.

Once you are ready with the relevant properties in your manifest file,
you can issue a `cf push` command from the directory where this file is
stored.

### Installing using Environment Variables

The following configuration is for Pivotal Web Services. You need to fill in `\<org>`, `\<space>`, `\<email>` and `\<password>` before running these commands.
Tasks are deployed directly from the Data Flow Server.
In the future, you will be able to launch tasks to multiple platforms. However, in 2.0, you can launch tasks to only a single
platform and the name must be `default`.

    cf set-env dataflow-server SPRING_PROFILES_ACTIVE: cloud
    cf set-env dataflow-server JBP_CONFIG_SPRING_AUTO_RECONFIGURATION: '{enabled: false}'
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_URL: https://api.run.pivotal.io
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_ORG: {org}
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_SPACE: {space}
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_DOMAIN: cfapps.io
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_USERNAME: <email>
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_PASSWORD: <password>
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_CONNECTION_SKIP_SSL_VALIDATION: true
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_SERVICES: postgreSQL
    cf set-env dataflow-server SPRING_CLOUD_DATAFLOW_TASK_PLATFORM_CLOUDFOUNDRY_ACCOUNTS[default]_DEPLOYMENT_MEMORY: 2048m

[[tip]]
| You must deploy Skipper first and then configure the URI location where the Skipper server runs.

The Spring Cloud Data Flow server defaults the remote Maven repository `https://repo.spring.io/libs-snapshot`.
This is an intentional bit of flexibility that lets you point to a remote repository of your choice.
The out-of-the-box applications that are supported by Spring Cloud Data Flow are available in Spring’s repository.
If you want to use them, set it as the remote repository, as the following example shows:

    cf set-env dataflow-server SPRING_APPLICATION_JSON '{"maven": { "remote-repositories": { "repo1": { "url": "https://repo.spring.io/libs-release" } } } }'

where `repo1` is the alias name for the remote repository.

Alternatively, you can use the `MAVEN_REMOTEREPOSITORIES[REPO1]_URL:` environment variable.

[[warning | SSL Validation]]
| Set _Skip SSL Validation_ to true only if you run on a Cloud Foundry
| instance that uses self-signed certificates (for example, in
| development). Do not use self-signed certificates for production.

[[tip | Single Sign-on Service]]
| If you deploy in an environment that requires you to sign on by using
| the Pivotal Single Sign-On Service, see
| [???](#configuration-cloudfoundry-security-sso) for information on how
| to configure the server.

You can now issue a `cf push` command and reference the Data Flow server
.jar file, as the following example shows:

    cf push dataflow-server -b java_buildpack -m 2G -k 2G --no-start -p spring-cloud-dataflow-server-{project-version}.jar
    cf bind-service dataflow-server my_postgres

### Register pre-built applications

All the pre-built streaming applications:

- Are available as Apache Maven artifacts or Docker images.
- Use RabbitMQ or Apache Kafka.
- Support monitoring via Prometheus and InfluxDB.
- Contain metadata for application properties used in the UI and code completion in the shell.

Applications can be registered individually using the `app register` functionality or as a group using the `app import` functionality.
There are also `bit.ly` links that represent the group of pre-built applications for a specific release which is useful for getting started.

You can register applications using the UI or the shell.
Even though we are only using two pre-built applications, we will register the full set of pre-built applications.

Since the Cloud Foundry installation guide uses RabbitMQ as the messaging middleware, register the RabbitMQ version of the applications.

**TODO screen shot instead of shell command**

```
app import --uri  http://bit.ly/Einstein-SR2-stream-applications-rabbit-maven
```

## Install on Local Machine

To run the server application locally (on your laptop or desktop) and target your Cloud Foundry installation, you can configure the Data Flow server by setting the following environment variables in a property file (for example, `myproject.properties`):

    spring.profiles.active=cloud
    jbp.config.spring.auto.reconfiguration='{enabled: false}'
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.url=https://api.run.pivotal.io
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.org={org}
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.space={space}
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.domain=cfapps.io
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.username={email}
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.password={password}
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].connection.skipSslValidation=false

    # The following command lets task applications write to their DB.
    # Note, however, that when the *server* runs locally, it cannot access that DB.
    # In that case, task-related commands that show executions do not work.
    spring.cloud.dataflow.task.platform.cloudfoundry.accounts[default].deployment.services=mysqlcups
    skipper.client.serverUri=https://<skipper-host-name>/api

You need to fill in `\{org}`, `\{space}`, `\{email}`, and `\{password}` before using the file in the following command.

[[warning | SSL Validation]]
| Set _Skip SSL Validation_ to true only if you run on a Cloud Foundry
| instance by using self-signed certificates (for example, in
| development). Do not use self-signed certificates for production.

[[tip]]
| You must deploy Skipper first and then configure the URI location of where the Skipper server is running.

Now you are ready to start the server application, as follows:

    java -jar spring-cloud-dataflow-server-{project-version}.jar --spring.config.additional-location=<PATH-TO-FILE>/foo.properties

**TODO bad references below**

[[tip]]
| All other parameterization options that were available when running
| the server on Cloud Foundry are still available. This is particularly
| true for [configuring defaults](#configuring-defaults) > [???](#configuring-defaults) for applications. To use them, substitute
| `cf set-env` syntax with `export`.

## Shell

The following example shows how to start the Data Flow Shell:

    $ java -jar spring-cloud-dataflow-shell-{scdf-core-version}.jar

**TODO add info on setting the url of the data flow server when starting**

## Monitoring

**TODO what to say about monitoring??**
