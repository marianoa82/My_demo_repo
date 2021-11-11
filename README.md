# Welcome to your new Salesforce Corporate Repository

## Ground Rules

This Repository was requested by Professional Services Team to support a customer's project implementation.

__Repository Owners (Salesforce Employee) is totally responsible by content stored inside this Repository, and who can access this environment. Be sure that your customer is aware about this Repository and always MAKE SURE that you're following our corporate compliance.__

## Adding collaborators to this Repository

Repository owners can add Internal Contributors (Salesforce Employees) and/or External Contributors.

* __Adding Internal Contributors (Employee):__ Invite collaborators to this Repository using the corporate e-mail. The contributor will be invited and must login in Github through Aloha. Remember to grant access as you need.

* __Adding External Contributors (Sub Contractors):__ Invite external contributors using their corporate e-mail and request to execute security configurations to grant access to the Repository:

   1. Enable 2FA: External contributors must need enable 2FA in their accounts:
   https://help.github.com/en/github/authenticating-to-github/securing-your-account-with-two-factor-authentication-2fa

   2. Create a Commit Signature: External contributors must need to create a commit signature:
   https://help.github.com/en/github/authenticating-to-github/managing-commit-signature-verification
   
## Branching strategy

You will have to decide on the git branching strategy you are going to follow. A recommendation is for you to use [GitFlow](https://nvie.com/posts/a-successful-git-branching-model/) - please go through the article and understand the difference between master, develop, feature branches and release branches. This strategy will enable the support of parallel development streams. 

## Setting up your development environment and working with a Salesforce DX project on a Org Development Model Strategy

### Adding a SSH public key in Github to streamline access

In order to facilitate your access through a git client, you can associate a public ssh certificate into your Github account, so you are not prompted every time that you want to execute a command against the remote repository to enter your credentials. You can do that following the instructions provided here: [Adding a new SSH key to your GitHub account](https://help.github.com/en/github/authenticating-to-github/adding-a-new-ssh-key-to-your-github-account)

### Configuring your development environment for a Salesforce DX project

The following software list is a recommendation for your development environment:

* Git client
* Salesforce CLI
* SourceTree (client git with a User Interface - optional but useful)
* Visual Studio Code with the following plug-gins:
    * GitLens
    * Salesforce Extension Pack
    * Salesforce CLI Integration
    * Salesforce Package.xml Generator Extension for VS Code (optional, but very useful)

You will also have to clone the Github repository locally. To do that, navigate to a folder where you want your project to be locally stored and execute the following command:
```
git clone git@github.com:Professiona-Services-LATAM/<REPO>.git
```

Now, you have to initiate a Salesforce DX project executing in the same directory the following command - more details about the initialization of a DX project can be found [here](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_create_new.htm):
```
force:project:create -n MyProject --template standard
```

### Connecting to the target sandboxes / orgs

You will need to authenticate your Salesforce CLI client against your development / testing / UAT / production sandboxes / orgs. To do that, in your Salesforce DX project folder execute the following command:

```
sfdx force:auth:web:login -r https://test.salesforce.com -a <SANDBOX_ALIAS>

OR

sfdx force:auth:web:login -a <ORG_ALIAS>
```

### Metadata retrieval

Now it's time to retrieve the updated metadata from your sandbox/org to your local Salesforce DX project. To do that, the utilization of the pluggin `Salesforce Package.xml Generator Extension for VS Code` is useful. You can launch the pluggin and select the metadata types and definitions you want to download and then create a new manifest file (package.xml) with the content.

Please, realize that the plugin will use the sandbox that is associated with the defaultusername configured in the file `<project_dir>/.sfdx/sfdx-config.json` - configure it properly to refer to the sandbox/org in which you want to download the metadata.

A word of caution regarding which metadata you want to download. It's recommended for you to download only the metadata that an organization had changed in the org. You should avoid to download/commit everything, since the project will become large, more complex to handle and the changes more difficult to determine. It's also helpful to maintain a manifest file only for retrieval operations and another for deployments, such as `<project_dir>/manifest/package-retrieval.xml` and `<project_dir>/manifest/package-baseDeploy.xml`. The reason for that is because sometimes during the project, you may be uncertain on which metadata your development touched, so you can download using a more comprehensive set of metadata and then compare the changes against the repo to determine exactly what you want to promote into the environment pipeline. 

Another aspect where a `package-retrieval.xml` is useful is when you want to download the permissions you configured. To understand how Metadata API handles permissions, please go through this article: [Dude where's my permissions](https://www.salesforcehacker.com/2013/05/dude-wheres-my-permission.html).

After you have your retrieval package.xml defined, you can execute the following command:
```
sfdx force:source:retrieve --manifest <RETRIEVAL_MANIFEST_FILE> -u <SANDBOX/ORG_ALIAS>

```

Upon gaining development maturity you will use the retrieval package less and less as you will be able to determine more easily the metadata your development touched. You will simply generate a temporary package.xml using the pluggin, download the metadata you want, delete the temporary package.xml and commit your changes.

### Metadata deployment / check 

After you determined which metadata you want to promote, apart from committing the new versions into the repository in the target branch you will have to keep a deployment manifest file, such as `<project_dir>/manifest/package-baseDeploy.xml` with the content of the package you are working on. This manifest file will be growing throughout the development of a release, with contributions from multiple team members. Probably this file is going to have some conflicts to be fixed whenever you are merging branches, since the same file (and sometimes the same line) would be changed by different development streams. 

And then, you will have to:

* (Optional) remove the converted metadata folder with the following command:

```
rm -rf <CONVERTED_METADATA_FOLDER>/
```

* (Optional) convert the source into metadata with the following command:

```
sfdx force:source:convert -r force-app/ -d <CONVERTED_METADATA_FOLDER> -x <DEPLOYMENT_MANIFEST_FILE>
```

* To test the deployment against another sandbox, executing the following command:

```
sfdx force:mdapi:deploy -u <SANDBOX/ORG_ALIAS> -d <CONVERTED_METADATA_FOLDER>/ -c -l RunLocalTests -w 10

OR, if you didn't convert the source into metadata

sfdx force:source:deploy -u <SANDBOX/ORG_ALIAS> -x <DEPLOYMENT_MANIFEST_FILE> -c -l RunLocalTests -w 10
```

* If any error, fix them and after everything is checked, commit your code into the target branch you are working on. 

* (Optional) if you don't have Github Workflow and Github Actions configured to automatically deploy your changes against a target org you can deploy the package executing the same command as above, removing the `-c`parameter.

### Destructive changes

Whenever you want to remove content of a target sandbox/org, you can use a destructive changes package and deploy it using SFDX commands. To do that execute the following steps:

* Create a directory into the <project_dir>, for instance: `mkdir <project_dir>/destructive`

* Create the following boilerplate package.xml inside the dir:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <version>47.0</version>
</Package>
```

* Create a destructiveChanges xml file in the same directory with the content that you want to be removed from the target sandbox/org, the sintax of this file is the same as the manifest files mentioned above. Here is an example of a destructive changes removing a custom field of the Account Object:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">    
    <types>
        <members>Account.CUSTOM_FIELD_1__c</members>
		<name>CustomField</name>
	</types>
</Package>
```

* Check/deploy your destructive changes with the following command (if you want to really delete, remove the -c parameter):
```
sfdx force:mdapi:deploy -d destructive -u <SANDBOX/ORG_ALIAS> -w 10 -c
```

## Deploy Automation with Github Actions

You can create [Github Workflows](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions) in your Repository that will execute [Github Actions](https://help.github.com/en/actions) and may deploy (or validate the deployment) automatically your code into target orgs upon events such as commit/push or pull request being created. We prepared for you a Github Action that will execute the following steps:
* Download and install Salesforce CLI on a Github runner;
* Decrypt a certificate stored in your repo;
* Auth in a target sandbox/org using the decrypted certificate; 
* Convert the source format into metadata format;
* Deploy/Check a pre-package (optional)
* Deploy/Check destructive changes (optional)
* Deploy/Check main package
* Execute Data Factory Apex (optional)

For you to create a Github Wofkflow and use it into your project please refer to the following repo README:
https://github.com/Professiona-Services-LATAM/sfdx-orgdev-build-deploy. All steps required to configure the workflow, the certificates and the connected app in the target sandbox/org are stated there. 

### Executing Github Workflows manually

In some moments of the project lifecycle it will be needed to trigger a Github workflow manually, for instance to update a training environment with a release. Normally there are some environments that don't have a lifecycle strictly connected with a repository branch, and, to be able to execute the Build&Deploy using the same process that will configure production will harden your package and your overall deployment process.

Unfortunately, Github doesn't provide on its User Interface a method to do that, but we can use a workaround and trigger the Workflow from our local machine using [Repository Dispatch](https://developer.github.com/v3/repos/#create-a-repository-dispatch-event).

To do that, you will need to follow these steps:

* Create a access token for your CLI and associate it with your account: https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line

* Associate the workflow with a repository dispatch event, such as:
```yml
# Unique name for this workflow
name: Custom - Deploy to TH01

# Definition when the workflow should run
on:
    repository_dispatch:
        types: deploy-th01

# Jobs to be executed
jobs:
    build-deploy:
        runs-on: ubuntu-latest
        steps:

            # Checkout the code in the pull request
            - name: 'Checkout source code'
              uses: Professiona-Services-LATAM/checkout@v2
              with:
                ref: ${{ github.event.client_payload.ref }}

            - name: 'Build and Deploy'
              uses: Professiona-Services-LATAM/sfdx-orgdev-build-deploy@v1.1
              with:
                type: 'sandbox'
                certificate_path: devops/server.key.enc
                decryption_key: ${{ secrets.DECRYPTION_KEY_NON_PRODUCTIVE }}
                decryption_iv: ${{ secrets.DECRYPTION_IV_NON_PRODUCTIVE }}
                client_id: ${{ secrets.CONSUMER_KEY_TH01 }}
                username: ${{ secrets.USERNAME_TH01 }}
                checkonly: false
                pre_manifest_path: manifest/package-preDeploy.xml
                destructive_path: destructive
                manifest_path: manifest/package-baseDeploy.xml
                data_factory: scripts/apex/CreateBaseData.apex
```

* Using curl, execute the follow command:
```
curl -v --data '{ "event_type": "<WORKFLOW_ACTION for instance: deploy-th01>", "client_payload": { "ref": "<REF/BRANCH here, for instance: feature/sprint11/marketshare>" } }' -H "Authorization: token <PERSONAL_TOKEN_HERE>" -H "Accept: application/vnd.github.everest-preview+json" --request POST https://api.github.com/repos/Professiona-Services-LATAM/<REPO_NAME>/dispatches
```

That's it, now just follow the action execution using Github UI.

## Keeping your release

Whenever you are working on a new feature that will be included into a release, keep the content of the release updated, mentioning the user stories that's part of the packages, any pre/pos deployment steps that may be required and some checks for the release engineer to validate the deployment. It's recommended to use a README.md file as the release notes for a package. 

## Highlights

* *Continuous Integration* it's not about software, or infrastructure. It's a development best practice that promotes the idea to integrate multiple development streams often and early. You just need a Control Version System to do CI, and not fancy software stack. Of course software will help, but my main point here is that this best practice is what it is: a practice - that needs to be followed in *daily basis* - not only when the sprint is over, or when the release is ready to be promoted. Then, it will be too late. One of the best way to do that, is to include in each and every user story a task for the developer to integrate his work as per the project strategy. His user story will be finished only when the metadata is integrated and the story is deployed into another environment. 

* As per the above point, you should *never* validate user stories in development sandboxes. 

* Better is for each developer to have his own development sandbox. Don't be afraid of merges - git will handle most of merge situation for you and in situation of conflicts it's better to detect them earlier rather than later. 

* The developer should often integrate his feature branch with develop, but also integrates develop (containing work from other streams) into his feature branch. Therefore, sometimes the developer will have to update his development sandbox with the changes being done by different streams.

* As stated above, it's useful to have more than one manifest file (package.xml), where each of them is used for it's own purpose:
	* The goal of retrieval manifest is to list / download all sandbox/org metadata for the repo - not necessarily all these metadata will be used on the promote/deploy package
	* The goal of the deployment manifest is to pile up the metadata that will be promoted to the environment pipelines. This means that each and every release (no matter where it's a major/minor/hotfix/patch) will have it's own deployment manifest file.

* It's useful during the development lifecycle for the developers to create it's own retrieval packages to download specific metadata. These should not be included in the repository. You can use .gitignore entries to segregate them. 

* Be mindful regarding the Salesforce releases calendar. You may work on a sandbox that's in a preview state containing features that's not available in your production org. Be extra caring of permissions that maybe introduced / removed by the release. 

* Upon handling permissions, it's recommended to include only the permissions that you are manipulating in the profiles / permission sets. Remember that: permissions are handled in their own way by the metadata API. Please, go again through [this article](https://www.salesforcehacker.com/2013/05/dude-wheres-my-permission.html).

* For projects containing multiple squads, a Salesforce release/devops engineer role is required. This role is responsible to define the branching strategy, communicate the policies to the team, enable the developers to follow the practices and approve / monitor pull requests to the main branches, including develop and master (production). Make sure to indicate this to your PM and work with the customer to define who is going to play that role. For projects containing only a single squad, probably the Technical Architect can easily play this role. 

Enjoy it!
