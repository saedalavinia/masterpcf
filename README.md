# Reference Project For the Deployment of Pivotal Cloud Foundry on AWS, including sample Apps, Monitoring and Logging Services
------
This guide helps you deploy an instance of Pivotal Cloud Foundry to your AWS, deploy two sample applications one which us RDS MySQL, and configure Log forwarding and Monitoring services on your Cloud Foundry to use New Relic and ELK Stack, all with less than 10 minutes of configuration.

By the end of this guide, you will be able to accomplish the followings:   
    
1. Deploy Pivotal Cloud Foundry (PCF) to you AWS; the default is to deploy PCF to us-east-1 region across 2 availability zones: 
> (i) Running PCF's CloudFormation Template on your AWS.  
> (ii) Deploying and starting an instance of PCF's Ops Manager.   
> (iii) Configuring, deploying, and starting an instance of PCF's Ops Manager Director (Bosh).   
> (iv) Configuring, deploying and starting PCF's Elastic Runtime VMs.   
> (v) Configuring an Administrative user on PCF. 

2. Deploy two sample applications to your PCF: 
> (i) Deploying and configuring and RDS MySQL Server for your sample ReST API.  
> (ii) Deploying a Spring Boot ReST API that uses the RDS MySQL server configured in (i).   
> (iii) Deploying a Spring Boot JSF Application that uses the ReST API to read and write data from the RDS MySQL configured in (i).   

3. Configuring an instance of a New Relic CF Service to monitor the the two applications. 
4. Configuring an ELK Stack and feeding the logs of the two applications: 
> (i) Deploying and starting the ELK Stack on AWS. 
> (ii) Configuring PCF Elastic Runtime to forward application logs to the ELK stack.  

### Minimum Prerequisites 
> (i) You must have an AWS Account and your AWS account must allow more than 20 (default) VMs in at least one region (Default is us-east-1)   
> (ii) You must have a registered domain (Preferably with Route 53), a certificate from a CA (Preferably AWS Certificate Manager)   
> (iii) You must have one or more SSH keys configured in your AWS.   
> (iv) You must have a PCF Account in order to download binaries.   
> (v) You must have a New Relic account in order to receive metrics from your applications.   

### Projects
  
This guide references the following projects: 
> (i)   [Optional] https://github.com/saedalav/ansible_controller_pcfdeployment   
> (ii)  https://github.com/saedalav/pcfdeployment    
> (iii) https://github.com/saedalav/addressdatabase  
> (iv)  https://github.com/saedalav/addressapi    
> (v)   https://github.com/saedalav/addressapiclient    
> (vi)  https://github.com/saedalav/newrelic-cf     
> (v)   https://github.com/saedalav/elkforpcf      

Clonse these projects into your local repository, follow the SETUP.md guide in each of the projects and modify the configuration to match your own environment and then execute it: 
```sh
git clone https://github.com/saedalav/ansible_controller_pcfdeployment.git   
git clone https://github.com/saedalav/pcfdeployment.git    
git clone https://github.com/saedalav/addressdatabase.git 
git clone https://github.com/saedalav/addressapi.git     
git clone https://github.com/saedalav/addressapiclient.git     
git clone https://github.com/saedalav/newrelic-cf.git      
git clone https://github.com/saedalav/elkforpcf.git      
```

The key setup points in each of these projects is restated below: 


# Automation of Pivotal Cloud Foundry deployment to AWS
*** 

This project enables you to fully automate (without any manual intervention) the deployment of Pivotal Cloud Foudry to Amazon Web Services using the default (basic) configuration using Ansible, as explained here: 
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform.html
Once deployed, you will be able to modify the basic configuration and customize the different aspect of the Pivotal Cloud Foudry. 

