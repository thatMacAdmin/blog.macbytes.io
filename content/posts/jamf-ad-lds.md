---
title: "AD LDS w/ ADAM Sync for jamf Pro"
date: 2020-03-19T17:59:36-04:00
draft: false
toc: true
images:
tags:
  - adlds
  - jamf
---

## Introduction
Let's say that you are standing up an on prem jamf Pro server for your organization. This is a big undertaking. There are a ton of things to consider. One of the core pieces of functionality that you will probably want, is to be able to scope policies and software items to existing active directory groups used by your organization. This is a fairly straightforward goal to achieve on the surface, you just put your active directory details into jamf Pro and away you go.

But what happens if your security team is not okay with your jamf Pro server touching AD? In organizations I have worked for in the past, this was the case. The logic goes something like this:

- The jamf Pro server is edge facing (its accessible from the internet).
- Active directory is not accessible from the edge.
- If someone compromises the jamf server they could potentially get credentials or otherwise attack AD directly from the jamf Pro server.
- So the jamf Pro server potentially makes AD accessible from the edge.
- This is bad!

So what can we do? The way I see it there are a few options:

- Don't use AD at all - An option but probably not why you are here...
- Make jamf Pro internal only - Not a great idea these days.
- Use something like NoMAD to store groups as an extension attribute (see this blog post here)
- Use an AD proxy - Not really that secure in reality
- Use an application directory with AD LDS with ADAM sync

You may have guessed it based on this posts title but I would suggest that the best option if you are not able to connect to AD directly is to use LDS with ADAM sync. If you set it up right, it is a secure TLS connection between LDS and your jamf Pro server using a username that exists nowhere else, so its useless if compromised.

With all that in mind, what will you need in order to set this up?

- Windows Server 2016 or 2019
- jamf Pro 10.x or later (this may work with 9.x but at this point you should be running 10)
- A valid Server SSL certificate with private key, root and intermediates
- A good text editor

This is a long article as there are a lot of steps, but stick with it and you will have something awesome at the end!

## Setting up AD LDS
### Install the server role
I am sure there are more command line ways to do this, but I like GUI's!

1. Open Server Manager
2. Click Manage --> Add Roles and Features
3. Click Next until you get to the 'Select server roles' screen
4. Check the box for: Active Directory Lightweight Directory Services
5. Click Next until the service installs
6. Click Finish

### Run the AD LDS setup wizard
Now that the server has the AD LDS role, we need to setup an instance on the server. To do that we need to run the Active Directory Lightweight Directory Services Setup Wizard. A lightweight directory can be setup two ways a unique istance or a clone of an existing instance. In our case we will setup a unique istance. When asked to enter the partition information, you probably want to structure it like this: DC=myApp,DC=companyDirectory,DC=com. In this instance your main directory that you are syncing from would be companyDirectory.com. Some guides will tell you to make the app directory a CN. This is a mistake as you will have issues syncing nested OU's if you do that. Make it DC.

Here are the steps:

1. Click Start --> Windows Administrative Tools --> Active Directory Lightweight Directory Services Setup Wizard
2. Click Next through the intro and select: A unique instance
3. At the next screen enter an instance name and description (remember the instance name, you will need it later)
4. You will then be prompted to choose your ports (the default is usually fine)
5. At the next screen ensure you select: Yes, create an application directory partition
6. For partition name enter something like: DC=myApp,DC=companyDirectory,DC=com
7. Choose the location you want to store the LDS data files (this could be a secondary drive etc.)
8. Select Network service account (the default) at the next screen unless yuou have a good reason not to
9. Select the account you want to be the default admin for the LDS instance - usually the account you are in now. We can add more admin accounts later once the directory is up and running.
10. Under Importing LDIF Files I select:
  - MS-AdamSyncMetadata.LDF (needed for ADAM Sync)
  - MS-InetOrgPerson.LDF
  - MS-User.LDF
  - MS-UserProxy.LDF (needed for user proxy authentication)
  - MS-UserProxyFull.LDF (needed for user proxy authentication)
