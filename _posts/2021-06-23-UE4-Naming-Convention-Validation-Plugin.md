---
published: true
layout: post
title: UE4 Naming Convention Validation plugin
category: UE4
tags: [ UE4, PlugIn ]
---

Over the last few years I have developed a few different UE4 plugins which are open source on GitHub. I figured this would be a good idea to talk about them here on this blog, as they proved to be very useful while developing our games at Fishing Cactus.

Let's start with the [Naming Convention Validation plugin](https://github.com/TheEmidee/UE4NamingConventionValidation) !

This plugin has the same base as the DataValidation plugin from UE. It provides the same editor context menu options, to validate naming of individual assets, or assets in a folder. It can also be run as a commandlet, on a continuous integration system for example. It will then output all the errors it could find in the log, which you can then parse using regular expressions for example.

But first things first, you can access the plugin configuration in the project settings. You have different basic options which are self-explanatory.

![Settings]({{ "/assets/img/namingconventionvalidation/Settings1.png" | absolute_url }})

But the real deal is with the Class Descriptions property:

![Class Descriptions]({{ "/assets/img/namingconventionvalidation/Settings2.png" | absolute_url }})

This is in the class descriptions that you can specify which prefix and / or suffix are required in the name of assets, depending on their type. If an asset matches multiple descriptions, there's the Priority property to filter them out.

Once everything is setup, you can right click in the content browser on some assets or folders, and select `Validate Assets Naming Convention`:

![Contextual Menu]({{ "/assets/img/namingconventionvalidation/ContextualMenu.png" | absolute_url }})

The results of the validation will then be added to the message log:

![Message Lo]({{ "/assets/img/namingconventionvalidation/MessageLog.png" | absolute_url }})

While the class descriptions are fine to validate individual assets, sometimes you want more elaborate conventions. For example, you may require all assets in a *Weapons* folder to have the weapon name in their asset name.

To achieve that, you can create a class which derives from `UEditorNamingValidatorBase`. You then just have to override the functions `CanValidateAssetNaming` and `ValidateAssetNaming`. This allows you to have more context about the asset, like its full path, and to implement more complex validation schemes.

You can find such an example [here](https://theemidee.github.io/UE4NamingConventionValidation/).

As with the data validation plugin bundled with UE, the plugin comes with a commandlet which allows you to validate the naming convention using the command line. This is most useful in the context of a continuous integration system, in which you can fail a job if an asset is wrongly named.

On one of our projects, we trigger that commandlet using Buildgraph, using the following XML node:

```
<Node Name="Naming Convention Validation" Requires="Compile SwarmsEditor Win64;">
    <Commandlet Name="NamingConventionValidation" Project="$(ProjectPath)" Arguments="-log -log=NamingConventionValidation.log -unattended -nopause -nullrhi" />
</Node>
```

After the commandlet is run, all the errors will be written in the log file, which you can parse using regular expressions. 

That's it for the quick presentation of that plugin. Thanks to it, we now enforce a real consistency in how the assets are named. This makes searching for assets in the editor faster, but has also other advantages, like avoiding errors when there's a need to programatically construct a path to an asset.

The plugin is open source, has a permissive licence, so feel free to use it! Of course, pull requests are opened if you want to contribute to the project.