### How the Project is organized
There are 8 Ansible roles, each of which carry a particular task in the deployment of Pivotal Cloud Foudry to AWS.  
1. cloudformation: Deploys the CloudFormation template for PCF on AWS as explained here:
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform-template.html
2. opsmanagerdeploy: Deploys and Start an AWS EC2 VM containg PCF's Operations Manager: 
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform-om-deploy.html
3. directorconfig: Partially Configures Operations Manager Director (Bosh) using Ops Manager Rest API as explained here: 
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform-om-config.html
* A bug in PCF's Operations Manager REST API does not allow this role to fully configure Bosh. In order to get around this bug, the following role modifies the Installation.yml file directly on Operations Manager VM. This is not normally recommended, but allows us to work around the bug. 
4. modifyserver: modifies Installation.yml in Operation Manager VM to finish the configuration of bosh. 
5. directorinstall: Triggers the installation of Operations Manager Director (Bosh) 
6. elasticruntimeupload: Uploads the image for PCF's Elastic Runtime to Operations Manager as explained here: 
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform-er-config.html
7. elasticruntimeconfig: Configures PCF's Elastic Runtime as explained here:
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform-er-config.html
8. createpushuser: creates a Cloud Foudry Super user to be used as administrator.

In the following section, the project setup is explained in detail. 

### Setting up the Project: 

##### Prerequisites
At a minimum, you need the following before you can use this project: 
>  An AWS Account in which, you are able to run more than 20 VMs (the default) in the region where you are planning to deploy Pivotal Cloud Foudry.

In order to run this project, you have two options: 
(i) (recommended) Use an Existing AWS AMI which already included all of the required dependencies to act as your Ansible Controller.
(ii) Prepare a linux enviornment in which you can run Ansible (and all of its dependencies). 

#### Using an Exsting AWS AMI for your Ansible Controller. 
if you wish to use this AMI (ami-025f2b14 for us-east-1), email me at saedalav@gmail.com so that I can share this AMI with you. Once you have the AMI, you must create an instance from this AMI and give it (at minimum) full EC2 and RDS IAM access. The modify the project (As explained in the following sections) and run the job. 
You may use the following project to further automate the deployment of Ansible Controller:
https://github.com/saedalav/ansible_controller_pcfdeployment

#### Prepare your own environment
if you chose to prepare your own linux environment, you must have the following dependencies installed: 
1) Python 2.7+ 
2) Ansible 2.0+
3) Python Boto Library (Boto, Boto3, Botodev)
4) CF-UAAC (Cloud Foudry User Accound and Authentication CLI) See here:
https://github.com/cloudfoundry/cf-uaac
5) CF-CLI (Cloud Foudry Command Line Interface) See here: 
https://github.com/cloudfoundry/cli
Additionally, you need: 
6) Pivotal Cloud Foundry Elastic Runtime 1.10.x Binaries. See here: https://network.pivotal.io/products/elastic-runtime
7) You must set AWS_SECRET_ACCESS_KEY and AWS_ACCESS_KEY environment variables. See here:
http://boto.cloudhackers.com/en/latest/boto_config_tut.html
8) One or more SSH key(s) for accessing EC2 instances in the desired region obtained from AWS.

### How to Modify the project
The project is setup such that for each role, you should only need to modfiy the vars/main.yml file unless you have a particular need that must be address before PCF is deployed. Otherwise, most of other changes can be applied after PCF is up and running using Ops Manager. 
Therefore, for each role: 
1) Modify the vars/main.yml 
Note: Many variables must be consitent accross all roles. For example, once you have set the region to us-east-1, you cannot change the value across roles.
Also note that while some variables can be left as default, others must be changed. In the following section, each variable is explained and user is told if they should change this value or not.

2) run the role in the order presented here. 
Alternatively, you can run deployAll.yml playbook to run all roles at once. However, this may not be the best idea if this is your first time. 

You must first download this project locally: 
```sh
$ git clone https://github.com/saedalav/pcfdeployment.git
$ cd pcfdeployment
```

#### Step 1: cloudformation Role
Modify roles/cloudformation/vars/main.yml. For example: 
```sh
vim roles/cloudformation/vars/main.yml
```

The following varibles can be configured: 


