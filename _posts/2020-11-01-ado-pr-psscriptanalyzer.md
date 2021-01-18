---
title: Azure DevOps PowerShell code analysis with Pull Request comments
author: Andrew Campbell
date: 2020-11-01
categories: []
tags: [AzureDevOps, PowerShell, PullRequest, CICD Pipeline]
pin: false
image: /assets/img/ado-analyzer/ado-analyzer-social2.png
description: Get static code analysis feedback on your PowerShell code hosted in Azure DevOps, using pipelines to create inline file comments within your pull requests.
---

## Overview

If you are using PowerShell you should be storing your code in a code repository. [GitHub](https://github.com){:target="_blank"} and [Azure DevOps](https://dev.azure.com){:target="_blank"} are great choices, but really any git repo will do. This isn't an article about why you should be using version control, I feel that's been covered at great length in the community already.

This article is going to walk through leveraging Azure DevOps to perform static code analysis using [PSScriptAnalyzer](https://github.com/PowerShell/PSScriptAnalyzer){:target="_blank"} as part of a Pull Request workflow. Our pull request will initiate PSScriptAnalyzer, gather the results, and post each issue as a comment on the offending line of code in the pull request.

The file and folder structure can be modified to suit the needs of any other repo, but for the purposes of this article we'll be using the following:

* `/azure-pipelines.yml`: The Azure DevOps pipeline yaml code; this installs PSScriptAnalyzer on the hosted agent, then runs our custom script to analyze a directory and post comments back to the Pull Request which initiated the pipeline.
* `/tests/Start-PSScriptAnalyzer.ps1`: The custom PowerShell script which runs PSScriptAnalyzer, performs a for-each loop through the results, and then posts a comment on the specific file and line where the issue was detected.
* `/scripts/example-script.ps1`: A sample script with a few errors that will cause PSScriptAnalyzer to produce warnings.

These files can all be downloaded from my GitHub repo here: [https://github.com/acampb/azuredevops-psscriptanalyzer-prcomments](https://github.com/acampb/azuredevops-psscriptanalyzer-prcomments){:target="_blank"}

---

## Create the Azure DevOps Pipeline

Start by adding the `Start-PSScriptAnalyzer.ps1` script to a new directory in your repo named `tests`. Grab the file from my GitHub repo above, or copy the code block below:

```powershell
[CmdletBinding()]
param (
    # Directory where PowerShell scripts to be tested are stored. Use a relative path like '../scripts'. Script Analyzer will recurse through subdirectories as well
    [Parameter(Mandatory = $true)]
    [string]
    $ScriptDirectory,

    # Comma separated list of specific PSScriptAnalyzer rules to exclude
    [Parameter(Mandatory = $false)]
    [string]
    $ScriptAnalyzerExcludeRules

)

function Add-PRComment {
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true)]
    [string]
    $Body
)
    Write-Verbose "Posting PR Comment via AzureDevOps REST API"

    # post the comment to the pull request
    try {
        $uri = "$($Env:SYSTEM_TEAMFOUNDATIONCOLLECTIONURI)$Env:SYSTEM_TEAMPROJECTID/_apis/git/repositories/$($Env:BUILD_REPOSITORY_NAME)/pullRequests/$($Env:SYSTEM_PULLREQUEST_PULLREQUESTID)/threads?api-version=5.1"
        Write-Verbose "Constructed URL: $uri"

        $response = Invoke-RestMethod -Uri $uri -Method POST -Headers @{Authorization = "Bearer $Env:SYSTEM_ACCESSTOKEN" } -Body $Body -ContentType application/json

        if ($null -eq $response) {
            Write-Verbose "Rest API posted OK"
        }
    }
    catch {
        Write-Error $_
        Write-Error $_.Exception.Message
    }
}

$ScriptAnalyzerRules = Get-ScriptAnalyzerRule -Severity Error, Warning, Information
$ScriptAnalyzerResult = Invoke-ScriptAnalyzer -Path $ScriptDirectory -Recurse -IncludeRule $ScriptAnalyzerRules -ExcludeRule $ScriptAnalyzerExcludeRules

if ( $ScriptAnalyzerResult ) {
    $ScriptAnalyzerResultString = $ScriptAnalyzerResult | Out-String
    Write-Warning $ScriptAnalyzerResultString

    # loop through each result and post to the azuredevops rest api
    foreach ($result in $ScriptAnalyzerResult) {
        # build the script path for the PR comment, drop the workdir from the path
        $ScriptPath = $result.ScriptPath -replace [regex]::Escape($Env:SYSTEM_DEFAULTWORKINGDIRECTORY), ""
        Write-Verbose "ScriptPath: $ScriptPath"
        Write-Verbose "Line Number: $($result.Line)"
        Write-Verbose "Message: $($result.Message)"


        # build the markdown comments
        # cannot be tabbed over to match indentation
        $markdownComment = @"
:warning: Script Analyzer found this issue with your code:

``$($result.Message)``
"@


        $body = @"
{
    "comments": [
        {
            "parentCommentId": 0,
            "content": "$markdownComment",
            "commentType": 1
        }
    ],
    "status": "active",
    "threadContext": {
        "filePath": "$ScriptPath",
        "leftFileEnd": null,
        "leftFileStart": null,
        "rightFileEnd": {
            "line": $($result.Line),
            "offset": 100
        },
        "rightFileStart": {
            "line": $($result.Line),
            "offset": 1
        }
    }
}
"@
        # post to the PR
        Add-PRComment -Body $body
    }

    throw "PSScriptAnalyzer found issues with your code"

} else {
    Write-Output "All Script Analyzer tests passed"

    $markdownComment = @"
:white_check_mark: Script Analyzer found no issues with your code! High Five! :hand:
"@

    Write-Verbose "Posting PR Comment via AzureDevOps REST API"

    $body = @"
{
    "comments": [
        {
            "parentCommentId": 0,
            "content": "$markdownComment",
            "commentType": 1
        }
    ],
    "status": "closed"
}
"@
    # post to the PR
    Add-PRComment -Body $body
}

```

![add-analyzer-script](/assets/img/ado-analyzer/add-analyzer-script.png)

I'm committing my changes directly to the `master` branch in the screenshot above. This is generally a bad practice; and later we'll setup a branch policy to prevent this from happening, forcing updates to `master` to occur through the pull request process.

For demonstrating the pipeline and PSScriptAnalyzer I've created an example script with several errors intentionally included so we can see PSScriptAnalyzer generate some warnings. If you want to use this file in your repo you can grab it here: [https://github.com/acampb/azuredevops-psscriptanalyzer-prcomments/blob/main/scripts/example-script.ps1](https://github.com/acampb/azuredevops-psscriptanalyzer-prcomments/blob/main/scripts/example-script.ps1){:target="_blank"}

Now let's create our Azure DevOps Pipeline yaml file (`azure-pipelines.yml`) in the root of our repo. Use the code block below, or grab the file directly from the GitHub repo.

```yaml
trigger:
  none

jobs:
- job: 'PSScriptAnalyzer'
  displayName: PSScriptAnalyzer
  pool:
    vmImage: 'ubuntu-latest'

  steps:
    - task: PowerShell@2
      displayName: 'Install ScriptAnalyzer'
      inputs:
        targetType: inline
        pwsh: true
        script: |
          Install-Module -Name 'PSScriptAnalyzer' -Scope 'CurrentUser' -Force
    - task: PowerShell@2
      displayName: 'Analyze and post PR Comment'
      env:
          SYSTEM_ACCESSTOKEN: $(System.AccessToken)
      inputs:
        targetType: filePath
        pwsh: true
        filePath: $(Build.Repository.LocalPath)/tests/Start-PSScriptAnalyzer.ps1
        arguments: '-ScriptDirectory $(Build.Repository.LocalPath)/scripts'
```

> Note: You may need to adjust the `filePath` parameter to where you stored the `Start-PSScriptAnalyzer.ps1` script, and the `arguments` to the directory where your PowerShell scripts are located.

![add-azure-pipelines](/assets/img/ado-analyzer/add-azure-pipelines.png)

Commit this file to the repo as well.

Simply creating the `azure-pipelines.yml` file does not actually create the Pipeline, we've just stored a yaml file in our repo. Follow these steps to create the Pipeline using our existing `azure-pipelines.yml` file.

![create-pipeline](/assets/img/ado-analyzer/create-pipeline.gif)

1. Click `Pipelines`
2. Click `Create Pipeline`
3. Select `Azure Repos Git` for code repo location
4. Select your git repo
5. Azure DevOps will most likely detect our `azure-pipelines.yml` file, however you may need to specify the file to use if you used a different file name, or have existing pipelines configured.
6. Do not run the pipeline, click the down arrow and click 'Save'
7. The default pipeline name will be based on the Repo name, click `...` and `Rename/move`, and change the name to **PSScriptAnalyzer**

> Note: The pipeline will not run correctly if it is started outside of a Pull Request. This is due to the way the `Start-PSScriptAnalyzer.ps1` script is written; it requires the pull request id to properly post a comment via the API.

---

## Configure Build Service Permissions

When the pipeline executes our `Start-PSScriptAnalyzer.ps1` script it is going to be running under the identity of the 'Build Service' user, and will attempt connect to the Azure DevOps REST API to add comments to our Pull Request. There are two pre-requisites for this to work correctly:

* **Access Token**: The `Start-PSScriptAnalyzer.ps1` script needs the access token for the Build Service user so it can authenticate with the Azure DevOps REST API. This is already configured in our `azure-pipelines.yml` file. This creates an Environmental variable named `SYSTEM_ACCESSTOKEN` which the script can use.

```yaml
env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

* **Permissions**: The 'Build Service' user itself needs to be granted permissions to interact with Pull Requests. By default this is not allowed and must be configured. Follow these steps to configure the permissions:

![build-service-permissions](/assets/img/ado-analyzer/build-service-permissions.gif)

1. Click `Project Settings`
2. Navigate to `Repos \ Repositories`
3. Click the `Permissions` tab
4. Select the `ProjectName Build Service` user
5. Change the permision for `Contribute to pull requests` to **Allow**

---

## Configure Build Verification Policy

We have all of our pre-requisites in place and are ready to tie everything together. We're going to accomplish that by configuring two settings:

* **Branch Policy**: This will stop anyone from committing directly to the `master` branch, and force them to use the pull request process.

* **Build Verification**: This will configure any pull request to run our pipeline, evaluate our code with PSScriptAnalyzer, and receive feedback through the pull request comment system.

Follow these steps to configure our policies:

![build-validation](/assets/img/ado-analyzer/build-validation.gif)

1. Click `Project Settings`
2. Navigate to `Repos \ Repositories`
3. Click on your repo name
4. Select the `Policies` tab
5. Under 'Branch Policies' click the `master` branch
6. Under Branch Policies set `Require a minimum number of reviewers` to **On**
7. Under Build Validation click the `+`
8. Select the `PSScriptAnalyzer` build pipeline and click **Save**

---

## Create a Pull Request to Validate

To confirm everything is working as expected let's create a new branch, add some code, and open a pull request.

![pull-request](/assets/img/ado-analyzer/pull-request.gif)

The build validation policy we created should kick off our PSScriptAnalyzer pipeline when the pull request is opened. In the overview the pull request you should see that the required check has not yet been run, and is either queued or in progress.

![pull-request-build](/assets/img/ado-analyzer/pull-request-build.png)

The `Start-PSScriptAnalyzer.ps1` script executed by the pipeline will evaulate our PowerShell code and when issues are found it will connect to the Azure DevOps REST API and post a comment. The comment will be linked to the specific file, and line of code identified by PSScriptAnalyzer, and provide the detailed error information.

![code-line-comment](/assets/img/ado-analyzer/code-line-comment.png)

If PSScriptAnalyzer finds issues with the PowerShell code the script will exit with an exception (after posting the comments). This will flag the build validation as failing, and prevent the pull request from being merged into `master`.

Updating the code failing the PSScriptAnalyzer tests and pushing new commits to our branch will trigger our pipeline to run again, and re-evaluate our code. If all of our code passes the PSScriptAnalyzer analysis our build will be marked as passing, and a comment will be left in the pull request indicating everything is correct.

![success-comment](/assets/img/ado-analyzer/success-comment.png)

## Troubleshooting

### PullRequestContribute error

```json
{"$id":"1","innerException":null,"message":"TF401027: You need
     | the Git 'PullRequestContribute' permission to perform this
     | action. Details: identity
     | 'Build\\0ef135cb-8cdd-4557-a536-e6f20b82b4b9', scope
     | 'repository'.","typeName":"Microsoft.TeamFoundation.Git.Server.GitNeedsPermissionException, Microsoft.TeamFoundation.Git.Server","typeKey":"GitNeedsPermissionException"
```

If you are encountering this error in your pipeline it means the project Build Service user account does not have access to contribute (ie, post a comment) to a pull request. Review the steps to [Configure Build Service Permissions](#configure-build-service-permissions)

### Add `-Verbose` for additional output

Both PowerShell tasks in the pipeline support adding the `-Verbose` switch for additional output. This can be helpful in troubleshooting if you are encountering another issue.

![add-verbose](/assets/img/ado-analyzer/add-verbose.png)

The task output in the pipeline execution will include all verbose output from the script.

![verbose-output](/assets/img/ado-analyzer/verbose-output.png)