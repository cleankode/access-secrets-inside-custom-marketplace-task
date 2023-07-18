# Azure DevOps using PowerShell

Here, we will explore few things that you might need when working in Azure Devops. Assuming you will be using PowerShell for scripting i.e

- Sign up for Azure DevOps
- Create Organization, make a note to the organization name (we can easly get it but just so you pay attention to it)
- Create Project, make a note to the project name too. This too will be easy to find out but just remember both names. It will be needed later when making API calls
- Setup Build pipeline for an existing project. Since the focus isn't to create fully functional application, please use a sample demo project. Lets consider ASP.NET Core app
- Then create also a release pipeline. Wire it so it the build pipeline you just did but make sure it it deploys to an empty stage. Why? you may wonder, because my friend, I don't want you to pay `$$$` by accident. What we we will explore is independent of whats the stage does so relax.

- Create a stage that preeceds your empty test or prod stage. Call it, Guard if you will. 
- Now, remember, what we want is to integrate our task within this Guard stage and do some magic. What magic you asked? Hmmm, okay stuff lets leave it with that. But seriously though, we will do things like:
  - check if the release was from the branch we all agreed up onbranch and that mostly is `master` or however you named it
  - check if there was at least one code reviewer
  - check if the required minimum test code coverage is met
  - create a bug and assign to the person who initiated the release when any of the minimium required criteria aren't met
  - fail the release when minimium requirement isn't met. Let them manually intervin if they want to but from automation perspective, minimum requirement not met means something is wrong.

## Create custom Visual Studio Marketplace Task using PowerShell

As part of your custom task, you might want to make a call to:

- access some resources that Azure DevOps (ADO) RESTFful API exposes such as `Release Detail`, `Code Coverage` of the build being deployed, `Work Items` associated with the build, and much more.
  - Personal Access Token (PAT) is required to access those APIs. I will come back on how to get PAT token later.
- access other resources from your own or external APIs depending on what the task needs to do.
  - this api most likely would like to check authorization beore acting on the resource (could be getting, updating, deleting or creating new resource.)

Note: Considering my readers being developers, when I refer `Resource` I meant it from `RESTful` API point of view and not resource as in Azure Resource :)

As you can see, in both cases, you would like to pass some sort of secret so that the Called service knows who the caller is, authorize and serves the request.

The token of some sort being a secret need to be properly protected. And the most appropriate place to keep it will be a KeyVault. Azure provide such service. Checkout here later (link to be included)

Now, whether the secrets are kept as a secret pipeline variable or fed into the pipeline from Azure KeyVault task, the PowerShell script you are using to build the task won't have the luxury to read Secret variables as it would non-secret variables.
I myself spent days on this looking for ways to have a hold on them. And I finally did.
Most articles you find telling you to do mapping here mapping over there, O no worries, they are accessible just like so, definetly using input in your task.json and all that never worked for me. I tried so many options and listing the different things I explored will only bore you so much that you might stop reading but instead here is what it works without spening as much time as I did.

There is an a Vsts API you need to call so it spits out ALL variables  (secrets included) that are accessible to the script in your task. Isn't that what I was looking and what you are looking now?
Here is it in action, just simply let it spit out the whole thing and pick the once you would like to use.

## How to access Secret Variables inside your custom marketplace task in PowerShell

```Powershell


https://github.com/microsoft/azure-pipelines-task-lib/blob/master/powershell/Docs/Commands.md

# This gets will return ALL Task Variables that you can access (including Secret variables)
$allTaskVariablesIncludingSecrets = Get-VstsTaskVariableInfo

# if you want, you can json convert it to see whats available during your debugging
$allTaskVariablesIncludingSecrets | ConvertTo-Json
#that will give you array of objects with three properties (Name, Secret and Value)
# [
#     {
#         "Name":  "SecretVariableName",
#         "Secret":  true,
#         "Value":  "***"
#     },
#     {
#     "Name":  "NotSecretVar",
#     "Secret":  false,
#     "Value":  "Some stuff here"
#     }
# ]

# Since our objective is to get hold of Secret varibales, lets filter them
$secVariables = $allTaskVariablesIncludingSecrets | Where-Object {$_.Secret -eq $true}
# If one of your Secret Variable is called 'SecretVariableName', here is how you access it
# Since each object has the following three properties (Name, Secret (this is bool) and Value)
$mySecretVarObject = $secVariables |  Where-Object {$_.Name -eq "SecretVariableName"}
$mySecret = $($mySecretVarObject.Value)
# This will give display *** for the valute but Length will show you the actual length. So you are good to use $mySecret in your script.
Write-Host "Value: $mySecret and Length: $($mySecret.Length)" 
# Simply use $mySecret the way you would any local variable. No special treatment or husle needed
```
