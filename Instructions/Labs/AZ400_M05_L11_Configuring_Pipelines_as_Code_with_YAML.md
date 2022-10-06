# Lab 11: Configuring Pipelines as Code with YAML
# Student lab manual

## Lab overview

Azure DevOps supports two types of version control, Git and Team Foundation Version Control (TFVC). Here is a quick overview of the two version control systems:

- **Team Foundation Version Control (TFVC)**: TFVC is a centralized version control system. Typically, team members have only one version of each file on their dev machines. Historical data is maintained only on the server. Branches are path-based and created on the server.

- **Git**: Git is a distributed version control system. Git repositories can live locally (such as on a developer's machine). Each developer has a copy of the source repository on their dev machine. Developers can commit each set of changes on their dev machine and perform version control operations such as history and compare without a network connection.

Git is the default version control provider for new projects. You should use Git for version control in your projects unless you have a specific need for centralized version control features in TFVC.

In this lab, you will learn how to establish a local Git repository, which can easily be synchronized with a centralized Git repository in Azure DevOps. In addition, you will learn about Git branching and merging support. You will use Visual Studio Code, but the same processes apply for using any Git-compatible client.

## Objectives

After you complete this lab, you will be able to:

-   configure CI/CD pipelines as code with YAML in Azure DevOps

## Instructions

#### Set up an Azure DevOps organization

1. On your lab VM open **Edge Browser** on desktop and navigate to https://dev.azure.com. Then click on **Sign into Azure DevOps** and login with the credentials provided in environment details tab.

    ![Azure DevOps](images/devops.png)

2. On the next page accept defaults and click on continue.

    ![Azure DevOps](images/m1-1.png)

3. On the **Get started with Azure DevOps** page click on **Continue**.

4. On the **Almost Done...** page fill the captcha and click on continue. 

    ![Azure DevOps](images/m1-2.png)

5. On the Azure Devops page click on **Azure DevOps** located at top left corner and then click on **Organization Setting** at the left down corner

    ![Azure DevOps](images/agent1.png)

6. In the **Organization Setting** window on the left menu click on **Billing** and select **Setup Billing** then click on save.

    ![Azure DevOps](images/agent3.png)
    ![Azure DevOps](images/agent4.png)    

7. On the **MS Hosted CI/CD** section under **Paid parallel jobs** enter value **1** and at the end of the page click on **Save**.

    ![Azure DevOps](images/agent2.png)

### Exercise 0: Configure the lab VM.

In this exercise, you will set up the prerequisites for the lab, which include the preconfigured Parts Unlimited team project based on an Azure DevOps Demo Generator template and a Visual Studio Code configuration.

#### Task 1: Configure the Parts Unlimited team project and add a self hosted agent

In this task, you will use Azure DevOps Demo Generator to generate a new project based on the **Parts Unlimited** template.

1.  In a new tab of Edge browser navigate to https://azuredevopsdemogenerator.azurewebsites.net. This utility site will automate the process of creating a new Azure DevOps project within your account that is prepopulated with content (work items, repos, etc.) required for the lab. 

    > **Note**: For more information on the site, see https://docs.microsoft.com/en-us/azure/devops/demo-gen.

1.  Click **Sign in** and if prompted sign with the credentials provided in environment details tab.
1.  If required, on the **Azure DevOps Demo Generator** page, click **Accept** to accept the permission requests for accessing your Azure DevOps subscription.
1.  On the **Create New Project** page, in the **New Project Name** textbox, type **Configuring Pipelines as Code with YAML**, in the **Select organization** dropdown list, select your Azure DevOps organization, and then click **Choose template**.

1.  In the list of templates, in the toolbar, click **General**, select the **PartsUnlimited-YAML** template and click **Select Template**.

1.  Back on the **Create New Project** page, click **Create Project**

    > **Note**: Wait for the process to complete. This should take about 2 minutes. In case the process fails, navigate to your Azure DevOps organization, delete the project, and try again.

1.  On the **Create New Project** page, click **Navigate to project**.

#### Task 2: Create Azure resources

In this task, you will create an Azure web app and an Azure SQL database by using the Azure portal.

1.  From the lab computer, start a web browser, navigate to the [**Azure Portal**](https://portal.azure.com), and sign in with the user account that has the Owner role in the Azure subscription you will be using in this lab and has the role of the Global Administrator in the Azure AD tenant associated with this subscription.
1.  In the Azure portal, in the toolbar, click the **Cloud Shell** icon located directly to the right of the search text box.
1.  If prompted to select either **Bash** or **PowerShell**, select **Bash**.

    >**Note**: If this is the first time you are starting **Cloud Shell** and you are presented with the **You have no storage mounted** message, select the subscription you are using in this lab, and select **Create storage**. 

1.  From the **Bash** prompt, in the **Cloud Shell** pane, run the following command to create a resource group (replace the `<region>` placeholder with the name of the Azure region closest to you such as 'eastus').

    ```bash
    RESOURCEGROUPNAME='az400m11l01-RG'
    LOCATION='<region>'
    az group create --name $RESOURCEGROUPNAME --location $LOCATION
    ```

1.  To create a Windows App service plan by running the following command:

    ```bash
    SERVICEPLANNAME='az400l11a-sp1'
    az appservice plan create --resource-group $RESOURCEGROUPNAME --name $SERVICEPLANNAME --sku B3
    ```
    
    > **Note**: If the `az appservice plan create` command fails with an error message starting with `ModuleNotFoundError: No module named 'vsts_cd_manager'`, then run the following commands and then re-run the failed command.

            ```bash
            az extension remove -n appservice-kube
            az extension add --yes --source "https://aka.ms/appsvc/appservice_kube-latest-py2.py3-none-any.whl"
            ```

1.  Create a web app with a unique name.

    ```bash
    WEBAPPNAME=partsunlimited$RANDOM$RANDOM
    az webapp create --resource-group $RESOURCEGROUPNAME --plan $SERVICEPLANNAME --name $WEBAPPNAME
    ```

    > **Note**: Record the name of the web app. You will need it later in this lab.

1.  Next, create an Azure SQL Server.

    ```bash
    USERNAME="Student"
    SQLSERVERPASSWORD="Pa55w.rd1234"
    SERVERNAME="partsunlimitedserver$RANDOM"

    az sql server create --name $SERVERNAME --resource-group $RESOURCEGROUPNAME \
    --location $LOCATION --admin-user $USERNAME --admin-password $SQLSERVERPASSWORD
    ```

1.  The web app needs to be able to access the SQL server, so we need to allow access to Azure resources in the SQL Server firewall rules.

    ```bash
    STARTIP="0.0.0.0"
    ENDIP="0.0.0.0"
    az sql server firewall-rule create --server $SERVERNAME --resource-group $RESOURCEGROUPNAME \
    --name AllowAzureResources --start-ip-address $STARTIP --end-ip-address $ENDIP
    ```

1.  Now create a database within that server.

    ```bash
    az sql db create --server $SERVERNAME --resource-group $RESOURCEGROUPNAME --name PartsUnlimited \
    --service-objective S0
    ```

1.  The web app you created needs the database connection string in its configuration, so run the following commands to prepare and add it to the app settings of the web app.

    ```bash
    CONNSTRING=$(az sql db show-connection-string --name PartsUnlimited --server $SERVERNAME \
    --client ado.net --output tsv)

    CONNSTRING=${CONNSTRING//<username>/$USERNAME}
    CONNSTRING=${CONNSTRING//<password>/$SQLSERVERPASSWORD}

    az webapp config connection-string set --name $WEBAPPNAME --resource-group $RESOURCEGROUPNAME \
    -t SQLAzure --settings "DefaultConnectionString=$CONNSTRING" 
    ```

### Exercise 1: Configure CI/CD Pipelines as Code with YAML in Azure DevOps

In this exercise, you will configure CI/CD Pipelines as code with YAML in Azure DevOps.

#### Task 1: Delete the existing pipeline

In this task, you will delete the existing pipeline.

1.  On the lab computer, switch to the browser window displaying the **Configuring Pipelines as Code with YAML** project in the Azure DevOps portal and, in the vertical navigational pane, select the **Pipelines**.

    > **Note**: Before configuring YAML pipelines, you will disable the existing build pipeline.

1.  On the **Pipelines** pane, select the **PartsUnlimited** entry. 
1.  In the upper right corner of the **PartsUnlimited** blade, click the vertical ellipsis symbol and, in the drop-down menu, select **Delete**.
1.  Write **PartsUnlimited** and click **Delete**.
1.  In the vertical navigational pane, select the **Repos > Files**. Make sure you are in the **master** branch (dropdown on top of **Files** window), on the **azure-pipelines.yml** file, click the vertical ellipsis symbol and, in the drop-down menu, select **Delete**. Commit the change on the master branch by clicking on **Commit** (leaving default options).

#### Task 2: Add a YAML build definition

In this task, you will add a YAML build definition to the existing project.

1.  Navigate back to the **Pipelines** pane in of the **Pipelines** hub. 
2.  In the **Create your first Pipeline** window, click **Create pipeline**. 

    > **Note**: We will use the wizard to automatically create the YAML definition based on our project.

3.  On the **Where is your code?** pane, click **Azure Repos Git (YAML)** option.
4.  On the **Select a repository** pane, click **PartsUnlimited**.
5.  On the **Configure your pipeline** pane, click **ASP<nolink>.NET** to use this template as the starting point for your pipeline. This will open the **Review your pipeline YAML** pane.

    > **Note**: The pipeline definition will be saved as a file named **azure-pipelines.yml** in the root of the repository. The file will contain the steps required to build and test a typical ASP<nolink>.NET solution. You can also customize the build as needed. In this scenario, you will update the **pool** to enforce the use of a VM running Windows 2019.

6.  Make sure  `trigger` is **master**.

    > **Note**: Review in Repos if your repository has **master** or **main** branch, organizations could choose default branch name for new repos: [Change the default branch](https://docs.microsoft.com/en-us/azure/devops/repos/git/change-default-branch?view=azure-devops#choosing-a-name). 

7.  On the **Review your pipeline YAML** pane, in line **10**, replace `vmImage: 'windows-latest'` with `vmImage: 'windows-2019'`.	

8.  Remove the **VSTest@2** task:	
    ```yaml	
    - task: VSTest@2	
      inputs:	
        platform: '$(buildPlatform)'	
        configuration: '$(buildConfiguration)'	
    ```	
9.  On the **Review your pipeline YAML** pane, click **Save and run**.	

10.  On the **Save and run** pane, accept the default settings and click **Save and run**.	
11.  On the pipeline run pane, in the **Jobs** section, click **Job** and monitor its progress and verify that it completes successfully. 

    > **Note**: Each task from the YAML file is available for review, including any warnings and errors.

#### Task 3: Add continuous delivery to the YAML definition

In this task, you will add continuous delivery to the YAML-based definition of the pipeline you created in the previous task.

> **Note**: Now that the build and test processes are successful, we can now add delivery to the YAML definition. 

1.  On the pipeline run pane, click the ellipsis symbol in the upper right corner and, in the dropdown menu, click **Edit pipeline**.
1.  On the pane displaying the content of the **azure-pipelines.yaml** file, in line **8**, following the `trigger` section, add the following content to define the **Build** stage in the YAML pipeline. 

    > **Note**: You can define whatever stages you need to better organize and track pipeline progress.

    ```yaml
    stages:
    - stage: Build
      jobs:
      - job: Build
    ```

1.  Select the remaining content of the YAML file and press the **Tab** key twice to indent it four spaces (it should be placed with same identation as ```job: Build```). 

    > **Note**: This way, everything starting with the `pool` section becomes part of the `job: Build`. 

1.  At the bottom of the file, add the configuration below to define the second stage.

    ```yaml
    - stage: Deploy
      jobs:
      - job: Deploy
        pool:
          vmImage: 'windows-2019'
        steps:
    ```

1.  Set the cursor on a new line at the end of the YAML definition. 

    > **Note**: This will be the location where new tasks are added.

1.  In the list of tasks on the right side of the code pane, search for and select the **Azure App Service Deploy** task.
1.  In the **Azure App Service deploy** pane, specify the following settings and click **Add**:

    - in the **Azure subscription** drop-down list, select the Azure subscription into which you deployed the Azure resources earlier in the lab, click **Authorize**, and, when prompted, authenticate by using the same user account you used during the Azure resource deployment.
    - in the **App Service name** dropdown list, select the name of the web app you deployed earlier in the lab. 
    - in the **Package or folder** text box, type `$(System.ArtifactsDirectory)/drop/*.zip`. 

    > **Note**: This will automatically add the deployment task to the YAML pipeline definition.

1.  With the added task still selected in the editor, press the **Tab** key twice to indent it four spaces, so that it listed as a child of the **steps** task.

    > **Note**: The **packageForLinux** parameter is misleading in the context of this lab, but it is valid for Windows or Linux. 

    > **Note**: By default, these two stages run independently. As a result, the build output from the first stage might not be available to the second stage without additional changes. To implement these changes, we will use one task to publish the build output at the end of the build stage and another to download it in the beginning of the deploy stage. 

1.  Place the cursor on a blank line at the end of the build stage to add another task. (right below  `task: VSBuild@1` )
1.  On the **Tasks** pane, search for and select the **Publish build artifacts** task. 
1.  On the **Publish build artifacts** pane, accept the default settings and click **Add**. 

    > **Note**: This will publish the build artifacts to a location that will be downloadable under the alias **drop**.

1.  With the added task still selected in the editor, press the **Tab** key twice to indent it four spaces (or press the **Tab** until task is indented as the ones above).

    > **Note**: You may also want to add an empty line before and after to make it easier to read.

1.  Place the cursor on the first line under the **steps** node of the **deploy** stage.
1.  On the **Tasks** pane, search for and select the **Download build artifacts** task.
1.  Click **Add**.
1.  With the added task still selected in the editor, press the **Tab** key twice to indent it four spaces. 

    > **Note**: Here as well you may also want to add an empty line before and after to make it easier to read.

1.  Add a property to the download task specifying the `artifactName` of `drop` (make sure to match the spacing):

    ```
    artifactName: 'drop'
    ```

1.  Click **Save**, on the **Save** pane, click **Save** again to commit the change directly into the master branch.

    > **Note**: This will automatically trigger a new build.

1.  The pipeline will look similar to this example  (**Reference your own subscription and webapp on last task**):

    ```
    trigger:
    - master

    stages:
    - stage: Build
      jobs:
      - job: Build
        pool:
            vmImage: 'windows-2019'

        variables:
            solution: '**/*.sln'
            buildPlatform: 'Any CPU'
            buildConfiguration: 'Release'

        steps:
        - task: NuGetToolInstaller@1

        - task: NuGetCommand@2
          inputs:
            restoreSolution: '$(solution)'

        - task: VSBuild@1
          inputs:
            solution: '$(solution)'
            msbuildArgs: '/p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:SkipInvalidConfigurations=true /p:PackageLocation="$(build.artifactStagingDirectory)"'
            platform: '$(buildPlatform)'
            configuration: '$(buildConfiguration)'
        - task: PublishBuildArtifacts@1
          inputs:
            PathtoPublish: '$(Build.ArtifactStagingDirectory)'
            ArtifactName: 'drop'
            publishLocation: 'Container'

    - stage: Deploy
      jobs:
      - job: Deploy
        pool:
            vmImage: 'windows-2019'
        steps:
        - task: DownloadBuildArtifacts@0
          inputs:
            buildType: 'current'
            downloadType: 'single'
            downloadPath: '$(System.ArtifactsDirectory)'
            artifactName: 'drop'
        - task: AzureRmWebAppDeployment@4
          inputs:
            ConnectionType: 'AzureRM'
            azureSubscription: 'YOUR-AZURE-SUBSCRIPTION'
            appType: 'webApp'
            WebAppName: 'YOUR-WEBAPP-NAME'
            packageForLinux: '$(System.ArtifactsDirectory)/drop/*.zip'
    ```

1.  In the web browser window displaying the Azure DevOps portal, in the vertical navigational pane, select the **Pipelines**.
1.  On the **Pipelines** pane, click the entry representing the newly configured pipeline.
1. Click on the most recent run (automatically started).
1.  On the **Summary** pane, monitor the progress of the pipeline run.
1.  If you notice a message stating that *This pipeline needs permissions to access a resource before this run can continue to Deploy*, click **View**, in the **Waiting for review** dialog box, click **Permit**, and, in the **Permit access?** pane, click **Permit** again.
1.  At the bottom of the **Summary** pane, click the **Deploy** stage to view details of the deployment.

    > **Note**: Once the task completes, your app will be deployed to an Azure web app.

#### Task 4: Review the deployed site

1.  Switch back to web browser window displaying the Azure portal and navigate to the resource group **az400m11l01-RG** open Azure web app blade displaying the properties of the Azure web app.
1.  On the Azure web app blade, click **Overview** and, on the overview blade, click **Browse** to open your site in a new web browser tab.
1.  Verify that the deployed site loads as expected in the new browser tab.

### Exercise 2: Remove the Azure DevOps billing

In this exercise, you will remove the Azure DevOps billing enabled in this lab to eliminate unexpected charges. 

#### Task 1: Remove the Azure DevOps billing 
    
In this task, you will remove pipeline billing to eliminate unnecessary charges.
    
1.  On the lab computer, switch to the browser window displaying Azure DevOps organization homepage and select **Organization Settings** at bottom left corner.
    
1.  Under **Organization Settings** select **Billing** and click on **Change billing** button to open Change billing pane.
    
1.  In the **Change billing** pane, select **Remove billing** setting and click on Save.

## Review

In this lab, you configured CI/CD pipelines as code with YAML in Azure DevOps.
