# [License Key Management](https://www.ibm.com/docs/en/engineering-lifecycle-management-suite/lifecycle-management/7.1.0?topic=servers-managing-licenses) for [ELM](https://jazz.net/)

## References

* [Managing licenses](https://www.ibm.com/docs/en/engineering-lifecycle-management-suite/lifecycle-management/7.1.0?topic=servers-managing-licenses)
* [IBM Common Licensing documentation](https://www.ibm.com/docs/en/common-licensing)
* [How to find IBM Engineering Lifecycle Management licenses](https://www.ibm.com/support/pages/how-find-ibm-engineering-lifecycle-management-licenses)


## Download LKS

(LKS = License Key Server)

Go to `https://www.ibm.com/support/fixcentral/` and choose:

* Product selector: IBM Common Licensing
* Version: All
* Platform: All

Then

* Browse for fixes

Then

* `IBM_Common_Licensing_ICL_9.0.0.2`

Download

* `IBM_License_key_Server_9002.zip`

Unzip

* Choose the your platform, Linux for example, and unzip to get `IBM_License_Key_Server_Linux`

Use Installation Manager to install the license key server on the chosen machine, from this installation repository (still taken Linux as an example):
`.../IBM_License_key_Server_9002/IBM_License_Key_Server_Linux/LKSSERVER_LINUX/disk1/`

You get something like:

* `/opt/IBM/LKS-9.0.0.2/`
  * `bin/`
  * `license/`
  * `logs/`
  * `swidtag/`


## Preparation

Gather information on the license key server machine:

```
cd /opt/IBM/LKS-9.0.0.2/bin
./lmutil lmhostid
```

replies something like:

```
lmutil - Copyright (c) 1989-2024 Flexera. All Rights Reserved.
The FlexNet host ID of this machine is ""a41fd5028d74 c654015c101f""
Only use ONE from the list of hostids.
```

And `hostname` replies something like `AP89-RGK`

## Operations

### Get your key(s)

Note: ELM 7.0.3 keys are still compatible with 7.1.

* [IBM Support - Licensing](https://www.ibm.com/support/pages/ibm-support-licensing-start-page)
* [Log in to the License Key Center](https://licensing.flexnetoperations.com/)
* go to your account
* go to "Get Keys"
* go to "IBM Tokens"
* select license keys from the set your need and click "Next"
  - for example "Item ordered" = "IBM Engineering Lifecycle Management Suite Token Subscription Lic"
* fill the form using the information you gathered from the license server, for example:
  - Number of keys to generate: 30 (enter what you purchased)
  - Server Configuration: Single License Server
  - Host ID Type: Ethernet Address
  - Host ID: a41fd5028d74
  - Hostname: AP89-RGK
  - Port: 27000 (it’s the default)
* and click "Generate" (*)
* from the page displaying the licences
  - click "Download License Keys" and save `license.dat` as `license703.dat` (any meaningful name actually)
    * or else copy and paste the text box content to a `license703.txt` file (for example)
  - click "Download Jazz Keys" to download `JazzTokens.zip` as `JazzTokens703.zip`
* unzip `JazzTokens703.zip`; it contains `.jar` files; choose the ones you need, for example
  - `EWM_v7.0.3_DEVELOPER-Token-Term-3AB95A8-22-16.jar` for EWM v7.0.3 or 7.1 developer licenses.

(*) Note:

* if it takes too long, it may still have worked
  - go to "View Keys by Host"
  - Host ID: a41fd5028d74
  - Host Type: SERVER
  - click "Search"
  - click the link with the hostname
  - click "Change"
  - look for a41fd5028d74

### Install your keys on the license key server

To start the LKS:

```
cd /opt/IBM/LKS-9.0.0.2/bin
startLKS.sh
```

To stop the LKS:

```
cd /opt/IBM/LKS-9.0.0.2/bin
stopLKS.sh
```

Note: you would have to delay a bit before starting again.

* replace the content of the `LKS_License.dat` file on the license key server (the name for this file might differ)) with the content of the `license.dat` file you downloaded from the license key center.
* update:

```
cd ${LKS}/bin
./lmutil lmreread
```

(if LKS is already started; otherwise: start it)

Check the log file: `/opt/Jazz/LKS-9.0.0.2/logs/lmgrd.log`

Fix any error and reread or stop+start if needed.

### Install your keys in the JTS

* go to the JTS admin page, then "License Key Management" page, like <https://test.jazz:9443/jts/admin#action=com.ibm.team.repository.admin.manageLicenses>
* in the "Floating License Server" section, add the `.jar` files you selected from `JazzTokens.zip`
  - you may see a warning "You have uploaded a floating license. Saving the changes will convert this server into a floating license server.", accept
* indeed, now, the "Server Information" section should show that this JTS "is a Floating License Server"
* the "Client Access License Types" and "Floating License Server" sections should show the tokens you uploaded
* at the bottom of the page, configure the "IBM Rational Common Licensing Service" to point to your FlexNet service, for example:
  - IBM Rational License Key Server: 27000@lks.jazz
* the "IBM Rational Common Licensing Service" status should be OK.


## Test

Assign yourself a license (and "Save").

Check on the license server what licenses are in use (i.e. someone is working on a project area):

```
cd ${LKS}/bin
./lmutil lmstat -a -c 27000@localhost
```

