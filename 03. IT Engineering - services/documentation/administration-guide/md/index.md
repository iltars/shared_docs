
# System Administration Guide

This article describes scenarios for administering a software component - **Deploy tools** (`CDJE`).

### Terms and Definitions

|      Term/abbreviation       | Definition                 |
|------------------------------|----------------------------|
| DevOps  |Development and Operations   |
| DPM     | DevOps Pipeline Management  |
| JDBC    | Java Data Base Connectivity |
| Jenkins | A continuous integration/continuous delivery and deployment (CI/CD) automation software |
| Nexus   | Nexus Repository Manager |
| Kubernetes/OpenShift | Container orchestration platform |
| UI      | User Interface |
| AFT     | Automated functional testing |
| DB      | Database |
| PROD    | Production environment |
| DBMS    | Database Management System |
| PMS     | Process Management System |
| User    | User account |
| FS      | Functional subsystem |
| SF      | Software distribution / distribution |

## Administration Scenarios

There are two primary configuration files to interact with the created **Pipeline** in Jenkins:
+ ***environment.json*** 
+ ***subsystems.json***

> These files are located in the common repository and are being used by the system administrator to carry out functional adjustments and manage environment-dependent parameters. 

After updating the `pipeline` distribution (refer to **[Installation Guide](../../installation-guide/md/index.md)** → **Update** → **Conducting a consecutive Pipeline migration**) using a **Service job** in the `common` repository the primary configuration file for the `pipeline` – ***environment.json*** is being created/updated. This file allows the administration of stand-dependent parameters shared among all deployed functional applications, including Nexus Repository addresses, deployment credentials, deployment modes, and optionally selected features. All possible parameters in the ***environment.json*** file are shown in the following code block.

### Working with the environment.json

All possible parameters in the ***environment.json*** file are presented in the following code block:

```json
{
  "!__default":
  {                                				                
      "doNotCheckInventoryOnNull": "false",
      "openshiftDeploySafeMode": "false",			    		
      "openShiftPrecheck": "warning",			    			
      "openShiftCheckConfNames": "off",                         
      "bluegreen":"false",          
      "approveGreen": "false",   
      "openshiftDeploySafeMode": "false",
      "openShiftPrecheck": "warning",	
      "openShiftCheckConfNames": "off", 
      "openShiftNewPasswords": "false",   
      "openshiftMultiClusters": "false",   
      "openshiftNginx": "false",         
      "openshiftSecretsFilters": "false",  
      "credentials": {
          "openshiftOpsPasswordsCred": "vault_cred_openshift_ops",                                                         
          "SshKeyCreds"         : "_template_",     
  [...]

}
```

### Segmenting environment.json to facilitate parameter grouping

Given the multitude of parameters listed in the ***environment.json***, a new functionality has been introduced in `release/D-01.038.917` allowing the split of the configuration file into several components. This feature is implemented by enabling the use of a special pseudo-directive: `@@include=path/filename.json` in `parameters` section within the file:

+ Files intended for inclusion in environment.json must be located within the common repository under the specified block.
+ Arbitrary directory and file names are permissible.
+ There are no restrictions on the number of pseudo-directives used or their nesting levels.

#### Example of Usage

`common` Repository Structure:

```yaml
common/b1
├─includes               
│  └───playbooks.json
│  └───ift.json
├─environment.json
```

1. Main configuration file: **_environment.json_**
    ```json
    {
        "_default": {
            ...
            "exampleParam": "test",
            "playbooks_fpi": "@@include=includes/playbooks.json",
            "exampleParam2": "test2",
            ..
        },
        "ift": "@@include=includes/ift.json"
    }
    ```

2. Data Source file **_includes/playbooks.json_**
    ```json
    [
      {
          "name": "PLAYBOOK_1",
          "id": 1
      },
      {
          "name": "PLAYBOOK_2",
          "id": 2
      }
    ]
    ```

3. Data Source file **_includes/ift.json_**
    ```json
    {
      ...
      "exampleParam": "anotherValue",
      "exampleParam2": "anotherValue",
      ...
    }
    ```

4. A newly transformed file (**_environment.json_**):
    ```json
    {
        "_default": {
            ...
            "exampleParam": "test",
            "playbooks_fpi": [
              {
                "name": "PLAYBOOK_1",
                "id": 1
              },
              {
                "name": "PLAYBOOK_2",
                "id": 2
              }
            ],
            "exampleParam2": "test2",
            ..
        },
        "ift": {
          ...
          "exampleParam": "anotherValue",
          "exampleParam2": "anotherValue",
          ...
        }
    }
    ```

The mentioned operations take place during the specified `"runtime":` with each execution. Therefore, configurations will be read from the final processed file (***environment.json***).

