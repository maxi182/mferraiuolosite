---
layout: post
title: "Do not over Sync!"
date: 2016-10-16 12:00:40 +01:00
categories: testing
author: Maximiliano Ferraiuolo
comments: true
tags: testing,android
---
You love Sync... you like ping to the server as often as you can... and you know it. Well, you also must concern about this as you probably know you are killing your phone battery as networking is the biggest battery hog there is, and we would be overloading the server.

# How networking is draining our battery?
Not only it has to initialise the chip for radio, also it keeps alive for an additional 20 to 60 seconds after you are done with your request (drawing power).. And you can have a worst situation where each one of your network request ends up waking up the radio.. and paying that cost.

![Diagram of the Android radio state machine](https://www.bignerdranch.com/img/blog/2016/04/mobile_radio_state_machine.png) 


# Types of networking requests.

OK, we already know why we should not over sync, *so when we do it?* The key here is to understand the difference between

* request that must happen RIGHT NOW
* request that could be DELAYED

![](https://dl.dropboxusercontent.com/u/37464129/max/Screen%20Shot%202016-10-17%20at%2011.12.53.png) 

1. **DO NOW** The user can take an action to get a request feed updated right now, so you will have to kick that request straight away.
2. **PUSH DATA and SERVER RESPONSE** is where we actually must start improving performance. Those are the requests that happen on a regular intervals that keeps data up to date but DON'T need to happen right this second.

# So, How we should emit this Syncs in a perform way?
Luckly, google play services provide us
# ** GCM Network Manager **


The [GCM Network Manager](https://developers.google.com/android/reference/com/google/android/gms/gcm/GcmNetworkManager) enables apps to register services that perform network-oriented tasks, where a task is an individual piece of work to be done. The API helps with scheduling these tasks, allowing Google Play services to batch network operations across the system.
The API also helps simplify common networking patterns, such as waiting for network connectivity, network retries and backoff. Essentially, the GCM Network Manager allows developers to focus a little less on networking issues and more on the actual app, by providing a straightforward API for scheduling network requests.

**Let's make GCM Network Manager handle those batching requests for us:**


First, add the appropriate dependency for the GCM Network Manager to the build.gradle file:

```java
 
    compile 'com.google.android.gms:play-services-gcm:9.6.1' 

```
Make sure you are not adding the whole google play service dependency which is going to add too many unnecessary libs, and you can pass your dex limit method count over 65k, just add play-services-gcm.

Next, we must declare a new Service in the Android Manifest inside <application>:


{% highlight xml %}
 <service android:name=".MyTaskService"
            android:permission="com.google.android.gms.permission.BIND_NETWORK_TASK_SERVICE"
            android:exported="true">
            <intent-filter>
                <action android:name="com.google.android.gms.gcm.ACTION_TASK_READY"/>
            </intent-filter>
        </service>
{% endhighlight %}


The name of the service (MyTaskService) is the name of the class that will extend GcmTaskService, which is the core class for dealing with GCM Network Manager. This service will handle the running of a task, where a task is any piece of individual work that we want to accomplish. The intent-filter of action SERVICE_ACTION_EXECUTE_TASK is to receive the notification from the scheduler of the GCM Network Manager that a task is ready to be executed.


Next, we’ll actually define the GcmTaskService, making a MyTaskService.java class that extends GcmTaskService,
due to extending GcmTaskService, we must implement the onRunTask method in CustomService:



```java
public class MyTaskService extends GcmTaskService {
@Override
public int onRunTask(TaskParams taskParams) {
        Log.i(TAG, "onRunTask");
        switch (taskParams.getTag()) {
                case TAG_TASK_ONEOFF_LOG:
                        Log.i(TAG, TAG_TASK_ONEOFF_LOG);
                        // This is where useful work would go
                        return GcmNetworkManager.RESULT_SUCCESS;
                case TAG_TASK_PERIODIC_LOG:
                        Log.i(TAG, TAG_TASK_PERIODIC_LOG);
                        // This is where useful work would go
                        return GcmNetworkManager.RESULT_SUCCESS;
                default:
                        return GcmNetworkManager.RESULT_FAILURE;
        }

    }

}
```
This is what's called when it's time for a task to be run. We check the tag of the TaskParams that’s been given as a parameter, as each tag will uniquely identify a different task that’s been scheduled. Normally some important networking task or logic would occur inside the case blocks, but this example just prints a log statement.


# Schedule the Task

A single task is one individual piece of work to be performed, and there are two types:

*  **one-off**
 
*  **periodic**

We provide the task with a window of execution, and the scheduler will determine the actual execution time. Since tasks are non-immediate, the scheduler can batch together several network calls to preserve battery life.
The scheduler will consider network availability, network activity and network load. If none of these matter, the scheduler will always wait until the end of the specified window.

Now, here’s how we would schedule a **one-off** task:



```
#!java


Task task = new OneoffTask.Builder()
              .setService(MyTaskService.class)
              .setExecutionWindow(0, 30)
              .setTag(LogService.TAG_TASK_ONEOFF_LOG)
              .setUpdateCurrent(false)
              .setRequiredNetwork(Task.NETWORK_STATE_CONNECTED)
              .setRequiresCharging(false)
              .build();

GcmNetworkManager.getInstance(context).schedule(task);

```

Using the builder pattern, we define all the aspects of our Task:

* **Service:** The specific GcmTaskService that will control the task. This will allow us to cancel it later.

* **Execution window:** The time period in which the task will execute. First param is the lower bound and the second is the upper bound (both are in seconds). This one is mandatory.

* **Tag:** We’ll use the tag to identify in the onRunTask method which task is currently being run. Each tag should be unique, and the max length is 100.

* **Update Current:** This determines whether this task should override any pre-existing tasks with the same tag. By default, this is false, so new tasks don’t override existing ones.
* **Required Network:** Sets a specific network state to run on. If that network state is unavailable, then the task won’t be executed until it becomes available.

* **Requires Charging:** Whether the task requires the device to be connected to power in order to execute.


And, here’s how we would schedule a **periodic** task:


```
#!java

Task task = new PeriodicTask.Builder()
                        .setService(MyTaskService.class)
                        .setPeriod(30)
                        .setFlex(10)
                        .setTag(LogService.TAG_TASK_PERIODIC_LOG)
                        .setPersisted(true)
                        .build();

GcmNetworkManager.getInstance(context).schedule(task);

```

 
* **Period:** Specifies that the task should recur once every interval at most, where the interval is the input param in seconds. By default, you have no control over where in that period the task will execute. This setter is mandatory.
* **Flex:** Specifies how close to the end of the period (set above) the task may execute. With a period of 30 seconds and a flex of 10, the scheduler will execute the task between the 20-30 second range.
* **Persisted:** Determines whether the task should be persisted across reboots. Defaults to true for periodic tasks, and is not supported for one-off tasks. Requires “Receive Boot Completed” permission, or the setter will be ignored.

**
Canceling Tasks**

We already know how to schedule task with GCM Networking, also we need to know how to cancel them.
You can **cancel all** tasks for a given GcmTaskService:

```
#!java

GcmNetworkManager.getInstance(context).cancelAllTasks(MyTaskService.class);

```

And you can also **cancel a specific task** by providing its tag and GcmTaskService:

```
#!java


GcmNetworkManager.getInstance(context).cancelTask(
        CustomService.TAG_TASK_PERIODIC_LOG,
        CustomService.class
);

```

In either case, remember that an in-flight task cannot be canceled.