11. A couple more next presses should finish the wizard up

Now we have the LDS instance role installed and the directory is running on your server! Lets move on to getting connected to it.

### Setup the ADSI edit tool
You are able to manage the LDS via the AD Directory Services PowerShell cmdlets, and I will provide a few examples below, but for many folks a GUI will be helpful. For that reason we will setup ADSI:

ADSI can be found at: Start --> Windows Administrative Tools --> ADSI Edit

Once open you will need to setup a connection to your server. If you have a DNS record setup for your server (you should) then use the FQDN, otherwise you can use localhost for the server address in the settings that follow:

1. Click Action --> Connect to
2. Enter the following settings:
  - Name: My Application LDS
  - Connection Point choose: Select or type a Distinguished Name or Naming Context
    - Enter: DC=myApp,DC=companyDirectory,DC=com
  - Computer choose: Select or type a domain or server
    - Enter: myADldsServer.company.com or localhost
  - For now do not check SSL (we will set this up later)
3. Click OK

That will create the connection. You can now browse and manage the directory with a nice GUI. We are now done with the initial directory setup! Next we will look at syncing your AD data to it.

## ADAM Sync
### Some Background
In order to sync an object from AD to AD LDS the schema for the attributes we want to sync must match exactlty. To make matters more complex, LDS by default, does not have the same schema as Active Directory. Even worse, if you have custom attributes you want to sync you will have to add those manually. If you fail to add the needed attributes, then your sync will fail, often with a cryptic hard to understand message. Fear not, I think I have covered most of the use cases. Read on for more.

For jamf Pro, we are going to focus on syncing Person, Group and Organizational Unit objects. AD contains a ton more stuff but we dont really need that for jamf. Our goals are to get username lookup and group association lookup.

### Import the AD 2K8 Schema
The first thing we will do is to import the default Active Directory 2008 schema. This will give us a good starting point from which to sync. To do that enter the following from an elevated command prompt:

```shell
cd C:\Windows\ADAM
ldifde –I –f ms-adamschemaw2k8.ldf –s localhost:389 –k –j . –c “cn=configuration,dc=x” #configurationNamingContext
```

### Edit your AdamSyncConf.xml
ADAM Sync is controlled with an XML based configuration file. We will start with the simplest possible configuration file to ensure we get a good full first sync. I would then advise that you add your attributes one by one and syncing after each add. This takes more time, but the first time you set this up, I promise if you try and sync too much at once you are going to run into issues and spend more time troubleshooting than if you just started small and worked your way up.

Here is a smaple config file you can edit and use:

```xml
<?xml version="1.0"?>
<doc>
 <configuration>
  <description>AD LDS Sync</description>
  <security-mode>object</security-mode>
  <source-ad-name>companyDirectory.com</source-ad-name>
  <source-ad-partition>dc=companyDirectory,dc=com</source-ad-partition>
  <source-ad-account>directory.user</source-ad-account>
  <account-domain>companyDirectory</account-domain>
  <target-dn>dc=myApp,dc=companyDirectory,dc=com</target-dn>
  <query>
   <base-dn>dc=companyDirectory,dc=com</base-dn>
   <object-filter>(&#124;(objectCategory=Person)(objectCategory=OrganizationalUnit)(objectCategory=Group))</object-filter>
   <attributes>
    <include>sAMAccountName</include>
    <include>member</include>
    <exclude></exclude>
   </attributes>
  </query>
  <schedule>
   <aging>
    <frequency>0</frequency>
    <num-objects>0</num-objects>
   </aging>
   <schtasks-cmd></schtasks-cmd>
  </schedule>
  <user-proxy>
  </user-proxy>
 </configuration>
 <synchronizer-state>
  <dirsync-cookie></dirsync-cookie>
  <status></status>
  <authoritative-adam-instance></authoritative-adam-instance>
  <configuration-file-guid></configuration-file-guid>
  <last-sync-attempt-time></last-sync-attempt-time>
  <last-sync-success-time></last-sync-success-time>
  <last-sync-error-time></last-sync-error-time>
  <last-sync-error-string></last-sync-error-string>
  <consecutive-sync-failures></consecutive-sync-failures>
  <user-credentials></user-credentials>
  <runs-since-last-object-update></runs-since-last-object-update>
  <runs-since-last-full-sync></runs-since-last-full-sync>
 </synchronizer-state>
</doc>

```

