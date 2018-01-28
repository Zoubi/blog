---
published: true
layout: post
title: Installed build of UE4 with console support
category: UE4
tags: [ UE4, BuildGraph, Jenkins, Console ]
---

[Unreal Engine 4](https://www.unrealengine.com), since the version 4.13, has in its kit a tool named [BuildGraph](https://docs.unrealengine.com/latest/INT/Programming/Development/BuildGraph/), which they use internally at Epic to create the installed engine we can install from their launcher.

We are in the beginning of the port of our next game on consoles. Developing for console with UE4 requires, as you may be know, to build the game using the sources of the engine. What's problematic is that the initial compilation time is really huge, opening the visual studio solution is slow, and every developer has to be aware of how to build the engine. Including artists or designers if they wish to deploy the game on the console to check how the game or a particular asset behaves...

To ease the development and allow us better iteration times, we thought of using BuildGraph to try to compile the engine with the support of the console platforms on our build server, and allow anyone on the team to use the engine as if they just had downloaded it from the Epic Launcher.

It turns out it is very possible, and very easy to do :)

The process is the same we use to compile our game, and to check the pull requests we want to merge do not break the code or the assets. 

We have a job on our [Jenkins ](https://jenkins.io/) continuous integration server, which polls on the develop branch of the fork of the engine. Whenever a new commit has been pushed on that branch, the job is executed and two simple steps happen:
- Checkout the branch locally
- Run the build graph script through a call to a bat command

Pretty easy isn't it?

If you have a look at the default [InstalledEngineBuild.xml](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Build/InstalledEngineBuild.xml) you will notice that if you set HostPlatformOnly to true, that will unset the other platforms:

```
<Do If="'$(HostPlatformOnly)' == true">
  <!-- Activate correct Target Platforms for host -->
  <Switch>
    <Case If="'$(HostPlatform)' == 'Win64'">
      <Property Name="WithWin64" Value="true"/>
      <Property Name="WithWin32" Value="true"/>
      <Property Name="WithMac" Value="false"/>
      <Property Name="WithLinux" Value="false"/>
    </Case>
    <Case If="'$(HostPlatform)' == 'Mac'">
      <Property Name="WithWin64" Value="false"/>
      <Property Name="WithWin32" Value="false"/>
      <Property Name="WithMac" Value="true"/>
      <Property Name="WithLinux" Value="false"/>
    </Case>
    <Case If="'$(HostPlatform)' == 'Linux'">
      <Property Name="WithWin64" Value="false"/>
      <Property Name="WithWin32" Value="false"/>
      <Property Name="WithMac" Value="false"/>
      <Property Name="WithLinux" Value="true"/>
    </Case>
    <Default>
      <Error Message="$(HostPlatform) is not supported for making an installed build"/>
    </Default>
  </Switch>
  <!-- Other Target Platforms always disabled -->
  <Property Name="WithAndroid" Value="false"/>
  <Property Name="WithIOS" Value="false"/>
  <Property Name="WithTVOS" Value="false"/>
  <Property Name="WithHTML5" Value="false"/>
  <Property Name="WithPS4" Value="false"/>
  <Property Name="WithXboxOne" Value="false"/>
  <Property Name="WithSwitch" Value="false"/>
</Do>
```

I then just set the console platform flags to true, to override those settings:

```
bat "${env.WORKSPACE}/Engine/Engine/Build/BatchFiles/RunUAT.bat" BuildGraph 
-target="Make Installed Build Win64" 
-script="${env.WORKSPACE}/Engine/Build/InstalledEngineBuild.xml" 
-set:HostPlatformOnly=true -set:WithPS4=true 
-set:WithXboxOne=true -set:WithSwitch=true 
-set:SavedOutput=V:/UE4Consoles
```

 One hour and half later, when the compilation finished, I realized that the output folder did not contain any console specific files...

I then removed the HostPlatformOnly option, and manually set every platform flag we need to true, and the platforms we don't need to false:

```
bat "${env.WORKSPACE}/Engine/Engine/Build/BatchFiles/RunUAT.bat" BuildGraph 
-target="Make Installed Build Win64" -script="${env.WORKSPACE}/Engine/Build/InstalledEngineBuild.xml" 
-set:WithWin64=true -set:WithPS4=true -set:WithXboxOne=true -set:WithSwitch=true 
-set:WithWin32=false -set:WithMac=false -set:WinLinux=false -set:WithIOS=false 
-set:WithAndroid=false -set:WithTVOS=false -set:WithHTML5=false -set:SavedOutput=V:/UE4Consoles
```

After an even longer compilation time (which was a good sign after all), I generated the Visual Studio solution file of our game, using that new Installed Engine as the source folder. And I was happy to see that the console platforms were correctly added to the project, and that I could deploy to the consoles directly from the editor.
