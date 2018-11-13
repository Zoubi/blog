---
published: true
layout: post
title: UE4 Unit Tests in Jenkins
category: UE4
tags: [ UE4, Jenkins, UnitTests, jUnit ]
---

On a personal project I try to work on during my spare time, I have written multiple unit tests to check some data is correct: for example, I check some given animation montages have the correct number of sections, which themselves are correctly named, etc...

The [documentation](https://docs.unrealengine.com/en-us/Programming/Automation) about the automation is pretty solid, and I encourage you to go have a look if you don't know what it is about.

It's pretty [straightforward](https://docs.unrealengine.com/en-us/Programming/Automation/UserGuide) to run the unit tests from within the editor, using the session frontend. But the task is a bit more complicated when you want to run the unit tests from within a jenkins job, and have the results parsed and displayed by jenkins.

The first step is to run the units tests from the command line. Here are the arguments I use in my jenkinsfile, based on [that thread](https://answers.unrealengine.com/questions/106978/run-automated-testing-from-command-line.html#answer-840447):

{% highlight terminal %}
bat returnStatus: true, script: "\"${env.UE_ROOT_FOLDER}\\Engine\\Binaries\\Win64\\UE4Editor-Cmd.exe\" 
    \"${env.WORKSPACE}\\${env.PROJECT_NAME}.uproject\" 
    -unattended -nopause 
    -NullRHI
    -ExecCmds=\"Automation RunTests ${env.PROJECT_NAME}\"
    -testexit=\"Automation Test Queue Empty\" -log -log=RunTests.log 
    -ReportOutputPath=\"${env.WORKSPACE}\\Saved\\UnitTestsReport\""
{% endhighlight %}

A few things to note here:
* I prefixed all my unit tests with my project name. This is why I use ${env.PROJECT_NAME} in **ExecCmds**.
* **NullRHI** is used to avoid  instantiate the whole editor. As of UE 4.20, this causes a crash when the tests are ended, but this has been fixed in UE 4.21
* **testexit** is used to let the editor know it has to exit when the message "Automation Test Queue Empty" is logged
* **unattended** and **nopause** are [documented](https://docs.unrealengine.com/en-US/Programming/Basics/CommandLineArguments#switches)
* **ReportOutputPath** is used to make UE4 generate a report in a JSON format in the destination folder

Once jenkins runs this step, you should have your tests runned, and the report generated. If that's not the case, here are 2 issues I encountered:
* in UE 4.20, if the Session Frontend tab is opened at the startup of the editor, despite the logs saying your tests were runned, they are not. This issue is fixed in UE 4.21, but the workaround is simple : close your session frontend tab, and close the editor, so that it does not show up again :)
* My jenkins nodes run on Windows machines, and are connected to the jenkins master from a service. I had crashes when running the unit tests because the editor could not create the rendering swap chain when launched from the service. I had to revert back to connecting the slaves to the master with the JNLP applet, but maybe there is a way to change the service so it has the correct permissions. To be investigated...

Now that UE runs the unit tests and generates the JSON report, we have to make jenkins use this report to display the test results.
The problem is that Jenkins can only natively use reports which fullful the [jUnit XML format](http://llg.cubic.org/docs/junit/). So we have to convert that JSON into a valid XML file.

Here is the format of the JSON created by UE:

{% highlight json %}
{
    "succeeded": 17,
    "succeededWithWarnings": 0,
    "failed": 1,
    "notRun": 0,
    "totalDuration": 10.075541496276855,
    "comparisonExported": true,
    "comparisonExportDirectory": "F:/Projects/RWC/Saved/TestReport/4369336",
    "tests": [
        {
            "testDisplayName": "AM_Dash_old",
            "fullTestPath": "RWC.AnimMontages.Dash.AM_Dash_old",
            "state": "Fail",
            "entries": [
                {
                    "event":
                    {
                        "type": "Error",
                        "message": "/Game/Characters/Player_Knight/Animations/Montages/AM_Dash_old - A dash montage must have exactly 4 sections: Back, Left, Right, Front",
                        "context": "",
                        "artifact": "00000000000000000000000000000000"
                    },
                    "filename": "f:\\projects\\rwc\\source\\rwceditor\\private\\tests\\rwcmontagesattacktests.cpp",
                    "lineNumber": 55,
                    "timestamp": "2018.11.12-15.01.02"
                }
            ],
            "warnings": 0,
            "errors": 1,
            "artifacts": []
        },
        {
            "testDisplayName": "AM_Attack_1H_Heavy_Charge",
            "fullTestPath": "RWC.AnimMontages.Attack.AM_Attack_1H_Heavy_Charge",
            "state": "Success",
            "entries": [],
            "warnings": 0,
            "errors": 0,
            "artifacts": []
        },
    ]
}
{% endhighlight %}

After numerous tries, here is what I got working so far:

{% highlight groovy %}
def runUnitTests() {
    stage( "Unit Tests" ) {
        try {
            bat returnStatus: true, script: "\"${env.UE_ROOT_FOLDER}\\Engine\\Binaries\\Win64\\UE4Editor-Cmd.exe\" \"${env.WORKSPACE}\\${env.PROJECT_NAME}.uproject\" -ExecCmds=\"Automation RunTests ${env.PROJECT_NAME}\" -unattended -nopause -testexit=\"Automation Test Queue Empty\" -log -log=RunTests.log -NullRHI -ReportOutputPath=\"${env.WORKSPACE}\\Saved\\UnitTestsReport\""
        } catch ( Exception e ) {
        }

        convertUnitTestsReport()
        junit 'Saved/UnitTestsReport/junit.xml'
    }
}

def convertUnitTestsReport() {
    def json = readFile file: 'Saved/UnitTestsReport/index.json', encoding: "UTF-8"
    // Needed because the JSON is encoded in UTF-8 with BOM
    json = json.replace( "\uFEFF", "" );

    def xml_content = getJUnitXMLContentFromJSON( json )

    writeFile file: "${env.WORKSPACE}\\Saved\\UnitTestsReport\\junit.xml", text: xml_content.toString()
}

@NonCPS
def getJUnitXMLContentFromJSON( String json_content ) {
    def j = new JsonSlurper().parseText( json_content )
    
    def sw = new StringWriter()
    def builder = new MarkupBuilder( sw )

    builder.doubleQuotes = true
    builder.mkp.xmlDeclaration version: "1.0", encoding: "utf-8"

    builder.testsuite( tests: j.succeeded + j.failed, failures: j.failed, time: j.totalDuration ) {
        for ( test in j.tests ) {
            builder.testcase( name: test.testDisplayName, classname: test.fullTestPath, status: test.state ) {
                for ( entry in test.entries ) { 
                    builder.failure( message: entry.event.message, type: entry.event.type, entry.filename + " " + entry.lineNumber )
                }
            }
        }
    } 

    return sw.toString()
}
{% endhighlight %}

A few things to note again here:
* UE4 outputs the JSON as an UTF-8 with BOM file. This caused the JSON parser to not be able to parse it. This is why I had to specify the UTF-8 encoding when I read the file content, and replaced the character *"\uFEFF"*
* I had to extract the actual conversion of the json content to the xml output in a separate function with the attribute *@NonCPS* to avoid jenkins errors about serialization. More infos [here](https://jenkins.io/blog/2017/02/01/pipeline-scalability-best-practice/). Also, not putting the attribute resulted in the XML to not have any closing tags. More infos [here](https://issues.jenkins-ci.org/browse/JENKINS-48875).
* Because of [security reasons](https://jenkins.io/doc/book/managing/script-approval/), I had to manually approve signatures of functions which are used in this script. So, if you have execution errors related to security, go to the **Script Approval** section of jenkins and manually approve.

If everything goes fine, you should have a jUnit.xml file in *Saved\UnitTestsReport*, and the step 

{% highlight groovy %}
junit 'Saved/UnitTestsReport/junit.xml'
{% endhighlight %}

should process this file, and let jenkins show you the test result in your job page.

About the way you write your unit tests in UE4:
* since the jUnit format only allows one **\<failure\>** child for each **\<testcase\>**, if your unit test spits out multiple errors, you will have to concatenate all errors into one FString you will pass to **AddError** (defined in **FAutomationTestBase**).
* I'm not sure it's very relevant to get the tests runned, but the flags I use in the test definitions are: 
{% highlight cpp %}
EAutomationTestFlags::ApplicationContextMask | EAutomationTestFlags::ProductFilter
{% endhighlight %}

Feel free to comment or to give your suggestions!