#### Automated delivery of configuration file changes from the distribution to the repository in the environment (migration)

To enable automated delivery of configuration file changes, the following conditions must be met:
1. Service job should be switched to configuration migration mode from the `common` distribution.
2. In the `common` distribution, the ***environment.json*** should be located along with other configuration files.
3. A new migration rule should be added to the ***playbooks.json_*** file in the `common` distribution.

> The conditions listed are described below:

##### Switching the Service job to configuration migration mode from the common distribution


To switch **Service job** into configuration migration mode it is necessary to specify a `flag` in the  ***environment<CHANNEL>.json*** file for the Service job at the installation-wide level in the **released** `pipeline` repository:

```json
{
  ...
  "migrateConfigurationWithPipeline": false,
  ...
}
```

1. This flag disables the migration of any *environment.json* files from the released `pipeline` repository by executing the **MIGRATION** script (`ARTIFACT_TYPE: PIPELINE`).
2. Enabling the flag allows the migration to be executed through a new **MIGRATION_CONFIGURATION** script (`ARTIFACT_TYPE: COMMON`).

##### Adding configuration template files to the common distribution

In the source code repository of the `common` distribution, a new ***environment.json*** must be created (with directives specified as `@@includes=...`), along with the files that are intended to enrich the ***environment.json*** (the structure is shown in the **Usage Examples** section).

> [!WARNING]  
> The files should be located in the directory: `src/main/resources/common`

##### Creating or updating the __migration-rules.yml file in the common distribution

The ***__migration-rules.yml*** file is utilized to configure the migration of data from the `common` distribution to the `common` repository for the environment, specifying which files to migrate and the migration process itself.

> [!WARNING]  
> he files should be located in the directory: `src/main/resources/common`

***__migration-rules.yml*** must contain the following:

1. Specify the basic migration behavior by copying the contents mentioned below::
    ```yaml
    # The rules are applied sequentially, from first to last
    migrationRules:
      # File paths are specified in  ant-style:
      # - https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/util/AntPathMatcher.html
      # - https://www.jenkins.io/doc/pipeline/steps/file-operations/
      - type: ignore
        items:
          - "**/__*__/**"
          - "**/__topology-mapping.yml"
          - "**/__migration-rules.yml"
          - "pipeline.yml"
          - "installer/system/efs/config/definitions/**"
          - "installer/system/efs/config/global_parameters/**"
          - "**/README.md"
      - type: rewrite
        items:
          - "**/ansible/sup2_common.json"
      - type: createIfNotExists
        items:
          - "**/ansible/secret.yml"
          - "**/ansible/_passwords.conf"
          - "**/ansible/nginx.conf"
          - "**/ansible/files/**"
          - "**/ansible/UTIL_SECURE/inventory"
          - "**/ansible/globalInventory"
          - "**/blackList.conf"
          - "**/jenkinsJobs.conf"
      - type: migrateConfTool
        items:
          - "installer/installer/system/efs/config/parameters/__migration.conf"
      - type: merge
        migratorOpts: "--never-delete '.*'"
        items:
          - "**/ansible/*.json"
          - "**/ansible/mq/*.json"
          - "**/ansible/kafka/*.json"
          - "**/ansible/kafka/*.yml"
          - "**/version.conf"
          - "**/multiClusters.json"
      - type: merge
        items:
          - "**/installer/system/efs/config/parameters/*.conf"
      - type: merge
        items:
          - "**/ansible/*conf.yml"
      - type: remove
        items: []
    ```

  > [!WARNING]  
  > When working with a freshly downloaded `common` distribution, instead of the above-mentioned, copy the following content:
  ```yaml
  # The rules are applied sequentially, from first to last
  migrationRules:
  # File paths are specified in  ant-style:
  # - https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/util/AntPathMatcher.html
  # - https://www.jenkins.io/doc/pipeline/steps/file-operations/
  - type: createIfNotExists
    items:
      - "**/hosts/globalInventory"
      - "**/parameters/_extra.conf"
      - "**/parameters/common.conf.yml"
      - "**/secrets/_passwords.conf"
      - "**/subsystems.json"

  - type: merge
    migratorOpts: "--never-delete '.*'"
    items:
      - "**/version.conf"
      - "**/multiClusters.json"
  ```


