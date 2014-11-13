---
layout: docs
title: Deploying to remote servers with AppVeyor Deployment Agent
---

# Deploying to remote servers with AppVeyor Deployment Agent

AppVeyor Deployment Agent (Deployment Agent) is a service running on remote server and helping to deploy select artifact as IIS website or Windows application/service.

### Table of contents

* [Software requirements](#software-requirements)
* [Installing AppVeyor Deployment Agent](#installing)
* [Unattended Deployment Agent installation](#unattended-installation)
* [What artifacts can be deployed](#deployable-artifacts)
* [How to get named artifacts](#named-artifacts)
* [Configuring deployment settings](#deployment-settings)
* [Deploying artifact package as IIS web site](#deploying-website)
* [Deploying artifact package as a Windows application](#deploying-windows-app)
* [Deploying artifact package as a Windows service](#deploying-windows-service)
* [Running PowerShell scripts on target server during deployment](#running-powershell)
* [Calling script block once per deployment](#calling-script-once-per-deployment)
* [Troubleshooting](#troubleshooting)


<a id="software-requirements"></a>
## Software requirements

The following is required on the server to run Deployment Agent:

- Windows Server 2012 (Windows 8) or newer
- .NET Framework 4.5.1 or newer
- Web Role (IIS) is installed if you are deploying web site


<a id="installing"></a>
## Installing AppVeyor Deployment Agent

1. Add new environment with **Agent** provider selected. Open environment settings and copy **Environment access key**.

2. [Download Deployment Agent]({{site.url}}/downloads/AppveyorDeploymentAgent.msi) (v2.1.142 from 10/04/2014)

3. Specify **Environment access key** during Deployment Agent installation.

4. Server is ready for deployment.


<a id="unattended-installation"></a>
## Unattended Deployment Agent installation

Run the following in PowerShell console:

    (new-object net.webclient).DownloadFile('http://appveyor-ftp.azurewebsites.net/AppveyorDeploymentAgent.msi', 'AppveyorDeploymentAgent.msi')
    msiexec /i AppveyorDeploymentAgent.msi /quiet /qn /norestart /log install.log ENVIRONMENT_ACCESS_KEY=<your_access_key>

> Replace `<your_access_key>` with your environment access key.


<a id="deployable-artifacts"></a>
## What artifacts can be deployed

Deployment Agent recognizes artifacts of two types which may contain either web application or Windows application/service:

* **Zip archive**
* **Web Deploy package**

To be deployable with Deployment Agent *artifact must have a name*. Name should not have any spaces. All unnamed artifacts are skipped by Deployment Agent provider.



<a id="named-artifacts"></a>
## How to get named artifacts

There are few possible ways of packaging artifact deployable by Agent:

1. When **Package Web Application projects** option is enabled on **Build** tab of project settings AppVeyor automatically publishes (applies web config transforms)
   and uploads VS.NET Web Application projects as artifacts named after the name of VS.NET project.

2. Specify **Deployment name** while adding artifact entry on **Artifacts** tab of project settings.

3. From script, for example **After build script**:

    `appveyor PushArtifact <zip_path> -Name MyApp`


<a id="deployment-settings"></a>
## Configuring deployment settings

Use **Provider settings** of Agent environment to configure *which* artifacts should be deployed by Agent and *how*. By default, nothing configured - nothing deployed.

Settings have format `<artifact_name>.<setting_name>` where `<artifact_name>` is artifact's **Deployment name**.

For example, let the build has the following artifacts:

![Artifacts](/site/docs/images/agent-deploy-artifacts.png)

In order for Deployment Agent to deploy that artifact as IIS web site **Provider settings** will be:

![Artifacts](/site/docs/images/agent-provider-settings.png)



<a id="deploying-website"></a>
## Deploying artifact package as IIS web site

    <artifact_name>.deploy_website: true

Other settings:

* `site_name` - The name of existing or new website, e.g. "Default Web Site".

* `application_name` - optional web application name (IIS virtual directory) to deploy web app into.

* `port` - Port of website binding.

* `ip` - IP address of website binding.

* `hostname` - Host header value of website binding.

* `write_access` - When set to `true` Agent sets **Modify** permissions for application pool identity on website root directory.

* `path` - Website root directory on the target server. If not specified and website already exists its root directory is not changed.
If not specified and website does not exists default directory path is `c:\appveyor\applications\<artifact_name>`.

* `remove_files` - Agent uses Web Deploy to synchronize website folder contents. By default, it only adds new files and modifies existing.
When `remove_files` is set to `true` Agent performs full content synchronization, i.e. deletes files at destination that don't exist in the package.

* `group` - Deployment group.

> When deploying web app from Web Deploy package you can use [Web Deploy parametrization](/docs/deployment/web-deploy#web-deploy-parametrization) with environment variables.

You can specify multiple bindings in `hostname`, `ip` and `port` separated by semi-colon. Below is an example of how 3 bindings can be configured:

- *:80:my-test101.com
- *:80:www.my-test101.com
- *:8090:

![agent-multiple-bindings](/site/docs/deployment/images/agent/agent-multiple-bindings.png)


<a id="deploying-windows-app"></a>
## Deploying artifact package as a Windows application

    <artifact_name>.deploy_app: true

Other properties:

* `path` - Application root directory. If not specified default application path is `c:\appveyor\applications\<artifact_name>`.

* `remove_files` - Remove additional files at destination.

* `group` - Deployment group.



<a id="deploying-windows-service"></a>
## Deploying artifact package as a Windows service

    <artifact_name>.deploy_service: true

Other properties:

* `service_executable` - File name of Windows service executable, e.g. `myapp.service.exe`. If not specified the first executable found in application directory will be used.

* `service_name` - The name of Windows service. If specified Windows service will be created.

* `service_display_name` - Display name of Windows service. If not specified `service_name` will be used.

* `path` - Application root directory. If not specified default application path is `c:\appveyor\applications\<artifact_name>`.

* `remove_files` - Remove additional files at destination.

* `group` - Deployment group.



<a id="running-powershell"></a>
## Running PowerShell scripts on target server during deployment

`before-deploy.ps1` PowerShell script in the root of application folder will be called **before** every succssful deployment.
`deploy.ps1` PowerShell script in the root of application folder will be called **after** every succssful deployment.

During scripts execution the following environment variables are available:

Job details:

* `APPVEYOR` - script runs in AppVeyor environment
* `CI` - script runs in AppVeyor environment
* `APPVEYOR_PROJECT_ID` - Unique system ID of project
* `APPVEYOR_PROJECT_NAME` - Project display name
* `APPVEYOR_PROJECT_SLUG` - Project slug that you can see in URL, e.g. myproject-123
* `APPVEYOR_BUILD_ID` - Unique system ID of build
* `APPVEYOR_BUILD_NUMBER` - Build number of deploying artifact
* `APPVEYOR_BUILD_VERSION` - Build version on deploying artifact
* `APPVEYOR_JOB_ID` - Unique system ID of deployment job
* `APPVEYOR_REPO_NAME` - Repository name in the form `owner-name/repo-name`
* `APPVEYOR_REPO_BRANCH` - Build branch
* `APPVEYOR_REPO_COMMIT` - Build commit ID (SHA)
* `APPVEYOR_REPO_COMMIT_AUTHOR` - Commit author
* `APPVEYOR_REPO_COMMIT_AUTHOR_EMAIL` - Commit author's email
* `APPVEYOR_REPO_COMMIT_TIMESTAMP` - Commit timestamp
* `APPVEYOR_REPO_COMMIT_MESSAGE` - Commit message

Application details:

* `APPLICATION_NAME` - application name (artifact deployment name)
* `APPLICATION_DEPLOY_WEBSITE` - `true` if artifact deployed as IIS web site
* `APPLICATION_PATH` - application or website root folder
* `APPLICATION_REMOVE_FILES` - perform full sync of package/application folder contents, i.e. remove additional files at destination
* `APPLICATION_SITE_NAME` - IIS web site name

Artifact details:

* `ARTIFACT_FILENAME` - artifact file name as in cloud storage
* `ARTIFACT_LOCALPATH` - local file name of downloaded artfact package
* `ARTIFACT_NAME` - artifact deployment name
* `ARTIFACT_SIZE` - package size in bytes
* `ARTIFACT_TYPE` - artifact type
* `ARTIFACT_URL` - artifact package download URL valid for 10 minutes


<a id="calling-script-once-per-deployment"></a>
### Calling script block once per deployment

<!--[TBD - explain some usage scenarios]-->

In your `before deploy.ps1` or `deploy.ps1` use the following code to run once per deployment/per cluster:

    if(Enter-OncePerDeployment "block_name")
    {
        # your code that must be run once per cluster
    }

> Replace `block_name` with some value identifying operations inside the block, e.g. "install_sql"


<a id="troubleshooting"></a>
## Troubleshooting

Open Event Viewer, expand **Applications and Services Logs** node and navigate to **Deployment Agent** event log.