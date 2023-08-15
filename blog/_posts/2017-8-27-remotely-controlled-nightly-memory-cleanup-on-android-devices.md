---
title: 'Remotely controlled nightly memory cleanup on Android devices'
layout: post
permalink: /remotely-controlled-nightly-memory-cleanup-on-android-devices/
dpsp_networks_shares:
    - 'a:5:{s:8:"facebook";i:0;s:9:"pinterest";i:0;s:6:"reddit";i:0;s:7:"twitter";i:0;s:8:"linkedin";i:0;}'
dpsp_networks_shares_total:
    - '0'
dpsp_networks_shares_last_updated:
    - '1690883323'
categories:
    - Mobile
    - Xamarin
tags:
    - Android
    - Azure
    - GCM
    - Xamarin.Forms
---

Imagine you have an enterprise application running on hundreds or thousands of mobile devices in the field. Your application is being used every day, and users prefer leaving the application in the foreground all the time, resulting in prolonged uptime.

While we all strive to create memory-responsible applications, the reality is that there may still be memory leaks. With intensive usage of the application described in the example above, even small memory leaks could eventually cause issues such as application malfunctions or breaks.

Our Android application is developed using Xamarin.Forms. The server runs on Azure and exposes a REST API developed using ASP.NET Web API. The server communicates with the mobile devices by sending messages through Azure Notification Hub and Google Cloud Messaging (GCM).

When we experienced memory-related issues on devices, we started brainstorming for possible solutions. Besides fixing all noticeable memory leaks, the best way to ensure that memory is in a good state is to restart the device. However, this method is intrusive, not easy to implement on Android, and generally discouraged. Nevertheless, it turns out that there are several applications on the Google Play Store that perform "fast rebooting". We chose FastReboot, a lightweight solution that "simulates a reboot by closing/restarting all core and user processes, thus freeing up memory".

![fast-reboot]({{ site.baseurl }}/assets/img/blog/Fast-Reboot-168x300.png)

## Solution

(Note: The Azure Notification Hub and GCM implementation are not the subject of this blog post. Keep in mind that Firebase Cloud Messaging (FCM) is now recommended over GCM, so you should use it for new implementations.)

For easier understanding, the code provided below is simplified (try-catch blocks and filtering logic have been removed).

### Server-side logic

We implemented a simple Web API method that only invokes the logic (in our case, implemented inside a worker task):

```csharp
[HttpGet]
public HttpResponseMessage FastReboot(string upn, string token)
{
    this.SecurePublicMethod(token);

    var connectionString = this.SettingProvider.ServiceBusConnectionString();
    WorkerTaskPump.Send(connectionString, WorkerTaskPump.WorkerTaskQueue, new FastRebootTask(upn));

    return new HttpResponseMessage(HttpStatusCode.Accepted);
}
```

Depending on your logic, the Web API method can accept other parameters. Below is a simplified implementation of FastRebootTask, responsible for obtaining mobile registration records and sending a message to every device. Typically, filtering logic is implemented here, so messages are sent only to specific devices.

```csharp
[Serializable]
[DataContract]
public class FastRebootTask : WorkerTaskBase
{

    protected override void ExecuteInternal(IKernel kernel)
    {
        var mailboxCenter = kernel.Get<IMailboxCenter>();
                
        var mailboxRegistrationRecords = mailboxCenter.GetMobileDeviceRegistrationsAsync().Result;

        foreach (var mailboxRegistrationRecord in mailboxRegistrationRecords.AsParallel())
        {
            this.SendFastRebootMessage(mailboxRegistrationRecord.Tag, mailboxCenter);
        }
    }

    private string SendFastRebootMessage(string destinationTag, IMailboxCenter mailboxCenter)
    {
        var mailboxMessageId = Guid.NewGuid().ToString("N").Substring(1, 8);
        var fastRebootMessage = new FastRebootMessage(
            mailboxMessageId,
            ServerManager.FastRebootTaskSourceTag,
            destinationTag,
            MailboxMessageStatus.None);

        mailboxCenter.PushAsync(fastRebootMessage);

        return mailboxMessageId;
    }
}
```

### Azure Scheduler

In the Azure portal, simply create a scheduler job that will invoke your Web API method URL. If you want it to be executed nightly, set a recurring job.

![](/assets/img/blog/xamarin-memory/Azure-Scheduler-249x300.png)


(Note: You could have filtering logic in place, creating multiple tasks, each invoking a different set of devices at different times.)

### Mobile solution

Fast Reboot frees up the memory of all applications in the background, leaving the one in the foreground working as expected. However, to ensure that the memory of our application is cleared as well, we needed to close our application before executing Fast Reboot. The solution was not straightforward, as killing the Android process removes all the intents that the process created. Nevertheless, we were able to utilize the Android Alarm Manager, which allows scheduling an action in the future. We implemented the following steps:

1. Schedule the Fast Reboot intent at "Now + 2 seconds".
2. Schedule the application intent at "Now + 10 seconds".
3. Kill the application.

This way, we let Fast Reboot clean all unnecessary memory left by processes (including the application, which is dead by that moment), and then start a fresh instance of the application.

```csharp
protected override async Task<bool> ProcessInternalAsync(FastRebootMessage mailboxMessage)
{
    this.ActivityService.LaunchFastReboot();
    this.ActivityService.RelaunchApp();

    return true;
}

public bool LaunchFastReboot()
{
    var alarmManager = (AlarmManager)Application.Context.GetSystemService(Context.AlarmService);
    var intent = Application.Context.PackageManager.GetLaunchIntentForPackage("com.greatbytes.fastreboot");

    if (intent != null)
    {
        var pendingServiceIntent = PendingIntent.GetActivity(Application.Context, 0, intent, PendingIntentFlags.CancelCurrent);
        alarmManager.Set(AlarmType.RtcWakeup, SystemClock.CurrentThreadTimeMillis() + ServiceManager.FastRebootPendingIntentDelay, pendingServiceIntent);
    }

    return intent != null;
}

public bool RelaunchApp()
{
    var alarmManager = (AlarmManager)Application.Context.GetSystemService(Context.AlarmService);

    var intent = Application.Context.PackageManager.GetLaunchIntentForPackage(Application.Context.PackageName);

    if (intent != null)
    {
        intent.AddFlags(ActivityFlags.ClearTask | ActivityFlags.NewTask);
        var pendingIntentId = ServiceManager.RelaunchNimbusPendingIntentId;

        var pendingServiceIntent = PendingIntent.GetActivity(Application.Context, pendingIntentId, intent, PendingIntentFlags.CancelCurrent);
        alarmManager.Set(AlarmType.ElapsedRealtimeWakeup, SystemClock.ElapsedRealtime() + ServiceManager.RelaunchNimbusPendingIntentDelay, pendingServiceIntent);
        Process.KillProcess(Process.MyPid());
    }

    return intent != null;
}
```

This solution works in all cases:
- The application is in the foreground.
- The application is in the background.
- The device is in sleep mode.

