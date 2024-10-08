# Enable Oracle Data Safe and monitor the Autonomous DB on a private endpoint

## Overview

This lab will show you how to monitor, from the security point of view, the ADB-S behind ORDS server with a private endpoint, to audit through the dashboard threats like, for example, failures in users logon. This just a basic employment of Data Safe, which allows administrators to make self-assessments on DBs registered, on-premises and Cloud, detect wrong configurations, and many other tasks. This is a beginner tutorial to show step-by-step how to implement the private endpoint access at an ADS-S in a private subnet, and leverage the pre-defined policies, such as the Logon Failures, that could be a sign of suspicious activity on DB.

For an in-depth approach to Data Safe, please refer to the LiveLab: [Get Started with Oracle Data Safe Fundamentals](https://livelabs.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=598), which explores all the service functionalities.

Estimated Time: 20 minutes

### Objectives

* Create a Dashboard with standard Audit Policies deployed by default on an ADB-S instance and detect login failures.

### Prerequisites

* Tenant admin role
* The Oracle Autonomous Transaction Processing database is named in a way like **&ltabrv>DB_XS** (created in Lab 1: Deployment)
* Cloud shell open to submit oci-cli commands, with environment variables coming from the initial Terraform setup available.
* Get the **&ltabrv>** code with the command `echo $TF_VAR_proj_abrv` and replace in the following steps

## Task 1: Register and Monitor an ADB instance

* Set the environment variables

```bash
<copy>
source ./terraform-env.sh
</copy>
```

* Get compartment OCID:

 ```bash
    <copy>
    export COMP_NAME=$(oci iam compartment get --compartment-id $TF_VAR_compartment_ocid --query data.name --raw-output)
    </copy>
```

* From Cloud Shell first, check if Data Safe it's enabled in your tenancy and Region:

```bash
    <copy>
    oci data-safe service get --compartment-id $TF_VAR_tenancy_ocid
    </copy>
```

* if already enabled, you should see `is-enabled=TRUE` in the response. In a positive case, you can skip the next steps related to Data Safe activation at the region/tenant level.

![Data_Safe_enabled](./images/data-safe-enabled.png " ")

* In the case Oracle Data Safe hasn't been activated previously in the region of your tenancy, you could do it manually from Web Console, under **Oracle Database**/**Data Safe** menu, click on **Enable Data Safe** button:

![Data_Safe_enabled_manually](./images/data-safe-enabled-manually.png " ")

* or via oci-cli:

```bash
    <copy>
       oci data-safe service enable --is-enabled true 
    </copy>
 ```

* You should see after the button is turned in **Data Safe Dashboard**:

![Data_Safe_on](./images/data-safe-on.png " ")

* from the **Identity & Security**/**Policies** main menu, click on the button **Create Policy**, and set:

    * **Name**: datasafe_policy
    * **Description**: test data safe policies
    * **Compartment**: choose the Lab Compartment
    * Select **Show manual editor**, and copy following policies, replacing <compartment_name\> and <group-name\> with Administrator role:

    ```text
    <copy>
    Allow group <group-name> to manage data-safe-family in compartment <compartment_name>
    Allow group <group-name> to manage autonomous-database in compartment <compartment_name>
    Allow group <group-name> to manage target-databases in compartment <compartment_name>
    </copy>
    ```

    **NOTE**: you could do the same task executing these oci-cli commands. Replace <ADM_GROUP\> with the group-name you want the policies to be applied to.

    ```bash
    <copy>
    export ADM_GROUP=<ADM_GROUP> 
    export statement="[\"Allow group ${ADM_GROUP} to manage data-safe-family in compartment ${COMP_NAME}\",\"Allow group ${ADM_GROUP} to manage autonomous-database in compartment ${COMP_NAME}\",\"Allow group ${ADM_GROUP} to manage target-databases in compartment ${COMP_NAME}\"]";
    echo "${statement}"  >statement.json;
    oci iam policy create --compartment-id $TF_VAR_compartment_ocid --name datasafe_policy --description 'test datasafe policies' --statements file://./statement.json;
    rm ./statement.json;
    </copy>
    ```

* Under **Oracle Database**/**Data Safe**, click on **Target Databases** and then, under **Connectivity Options** on the **Private Endpoints** menu. For further info about [private endpoint](https://docs.oracle.com/en/cloud/paas/data-safe/admds/create-oracle-data-safe-private-endpoint.html#GUID-B601106D-563A-42BE-BEBC-FCC2E97ACFF0)
* Click on **Create Private Endpoint**:
 ![private_endpoint](./images/private-endpoint.png " ")
* Set the following parameters:
    * **Name**: **PE\_&ltabrv\>DB\_XS**
    * **Compartment**: choose your [compartment_name]
    * **Virtual Cloud Network** in [compartment_name]: **&ltabrv\>-vcn**
    * **Subnet** in [compartment_name]: **&ltabrv\>-subnet-private**
    * **Network Security Groups**: choose **&ltabrv\>-security-group-adb**
  as shown in the following picture, where compartment_name is "securityworkshop":

   ![private_endpoint_creation](./images/private-endpoint-creation.png " ")

   Click on button **Create Private Endpoint** and after a few minutes you should see **Active**:
   ![private_endpoint_list](./images/private-endpoint-list.png " ")

* and clicking on **PE\_<abrv\>DB\_XS**, you can access the details of the private end-point created:
   ![private_endpoint_created](./images/private-endpoint-created.png " ")

   In this way, **Data Safe** services have a private channel to access DBs not publicly accessible.

* From **Data Safe** / **Overview**, we are ready to register Autonomous DB through a wizard:
    ![ADB_Wiz](./images/adb-wiz.png " ")

    * **Select Database in** compartment, in our case **&ltabrv\>DB\_XS** target:
    ![ADB_Wiz_DB](./images/adb-wiz-db.png " ")

    * Set **Connectivity Option** leveraging Existing Private Endpoint, **PE\_&ltabrv\>DB\_XS**:
    ![ADB_Wiz_PE](./images/adb-wiz-pe.png " ")

    * Choose to add ingress/egress rules to an existing Network Security Group for ADB-S, i.e. **&ltabrv\>-security-group-adb**:
    ![ADB_Wiz_NSG](./images/adb-wiz-nsg.png " ")

    * **Review and Submit**, and click on the **Register** button at the bottom of the page:
    ![ADB_Wiz_Sub](./images/adb-wiz-sub.png " ")

    * at the end of process, you should see something like this. This will take a few minutes:
    ![ADB_Wiz_End](./images/adb-wiz-end.png " ")

* In ADB-S details, you can check if the DB instance has been correctly registered to be monitored by Data Safe:
![DB_Details](./images/db-details.png " ")

* Under **Oracle Database**/**Data Safe** menu and **Target Database** left menu, on your compartment, you should see under **Target Databases** list the **&ltabrv\>DB_XS** instance it has been just registered:
![Target_DB](./images/target-db.png " ")

## Task 2: Configure Auditing on registered ADB-S instance

* Click on **Data Safe** main menu, to watch the main dashboard, under **Security Center**/**Dashboard**:
![Data_Safe_Dashboard](./images/data-safe-dashboard.png " ")

* under **Security Center / Activity Auditing / Audit Profiles** :
![Security_Center](./images/security-center.png " ")

    select **&ltabrv\>DB_XS** target details:
![Target_DB_Dash](./images/target-db-dash.png " ")

* from **Data Safe / Security Center / Activity Auditing /Audit Policy / Audit Policy Details** , click **Retrieve** Button, to get actual policy pre-deployed on Autonomous instances.

    ![Policy_Details](./images/policy-details.png " ")

* In **Audit Policy Details**, **ORA\_LOGON\_FAILURES** policy is one of the **Oracle Pre-defined Policies** enabled by Default on Autonomous DB for all users. Click on the **View Details** link and leave as already set:
![Pre_def_policies](./images/pre-def-policies.png " ")

   You could eventually modify the policy to apply it to a restricted number of users. In this case, you have to click finally on **Update and Provision** after modified.

* under **Security Center / Activity Auditing / Audit Profiles**:
![Security_Center_Prof](./images/security-center-prof.png " ")

    select **&ltabrv\>DB_XS**:
    ![Security_Center_DB](./images/security-center-db.png " ")

    to see target details:
    [Audit_Trail](./images/audit_trail.png " ")

  * Start Audit Trail: under **Data Safe / Security Center / Activity Auditing / Audit Trails / Audit Trails Details**,
    ![Auditing](./images/audit.png " ")

* Select **Target Database**: **&ltabrv\>DB_XS**

  * Click on the **Start** button:
    ![Audit_Trail_Start](./images/audit-trail-start.png " ")

  * and set the current date/time for **Select Start Date**:
    ![Audit_Trail_TargetDB](./images/audit-trail-targetdb.png " ")

  * At the end Audit Trail page should show **Collection State**: "COLLECTING":
    ![Audit_Trail_Collecting](./images/audit-trail-collecting.png " ")

* At this stage, from **Data Safe / Security Center / Dashboard** you can have an overall security status overview, in which you can notice that as started a security assessment on DB instance that shows the first useful advices about **High**/**Medium** risks:

    ![Dashboard](./images/dash-board.png " ")

## Task 3: Generate logon failure events on the Autonomous DB instance

* In a terminal session on the local desktop, following the instructions reported in [Access Bastion Service](/workshops/paid/?lab=bastion&nav=new), connect through SQL client to ADB-S as admin 3 times with a wrong password, to simulate the intent to access as Admin to ADB-S, in the highly unlikely hypothesis that all DB access barriers have been breached:

```text
    <copy>
    sql ADMIN/MyWrongPWD@$CONN_string
    </copy>
```

* After a few minutes, from **Data Safe / Security Center / Activity Auditing** look at **Failed Login Activity** graph that shows incorrect login attempts done at previous steps:

    ![Auditing_Login](./images/auditing-login.png " ")

* From **Data Safe / Security Center / Activity Auditing** you will have more details about failed login just done. If you don't see any events, wait a moment to see the effects of the previous logon:
    ![Dashboard_Login](./images/dashboard-login.png " ")

Once the setup and testing of access have been completed you are ready to **Proceed to the next lab.**

## Learn More

* Ask for help and connect with developers on the [Oracle DB Microservices Slack Channel](https://bit.ly/oracle-database-microservices-slack)  
Search for and join the `oracle-db-microservices` channel.
* LiveLab: [Get Started with Oracle Data Safe Fundamentals](https://livelabs.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=598)

## Acknowledgements

* **Author** - Andy Tael, Developer Evangelist;
               Corrado De Bari, Developer Evangelist
* **Last Updated By/Date** - Andy Tael, August 2022