| Variable        | Value          | Remarks   |
| :-------------: |:-------------:| :-----:|
| StackName     | The name of the clodu formation stack | Leave it as is |
| Region      | the region in which PCF is deployed      |    |
| TemplateLocation | the cloudformation template location      |    You must leave it as is |
| NATKeyPai | the SSH Key used by NAT VM | Must choose an existing key in your AWS | 
| NATInstanceType | NAT VM Instance Type | Leave it as is | 
| OpsManagerIngress | Range of Ingress IPs for Ops Manager | Leave it as is |
| RdsDBName | name of the RDS DB for Ops Man | Up to the user | 
| RdsUserName |  username of RDS DB | Up to the user | 
| RDsPassword | password for RDS | Chose a secure password or user Vault | 
| SSLCertificateARN | the ARN for the AWS Certificate Manager | See Note 1 | 
| OpsManagerTemplate | the location of the OpsManagerTemplate | See Note 2 | 

* Note 1: You must create a Certificate to be used for your PCF using AWS Certificate Manger, capture its ARN and use it here. For more information, see here: 
http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/ssl-server-cert.html#create-cert
* Note 2: If you leave this as blank, the default OpsManager template will be used which is highly recommended. If you wish to modify that, then provide the S3 location for a new ops-manager.json file. 

#### Step 2: opsmanagerdeploy
Modify roles/opsmanagerdeploy/vars/main.yml. For example: 
```sh
vim roles/opsmanagerdeploy/vars/main.yml
```

The following varibles can be configured: 


| Variable        | Value          | Default is accepted?   |
| :-------------: |:-------------:| :-----:|
|StackName | the name of the CloudFormation stack | Must match the value in previous Role |
|Region | the region to which PCF is deployed | Must match the value in previous Role |
|NatKeyPair | the SSH Key used to access Ops Manager | Provide an existing EC2 SSH Key | 
|OpsMangInstanceType | the InstanceType for Ops Manager | User to choose. default is sufficient|
|AMIID | the AMI ID for the existing Ops Man AMI | see Note 1 | 
| Route53Zone | Route 53 Domain to be used with Ops Man | User to choose | 
| Route53Record | Route 53 Record to be used with Ops Man | User to choose | 
| Route53Type | Route 53 Record Type | Leave it as is | 
| DecryptionPassphrase | Decrpytion Passphrase for Ops Man's data | User to choose or use Vault |
| AdminUsername | Username to be used for Ops Man | User to choose | 
| AdminPassword | Password for Ops Man Admin | User to choose or user Vault | 
* Note 1: This must a pre-existing AMI ID provided by Pivotal for the Operations Manager of your choise. For a list of available AMI in each region, see: 
https://network.pivotal.io/products/ops-manager

#### Step 3: directorconfig
Modify roles/directorconfig/vars/main.yml. For example: 
```sh
vim roles/directorconfig/vars/main.yml
```

The following varibles can be configured: 


| Variable        | Value          | Default is accepted?   |
| :-------------: |:-------------:| :-----:|
|StackName | the name of the CloudFormation stack | Must match the value in previous Role |
|DecryptionPassphrase | Decrpytion Passphrase for Ops Man's data | Must match the value in previous Role |
| AdminUsername | Username to be used for Ops Man | Must match the value in previous Role | 
| AdminPassword | Password for Ops Man Admin | Must match the value in previous Role | 
| Route53Record | Route 53 Record for Ops Man | Must match the value in previous Role | 
|Region | the region to which PCF is deployed | Must match the value in previous Role |
|KeyPair | the SSH Key used to access Ops Manager | Must match the value in previous Role | 
|ssh_private_key | SSH Key for BOSH | See Note 1 |
* Note 1: You must enter the value in the following format: 
"{{ lookup('file', 'path/to/yourkey/yourkey.pem') }}"


#### Step 4: modifyserver
Variable are the same as the previous role.
```sh
cp roles/directorconfig/vars/main.yml roles/modifyserver/vars/main.yml
```




#### Step 5: directorInstall
Variable are the same as the previous role.
```sh
cp roles/directorconfig/vars/main.yml roles/directorInstall/vars/main.yml
```

