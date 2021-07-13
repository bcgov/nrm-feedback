### Table of Contents

<!-- TOC depthTo:1 -->

- [NRM Fider](#nrm-fider)
  - [Prerequisites](#prerequisites)
  - [Files](#files)
  - [Build](#build)
    - [Custom Image](#custom-image)
    - [Out-of-the-box Image](#out-of-the-box-image)
  - [Deploy](#deploy)
    - [Database Deployment](#database-deployment)
      - [Prepare DB for application install](#prepare-db-for-application-install)
    - [Application Deployment](#application-deployment)
      - [Log into the Fider installation](#log-into-the-fider-installation)
      - [Reset database account privileges](#reset-database-account-privileges)
  - [Example Deployment](#example-deployment)
    - [Database Deployment](#database-deployment-1)
      - [Prepare DB for application install](#prepare-db-for-application-install-1)
    - [Application Deployment](#application-deployment-1)
      - [Log into the Fider app](#log-into-the-fider-app)
  - [Using Environmental variables to deploy](#using-environmental-variables-to-deploy)
    - [Set the environment variables](#set-the-environment-variables)
    - [Database Deployment](#database-deployment-2)
      - [Prepare DB for application install](#prepare-db-for-application-install-2)
    - [Application Deployment](#application-deployment-2)
      - [Log into the Fider app](#log-into-the-fider-app-1)
  - [Integration with Platform SSO KeyCloak Shared instance](#integration-with-platform-sso-keycloak-shared-instance)
  - [FAQ](#faq)
  - [TODO](#todo)
    - [Done](#done)
  - [SMTPS Issue](#smtps-issue)

<!-- /TOC -->

# NRM Fider

OpenShift templates for customized image of [Fider](https://github.com/getfider/fider), used within Natural Resources Ministries and ready for deployment on [OpenShift](https://www.openshift.com/). [Fider](https://getfider.com/) is an open-source [Go](https://golang.org/) application with a [PostgreSQL](https://www.postgresql.org/) relational database for persistent data.

BC Gov's [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) configuration does not support [SMTPS](https://en.wikipedia.org/wiki/SMTPS), but Fider's SMTP Go library enforces smtps, so we have forked the code and removed the SMTPS authn from our deployment. This means that this image can ONLY be deployed within the BC Gov network, as only devices on the BC Gov network can see the standard BC Gov SMTP Server.

## Prerequisites

Network Security Policy is in place:

```bash
oc -n 599f0a-tools process -f https://raw.githubusercontent.com/BCDevOps/platform-services/master/security/aporeto/docs/sample/quickstart-nsp.yaml NAMESPACE=599f0a-tools | oc -n 599f0a-tools create -f -
```

For builds:

- Administrator access to an [Openshift](https://console.apps.silver.devops.gov.bc.ca/k8s/cluster/projects) Project namespace

Once built, this image may be deployed to a separate namespace with the appropriate `system:image-puller` role.

For deployments:

- Administrator access to an [Openshift](https://console.apps.silver.devops.gov.bc.ca/k8s/cluster/projects) Project namespace
- the [oc](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html) CLI tool, installed on your local workstation
- access to this public [GitHub Repo](./)

Once deployed, any visitors to the site will require a modern web browser (e.g. Chrome, Edge, FF, Opera etc.).

## Files

- [OpenShift Fider app template](/ci/openshift/fider-bcgov.dc.yaml) for Fider application
- [OpenShift Database service template](/ci/openshift/postgresql.dc.yaml) for PostgreSQL Database, acting as the datastore for the Fider application

## Build

### Custom Image

To create an image stream using this forked code (replace `<tools-namespace>` with your `-tools` project namespace).

> oc -n &lt;tools-namespace&gt; create istag fider-bcgov:latest

> oc -n &lt;tools-namespace&gt; process -f ci/openshift/fider-bcgov.bc.yaml | oc -n &lt;tools-namespace&gt; apply -f -

> oc -n &lt;tools-namespace&gt; start-build nrm-feedback

Tag with the correct release version, matching the `major.minor` release tag at the source [repo](https://github.com/getfider/fider/releases). For example:

> oc -n &lt;tools-namespace&gt; tag fider-bcgov:latest fider-bcgov:0.18.0

NOTE: To update our Fider image, we would update the submodule (e.g. `0.19.0`) and then tag _this_ build as `latest`, and plan for a re-deploy using the newer image.

> git submodule update --init --recursive

### Out-of-the-box Image

If and when we solve the SMTPS issue, we should build directly off the Fider image, referencing `https://github.com/getfider/fider` in the deployment, and no longer use custom builds.

## Deploy

### Database Deployment

Deploy the DB using the correct FEEDBACK_NAME parameter (e.g. an acronym that is prefixed to `fider`):

> oc -n &lt;project&gt; new-app --file=./ci/openshift/postgresql.dc.yaml -p FEEDBACK_NAME=&lt;feedback&gt;

All DB deployments are based on the out-of-the-box [OpenShift Database Image](https://docs.openshift.com/container-platform/3.11/using_images/db_images/postgresql.html).

#### Prepare DB for application install

Although Fider is setup to _auto-install_ upon deployment, the OpenShift DB template disallows the application account to install DB extensions. Therefore, run the the following via `oc rsh`, with the correct FEEDBACK_NAME and credentials:

> oc -n &lt;project&gt; rsh $(oc -n &lt;project&gt; get pods | grep -v -e "<feedback>-postgresql.\*-deploy" | grep Running | awk '{print $1}')

> psql ${POSTGRESQL_DATABASE} -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"

> exit

NOTE: the `${POSTGRESQL_DATABASE}` text is exactly as written, since the app has access to these environment variables (set during the `new-app` step above).

### Application Deployment

Deploy the Application specifying:

- the feedback-specific parameter (i.e. `<feedback>fider`)
- your project namespace that contains the image, and
- a `@gov.bc.ca` email account that will be used with the `apps.smtp.gov.bc.ca` SMTP Email Server:

> oc -n &lt;project&gt; new-app --file=./ci/openshift/fider-bcgov.dc.yaml -p FEEDBACK_NAME=&lt;feedback&gt;fider -p IS_NAMESPACE=&lt;tools-namespace&gt; -p EMAIL_SMTP_USERNAME=&lt;Email.Address&gt;@gov.bc.ca

#### Log into the Fider installation

After sixty seconds, the application will have finished the initial install. Open the app in a browser, to set the admin user. The URL will be of the form `https://<xyz>fider.apps.silver.devops.gov.bc.ca/`.

#### Reset database account privileges

Revoke the superuser privilege afterwards:

> oc -n &lt;project&gt; rsh $(oc -n &lt;project&gt; get pods | grep xyz-postgresql- | grep Running | awk '{print $1}')

> psql ${POSTGRESQL_DATABASE} -c "ALTER USER ${POSTGRESQL_USER} WITH NOSUPERUSER"

NOTE that the `${POSTGRESQL_DATABASE}` text is exactly as written, since the app has access to these environment variables (set during the `new-app` step).

## Example Deployment

As a concrete example of a feedback initiative with the acronym `acme`, deployed in the project namespace `599f0a-dev`, here are the steps:

<details><summary>Deployment Steps</summary>

### Database Deployment

> oc -n 599f0a-dev new-app --file=./ci/openshift/postgresql.dc.yaml -p FEEDBACK_NAME=acmefider

```bash
--> Deploying template "599f0a-dev/nrmfeedback-postgresql-dc" for "./ci/openshift/postgresql.dc.yaml" to project 599f0a-dev

     nrmfeedback-postgresql-dc
     ---------
     A new PostgreSQL application has been created in your project

     * With parameters:
        * Feedback Name=acmefider
        * Memory Limit=512Mi
        * PostgreSQL Connection Password=kTl3UhrB5cFI5hF4 # generated
        * Database Volume Capacity=2Gi

--> Creating resources ...
    secret "acmefider-postgresql" created
    persistentvolumeclaim "acmefider-postgresql" created
    deploymentconfig.apps.openshift.io "acmefider-postgresql" created
    service "acmefider-postgresql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/acmefider-postgresql'
    Run 'oc status' to view your app.

```

#### Prepare DB for application install

After thirty seconds, the database pod should be up.

> oc -n 599f0a-dev rsh $(oc -n 599f0a-dev get pods | grep -v -e "acmefider-postgresql.\*-deploy" | grep Running | awk '{print $1}')

Run the install commands in this shell:

> psql ${POSTGRESQL_DATABASE} -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"

The database responds with:

```bash
ALTER ROLE
```

Type `exit` to exit the remote shell.

### Application Deployment

> oc -n 599f0a-dev new-app --file=./ci/openshift/fider-bcgov.dc.yaml -p FEEDBACK_NAME=acmefider -p IS_NAMESPACE=599f0a-tools EMAIL_SMTP_USERNAME=Porky.Pig@gov.bc.ca

```bash
--> Deploying template "599f0a-dev/nrmf-feedback-dc" for "./ci/openshift/fider-bcgov.dc.yaml" to project 599f0a-dev

     nrmf-feedback-dc
     ---------
     A new Fider application, customized with noTLS, has been created in your project

     * With parameters:
        * Namespace=599f0a-tools
        * Image Stream=fider-bcgov
        * Version of Fider Product Feedback=0.18.0
        * Feedback Product Name=acmefider
        * Fider Go Environment=production
        * Fider application logging level=INFO
        * Fider Go Environment=TywM6VICeJ2J0jgmyn4gwS0F7km3f5sq # generated
        * SMTP Host=apps.smtp.gov.bc.ca
        * SMTP Port=25
        * SMTP User=Gary.T.Wong@gov.bc.ca
        * SMTP No-Reply Address=noreply@gov.bc.ca
        * Google SoMe AppID=
        * Google SoMe Secret=
        * FaceBook SoMe AppID=
        * FaceBook SoMe Secret=
        * GitHub SoMe AppID=
        * GitHub SoMe Secret=
        * CPU_LIMIT=500m
        * MEMORY_LIMIT=1Gi
        * CPU_REQUEST=500m
        * MEMORY_REQUEST=1Gi
        * REPLICA_MIN=2
        * REPLICA_MAX=3

--> Creating resources ...
    secret "acmefider-jwt" created
    deploymentconfig.apps.openshift.io "acmefider-app" created
    service "acmefider" created
    route.route.openshift.io "acmefider" created
--> Success
    Access your application via route 'acmefider.apps.silver.devops.gov.bc.ca'
    Run 'oc status' to view your app.
```

#### Log into the Fider app

After sixty seconds, the application will be ready to accept connections at https://acmefider.apps.silver.devops.gov.bc.ca. Initially, this redirect you to https://acmefider.apps.silver.devops.gov.bc.ca/signup, bringing you the screen:
![Admin SignUp](/docs/screenshots/SignUp.png)

Once you've confirmed the setup detail, you'll be sent a confirmation email, and you'll have to click the embedded link:
![Confirmation Email](/docs/screenshots/ConfirmEmail.png)

</details>

## Using Environmental variables to deploy

As this is a template deployment, it may be easier to set environment variable for the deployment, so using the same PROJECT of `599f0a-dev` and a new FEEDBACK project of `eao`:

<details><summary>Deployment Steps</summary>

### Set the environment variables

On a workstation logged into the OpenShift Console:

```bash
export TOOLS=599f0a-tools
export PROJECT=599f0a-dev
export FEEDBACK=eao
```

### Database Deployment

> oc -n ${PROJECT} new-app --file=./ci/openshift/postgresql.dc.yaml -p FEEDBACK_NAME=${FEEDBACK}fider

```bash
--> Deploying template "599f0a-dev/nrmfeedback-postgresql-dc" for "./ci/openshift/postgresql.dc.yaml" to project 599f0a-dev

     nrmfeedback-postgresql-dc
     ---------
     A new PostgreSQL application has been created in your project

     * With parameters:
        * Feedback Name=eaofider
        * Memory Limit=512Mi
        * PostgreSQL Connection Password=yOmOOBMbY8k3aHiy # generated
        * Database Volume Capacity=2Gi

--> Creating resources ...
    secret "eaofider-postgresql" created
    persistentvolumeclaim "eaofider-postgresql" created
    deploymentconfig.apps.openshift.io "eaofider-postgresql" created
    service "eaofider-postgresql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/eaofider-postgresql'
    Run 'oc status' to view your app.
```

#### Prepare DB for application install

After thirty seconds, the database pod should be up.

> oc -n ${PROJECT} rsh $(oc -n ${PROJECT} get pods | grep -v -e "${FEEDBACK}fider-postgresql-.-deploy" | grep Running | grep fider | awk '{print $1}')


```bash
psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"
```

The database responds with:

```bash
ALTER ROLE
```

Type `exit` to exit the remote shell.

### Application Deployment

> oc -n ${PROJECT} new-app --file=./ci/openshift/fider-bcgov.dc.yaml -p FEEDBACK_NAME=${FEEDBACK}fider -p IS_NAMESPACE=\${TOOLS} -p EMAIL_SMTP_USERNAME=Daffy.Duck@gov.bc.ca

```bash
--> Deploying template "599f0a-dev/nrmf-feedback-dc" for "./ci/openshift/fider-bcgov.dc.yaml" to project 599f0a-dev

     nrmf-feedback-dc
     ---------
     A new Fider application, customized with noTLS, has been created in your project

     * With parameters:
        * Namespace=599f0a-tools
        * Image Stream=fider-bcgov
        * Version of Fider Product Feedback=0.18.0
        * Feedback Product Name=eaofider
        * Fider Go Environment=production
        * Fider application logging level=INFO
        * Fider Go Environment=c1B5usKLsfCQE3DGeoxqS10ROlgOhtQ4 # generated
        * SMTP Host=apps.smtp.gov.bc.ca
        * SMTP Port=25
        * SMTP User=Gary.T.Wong@gov.bc.ca
        * SMTP No-Reply Address=noreply@gov.bc.ca
        * Google SoMe AppID=
        * Google SoMe Secret=
        * FaceBook SoMe AppID=
        * FaceBook SoMe Secret=
        * GitHub SoMe AppID=
        * GitHub SoMe Secret=
        * CPU_LIMIT=500m
        * MEMORY_LIMIT=1Gi
        * CPU_REQUEST=500m
        * MEMORY_REQUEST=1Gi
        * REPLICA_MIN=2
        * REPLICA_MAX=3

--> Creating resources ...
    secret "eaofider-jwt" created
    deploymentconfig.apps.openshift.io "eaofider-app" created
    horizontalpodautoscaler.autoscaling "eaofider" created
    service "eaofider" created
    route.route.openshift.io "eaofider" created
--> Success
    Access your application via route 'eaofider.apps.silver.devops.gov.bc.ca'
    Run 'oc status' to view your app.
```

#### Log into the Fider app

After sixty seconds, you may navigate to the setup page at:
https://iitdfider.apps.silver.devops.gov.bc.ca/signup

![Admin SignUp](/docs/screenshots/SignUp.png)

Once you've confirmed the setup detail, you'll be sent a confirmation email like this, and you'll have to click the link:
![Confirmation Email](/docs/screenshots/ConfirmEmail.png)

When finished, remember to unset the environment variables:

```bash
unset TOOLS PROJECT FEEDBACK
```

</details>


## Integration with Platform SSO KeyCloak Shared instance

After requesting access via https://github.com/BCDevOps/devops-requests, use the GUI to navigate to:

Administration -> Site Settings

Authentication -> Add New

```bash
OAuth2
Client ID     : dds-portfolio
Client Secret : f740feed-4780-4668-b1cf-0fa0f069ea90
Authorize URL : https://dev.oidc.gov.bc.ca/auth/realms/onestopauth-basic/protocol/openid-connect/auth
Token URL     : https://dev.oidc.gov.bc.ca/auth/realms/onestopauth-basic/protocol/openid-connect/token
Scope: openid, email, address, phone, roles
Profile API URL: (empty)
JSON PATH
	ID: 	sub
	Name: 	name
	Email: email	
```

## FAQ

- To login into the database, open the DB pod terminal (via OpenShift Console or oc rsh) and enter:

```bash
psql -U ${POSTGRESQL_USER} ${POSTGRESQL_DATABASE}
```

- to clean-up database deployments:

  To re-deploy _just_ the database, first idle the database service and then delete the deployed objects from the last run, with the correct FEEDBACK_NAME, such as:

  ```bash
  oc -n <project> idle <feedback>fider-postgresql
  oc -n <project> delete secret/<feedback>fider-postgresql svc/<feedback>fider-postgresql pvc/<feedback>fider-postgresql dc/<feedback>fider-postgresql
  ```

  NOTE: This resets the persistent database, so you will lose all your Fider data. If you wish to only redeploy the DB runtime but not reset the data, then omit the `pvc/<feedback>fider-postgresql` object from the above command.

  or if using environment variables:

```bash
  oc -n ${PROJECT} idle ${FEEDBACK}fider-postgresql
  oc -n ${PROJECT} delete secret/${FEEDBACK}fider-postgresql svc/${FEEDBACK}fider-postgresql pvc/${FEEDBACK}fider-postgresql dc/${FEEDBACK}fider-postgresql
```

- to clean-up application deployments:

```bash
  oc -n <project> delete dc/<feedback>fider-app svc/<feedback>fider route/<feedback>fider secret/<feedback>fider-jwt hpa/<feedback>fider
```

Or if using environment variables:

```bash
  oc -n ${PROJECT} delete dc/${FEEDBACK}fider-app svc/${FEEDBACK}fider route/${FEEDBACK}fider secret/${FEEDBACK}fider-jwt hpa/${FEEDBACK}fider
```

- to reset _all_ deployed objects (this will destroy all data and persistent volumes). Only do this on a botched initial install or if you have the DB backed up and ready to restore into the new wiped database.

  `oc -n <project> delete all,secret,pvc,hpa -l app=<feedback>fider`

  or if using environment variables:

  `oc -n ${PROJECT} delete all,secret,pvc,hpa -l app=${FEEDBACK}fider`

- Git SubModule was created via:

```bash
git submodule add https://github.com/getfider/fider
```

    NOTE: In hindsight, we should've explicitly tracked the master branch

    > git submodule add -b master https://github.com/getfider/fider

- To update this LimeSurvey git submodule from the upstream repo, from root of repo:

```bash
  git submodule update --remote
```

## TODO

- document KeyCloak integration
- test out application upgrade (e.g. Fider updates their version)
- check for image triggers which force a reploy (image tags.. latest -> v0.19.0)

### Done

- health checks for application containers
- appropriate resource limits (multi-replica pods supported)
- created fider-bcgov.bc.json file
- integrated with apps.smtp.gov.bc.ca:25 without TLS (e.g. x509 error due to vwall.gov.bc.ca on cert)
- health checks for each of the database container
- convert ci/openshift/_.json to _.yaml
- tested DB backup/restore and transfer with [Backup-Containers](https://github.com/BCDevOps/backup-container)

## SMTPS Issue

1. When configured with apps.smtp.gov.bc.ca

```bash
ERROR [2019-11-30T02:54:11Z] [WEB] [6N0EEiyyCVMpNWxD3w9N7J6XjVnlFT6Y] Error Trace:
- app/handlers/apiv1/invite.go:31
- failed to send email with template invite_email (app/pkg/email/smtp/smtp.go:104)
- x509: certificate is valid for vwall.gov.bc.ca, www.vwall.gov.bc.ca, not apps.smtp.gov.bc.ca
```

2. When using straight non-TLS STMP connection

```bash
ERROR [2019-11-30T03:17:59Z] [WEB] [7g7GXPSR57065JvQYioR9nvj7Z7Zy0pJ] Error Trace:
- app/handlers/apiv1/invite.go:31
- failed to send email with template invite_email (app/pkg/email/smtp/smtp.go:104)
- dial tcp 74.125.20.108:587: connect: connection timed out
```

I suspect `apps.smtp.gov.bc.ca` starts the TLS handshake which tells Fider SMTP utility to expect a properly configured SSL Certificate.
