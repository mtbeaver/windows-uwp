---
author: TylerMSFT
title: Group background task registration
description: Register/unregister background tasks as part of a group to isolate those registrations.
ms.author: twhitney
ms.date: 04/05/2017
ms.topic: article
ms.prod: windows
ms.technology: uwp
keywords: windows 10, background task
---

# Group background task registration

**Important APIs**

[BackgroundTaskRegistrationGroup class](https://docs.microsoft.com/en-us/uwp/api/windows.applicationmodel.background.backgroundtaskregistrationgroup)

Background tasks can now be registered in a group, which you can think of as a logical namespace. This isolation helps ensure that different components of an app, or different libraries, don’t interfere with each other’s background task registration.

When an app and the framework (or library) it uses registers a background task with the same name, the app could inadvertently remove the framework's background task registrations. App authors could also accidently remove framework and library background task registrations because they could unregister all registered background tasks by using [BackgroundTaskRegistration.AllTasks](https://docs.microsoft.com/en-us/uwp/api/windows.applicationmodel.background.backgroundtaskregistration#Windows_ApplicationModel_Background_BackgroundTaskRegistration_AllTasks).  With groups, you can isolate your background task registrations so this doesn't happen.

## Features of groups

* Groups can be uniquely identified by a GUID. They can also have an associated friendly name string which is easier to read while debugging.
* Multiple background tasks can be registered in a group.
* Background tasks registered in a group won't appear in [BackgroundTaskRegistration.AllTasks](https://docs.microsoft.com/en-us/uwp/api/windows.applicationmodel.background.backgroundtaskregistration#Windows_ApplicationModel_Background_BackgroundTaskRegistration_AllTasks). Thus apps that currently use **BackgroundTaskRegistration.AllTasks** to unregister their tasks won't inadvertently unregister background tasks registered in a group. See [Unregister background tasks in a group example](#unregister-background-tasks-in-a-group-example) below to see how to unregister all background triggers that have been registered as part of a group.
* Each Background Task Registration will have a Group property to determine which group it is associated with.

## Register a background task in a group

The following shows how to register a background task (triggered by a time zone change, in this example) as part of a group.

```csharp
private const string groupFriendlyName = "myGroup";
private const string groupId = "3F2504E0-4F89-41D3-9A0C-0305E82C3301";
private const string myTaskName = "My Background Trigger";

public static void RegisterBackgroundTaskInGroup()
{
   BackgroundTaskRegistrationGroup group = BackgroundTaskRegistration.GetTaskGroup(groupId);
   bool isTaskRegistered = false;

   // See if this task already belongs to a group
   if (group != null)
   {
       foreach (var taskKeyValue in group.AllTasks)
       {
           if (taskKeyValue.Value.Name == myTaskName)
           {
               isTaskRegistered = true;
               break;
           }
       }
   }

   // If the background task is not in a group, register it
   if (!isTaskRegistered)
   {
       if (group == null)
       {
           group = new BackgroundTaskRegistrationGroup(groupId, groupFriendlyName);
       }

       var builder = new BackgroundTaskBuilder();
       builder.Name = myTaskName;
       builder.TaskGroup = group; // we specify the group, here
       builder.SetTrigger(new SystemTrigger(SystemTriggerType.TimeZoneChange, false));

       // Because builder.TaskEntryPoint is not specified, OnBackgroundActivated() will be raised when the background task is triggered
       BackgroundTaskRegistration task = builder.Register();
   }
}
```

## Unregister background tasks in a group

The following shows how to unregister background tasks that were registered as part of a group.
Because background tasks registered in a group don't appear in [BackgroundTaskRegistration.AllTasks](https://docs.microsoft.com/en-us/uwp/api/windows.applicationmodel.background.backgroundtaskregistration#Windows_ApplicationModel_Background_BackgroundTaskRegistration_AllTasks), you must iterate through the groups, find the background tasks registered to each group, and unregister them.

```csharp
private static void UnRegisterAllTasks()
{
    // Unregister tasks that are part of a group
    foreach (var groupKeyValue in BackgroundTaskRegistration.AllTaskGroups)
    {
        foreach (var groupedTask in groupKeyValue.Value.AllTasks)
        {
            groupedTask.Value.Unregister(true); // passing true to cancel currently running instances of this background task
        }
    }

    // Unregister tasks that aren't part of a group
    foreach(var taskKeyValue in BackgroundTaskRegistration.AllTasks)
    {
        taskKeyValue.Value.Unregister(true); // passing true to cancel currently running instances of this background task
    }
}
```