#### Step 6: elasticruntimeupload
Modify roles/elasticruntimeupload/vars/main.yml. For example: 
```sh
vim roles/elasticruntimeupload/vars/main.yml
```

All variables are the same as previous step except for the following: 


| Variable        | Value          | Default is accepted?   |
| :-------------: |:-------------:| :-----:|
|ElasticRuntimeFilePath | the location fo the Elastic Runtime Binaries | See Note 1 |
Note 1: You must download your desired version of PCF Elastic Runtime from here:
https://network.pivotal.io/products/elastic-runtime
and then set ElasticRuntimeFilePath.


#### Step 7: elasticruntimeconfig
Modify roles/elasticruntimeconfig/vars/main.yml. For example: 
```sh
vim roles/elasticruntimeconfig/vars/main.yml
```

All variables are the same as previous step except for the following new variables: 


| Variable        | Value          | Default is accepted?   |
| :-------------: |:-------------:| :-----:|
|ElasticRuntimeDBAEmail | Email Address for Elastic Runtime DBA | User to choose a value |
|RuntimeAppDomain|The domain in which CF Apps run | See Note 1 |
|RuntimeSystemDomain| The domain in which CF System apps run | See Note 1 |
* Note 1: For PCF's recommandation, please see Step 4: Add CNAME Record for Your Custom Domain in:
https://docs.pivotal.io/pivotalcf/1-10/customizing/cloudform-er-config.html
 

#### Step 8: createpushuser
Modify roles/createpushuser/vars/main.yml. For example: 
```sh
vim roles/createpushuser/vars/main.yml
```

The following varibles can be configured: 


| Variable        | Value          | Default is accepted?   |
| :-------------: |:-------------:| :-----:|
|pushUserName| the primary username to execute cf push | User to choose | 
|pushPassword| password for the primary user| User to choose or use Vault|
|pushEmail| Primary user's email address | User to choose| 
|defautOrg|the default Org for the primary user | User to choose |
|dfaultSpace|the default Space for the primary user | User to choose|

Once all variables are set, you may either run each role separately (recommended if it is your first time) or run all the roles sequencially useing deployAll.yml 
```sh
ansible-playbook deployAll.yml -vvv
```

Alternatively, you can run (or schedule job.sh) which runs the ansble command and emails the user the result (if a mail client is setup. Following variables must be setup: 
email= The email address at which the job result are to be emailed
output_path= The location of the output file
root_directory= The project's root path name

### What Exactly each Role does: 
In this section, we briefly go over the key tasks in each role 
#### Step 1: cloudformation Role
> Runs a Cloudformation module task to deploy PCF's CloudFormation Template  
#### Step 2: opsmanagerdeploy Role
> Gathers information from the output of the cloudformation stack   
> Runs an ec2 module to lunch the Ops Manager Instance   
> Wati for Ops Manager to become available   
> Creates a Route53 record   
> Uses Ops Manager API to configure the Admin Username and password for Ops Manager. As this task also decrypt the data, it is time consuming and as such, Ansible wait on it and retries 3 times  
> Once the Ops Manager Security is configured, Ansible paused for another 2 minutes so that the Authentication Service starts. This step is needed if all roles are executed sequencially and ensures that authentication is running before attempting to use the API   
#### Step 3: directorconfig Role
> Obtains a UAA_ACCESS_TOKEN from Ops Manager so that it can access its REST API
> Regather the CloudFormation Stack facts to be used as input in configuration
> Uses Ops Manager Rest API to configure Bosh 
#### Step 4: modifyserver Role
> As mentioned, this role  SSH into the Ops Manager VM, decrypts the Installation.yml file, modifies its content (to get around the Rest API bug where the value of Database and S3 cannot be set to external) and decrpyts Instllation.yml and places back into its location. After this, Bosh is ready to be installed
#### Step 5: directorinstall Role
> This step trigers the installation of Bosh, then waits for 10 minutes during which Bosh Installation continues. After 10 minutes, it uses the rest API to see if the Installation was successful or not. If it was incomplete, it wait for another minute before checking again. The wait if only necessary if the steps are executed sequencially. 
#### Step 6: elasticruntimeupload Role
> This step uses Operation Managers API to upload Elastic Runtime to Ops Manager. This is a very time consuming and I/O intensive task. 
#### Step 7: Elasticruntime Config
> This role first reobtains API keys to ensure it can access Ops Manager Rest API
> It retrieves Cloudformation facts and sets variables
> It then uses Operation Manager Rest API to configure Elastic Runtime and prepare it for Installation.
> It also creates two CNAME record in Route 53. If you already have the required CNAME and do not need to configure it in AWS, you may remove this step. 
> It then triggers the installation of Elastic Runtime
> It wait for 1 hour before checking the status of the Installation. If it has not completed, it wait for 5 minutes before rechecking (up to 20 times). 
#### Step 8: createpushuser 
> This role uses CF-UAAC to create a User and grant all rights to that user (Admin)
> It then creates the default Org and Space


# Address API 
Address API provides programmatic acces to read and write the Address data. If you wish to use the address API, please refer to README.MD as this is inteded for the application setup

### Setting up the Project: 
Address API is a Maven-based Spring Boot application with an embedded tomcat server. Packaging this application in maven produces a runable jar that you can run with "java -jar" command locally or on any PAAS that is able to run java. This project is particularly tested with Cloud Foundry (a manifest.yml for Pivotal Cloud Foundry is included). 

Prerequisite:
i) you must have Maven installed in your environment. Test with 
```sh
$ mvn --version
```
If you do not have maven installed, see here:
https://maven.apache.org/install.html
    
