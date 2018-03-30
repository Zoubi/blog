---
published: true
layout: post
title: Multi-platforms UE4 game on Jenkins
category: UE4
tags: [ UE4, Jenkins, GitHub ]
---

In a previous [post]({{ site.base_url }}/ue4/2018/01/28/Installed-build-of-UE4-with-console-support.html) I quickly said we use [Jenkins](https://jenkins.io/) to automate our builds of our in-development game. I thought it would be interesting to explain the process, and share what I came up with our configuration file.

We use the [git-flow](https://github.com/nvie/gitflow) methodology on our project. This means that whenever we develop a new feature, or want to fix a bug, we create a branch from the tip of develop, then push it on GitHub, where it is reviewed by the peers. If everything is OK, the branch is then merged in develop.

The problem with that approach, is that we often had cases where the game was broken because an asset would not be cooked anymore after some code change. Another common issue we had was that a specific platform was not compiling anymore because of compilation errors. We could have asked every member of the team to make sure that nothing is broken each time a pull request is created, but this would have obviously been a lot of lost time. 

This is why we used a continuous integration tool, Jenkins, to do all those checks for us.

In summary, here is what we wanted Jenkins to automate for us:
* for each opened PR: cook only the Win64 platform, compile all platforms
* for the currently opened feature branch : cook and compile all platforms
* for the develop and master branches: cook, compile, archive all platforms

First, let's head to Jenkins and add a new job of type [Multibranch pipeline](https://jenkins.io/doc/book/pipeline/multibranch/#creating-a-multibranch-pipeline). We have 2 kinds of branches we need to find on GitHub, which must trigger a job on Jenkins : Pull requests and some regular branches (develop and master).

We then need 2 *Branch Sources* entries to discover the branches:

* Pull requests:

![Pull Requests config]({{ "/assets/img/jenkins_multibranch_pull_requests_config.png" | absolute_url }})

Notice the advanced clone behaviours section which will allow to not download the entire repository history, which will speed up the job, and lower the disk footprint.

* Branches:

![Branches config]({{ "/assets/img/jenkins_multibranch_branch_config.png" | absolute_url }})

We keep the same clone behaviours section, we manually filter the branches we are interested in, in the filter section.

The last thing to setup is to tell jenkins where to find the [Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile/), which is basically a configuration file written in [Groovy](http://groovy-lang.org/). In our case, we use a scripted jenkins file (as opposed to a declarative jenkins file).

![Jenkinsfile config]({{ "/assets/img/jenkins_multibranch_other_config.png" | absolute_url }})

Without further ado, here is the last iteration of our jenkinsfile, with comments to explain some portions of the code:

{% highlight groovy %}

env.PROJECT_NAME = "YourGameName"

// If the job to run targets a branch which is already being built, cancel the already running job
abortPreviousRunningBuilds()

def tasks = [:]

// Expose some properties in the UI of jenkins when we run the job manually
properties([
    parameters([
        string( name: "ROOT_ARCHIVE_DIRECTORY", defaultValue: 'V:/', trim: true ),
        booleanParam( name : "DEPLOY_BUILD", defaultValue: false )
    ]),
    buildDiscarder( logRotator( numToKeepStr: '1' ) )
])

stage( 'Check parameters' ) {
    try {
        if ( params.ROOT_ARCHIVE_DIRECTORY == "" ) {
            error "ROOT_ARCHIVE_DIRECTORY must be set !"
        }
    } catch ( Exception err ) {
        slackSend channel: 'jenkins', color: 'danger', message: "Build failed : ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
        currentBuild.result = "FAILURE"
    }
}

/* 
Each platform will be built in parallel. Parallelize accross all the slaves
Better parallelize each platform than each pull request, to save time when only 
a few jobs are running
*/
[ 'Win64', 'XboxOne', 'PS4', 'Switch' ].each {
    tasks[ it ] = {

        env.BRANCH_TYPE = getBranchType( env.BRANCH_NAME )
        env.DEPLOYMENT_ENVIRONMENT = getBranchDeploymentEnvironment( env.BRANCH_TYPE )
        env.CLIENT_CONFIG = getClientConfig( env.DEPLOYMENT_ENVIRONMENT )

        String labels = getNodeLabels()

        // Some jobs must be executed exclusively on a dedicated node of jenkins
        node( labels ) {
            // Some nodes store the job workspace (and the UE4 installed build) 
            // in a different directory
            env.WORKSPACE = getWorkSpace()

            ws( env.WORKSPACE ) {

                // For shipping builds, clear to force a full rebuild + full cook
                if ( env.DEPLOYMENT_ENVIRONMENT == "shipping" ) {
                    deleteDir()
                }

                stage( 'Checkout' ) {
                    sendMessageToSlack( "Build started", it, "#0000FF" )
                    checkout scm
                }

                try {
                    // Always build manually the editor due to a bug: 
                    // using RunUAT does not always compile the UE4Editor-XXX.dll
                    buildEditor( it )
                    // Do the actual build cook run; All specialized work is handled here
                    buildCookRun( it )
                    sendMessageToSlack( "Successfully processed", it, "good" )
                } catch ( Exception err ) {
                    sendMessageToSlack( "Failed to process", it, "danger", "Reason : " + err.toString() )
                    
                    error "Failed to process " + it + " : " + err.toString()
                }
            }
        }
    }
}

// Run all tasks; We are done!
parallel tasks

// ------------------------------------//
// All the helper functions used above //
// ------------------------------------//

def getBranchType( String branch_name ) {
    if ( branch_name =~ ".*develop" ) {
        return "development"
    } else if ( branch_name =~ ".*release/.*" ) {
        return "release"
    } else if ( branch_name =~ ".*master" ) {
        return "master"
    }

    return "test";
}

def getBranchDeploymentEnvironment( String branch_type ) {
    if ( branch_type == "development" ) {
        return "development"
    } else if ( branch_type == "release" ) {
        return "release"
    }else if ( branch_type == "master" ) {
        return "shipping"
    }
    
    return "testing";
}

def getClientConfig( String environment_deployment ) {
    if ( environment_deployment == "shipping" ) {
        return "Shipping"
    }

    // release and development return Development
    return "Development"
}

def sendMessageToSlack( String message, String platform, String color, String suffix = "" ) {

    String full_message = message + " : #${env.BUILD_NUMBER} - ${env.JOB_NAME} - " + platform + " On ${env.NODE_NAME} (<${env.BUILD_URL}|Open>)"

    if ( !( suffix?.trim() ) ) {
        full_message += " " + suffix
    }

    slackSend channel: 'jenkins', color: color, message: full_message
}

// Manually build the editor of the game using UnrealBuiltTool
def buildEditor( String platform ) {
    stage ( "Build Editor Win64 for " + platform ) {
        bat getUE4DirectoryFolder() + "/Engine/Binaries/DotNET/UnrealBuildTool.exe ${env.PROJECT_NAME}Editor Win64 Development ${env.WORKSPACE}/${env.PROJECT_NAME}.uproject"
    }
}

def buildCookRun( String platform ) {

    // Dont archive for bugfix / hotfix / etc...
    Boolean can_archive_project = ( env.DEPLOYMENT_ENVIRONMENT == "development" 
        || env.DEPLOYMENT_ENVIRONMENT == "shipping" )

    // Cook if we want to archive (obviously) and always cook on Win64 to check PRs won't break
    Boolean can_cook_project = can_archive_project || ( platform == "Win64" )

    stage ( "Build " + platform ) {
        bat getUATCommonArguments( platform ) + getUATBuildArguments()
    }

    if ( can_cook_project ) {
        stage ( "Cook " + platform ) {

            // Some platforms may need specific commands to be executed before the cooker starts
            executePlatformPreCookCommands( platform )
            bat getUATCommonArguments( platform ) + getUATCookArguments( platform, env.CLIENT_CONFIG, can_archive_project )
            executePlatformPostCookCommands( platform )
        }
    }
}

def getWorkSpace() {
    return getWorkSpaceRootFolder() + getWorkSpaceFolderName()
}

def getArchiveDirectory( String client_config ) {
    return env.ROOT_ARCHIVE_DIRECTORY + env.PROJECT_NAME + "/" + client_config
}

def getWorkSpaceFolderName() {
    // Shipping jobs always run on the master node, in a specific folder
    if ( env.DEPLOYMENT_ENVIRONMENT == "shipping" ) {
        return "Jenkins_${env.PROJECT_NAME}_Master"
    } else {
        // Use same folder for PRs and develop since we now have multiple nodes working in parallel
        return "Jenkins_${env.PROJECT_NAME}"
    }
}

def getWorkSpaceRootFolder() {
    if ( env.NODE_NAME == "master" ) {
        return "D:/"
    }

    return "C:/"
}

def getNodeLabels() {
    // Do all shipping builds on master for storage purposes
    if ( env.DEPLOYMENT_ENVIRONMENT == "shipping" ) {
        return "master"
    }

    return "ue4"
}

def getUE4DirectoryFolder() {
    if ( env.NODE_NAME == "master" ) {
        return "D:/UE4/Windows"
    }

    return "C:/UE4/Windows"
}

def getUATCommonArguments( String platform ) {
    String result = getUE4DirectoryFolder() + "/Engine/Build/BatchFiles/RunUAT.bat BuildCookRun -project=${env.WORKSPACE}/${env.PROJECT_NAME}.uproject -utf8output -noP4 -platform=" + platform + " -clientconfig=" + env.CLIENT_CONFIG

    result += getUATCompileFlags()

    if ( env.CLIENT_CONFIG == "Shipping" ) {
        result += " -clean"
    }

    return result
}

def getUATCompileFlags() {
    // -nocompile because we already have the automation tools
    // -nocompileeditor because we built it before
    return " -nocompile -nocompileeditor -installed -ue4exe=UE4Editor-Cmd.exe"
}

def getUATBuildArguments() {
    // build only. dont cook. This is done in a separate stage
    return " -build -skipcook"
}

def getUATCookArguments( String platform, String client_config, Boolean archive_project ) {
    String result = " -allmaps -cook"
    
    result += getUATCookArgumentsFromClientConfig( client_config)
    result += getUATCookArgumentsForPlatform( platform )

    if ( archive_project ) {
        result += " -pak -package -stage -archive -archivedirectory=" + getArchiveDirectory( client_config )
    }

    return result;
}

def getUATCookArgumentsFromClientConfig( String client_config ) {
    // Do not cook what has already been cooked if possible
    if ( client_config == "Development" ) {
        return " -iterativecooking"
    }
    // but not in shipping; Do a full cook.
    else if ( client_config == "Shipping" ) {
        return " -distribution"
    }
}

def getUATCookArgumentsForPlatform( String platform ) {
    String result = ""

    // See https://docs.unrealengine.com/latest/INT/Engine/Basics/Projects/Packaging/
    if ( platform != "PS4" ) {
        result += " -compressed"
    }

    if ( params.DEPLOY_BUILD ) {
        if ( platform == "PS4" ) {
            result += " -deploy -cmdline=\" -Messaging\" -device=PS4@192.168.XXX.XXX"
        }
        else if ( platform == "XboxOne" ) {
            result += " -deploy -cmdline=\" -Messaging\" -device=XboxOne@192.168.YYY.YYY"
        }
    }

    return result
}

def executePlatformPreCookCommands( String platform ) {
    if ( platform == "PS4" ) {
        
    }
}

def executePlatformPostCookCommands( String platform ) {
    if ( platform == "PS4" ) {
        
    }
}

// Note you will have to add some exceptions in the Jenkins security options to allow this function to run
def abortPreviousRunningBuilds() {
  def hi = Jenkins.instance
  def pname = env.JOB_NAME.split('/')[0]

  hi.getItem( pname ).getItem(env.JOB_BASE_NAME).getBuilds().each{ build ->
    def exec = build.getExecutor()

    if ( build.number < currentBuild.number && exec != null ) {
      exec.interrupt(
        Result.ABORTED,
        new CauseOfInterruption.UserInterruption(
          "Aborted by #${currentBuild.number}"
        )
      )
      println("Aborted previous running build #${build.number}")
    }
  }
}
{% endhighlight %}

That's a pretty big file, result of many weeks of iteration. It could be more consistent when using environment variables or function arguments, but at least it gets the job done for us :)

I hope you will find it useful. You can find that file [here](https://github.com/Zoubi/UE4JenkinsScripts/blob/master/UE4Game_Jenkinsfile), along a few other scripts we use here.

Do not hesitate to comment, to give your suggestions!