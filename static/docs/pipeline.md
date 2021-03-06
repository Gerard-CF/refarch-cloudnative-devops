# Bluemix DevOps toolchain deep dive

*This project is part of the 'IBM Cloud Native Reference Architecture' suite, available at
https://github.com/ibm-cloud-architecture/refarch-cloudnative*

Here is the instruction of how to build your DevOps toolchain and pipelines from scratch.

### Create a Bluemix DevOps Toolchain

Click this [link](https://new-console.ng.bluemix.net/devops?cm_mmc=developerWorks-_-dWdevcenter-_-devops-services-_-lp&cm_mc_uid=50560713550714744636050&cm_mc_sid_50200000=1474581316) to bring up Bluemix DevOps dashboard, click the Toolchains tab and click Create a Toolchain.  

Under Advanced section, click **Build your own toolchain**.

Give a name like "refarch-cloudnative-toolchain-qa", then click **Create**

Then you can review the toolchain (the page will automatically refresh to the toolchain page)  
Now, you can start building the pipeline and associated DevOps tools for each component.

### Develop the toolchain for SocialReview Microservice

A toolchain here is a logical concept of a set of tools, utility that can help you implement CI/CD on IBM Bluemix platform. In our specific example, it'll be mainly github issue, github repository and Bluemix DevOps pipeline that build and deploy your application component.

In Toolchain page, click the **+** add button to add a tool to your toolchain.   
Click **github** from the options of tools integration panel.  
Select **Clone** as Repository type,   
Enter the New Repository name as: **refarch-cloudnative-micro-socialreview**,  
And Enter the git repository link for the socialreviw microservice: https://github.com/ibm-cloud-architecture/refarch-cloudnative-micro-socialreview,  
You can choose to enable the git issue, then it will be part of the tool chain.  
Click **Create Integration** button.

![Create Github](static/imgs/socialreview_github_tool.png?raw=true)  
You may need to Grant your github access for the toolchain.  

Now, you can add Bluemix DevOps pipeline to your toolchain. Back in the Toolcahin -> Tool Integration page, click the **+** button to add another tool,  
Choose **Delivery Pipeline**  
Give a name like *socialreview-delivery-pipeline*,  then click **Create Integration**  

Now, you should see 3 tools in your toolchains as below:  
![Socialreview toolchain](static/imgs/socialreview_micro_toolchain.png?raw=true)  

Click on the Delivery Pipeline you just created to configure the build and deployment process.  
Click **ADD STAGE** button. Name the stage as "Build Micorservice", under the **INPUT** tab,  
Choose Input Type as SCM Repository, and Specify the Git Repository as the **refarch-cloudnative-micro-socialreview** you just create, it will automatically fill in the Git URL.  
Set the Stage Trigger as *Run jobs whenever a change is pushed to Git* as shown below:  

![Socialreivew Build Stage](static/imgs/socialreview_build_stage_git.png?raw=true)  

Then click the **JOBS** tab, click the **ADD JOB** button, select **Build** type,  
In Build job, select the Builder Type as **IBM Container Service**  
Fill in the Bluemix Target, Organization and Space information.  
Then, in the Build Script section, remove the existing script, then copy and paste the following script in there:  

```
#!/bin/bash

# The default Gradle version is 1.12
# To use Gradle 2.9, uncomment the following line:
# export PATH="$GRADLE2_HOME/bin:$PATH"
export JAVA_HOME=~/java8
export PATH=$JAVA_HOME/bin:$PATH
./gradlew build -x test
./gradlew docker
ls -al docker
cp -R docker/* .
echo "gradle build done"

# The following colors have been defined to help with presentation of logs: green, red, label_color, no_color.  
log_and_echo "$LABEL" "Starting build script"

# The IBM Container BM Containers plug-in (cf ic), Git client (git), and IDS Inventory CLI (ids-inv) have been installed.
# Based on the organization and space selected in the Job credentials are in place for both IBM Container Service and IBM Bluemix
#####################
# Run unit tests    #
#####################
log_and_echo "$LABEL" "No unit tests cases have been checked in"

######################################
# Build Container via Dockerfile     #
######################################

REGISTRY_URL=${CCS_REGISTRY_HOST}/${NAMESPACE}
FULL_REPOSITORY_NAME=${REGISTRY_URL}/${IMAGE_NAME}:cloudnative
# If you wish to receive slack notifications, set SLACK_WEBHOOK_PATH as a property on the stage.

if [ -f Dockerfile ]; then
    log_and_echo "$LABEL" "Building ${FULL_REPOSITORY_NAME}"
    ${EXT_DIR}/utilities/sendMessage.sh -l info -m "New container build requested for ${FULL_REPOSITORY_NAME}"
    # build image
    BUILD_COMMAND=""
    if [ "${USE_CACHED_LAYERS}" == "true" ]; then
        BUILD_COMMAND="build --pull --tag ${FULL_REPOSITORY_NAME} ${WORKSPACE}"
        ice_retry ${BUILD_COMMAND}
        RESULT=$?
    else
        BUILD_COMMAND="build --no-cache --tag ${FULL_REPOSITORY_NAME} ${WORKSPACE}"
        ice_retry ${BUILD_COMMAND}
        RESULT=$?
    fi

    if [ $RESULT -ne 0 ]; then
        log_and_echo "$ERROR" "Error building image"
        ice_retry info
        ice_retry images
        ${EXT_DIR}/print_help.sh
        ${EXT_DIR}/utilities/sendMessage.sh -l bad -m "Container build of ${FULL_REPOSITORY_NAME} failed. $(get_error_info)"
        exit 1
    else
        log_and_echo "$SUCCESSFUL" "Container build of ${FULL_REPOSITORY_NAME} was successful"
        ${EXT_DIR}/utilities/sendMessage.sh -l good -m "Container build of ${FULL_REPOSITORY_NAME} was successful"
    fi  
else
    log_and_echo "$ERROR" "Dockerfile not found in project"
    ${EXT_DIR}/utilities/sendMessage.sh -l bad -m "Failed to get Dockerfile. $(get_error_info)"
    exit 1
fi  

######################################################################################
# Copy any artifacts that will be needed for deployment and testing to $WORKSPACE    #
######################################################################################
echo "IMAGE_NAME=${FULL_REPOSITORY_NAME}" >> $ARCHIVE_DIR/build.properties
```   
Click the **Save** button.  

Now, back in Delivery Pipeline, you need to add another stage to Create the Cloudant datbase service and deploy the container to Bluemix. Click the **ADD STAGE** button again.  
Name the stage as "Deploy"  
Under the **INPUT** tab, enter/select following:  
  Input Type:   Build Artifacts  
  Stage:        Build Microservices (*this is the Build stage you created earlier*)  
  Job:          Build  

Then, check the "Run jobs when the previous stage is completed." This will trigger the deploy when the build phase completes.   
Switch to the **JOBS** tab, click **ADD JOB**  
Name the Job as "Prepare Cloudant"  
Choose Deploy Type as **Cloud Foundry**  
Fill in the Bluemix information: Target, Organization and Space.  
For Application Name, enter **socialreviews-microservice-{yourinstance}**. Replace the {yourinstance} with some arbitrary string you can uniquely identify your application in your environment.  
Then, Under the Deploy Script, replace the existing script with the following script:  
```
#!/bin/bash
cf create-service cloudantNoSQLDB Shared refarch-cloudantdb
cf create-service-key refarch-cloudantdb refarch-cloudantdb-credential

```  
This will create a cloudant service and security credential associated with it if one doesn't exist (otherwise, it'll skip the service and credential creation).   