ii) if you wish to run this project locally, you must have Java 8 installed. If you wish to run this project on cloud foundry (recommended), you must have Cloud Foudry Command Line Interface installed. If you do not have Cf-cli installed, see here:
https://github.com/cloudfoundry/cli

iii) Before deploying this project, you must setup the MySQL database for persistent storage. You must run script.sql file against the database and record the following information: 
- Public URL
- Port number
- Database Name
- Username
- Password

Alternatively, you can take a look at the 'addressdatabase Project' (https://github.com/saedalav/addressdatabase) if you wish to automatically provision the database in AWS RDS using Ansible. 

1. Clone this project: 
```sh
$ git clone https://github.com/saedalav/addressapi.git
$ cd addressapi
```
2. Modify src/main/resources/aplication.properties and enter your database information in the following format: 
spring.datasource.url=jdbc:mysql://DATABASE_URL:DB_PORT/address
spring.datasource.username=USERNAME
spring.datasource.password=PASSWORD
```sh
$ vim src/main/resources/aplication.properties
$ [Modify the file]
$ wq
```

3. From the project's root directory (where the pom.xml file is present), run Maven's package in order to package the project into a runnable jar 
```sh
$ mvn clean package
```
This will produce addressapi-0.0.1-SNAPSHOT.jar in target directory. This is the runnable jar that you can run either locally or in Cloud Foudry 

4. Run the addressapi-0.0.1-SNAPSHOT.jar 
(a) To run the application locally with Java 8, 
```sh
$ java -jar addressapi-0.0.1-SNAPSHOT.jar
```
The application will be available at http://localhost:8080/. Test your application by invoking different REST API methods. 

(b) To run the application on your Cloud Foudry: 
Open manifest.yml file and modify the default values if you wish. then: 
```sh
$ cf login -a API_URL -u USERNAME -p PASSWORD (Then Select Org and workspace)
$ cf push
```
The application will then be available in https://yourappspace.fqdn/addressapi. Test your application by invoking different REST API methods. 

# Address API Client Application
Address API Client Application is a client of Address API.

### Setting up the Project: 
Address API Client is a Maven-based Spring Boot + JSF application with an embedded tomcat server. Packaging this application in maven produces a runable war that you can run with "java -jar" command locally or on any PAAS that is able to run java. This project is particularly tested with Cloud Foundry (a manifest.yml for Pivotal Cloud Foundry is included). 

Prerequisite:
i) you must have Maven installed in your environment. Test with 
```sh
$ mvn --version
```
If you do not have maven installed, see here:
https://maven.apache.org/install.html
    