Here are the things you need to edit:

**description:** This can be anything you want

**source-ad-name:** This should be the root of your company directory

**source-ad-partition:** This should be the DN for the source ad

**source-ad-account:** An account with at least read priviliges in the source directory

**account-domain:** The domain the account is in

**target-dn:** This is the DN we used when we made the LDS instance

**base-dn:** This is the DN where you want to start syncing (you can have more than one.. but don't)

**object-filter:** I would leave this as is.. more on that later

**attributes:** always use include, just include the attributes you want to sync

### A note on the base-dn
On the surface it would seem that limiting the base-dn to only the OU's that you want to sync and having two or three of them would make the sync targeted and faster. Unfortunatly because of the way dirsync works this is not the case. If you don't use the root DN, the sync will take 10 to 20 times as long and will generate a huge log file. Besides, dirsync will scan every item in the directory no matter what. Just use the root DN for this. You will be happier.

### The object filter
The object filter allows you to filter the objects that ADAM will sync. This is important particularly in large directories where there are a ton of objects. The filter I have ensures that we get users, goups AND OU's. We need to ensure that OU's are included because although the OU strcuture is created on the initial sync, if something is moved inside the directory, it may not update correctly if OrganizationlUnit is not in the filter. If you are really good with LDAP Search filters and you want to tweak what I have here, go for it, otherwise i'd just leave it as is.

### Install the AdamSyncConf.xml
Now that you have your initial sync setup you need to install the sync configuration. To do that open an elevated command prompt and enter:

```shell
C:\Windows\ADAM\adamsync.exe /install localhost:389 C:\path\to\adamSyncConf.xml
```

### Let's Sync
At this point we should be ready to do a basic sync. Open a command prompt. Enter the following command to sync objects:

```shell
C:\Windows\ADAM\adamsync.exe /sync localhost:389 dc=myApp,dc=companyDirectory,dc=com /log C:\path\to\log.log
```

We want to make sure we use the /log tag so that if there are any errors we can see what might be going on. I will discuss some common errors and how to handle them later in the article. One more thing to note, if the account you are using to run the sync cannot access the log, or the log folder to create a new log, you sync will instantly fail. So be sure permissions on the log and log folder are set correctly.

All being well you should now have an AD LDS instance with your sAMAccountName and group member attribute synced! You can now add attributes one at a time and re-sync using the same command above. Once you get all your attributes included you are done! But what if you have custom attributes or you are getting errors about missing attributes from a class? Read on!

### Editing your schema so objects include other attributes

Sometimes you may need to add an attribute in your schema to an existing class object. You will know you need to do this if you get an error in your logs like:

Object Class Violation
Adding atttributes: list of attributes here

The attributes listed need to be added to that objects class. To do that do the following:

1. Open an elevated command prompt
2. Enter: regsvr32 schmmgmt.dll

Now do  the following:

1. Launch mmc.exe with the user you set at the LDS admin
2. Add the AD Schema snap in
3. Right click the default Active Directory Schema and choose: Change Active Directory Controller
4. Click the radio button this domain controller or AD LDS instance
5. At the top of the name field click on <Type a Directory Server name> this will allow you to enter a hostname
  - Enter: localhost
6. Once it scans the server, with localhost selected hit OK
7. Hit Yes on the pop-up about the forest root dialog that pops up
8. Expand out the tree and select classes
9. Select the class you want to edit, usually user, group, person or organizationUnit
10. Right click the class and choose properties
11. Choose the attributes tab
12. Click Add
13. Find the attibute you want and click OK
14. Click Apply then OK

Now you can add the attibute as in include to your sync configuration, install the configuration and re-sync.

### Adding custom attributes to LDS
If you are anything like the companies I have set this up for in the past you will have custom attributes that you need to sync. To do this we will need to go through a few different steps. First you will need to generate an ldf file with the attribute differences between your source AD and the target LDS instance. To do that we will use a tool called AD DS/LDS Schema Analyzer.

1. Open AD DS/LDS Schema Analyzer it can be found at this path:
```shell
C:\Windows\ADAM\ADSchemaAnalyzer.exe
```
2. Choose File --> Target Schema
3. Enter the details for your source directory (not AD LDS) and press OK
4. Choose File --> Base Schema
5. Enter the details for your AD LDS instance and press OK
6. Choose Schema --> Hide present elements
7. Choose Schema --> Mark all non present elements as included
8. Finall export an ldf file by selecting: File --> Create LDIF
9. Save this file somewhere you can find again

Now we have an LDIF file with all the differences between your source AD and your LDS instance. You could just import all of these differences so that both directories match, but in large complex directories this can fail causing other issues. My recommendation would be to create individual ldf files for each attribute you want to import and then import them one by one. Because we are only syncing select attributes you will probably only have a few.

This is an example of such a file for a site code attribute, I have redacted a few elements since I pulled this from a real directory. But all I did was copy paste this attribute from the huge file containing a few thousand differences into its own file:

```
# Attribute: sitecode
dn: cn=sitecode,cn=Schema,cn=Configuration,dc=X
changetype: add
objectClass: attributeSchema
attributeId: [redacted]
ldapDisplayName: sitecode
attributeSyntax: 2.5.5.12
adminDescription: Company Attribute
adminDisplayName: Company-Attribute10
# schemaIDGUID: [redacted]
schemaIDGUID:: [redacted]
oMSyntax: 64
isMemberOfPartialAttributeSet: TRUE
isSingleValued: TRUE
systemOnly: FALSE
rangeLower: 1
rangeUpper: 1024
```

Now we need to import this into the LDS schema. From an command prompt enter:
```shell
ldifde -i -u -f savedLDF.ldf -s localhost -j . -c "cn=Configuration,dc=X” #configurationNamingContext"
```

You can repeat this command for any additional attributes you may have saved that you want to import. Then follow the instructions above to edit the class object schema to include the new attribute.

Finally run a sync again. Assuming you get no errors you now have a fully synced directory! Lets get LDAPS enabled and create a read account for jamf Pro to use!

## Enable User Proxy Objects
### Background
User proxy objects enable you to do authentication forwarding. This means that your LDS does not store credentials. Its stores a proxy representation of the object and sends the authentication request to the source directory. This works well if you want to use LDAP authentication off your network, but cannot store credentials on servers that touch items in the DMZ!

### Edit the AdamSynConfig.xml
Adding the functionality is as simple as adding two lines in between the user proxy tags and a single additional include to your sync configuration xml:

```xml
<include>objectSID</include>
```

```xml
<user-proxy>
    <source-object-class>user</source-object-class>
    <target-object-class>userProxyFull</target-object-class>
</user-proxy>
```

Now re-install the configuration and re-sync.

## Enable LDAPS
### Background
When you setup AD LDS, by default it will attempt to enable both LDAP (389) and LDAPS (636). In most situations however your server is probably setup with a self signed certificate and this will not be suitable for LDAPS. You will need a valid SSL certificate complete with leaf, intermediates and root. Once you have that, either from your internal PKI or a third party certificate issuer, follow these steps to get it imported.

### Certificate requirements
Your SSL Certificate must have the following:
- Server FQDN must be in subject and SAN
- Certificate must be set for the Service Authentication OID
- Private key must exist

### Importing certificates
Windows Server uses Schannel to manage certificates somewhat automagically. We need to import them into the certificate store of the service account setup when we created the AD LDS instance. The service acount will be named after your LDS instance name. To import the certificate do the following:

1. Launch an elevated mmc session
2. File --> Add Snap-In --> Certificates --> Add
3. When prompted select Service Account then local computer
4. From the list of service account select your ADAM service account (instance name)
5. Expand out: Certificates --> ADAM_serviceAccount\Personal
6. Right click ADAM_serviceAccount\Personal --> All tasks --> Import
7. Follow wizard to import certificate into the personal store

### A note on roots and intermediates
When you import your certificate into the store you will likley import the root and intermediates certs into the personal store as well. This can cause issues. Many intermediates might also be tagged for server authentication and this can confuse Schannel the cert manager. Schannel will always select the cert with the oldest expiration date that is tagged for Service Authentication. If one of your intermediates is setup this way, then you will have issues since you dont have the private key for those certs. This will cause the setup of LDAPS to fail.

To be safe and to ensure that Schannel selects the correct cert, only put the leaf cert (server cert) and its private key in the personal store. Place the roots and intermediate in the corresponding folders.

### Setting key permissions
Now that we have the certificate installed into the service account's store we just need to grant permission to the keys. To do this do the following:

1. Open this folder:
```shell
C:\ProgramData\Microsoft\Crypto\RSA
```
2. Right click on the MachineKeys folder
3. Choose: Properties --> Security --> Advanced --> Add
4. Click Select a principal
5. Enter: NETWORK SERVICE
6. Click: Check Names
7. Click: OK --> OK --> Apply --> OK --> OK

### Restart the LDS Service

1. Open the Services control window
2. Location your LDS service (instance name)
3. Click: Restart

If everything went well you should be able to do the following:

1. Open ADSI
2. Right click your LDS instance and choose: Settings
3. Check the box for SSL then click: OK

If you dont get an error you should be good to go!

## Get jamf Pro linked
### Create a local read only account
We will need to create a local read only account inside of the LDS instance. This will ensure that jamf Pro can read the LDS instance but if the credentials are compromised, they cannot be used on the source directory!

1. Open ADSI
2. Right click the root DN
3. Choose: New --> Object --> OrganizationalUnit
4. For value enter: jamfProUsers
5. Right click your new OU
3. Choose: New --> Object --> user
4. For value enter: jamfProRead

### Set account password
Now that your account is created we need to set a password. Remember that many server environment have much stronger password requirements than a normal desktop environment. For this reason I would recommend a long strong password. To set the account password:

1. Right click the jamfProRead account
2. Choose reset password
3. Follow the prompts to reset the password

### Enable account
Once the password is set, its tempting to add the data into your jamf Pro LDAP connector but if you do it will fail. All LDS accounts are diabled by default. To enable the account do the following:

1. Right click the jamfProRead account
2. Choose properties
3. Locate the key: msDS-UserAccountDisabled
4. Double click the property to edit it
5. Set the property to FALSE and click OK

You should now be able to follow the jamf Pro standard documentation to add a manual LDAP directory! That is outside the scope of this article but your jamf technical account manager should be able to help.

## Setup a Scheduled Task

## Troubleshooting
### DNManip Error
You may occasionally come accross an error that has very little information. It will just have a GUID and the error:

```shell
DNManip
```

This is cryptic. What it means is that the object in question is a conflict object. Conflict objects have a special character in them that LDS cannot sync. This snippit of PowerShell can help you find those records in your source directory:

```powershell
Get-ADobject -LDAPFilter:"(objectClass=user)" | ?{$_.DistinguishedName.Contains($_.objectGUID)} | %{write-host $_}
```

Once you find the objects you should either remediate them yourself or work with you AD team to remediate them. Once you do your sync should proceed as normal.

### Force Sync
If you just cannot get past some errors, you can always force sync. This can be useful in large complex environments:

```shell
C:\Windows\ADAM\adamsync.exe /sync localhost:389 dc=myApp,dc=companyDirectory,dc=com /force -1 /log C:\path\to\log.log
```