Add Another **JOB**, select "Deploy" job type.  
Name it "Deploy Container"  
Select **IBM Containers on Bluemix** as Deployer Type,  
Fill in Bluemix information: Target, Organization and Space.  
You enter the Name field as **micro-socialreview-group-{yourinstance}**  
Under the Optional deploy arguments field, enter the following:  

`-m 512 --auto -n socialreviewserviceqa -d mybluemix.net`  

Then, replace the **Deployer script** with the following script:  
```
#!/bin/bash
# The following are some example deployment scripts.  Use these as is or fork them and include your updates here:
echo -e "${label_color}Starting deployment script${no_color}"


# To view/fork this script goto: https://github.com/Osthanes/deployscripts
# git_retry will retry git calls to prevent pipeline failure on temporary github problems
# the code can be found in git_util.sh at https://github.com/Osthanes/container_deployer
git_retry clone https://github.com/ssibm/deployscripts.git deployscripts

# get Cloudant information
log_and_echo "Build SocialReview Docker Container"
cf service-key refarch-cloudantdb refarch-cloudantdb-credential|grep -E 'host|username|password'|sed 's/[\" \"|"|,]//g'|sed 's/:/=/g' > cloudant.env
. cloudant.env
OPTIONAL_ARGS="${OPTIONAL_ARGS} -e "eureka.client.serviceUrl.defaultZone=${EUREKA_REGISTRY_URL}" -e "cloudant.username=$username" -e "cloudant.password=$password" -e "cloudant.host=https://$host""
log_and_echo "Docker arguments: " $OPTIONAL_ARGS


/bin/bash deployscripts/deploygroup.sh

RESULT=$?

# source the deploy property file
if [ -f "${DEPLOY_PROPERTY_FILE}" ]; then
  source "$DEPLOY_PROPERTY_FILE"
fi

#########################
# Environment DETAILS   #
#########################
# The environment has been setup.
# The Cloud Foundry CLI (cf), IBM Container Service CLI (ice), Git client (git), IDS Inventory CLI (ids-inv) and Python 2.7.3 (python) have been installed.
# Based on the organization and space selected in the Job credentials are in place for both IBM Container Service and IBM Bluemix

# The following colors have been defined to help with presentation of logs: green, red, label_color, no_color.
if [ $RESULT -ne 0 ]; then
    echo -e "${red}Executed failed or had warnings ${no_color}"
    ${EXT_DIR}/print_help.sh
    exit $RESULT
fi
echo -e "${green}Execution complete${no_label}"

```  

Click the **SAVE** button to save the Deploy stage. Now, back in your Delivery Pipeline, you should see something like following:  

![Socialreivew Pipeline](static/imgs/socialreview_delivery_pipeline.png?raw=true)  

Now, you can execute the pipeline by clicking the **Play** button in *Build Microservice* Stage. This will trigger the Build process as well as create the cloudant and deploy the Socialreview microservices to Bluemix container.  

### Next Pipeline  