ii) if you wish to run this project locally, you must have Java 8 installed. If you wish to run this project on cloud foundry (recommended), you must have Cloud Foudry Command Line Interface installed. If you do not have Cf-cli installed, see here:
https://github.com/cloudfoundry/cli

iii) You must know the URL at which Address API is available. 

1. Clone this project: 
```sh
$ git clone https://github.com/saedalav/addressapiclient.git
$ cd addressapiclient
```
2. Modify /src/main/java/me/alavinia/saed/restclient/CallRest.java and enter the URL at which Address API is available in the following format: 
private static final String URL = "https://addressapi_url/";
```sh
$ vim /src/main/java/me/alavinia/saed/restclient/CallRest.java
$ [Modify the file]
$ wq
```

3. From the project's root directory (where the pom.xml file is present), run Maven's package in order to package the project into a runnable jar 
```sh
$ mvn clean package
```
This will produce target/addressapiclient-0.0.1-SNAPSHOT.war in target directory. This is the runnable jar that you can run either locally or in Cloud Foudry 

4. Run the addressapiclient-0.0.1-SNAPSHOT.war
(a) To run the application locally with Java 8, 
```sh
$ java -jar target/addressapiclient-0.0.1-SNAPSHOT.war
```
The application will be available at http://localhost:8080/. (Make sure this doesn't conflict with your Address API application. if you are running both of them locally, you must modify the application.properties file and set the port key to another value such 8081). Test your application by attempting to Add a new member or address. 

(b) To run the application on your Cloud Foudry: 
Open manifest.yml file and modify the default values if you wish. then: 
```sh
$ cf login -a API_URL -u USERNAME -p PASSWORD (Then Select Org and workspace)
$ cf push
```

# Deploying NewRelic Service to Monitor Addressapi and Adressapiclient Applications on PCF
*** 

### Setting up the Project: 

##### Prerequisites: 
You need to have a valid New Relic Account. See here: 
https://rpm.newrelic.com

Then, login to your New Relic Account and record the License Key from here:
https://rpm.newrelic.com/accounts


1. Clone this Project 
```sh
git clone https://github.com/saedalav/newrelic-cf.git 
cd newrelic-cf.git
```
2. Edit manifest.yml file and update your Username, password, and New Relic License Key. 
```sh 
vim manifest.yml
[ Update ] 
:wq
```

3. Execute the following commands: 

```sh 
cf login -a URL -u USERNAME -p PASSWORD
cf push 
cf create-service-broker newrelicservicebroker saedalav mypass https://newrelicservicebrokerapp.apps.alavinia.me 
cf enable-service-access newrelic
cf create-service newrelic standard newrelicserviceinstance
cf bind-service addressapi newrelicserviceinstance
cf bind-service addressapiclient newrelicserviceinstance
cf restage addressapi
cf restage addressapiclient
```




# Configuring PCF to Forward Application Logs to your ELK  
*** 

### Setting up the Project: 

##### Prerequisites: 
You need to have an ELK Stack and know the following information: 
* The URL and Port Numeber at which Logstash consumes Logs. For example:
syslog://ip_address:5000/
* The Username and Password for your Kibana Web Application. 

If you wish to deploy an instance of Logstash in AWS, ami-40e3a356 (US-EAST-1) is available and setup to consume logs as syslog://ip_address:5000/ and Kibana is configured with Username: kibanaadmin and Password: mypass. 
Email me at saedalav@gmail.com to gain access to this AMI as it is currently Private. 

This guide assumes that you have two application for which you would like to forward logs to your ELK Stack: 
- addressapi
- addressapiclient
And that Logstash is configured to consume logs at:
syslog://elk.alavinia.me:5000/ 


1. Execute the following commands: 

```sh 
cf login -a URL -u USERNAME -p PASSWORD
cf cups logstash-drain -l syslog://elk.alavinia.me:5000/
cf bind-service addressapi logstash-drain
cf bind-service addressapiclient logstash-drain
cf restage addressapi
cf restage addressapiclient
