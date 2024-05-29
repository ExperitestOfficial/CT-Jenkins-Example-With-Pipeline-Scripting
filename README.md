# CT-Jenkins-Example-With-Pipeline-Scripting

In this example we will look at how we can create an automated CICD pipeline to execute Appium Scripts against iOS and Android Devices on Digital.ai's Continuous Testing Platform.

For this example I am using a Pipeline Project created:

![image](https://user-images.githubusercontent.com/71343050/183140704-b8b65a70-69e6-46b5-8511-0a420d16821d.png)

The reason for this is I want to build stages for different phases I have throughout this execution. By having different stages, I can better debug and understand when a failure occurs.

I will be writing this code in Groovy, using the "Pipeline script":

![image](https://user-images.githubusercontent.com/71343050/183142120-3879fecd-4d08-482e-9f7a-ae787d3f6a1b.png)

Let's look at the stages that I'll build out:

**Preparation**

```
stage('Preparation') { 
        git 'https://github.com/raheekhandigitalai/ExperiBankDemoApplication.git'
        mvnHome = tool 'M3'
}
```

The preparation stage is calling out to a GitHub repository where all the Appium Scripts are stored.
Since I am using Maven to execute my Appium Scripts, I am also defining a global variable for Maven is being set as "M3" in order to be able to run Maven based commands.

------

**Build**

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

In the Build Stage, I am using Maven commands to run the Appium Scripts. Depending on if the Tests are ran from a Unix based platform or Windows, I've put in a conditional statement to run according to the platform. This will trigger tests from the GitHub Repository.

------

**Results**

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

In the Results stage, I am utilizing Digital.ai's Continuous Testing APIs that are publically available. In order to retrieve the Test Results, I need to first create a "Test View". After that, I am retrieving Test Results from the created Test View and applying a filter to only give me the results from this Jenkins Build Run.

A Test View is simply a view that has test results. Think of Test View as a container, we need a container to fetch the information about the container properties.

[Create Test View API Documentation](https://docs.digital.ai/bundle/TE/page/rest_api_-_testview.html#RestAPI-TestView-CreateTestViewsGroup)

[Get Filtered Test Results from Test View Documentation](https://docs.digital.ai/bundle/TE/page/rest_api_-_testview.html#RestAPI-TestView-GetTestCounts)

------

**Clean Up**

```
stage('Tear Down') {
        triggerResponseDeleteTestView = sh(script: '''curl --location --request DELETE \'''' + CLOUD_URL + '''/reporter/api/testView/''' + testViewId + '''\' --header \'Content-Type: application/json\' --header \'Authorization: Bearer ''' + ACCESS_KEY + '''\'''', returnStdout: true).trim()
        echo triggerResponseDeleteTestView
}
```

In the Clean Up stage, I am deleting the Test View, otherwise if we were to create Test Views and keep them every time, it would just clog up the Reporter unnecessarily.

[Delete Test View API Documentation](https://docs.digital.ai/bundle/TE/page/rest_api_-_testview.html#RestAPI-TestView-DeleteTestView)

------

Keep in mind that the Stages does not have to be built out like this, or even with this Logic. For this particular scenario this is what I want to happen. Feel free to explore and experiment with that works best for you.

To get the Full Script Example, see Pipeline_Groovy_Script.txt
