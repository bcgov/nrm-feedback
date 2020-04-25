### Table of Contents

<!-- TOC depthto:2 -->

- [NRM Fider](#nrm-fider)
  - [Prerequisites](#prerequisites)
  - [Files](#files)
  - [Build](#build)
  - [Deploy](#deploy)
  - [Perform Fider installation](#perform-fider-installation)
  - [Example Deployment](#example-deployment)
  - [Using Environmental variables to deploy](#using-environmental-variables-to-deploy)
  - [FAQ](#faq)
  - [TODO](#todo)
  - [SMTPS Issue](#smtps-issue)

<!-- /TOC -->


# NRM Fider

OpenShift templates for customized image of [Fider](https://github.com/getfider/fider), used within Natural Resources Ministries and ready for deployment on [OpenShift](https://www.openshift.com/).  [Fider](https://getfider.com/) is an open-source [Go](https://golang.org/) application with a [PostgreSQL](https://www.postgresql.org/) relational database for persistent data. 

BC Gov's [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) configuration does not support [SMTPS](https://en.wikipedia.org/wiki/SMTPS), but Fider's SMTP Go library enforces smtps, so we have forked the code and removed the SMTPS authn from our deployment.  This means that this image can ONLY be deployed within the BC Gov network, as only devices on the BC Gov network can see the standard BC Gov SMTP Server.

## Prerequisites

For builds:

* [Project Administrator](https://docs.openshift.com/container-platform/3.11/architecture/additional_concepts/authorization.html#roles) access to an [Openshift](https://console.pathfinder.gov.bc.ca:8443/console/catalog) Project namespace

Once built, this image may be deployed to a separate namespace with the appropriate `system:image-puller` role. 

For deployments:

* [Project Administrator](https://docs.openshift.com/container-platform/3.11/architecture/additional_concepts/authorization.html#roles) access to an [Openshift](https://console.pathfinder.gov.bc.ca:8443/console/catalog) Project namespace
* the [oc](https://docs.openshift.com/container-platform/3.11/cli_reference/get_started_cli.html) CLI tool, installed on your local workstation
* access to this public [GitHub Repo](./)

Once deployed, any visitors to the site will require a modern web browser (e.g. IE11, Edge, FF, Chrome, Opera etc.).

## Files

* [OpenShift Fider app template](/openshift/fider.dc.json) for Fider application
* [OpenShift Database service template](/openshift/postgresql.dc.json) for PostgreSQL Database, acting as the datastore for the Fider application

## Build

### Custom Image

To create an image stream using this forked code (replace `<tools-namespace>` with your `-tools` project namespace).

> oc -n &lt;tools-namespace&gt;  process -f openshift/fider.bc.json | oc -n &lt;tools-namespace&gt;  apply -f -

Tag with the correct release version,  matching the `major.minor`  release tag at the source [repo](https://github.com/getfider/fider/releases).  For example:

> oc -n &lt;tools-namespace&gt; tag fider-notls:latest fider-notls:0.18.0

### Out-of-the-box Image

If and when we solve the SMTPS issue, we should build directly off the Fider image, for example:

1. Import a specific tagged release:

> oc -n &lt;tools-namespace&gt; import-image fider:0.18.0 --from=getfider/fider:0.18.0 --confirm --insecure --reference-policy=local

2. Once imported with the correct release number, tag it to explictly show the version:

> oc -n &lt;tools-namespace&gt; tag fider:latest fider:0.18.0

NOTE: To remove tags, the syntax is:

```bash
oc -n <tools-namespace> tag -d fider:latest
oc -n <tools-namespace> tag -d fider:v0.18.0
```

NOTE: To update our Fider image, we would import the new version (e.g. `0.19.0`) and then tag *this* as `latest`, and plan for a re-deploy using the newer image.

## Deploy

### Database Deployment

Deploy the DB using the correct FEEDBACK_NAME parameter (e.g. an acronym that is prefixed to `fider`):

> oc -n &lt;project&gt; new-app --file=openshift/postgresql.dc.json -p FEEDBACK_NAME=&lt;feedback&gt;

All DB deployments are based on the out-of-the-box [OpenShift Database Image](https://docs.openshift.com/container-platform/3.11/using_images/db_images/postgresql.html).

### Application Deployment

Deploy the Application specifying:   
* the feedback-specific parameter (i.e. `<feedback>fider`)
* your project namespace that contains the image, and 
* a `@gov.bc.ca` email account that will be used with the `apps.smtp.gov.bc.ca` SMTP Email Server:

> oc -n &lt;project&gt; new-app --file=./openshift/fider.dc.json -p FEEDBACK_NAME=&lt;feedback&gt;fider -p IS_NAMESPACE=&lt;tools-namespace&gt; EMAIL_SMTP_USERNAME=&lt;Email.Address&gt;@gov.bc.ca

## Perform Fider installation

Although Fider is setup to *auto-install* upon deployment, the OpenShift DB template disallows the application account to install DB extensions.  Therefore, run the the following via `oc rsh`, with the correct FEEDBACK_NAME and credentials:

> oc -n &lt;project&gt; rsh $(oc -n &lt;project&gt; get pods | grep &lt;feedback&gt;fider-postgresql- | grep Running | awk '{print $1}')

> psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"

NOTE that the `${POSTGRESQL_DATABASE}` text is exactly as written, since the app has access to these environment variables (set during the `new-app` step above).

### Log into the Fider installation

Once the application has finished the initial install, open the app in a browser, to set the admin user.  The URL will be of the form `https://<xyz>fider.pathfinder.gov.bc.ca/`.

### Reset database account privileges

Revoke the superuser privilege afterwards:

> oc -n &lt;project&gt; rsh $(oc -n &lt;project&gt; get pods | grep xyz-postgresql- | grep Running | awk '{print $1}')

> psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH NOSUPERUSER"

NOTE that the `${POSTGRESQL_DATABASE}` text is exactly as written, since the app has access to these environment variables (set during the `new-app` step).

## Example Deployment

As a concrete example of a feedback initiative with the acronym `acme`, deployed in the project namespace `csnr-devops-lab-deploy`, here are the steps:

<details><summary>Deployment Steps</summary>

### Database Deployment

> oc -n csnr-devops-lab-deploy new-app --file=openshift/postgresql.dc.json -p FEEDBACK_NAME=acmefider

```bash
--> Deploying template "csnr-devops-lab-deploy/nrmfeedback-postgresql-dc" for "openshift/postgresql.dc.json" to project csnr-devops-lab-deploy

     * With parameters:
        * Feedback Name=acmefider
        * Memory Limit=512Mi
        * PostgreSQL Connection Password=wAktZce2pIt5O8Y3 # generated
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

### Application Deployment

> oc -n csnr-devops-lab-deploy new-app --file=./openshift/fider.dc.json -p FEEDBACK_NAME=acmefider -p IS_NAMESPACE=csnr-devops-lab-tools EMAIL_SMTP_USERNAME=Porky.Pig@gov.bc.ca

```bash
--> Deploying template "csnr-devops-lab-deploy/nrmf-feedback-dc" for "./openshift/fider.dc.json" to project csnr-devops-lab-deploy

     * With parameters:
        * Namespace=csnr-devops-lab-tools
        * Image Stream=fider-notls
        * Version of Fider Product Feedback=0.18.0
        * Feedback Product Name=acmefider
        * Fider Go Environment=production
        * Fider application logging level=INFO
        * Fider Go Environment=3KtO32hUVd1VbpAgudiskTNekps3jtzb # generated
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
        * CPU_REQUEST=100m
        * MEMORY_REQUEST=256Mi
        * REPLICA_MIN=2
        * REPLICA_MAX=2

--> Creating resources ...
    secret "acmefider-jwt" created
    deploymentconfig.apps.openshift.io "acmefider-app" created
    horizontalpodautoscaler.autoscaling "acmefider" created
    service "acmefider" created
    route.route.openshift.io "acmefider" created
--> Success
    Access your application via route 'acmefider.pathfinder.gov.bc.ca'
    Run 'oc status' to view your app.
```

#### Perform Fider installation

After 10 - 30 seconds, at least one pod should be up; remote into any one of these pods:

> oc -n csnr-devops-lab-deploy rsh $(oc -n csnr-devops-lab-deploy get pods | grep acmefider-postgresql- | grep Running | awk '{print $1}')

Run the install commands in this shell:

> psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"

The database responds with:
```bash
ALTER ROLE
```

Type `exit` to exit the remote shell.

### Log into the Fider app

The app is at https://acmefider.pathfinder.gov.bc.ca but will initially redirect you to https://acmefider.pathfinder.gov.bc.ca/signup, bringing you the screen:
![Admin SignUp](/docs/images/SignUp.png)

Once you've confirmed the setup detail, you'll be sent a confirmation email, and you'll have to click the embedded link:
![Confirmation Email](/docs/images/ConfirmEmail.png)
</details>

## Using Environmental variables to deploy

As this is a template deployment, it may be easier to set environment variable for the deployment, so using the same PROJECT of `csnr-devops-lab-deploy` and a new FEEDBACK project of `iitd`:

<details><summary>Deployment Steps</summary>

### Set the environment variables

On a workstation logged into the OpenShift Console:

```bash
export TOOLS=csnr-devops-lab-tools
export PROJECT=csnr-devops-lab-deploy
export FEEDBACK=iitd
```

### Database Deployment

> oc -n ${PROJECT} new-app --file=openshift/postgresql.dc.json -p FEEDBACK_NAME=${FEEDBACK}fider

```bash
--> Deploying template "csnr-devops-lab-deploy/nrmfeedback-postgresql-dc" for "openshift/postgresql.dc.json" to project csnr-devops-lab-deploy

     * With parameters:
        * Feedback Name=iitdfider
        * Memory Limit=512Mi
        * PostgreSQL Connection Password=In0Yb6djOMHESKh5 # generated
        * Database Volume Capacity=2Gi

--> Creating resources ...
    secret "iitdfider-postgresql" created
    persistentvolumeclaim "iitdfider-postgresql" created
    deploymentconfig.apps.openshift.io "iitdfider-postgresql" created
    service "iitdfider-postgresql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/iitdfider-postgresql' 
    Run 'oc status' to view your app.
```

### App Deployment

> oc -n ${PROJECT} new-app --file=./openshift/fider.dc.json -p FEEDBACK_NAME=${FEEDBACK}fider -p IS_NAMESPACE=${TOOLS} EMAIL_SMTP_USERNAME=Daffy.Duck@gov.bc.ca

```bash
--> Deploying template "csnr-devops-lab-deploy/nrmf-feedback-dc" for "./openshift/fider.dc.json" to project csnr-devops-lab-deploy

     * With parameters:
        * Namespace=csnr-devops-lab-tools
        * Image Stream=fider-notls
        * Version of Fider Product Feedback=0.18.0
        * Feedback Product Name=iitdfider
        * Fider Go Environment=production
        * Fider application logging level=INFO
        * Fider Go Environment=a7e8XLIJ57S2JR1vy7Nx5T73yfBk7ICN # generated
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
        * CPU_REQUEST=100m
        * MEMORY_REQUEST=256Mi
        * REPLICA_MIN=2
        * REPLICA_MAX=2

--> Creating resources ...
    secret "iitdfider-jwt" created
    deploymentconfig.apps.openshift.io "iitdfider-app" created
    horizontalpodautoscaler.autoscaling "iitdfider" created
    service "iitdfider" created
    route.route.openshift.io "iitdfider" created
--> Success
    Access your application via route 'iitdfider.pathfinder.gov.bc.ca' 
    Run 'oc status' to view your app.
```

#### Perform Fider installation

After 10 - 30 seconds, at least one pod should be up; remote into any one of these pods:

> oc -n ${PROJECT} rsh $(oc -n ${PROJECT} get pods | grep ${FEEDBACK}fider-postgresql- | grep Running | awk '{print $1}')

```bash
sh-4.2$ psql ${POSTGRESQL_DATABASE}  -c "ALTER USER ${POSTGRESQL_USER} WITH SUPERUSER"
ALTER ROLE
```
Type `exit` to exit the remote shell.

### Log into the Fider app

The setup page is at:
https://${FEEDBACK}fider.pathfinder.gov.bc.ca/signup

![Admin SignUp](/docs/images/SignUp.png)

Once you've confirmed the setup detail, you'll be sent a confirmation email like this, and you'll have to click the link:
![Confirmation Email](/docs/images/ConfirmEmail.png)

When finished, remember to unset the environment variables:

```bash
unset TOOLS PROJECT FEEDBACK
```

</details>


## FAQ

* To login into the database, open the DB pod terminal (via OpenShift Console or oc rsh) and enter:

```bash
psql -U ${POSTGRESQL_USER} ${POSTGRESQL_DATABASE}
```

* to clean-up database deployments:

  To re-deploy *just* the database, first idle the dateabase service and then delete the deployed objects from the last run, with the correct FEEDBACK_NAME, such as:

  ```bash
  oc -n <project> idle <feedback>fider-postgresql
  oc -n <project> delete secret/<feedback>fider-postgresql svc/<feedback>fider-postgresql pvc/<feedback>fider-postgresql dc/<feedback>fider-postgresql
  ```

  NOTE: This resets the persistent database, so you will lose all your Fider data.  If you wish to only redeploy the DB runtime but not reset the data, then omit the `pvc/<feedback>fider-postgresql` object from the above command.

  or if using environment variables:

```bash
  oc -n ${PROJECT} idle ${FEEDBACK}fider-postgresql
  oc -n ${PROJECT} delete secret/${FEEDBACK}fider-postgresql svc/${FEEDBACK}fider-postgresql pvc/${FEEDBACK}fider-postgresql dc/${FEEDBACK}fider-postgresql
```

* to clean-up application deployments:

```bash
  oc -n <project> delete dc/<feedback>fider-app svc/<feedback>fider route/<feedback>fider secret/<feedback>fider-jwt hpa/<feedback>fider
```

Or if using environment variables:

```bash
  oc -n ${PROJECT} delete dc/${FEEDBACK}fider-app svc/${FEEDBACK}fider route/${FEEDBACK}fider secret/${FEEDBACK}fider-jwt hpa/${FEEDBACK}fider
```

* to reset *all* deployed objects (this will destroy all data and persistent volumes).  Only do this on a botched initial install or if you have the DB backed up and ready to restore into the new wiped database.

  `oc -n <project> delete all,secret,pvc,hpa -l app=<feedback>fider`

  or if using environment variables:

  `oc -n ${PROJECT} delete all,secret,pvc,hpa -l app=${FEEDBACK}fider`


* Git SubModule was created via:

```bash
git submodule add https://github.com/getfider/fider
cd fider
git apply ../bcgov-notls.patch 
```

  * To update this LimeSurvey git submodule from the upstream repo, from root of repo:

```
  git submodule update --remote 
```

## TODO

* test DB backup/restore and transfer
* convert ./openshift/*.json to *.yml
* test out application upgrade (e.g. Fider updates their version)
* check for image triggers which force a reploy (image tags.. latest -> v0.19.0)

### Done

* health checks for application containers
* appropriate resource limits (multi-replica pods supported)
* created fider.bc.json file
* integrated with apps.smtp.gov.bc.ca:25 without TLS (e.g. x509 error due to vwall.gov.bc.ca on cert)
* health checks for each of the database container

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
