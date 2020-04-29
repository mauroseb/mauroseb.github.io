---
layout: posts
title: "OCP 4.3 on Azure Tests"
date: 2020-04-02
categories: [blog]
tags: [ ocp, azure, kubernetes ]
author: mauroseb
---

## Intro

These are my first notes testing the official procedure to deploy OCP4.3 on Azure. [^1]

## Table of Contents

 1. [Prerequisites](##Prerequisites)
 2. [Install](##Install)


## Prerequisites

The BOM to start with:

 - Fedora (31) workstation I will be working with 500 MB free local disk
 - Install az CLI if not there [^2] [^3]
 - Valid azure account with the following:
   - At least a Pay-As-You-Go Subscription
     - free-tier sub does not allow to request a resource limit increase
   - Service Principal with "User Access Administrator" Role and a few other privileges
   - Proper limits set
     - The only limit that would need to change from default is [^4]:
       - Compute-VMs (vCPUs): 34 (default is 10)
     - The rest of required resource amounts should be okay with defaults in place:
       - VNet: 1
       - Network Interfaces: 6
       - NSGs: 2
       - Network Load Balancers: 3
       - Public IPs: 3
       - Private IPs: 7


             $ az vm list-usage --location "westcentralus" -o table
             Name                               CurrentValue    Limit
             ---------------------------------  --------------  -------
             Availability Sets                  0               2500
             Total Regional vCPUs               0               34      <=
             Virtual Machines                   0               25000
             Virtual Machine Scale Sets         0               2500
             Dedicated vCPUs                    0               3000
             Total Regional Low-priority vCPUs  0               10
             Standard DAv4 Family vCPUs         0               0


    - Existing DNS zone created for the cluster (i.e. t1.oddi.info)

          $ dig +short t1.oddi.info ns
          ns2-08.azure-dns.net.
          ns1-08.azure-dns.com.


  - Create Service Principal

    1. Choose the right subscription for the account

           $ az account set -s XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

    2. Create SP

           $ az ad sp create-for-rbac --role Contributor --name ocp43
           Changing "ocp43" to a valid URI of "http://ocp43", which is the required format used for service principal names
           Creating a role assignment under the scope of "/subscriptions/XXXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
           Retrying role assignment creation: 1/36
           {
           "appId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
           "displayName": "ocp43",
           "name": "http://ocp43",
           "password": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
           "tenant": "XXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
           }


    3. Add User Access Administrator role to the SP [^4] [^5]

           $ az role assignment create --role "User Access Administrator" --assignee-object-id $(az ad sp list --filter "appId eq 'XXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX'"  | jq '.[0].objectId' -r)
           {
           "canDelegate": null,
           "id": "/subscriptions/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/providers/Microsoft.Authorization/roleAssignments/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
           "name": "b40a2165-a2cb-403c-b9fd-dd492b421774",
           "principalId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXXXXXX",
           "principalType": "ServicePrincipal",
           "roleDefinitionId": "/subscriptions/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/providers/Microsoft.Authorization/roleDefinitions/18d7d88d-d35e-4fb5-a5c3-7773c20a72d9",
           "scope": "/subscriptions/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
           "type": "Microsoft.Authorization/roleAssignments"
            }


    4. Add Azure Active Directory Graph permission where 00000002-0000-0000-c000-000000000000 is the resource App ID for the Windows Azure AD and 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7 corresponds to a defined role to manage new apps that this app creates or owns [^6]:

           $ az ad app permission add --id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX --api 00000002-0000-0000-c000-000000000000 --api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role
           Invoking "az ad app permission grant --id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX --api 00000002-0000-0000-c000-000000000000" is needed to make the change effective


    5. Approve permissions

           $ az ad app permission grant --id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX --api 00000002-0000-0000-c000-000000000000
            {
              "clientId": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
              "consentType": "AllPrincipals",
              "expiryTime": "2021-04-01T00:26:30.437474",
              "objectId": "I5VYFa60pkGfD1vXlfF3D3hgJ9wpnM1AqMNPAjHcVgM",
              "odata.metadata": "https://graph.windows.net/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/$metadata#oauth2PermissionGrants/@Element",
              "odatatype": null,
              "principalId": null,
              "resourceId": "dc276078-9c29-40cd-a8c3-4f0231dc5603",
              "scope": "user_impersonation",
              "startTime": "2020-04-01T00:26:30.437474"
            }


 - Create a cluster in ```cloud.redhat.com```, pick Azure as infrastructure provider, method IPI and download the oc client, openshift-installer and pull-secret


## Install

I am running different tests based on the IPI documentation. 

 1. SSH KEY

    1.a. Generate an SSH key for passwordless-auth if you I do not have one

    1.b. Run ssh-agent in background and add your key to it

        $ eval "$(ssh-agent -s)"
        $ ssh-add ~/.ssh/id_rsa


 2. Ensure you already have downloaded and added the oc client and installer current version (4.3.8) in your path

 3. Run intsallation with customizations
 
    3.a. Create install config
    
        $ openshift-install create install-config --dir=ocp4-on-azure-test2
        ? SSH Public Key /home/maur0x/.ssh/id_rsa.pub
        ? Platform azure
        ? Region westeurope
        ? Base Domain t1.oddi.info
        ? Cluster Name ocp43-b
        ? Pull Secret [? for help] ***************************************

    3.b. Edit install-config.yaml to add customizations
    
    3.c. Make a backup of the modified file
    
    3.d. Deploy
     
        $ openshift-install create cluster --dir=ocp4-on-azure-test2 --log-level=info

    Alternatively for step 3. the default configuration with 3 masters and 3 worker nodes can be deployed without any of the above substeps.
    
        $ openshift-install create cluster --dir=ocp4-on-azure-test1 --log-level=info


## References

 [^1]: https://docs.openshift.com/container-platform/4.3/installing/installing_azure/installing-azure-customizations.html#installing-azure-customizations
 
 [^2]: https://docs.microsoft.com/nl-nl/cli/azure/install-azure-cli-yum?view=azure-cli-latest
 
 [^3]: https://docs.microsoft.com/nl-nl/cli/azure/get-started-with-azure-cli?view=azure-cli-latest
 
 [^4]: https://docs.openshift.com/container-platform/4.3/installing/installing_azure/installing-azure-account.html#installation-azure-increasing-limits_installing-azure-account
 
 [^5]: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/openshift-container-platform-4x

 [^6]: https://blogs.msdn.microsoft.com/aaddevsup/2018/06/06/guid-table-for-windows-azure-active-directory-permissions/
