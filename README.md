# CT-Jenkins-Example-With-Pipeline-Scripting

In this example, we will take a look at how we can use Pipeline orchestration from Jenkins and start taking it one step further by building out custom logic using Pipeline Scripting. We will see how we can leverage Digital.ai's Continuous Testing platform APIs to get Test Results from the Automated Test Executions.

For this example I am using a Pipeline Project created:

![image](https://user-images.githubusercontent.com/71343050/183140704-b8b65a70-69e6-46b5-8511-0a420d16821d.png)

Choosing the Pipeline option provides the ability to build "stages" for the different phases you may have throughout a Test Execution which helps with debugging and understanding failures when they occur. For example, a Test Automation may have phases such as:

1. Setting up the Environment
2. Building and Running the Tests
3. Getting Test Results back


I will be writing this code in Groovy, using the "Pipeline Script" option:

![image](https://user-images.githubusercontent.com/71343050/183142120-3879fecd-4d08-482e-9f7a-ae787d3f6a1b.png)

Let's look at the stages that I'll build out:

**Preparation**

The preparation stage is calling out to a GitHub repository where all the Appium Scripts are saved.
Since I am using Maven to execute my Appium Scripts, I am also defining a global variable for Maven which is being set as "M3" in order to be able to run Maven based commands.

```
stage('Preparation') { 
        git 'https://github.com/raheekhandigitalai/ExperiBankDemoApplication.git'
        mvnHome = tool 'M3'
}
```

------

**Build**

In the Build Stage, I am using Maven commands to run the Appium Scripts. Depending on if the Tests are ran from a Unix or Windows based platform, I've put in a conditional statement to run according to the platform.

```
stage('Build') {
        withEnv(["MVN_HOME=$mvnHome"]) {
                if (isUnix()) {
                        sh '"$MVN_HOME/bin/mvn" -Dmaven.test.failure.ignore clean install'
                } else {
                        bat(/"%MVN_HOME%\bin\mvn" -Dmaven.test.failure.ignore clean install/)
                }
        }
}
```

------

**Results**

In the Results stage, I am utilizing Digital.ai's Continuous Testing APIs, available to all customers. In order to retrieve the Test Results, I need to first create a "Test View". After that, I am retrieving Test Results from the created Test View by applying a filter to fetch results specific from this Jenkins Build.

A Test View is a view that stores Test Results. Think of Test View as a container, we need a container to fetch the information about the container properties.

[Create Test View API Documentation](https://docs.digital.ai/bundle/TE/page/rest_api_-_testview.html#RestAPI-TestView-CreateTestViewsGroup)

[Get Filtered Test Results from Test View Documentation](https://docs.digital.ai/bundle/TE/page/rest_api_-_testview.html#RestAPI-TestView-GetTestCounts)

```
stage('Report Summary') {
        triggerResponseCreateTestView = sh(script: '''curl --location --request POST \'''' + CLOUD_URL + '''/reporter/api/testView\' --header \'Content-Type: application/json\' --header \'Authorization: Bearer ''' + ACCESS_KEY + '''\' --data \'{"name": "TestViewFromJenkins", "byKey": "date", "groupByKey1": "device.os", "groupByKey2": "device.version"}\'''', returnStdout: true).trim()
        echo "Response from Create View: ${triggerResponseCreateTestView}"
        
        testViewId = getJsonObjectFromResponseOutputID(triggerResponseCreateTestView, "id")
        testViewName = getJsonObjectFromResponseOutputName(triggerResponseCreateTestView, "name")
        
        echo "testViewId: ${testViewId}"
        echo "testViewName: ${testViewName}"
        echo "jenkinsbuildnr: ${JENKINS_BUILD_NUMBER}"
        
        triggerResponseGetTestViewResults = sh(script: '''curl --location --request GET \'''' + CLOUD_URL + '''/reporter/api/testView/''' + testViewId + '''/summary?filter=%7B%22Jenkins_Build_Number%22%3A%225%22%7D\' --header \'Authorization: Bearer ''' + ACCESS_KEY + '''\'''', returnStdout: true).trim()
        // triggerResponseGetTestViewResults = sh(script: '''curl --location --request GET \'''' + CLOUD_URL + '''/reporter/api/testView/''' + testViewId + '''/summary?filter=%7B%22Jenkins_Build_Number%22%3A%22''' + JENKINS_BUILD_NUMBER + '''%22%7D\' --header \'Authorization: Bearer ''' + ACCESS_KEY + '''\'''', returnStdout: true).trim()
        echo triggerResponseGetTestViewResults
        echo "Response from Get Test View Results: ${triggerResponseGetTestViewResults}"
    
        passedCount = getJsonObjectFromResponseOutput(triggerResponseGetTestViewResults, "passedCount")
        failedCount = getJsonObjectFromResponseOutput(triggerResponseGetTestViewResults, "failedCount")
        incompleteCount = getJsonObjectFromResponseOutput(triggerResponseGetTestViewResults, "incompleteCount")
        skippedCount = getJsonObjectFromResponseOutput(triggerResponseGetTestViewResults, "skippedCount")
        totalCount = getJsonObjectFromResponseOutput(triggerResponseGetTestViewResults, "_count_")
        
        echo "Passed Count: ${passedCount}"
        echo "Failed Count: ${failedCount}"
        echo "Incomplete Count: ${incompleteCount}"
        echo "Skipped Count: ${skippedCount}"
        echo "Total Count: ${totalCount}"
}
```

------

**Clean Up**

In the Clean Up stage, I am deleting the Test View.

[Delete Test View API Documentation](https://docs.digital.ai/bundle/TE/page/rest_api_-_testview.html#RestAPI-TestView-DeleteTestView)

```
stage('Tear Down') {
        triggerResponseDeleteTestView = sh(script: '''curl --location --request DELETE \'''' + CLOUD_URL + '''/reporter/api/testView/''' + testViewId + '''\' --header \'Content-Type: application/json\' --header \'Authorization: Bearer ''' + ACCESS_KEY + '''\'''', returnStdout: true).trim()
        echo triggerResponseDeleteTestView
}
```

------

To get the Full Script Example, see ```Pipeline_Groovy_Script.txt```
