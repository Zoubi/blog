---
layout: post
title: Build a UE4 multi-platforms game on Jenkins
category: UE4
tags: [ UE4, Jenkins, GitHub ]
---

In a previous [post]({{ site.base_url }}/ue4/2018/01/28/Installed-build-of-UE4-with-console-support.html) I quickly said we use [Jenkins](https://jenkins.io/) to automate our builds of our in-development game. I thought it would be interesting to explain the process, and share what I came up with our configuration file.

We use the [git-flow](https://github.com/nvie/gitflow) methodology on our project. This means that whenever we develop a new feature, or want to fix a bug, we create a branch from the tip of develop, then push it on GitHub, where it is reviewed by the peers. If everything is OK, the branch is then merged in develop.

The problem with that approach, is that we often had cases where the game was broken because an asset would not be cooked anymore after some code change. Another common issue we had was that a specific platform was not compiling anymore because of compilation errors. We could have asked every member of the team to make sure that nothing is broken each time a pull request is created, but this would have obviously been a lot of lost time. 

This is why we used a continuous integration tool, Jenkins, to do all those checks for us.

In summary, here are what we wanted Jenkins to automate for us:
* for each opened PR: cook only the Win64 platform, compile all platforms
* for the currently opened feature branch : cook and compile all platforms
* for the develop and master branches: cook, compile, archive all platforms

you don't always know before a branch is merged in develop, that it's impossible to cook the game data

![My helpful screenshot]({{ "/assets/img/jenkins_multibranch_pull_requests_config.png" | absolute_url }})
![My helpful screenshot]({{ "/assets/img/jenkins_multibranch_branch_config.png" | absolute_url }})

```
abortPreviousRunningBuilds()

def tasks = [:]

properties([
    parameters([
        string( name: "ROOT_DEPLOY_DIRECTORY", defaultValue: 'V:/ShiftQuantum', trim: true ),
        booleanParam( name : "DEPLOY_BUILD", defaultValue: false )
    ]),
    buildDiscarder( logRotator( numToKeepStr: '1' ) )
])

stage( 'Check parameters' ) {
    try {
        if ( params.ROOT_DEPLOY_DIRECTORY == "" ) {
            error "ROOT_DEPLOY_DIRECTORY must be set !"
        }
    } catch ( Exception err ) {
        slackSend channel: 'jenkins', color: 'danger', message: "Build failed : ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
        currentBuild.result = "FAILURE"
    }
}

[ 'Win64', 'XboxOne', 'PS4', 'Switch' ].each {
    tasks[ it ] = {

        env.BRANCH_TYPE = getBranchType( env.BRANCH_NAME )
        env.DEPLOYMENT_ENVIRONMENT = getBranchDeploymentEnvironment( env.BRANCH_TYPE )
        env.CLIENT_CONFIG = getClientConfig( env.DEPLOYMENT_ENVIRONMENT )

        String labels = getNodeLabels()

        node( labels ) {
            env.WORKSPACE = getWorkSpace()

            ws( env.WORKSPACE ) {

                if ( env.DEPLOYMENT_ENVIRONMENT == "shipping" ) {
                    deleteDir()
                }

                stage( 'Checkout' ) {
                    sendMessageToSlack( "Build started", it, "#0000FF" )
                    checkout scm
                }

                // Do not specify the platform. The build tool will automatically create one for us
                env.DEPLOY_DIRECTORY = "${params.ROOT_DEPLOY_DIRECTORY}/${env.CLIENT_CONFIG}"

                try {
                    buildEditor( it )
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

parallel tasks

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

    return "Development"
}

def sendMessageToSlack( String message, String platform, String color, String suffix = "" ) {

    String full_message = message + " : #${env.BUILD_NUMBER} - ${env.JOB_NAME} - " + platform + " On ${env.NODE_NAME} (<${env.BUILD_URL}|Open>)"

    if ( !( suffix?.trim() ) ) {
        full_message += " " + suffix
    }

    slackSend channel: 'jenkins', color: color, message: full_message
}

def buildEditor( String platform ) {
    stage ( "Build Editor Win64 for " + platform ) {
        bat getUE4DirectoryFolder() + "/Engine/Binaries/DotNET/UnrealBuildTool.exe ShiftQuantumEditor Win64 Development " + env.WORKSPACE + "/ShiftQuantum.uproject"
        warnings canComputeNew: false, canResolveRelativePaths: false, categoriesPattern: '', consoleParsers: [[parserName: 'MSBuild'], [parserName: 'UE4BlueprintCompiler']], defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', unHealthy: ''
    }
}

def buildCookRun( String platform ) {

    // Dont archive for bugfix / hotfix / etc...
    Boolean can_archive_project = ( env.DEPLOYMENT_ENVIRONMENT == "development" || env.DEPLOYMENT_ENVIRONMENT == "shipping" )
    // Cook if we want to archive (obviously) and always cook on Win64 to check PRs won't break
    Boolean can_cook_project = can_archive_project || ( platform == "Win64" )

    stage ( "Build " + platform ) {
        bat getUATCommonArguments( platform ) + getUATBuildArguments()
        warnings canComputeNew: false, canResolveRelativePaths: false, categoriesPattern: '', consoleParsers: [[parserName: 'MSBuild'], [parserName: 'UE4BlueprintCompiler']], defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', unHealthy: ''
    }

    if ( can_cook_project ) {
        stage ( "Cook " + platform ) {

            executePlatformPreCookCommands( platform )
            bat getUATCommonArguments( platform ) + getUATCookArguments( platform, env.CLIENT_CONFIG, can_archive_project )
            executePlatformPostCookCommands( platform )
        }
    }
}

def getWorkSpace() {
    return getWorkSpaceRootFolder() + getWorkSpaceFolderName()
}

def getWorkSpaceFolderName() {
    if ( env.DEPLOYMENT_ENVIRONMENT == "shipping" ) {
        return "Jenkins_ShiftQuantum_Master"
    } else {
        // Use same folder for PRs and develop since we now have multiple nodes working in parallel
        return "Jenkins_ShiftQuantum"
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
    // -nocompile to win some seconds because automation tools are already built on the build server
    String result = getUE4DirectoryFolder() + "/Engine/Build/BatchFiles/RunUAT.bat BuildCookRun -project=" + env.WORKSPACE + "/ShiftQuantum.uproject -utf8output -noP4 -platform=" + platform + " -clientconfig=" + env.CLIENT_CONFIG

    result += getUATCompileFlags()

    if ( env.CLIENT_CONFIG == "Shipping" ) {
        result += " -clean"
    }

    return result
}

def getUATCompileFlags() {
    return " -nocompile -nocompileeditor -installed -ue4exe=UE4Editor-Cmd.exe"
}

def getUATBuildArguments() {
    return " -build -skipcook"
}

def getUATCookArguments( String platform, String client_config, Boolean archive_project ) {
    String result = " -allmaps -cook"
    
    result += getUATCookArgumentsFromClientConfig( client_config)
    result += getUATCookArgumentsForPlatform( platform )

    if ( archive_project ) {
        result += " -pak -package -stage -archive -archivedirectory=" + env.DEPLOY_DIRECTORY
    }

    return result;
}

def getUATCookArgumentsFromClientConfig( String client_config ) {
    if ( client_config == "Development" ) {
        return " -iterativecooking"
    }
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
            result += " -deploy -cmdline=\" -Messaging\" -device=PS4@192.168.2.217"
        }
        else if ( platform == "XboxOne" ) {
            result += " -deploy -cmdline=\" -Messaging\" -device=XboxOne@192.168.1.13"
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
```