2. Add a new group for the migration of configurations:
    ```yaml
    environment:
      # Migration of environment.json file
      - command: merge
        parameters:
          sourceDir: "{{ sourcePath }}/environments"
          targetDir: "{{ targetPath }}"
          resultDir: "{{ targetPath }}"
          includes:
            - "environment.json"
          mergeConfiguration:
              includes:
                - "__default(\\.+)?"
                - "{{ ENVIR }}(\\.+)?"
              rewrite:
                - "__default(\\.+)?"
              keep:
                - "{{ ENVIR }}(\\.+)?"

      # Migration of environment.json file components
      - command: merge
        parameters:
          sourceDir: "{{ sourcePath }}/environments/includes"
          targetDir: "{{ targetPath }}/includes"
          resultDir: "{{ targetPath }}/includes"
          includes:
            - "*.json"
          mergeConfiguration:
            rewrite:
              - ".*"
    ```

   > [!CAUTION] 
   > If the files are located in directories other than originally specified, it is necessary to adjust the template for the designated directories.

It's worth noting that within the description of the ***__migration-rules.yml*** file, the use of templates is also allowed. At this stage, there are three placeholders available:

1. `sourcePath` - the source directory (where the `common` software distribution was originally placed).
2. `targetPath` - the target directory (where the `common` repository with the selected blocks was downloaded from the active environment).
3. `ENVIR` - the name of the environment/'stand'.

> [!NOTE]
> A new new version of the `common` software distribution should be build after making all the modifications to the source code.

##### Initiating the Service job

> [!TIP]
> Before Initiating it is reccomened to perform a clean reconfiguration of the Service job by launching a Jenkins job with empty parameters.

To initiate the migration of configurations using the **Service job**, launch it with the following parameters:
```yaml
# A component to be updated
ARTIFACT_TYPE: "COMMON"
# Targeted version
ARTIFACT_VERSION: <Version of the previously released `common` distribution>
# Stage selection (stage – is a specified part of the component that will be updated)
PARAMETERS:
  - MIGRATION_CONFIGURATION
```

### Managing the subsystems.json

Following the Pipeline update (refer to [**Installation Guide**](../../installation-guide/md/index.md) → **Conducting Sequential Subsystems Migration**), the primary configuration file for installed configurations of functional applications - subsystems.json, is being created or updated in the `common` repository through the **Service job**.

***subsystems.json*** file contents example:

```
{"!__default"
  :{                                // Default Settings Block. AVOID MAKING ANY CHANGES IN THIS BLOCK — IT IS REGULARLY UPDATED AND OVERWRITTEN

  "classifier": "distrib",          // Classifier of the functional subsystem software distribution in the Nexus repository
  "groupId": "Nexus_PROD",          // Group ID of the functional subsystem software distribution in the Nexus repository
  "packaging": "zip",               // File extension with the functional subsystem software distribution
  "strict": "false",                // Migration mode for functional subsystem settings (true - only files starting with <fpi_name> will be migrated, false - all files will be migrated)
  "limit": 100,                     // Limitation on the number of distribution versions in the Jenkins menu
  "emailList": ["<email>"],         // Email addresses for receiving notifications on the status of the AD Pipeline
  "at": {                           // Block with default API test distribution credentials
    "groupId": "Nexus_PROD",        // Group ID for the API test distribution
    "branch": "master",             // Default repository branch for API test settings
    "classifier": "distrib",        // Default classifier for the API test distribution
    "packaging": "zip",             // Default extension for the API test distribution
    "timeout": 100                  // Time limit, after which the automated tests will be stopped and an error message logged
  },

  [...]
  
}
```

## System Log Events

The **Deploy Tools** software component lacks a built-in system log. Events from each run of the Deploy tools are logged using Jenkins service and stored in the job run logs.

> Audit events for the Deploy tools software component are not generated.

## Monitoring Events

Monitoring events are not currently present for the **Deploy tools** software component.

## Common Issues and Solution

1. **`Issue:`** Repositories for `common`, `pipeline`, or functional subsystem repositories has not been created, or there is no access to these mentioned (requires Read/Write access).
    > [!IMPORTANT]  
    > This error may also occur when no initial **Commit** has been made to the created repositories.

    ```
    11:43:40  ERROR: Error cloning remote repo 'origin'
    11:43:40  hudson.plugins.git.GitException: Command "git fetch --tags --progress ssh://<git@stash devops sandbox>/CI00380023_efs_ukofl_pipeline_mmv.git +refs/heads/*:refs/remotes/origin/*" returned status code 128:
    11:43:40  stdout:
    11:43:40  stderr: Repository not found
    11:43:40  The requested repository does not exist, or you do not have permission to access it.
    11:43:40  fatal: Could not read from remote repository.
    11:43:40  
    11:43:40  Please make sure you have the correct access rights
    11:43:40  and the repository exists.
    11:43:40  
    ```

    ***

    ```
    11:45:04  The build was interrupted due to an error:
    11:45:04  <..>.devops.UFSResultException: hudson.AbortException: Error cloning remote repo 'origin'
    ```

    **`Solution:`** Create the absent repository/repositories and ensure access to them for the designated user account.
