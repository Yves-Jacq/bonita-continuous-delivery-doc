# How to manage Living Apps Configuration

<span class="label label-danger">Supported from Bonita 7.8.0 onwards</span> This tutorial describes how to configure your Bonita Living Application from the command line using BCD.

## How it works

Since BCD 3.0.0 and with Bonita 7.8.0 onwards, if you build a Living Application you will get separately:
- a zip file which contains binaries
- a bconf file which contains the configuration for the target [environment](https://documentation.bonitasoft.com/bonita/${bonitaDocVersion}/environments) (Local by default)

Here is an example of a build result:
```
bonita-vacation-management-example
├── target
   └── bonita-vacation-management-example-Test-20181003140237.zip
   └── bonita-vacation-management-example-Test-20181003140237.bconf
```

It's possible to extract the configuration to check it and also override (merge) some parameters if needed.

### Extract configuration

You can extract the configuration if you want to check it or modify it

```bash
bcd -s scenarios/build_and_deploy.yml -y livingapp extract-conf \
-p bonita-vacation-management-example/target/bonita-vacation-management-example-Test-20181003140237.bconf \
-o scenarios/build_and_deploy_Test.yml
```

The configuration looks like this :
```yaml
---
processes:
- name: "Modify Pending Vacation Request"
  version: "1.4.1"
  parameters:
  - name: "calendarApplicationName"
    value: "Bonitasoft-NewVacationRequest/1.4.0"
    type: "String"
  - name: "calendarCalendarId"
    value: "mydomain.com_4gc5656x7f57cfsrejgb@group.calendar.google.com"
    type: "String"
```

As it may contain sensitive data, it's recommended to encrypt your configuration using [vault](how_to_use_bcd_with_data_encrypted):

```bash
ansible-vault encrypt scenarios/build_and_deploy_Test.yml
New Vault password:
Confirm New Vault password:
Encryption successful
```

You can also just check if there are parameters that have no value for this environment:

```bash
bcd -s scenarios/build_and_deploy.yml -y livingapp extract-conf --without-value \
-p bonita-vacation-management-example/target/bonita-vacation-management-example-Test-20181003140237.bconf \
-o scenarios/build_and_deploy_Test_missing_parameters.yml
```

Notes :
- If you omit to specify -o, the name of the output file by default is __parameters.yml__ and it will be created in the same directory of the original ***bconf*** file.
- If all parameters are set, no file will be created.

### Merge configuration

You may want to complete or override some parameter values coming from your Living App repository, to do that you can modify the output file of the __extract-conf__ command and ***merge*** with your ***bconf*** file.

```bash
bcd -s scenarios/build_and_deploy.yml -y livingapp merge-conf \
-p bonita-vacation-management-example/target/bonita-vacation-management-example-Test-20181003140237.bconf \
-i scenarios/build_and_deploy_Test.yml \
-o /tmp/bonita-vacation-management-example-Test-20181003140237-modified.bconf
```

Note : the content of bconf file is not encrypted so it's recommended to clean them after usage.

#### Override parameters with the same name
You may have the same parameter name in more than one processes and you want to override them in all processes, to do that you can create an ***yml*** file as shown:

```yaml
---
global_parameters:
  - name: "ParameterNameInAllProcesses"
    value: "SameValueInAllProcess"
    type: "String"
```

::: info
Important:
A specific parameter setting has priority over a global parameter configuration.
:::

**Example**:
Let assume that these processes __P1, P2, P3__ have all these three paremeters: ***calendarApplicationName***, ***emailNotificationSender***, ***emailServerUseSSL***.

```yaml
---
processes:
- name: "P1"
  version: "1.4.1"
  parameters:
  - name: "calendarApplicationName"
    value: "Bonitasoft-NewVacationRequest/1.4.0"
    type: "String"
  - name: "emailNotificationSender"
    value: "cancelvacationconfirmation@mail.com"
    type: "String"
- name: "P2"
  version: "1.4.1"
  parameters:
  - name: "calendarApplicationName"
    value: "Bonitasoft-NewVacationRequest/1.4.0"
    type: "String"
- name: "P3"
  version: "1.4.1"
  parameters:
  - name: "calendarApplicationName"
    value: "Bonitasoft-NewVacationRequest/1.4.0"
    type: "String"
global_parameters:
  - name: "emailNotificationSender"
    value: "vacation-notification@mail.com"
    type: "String"
  - name: "emailServerUseSSL"
    value: true
    type: "Boolean"
```

The result of __merge-conf__ will be:

* The value of ***emailServerUseSSL*** in __global_parameters__ will override __P1, P2, P3__.
* The value of ***emailNotificationSender*** in __global_parameters__ will override only __P2 and P3__ because the setting of ***emailNotificationSender*** in __P1__ has priority.
* The value of ***emailNotificationSender*** in __P1__ will override only the parameter of __P1__.
