---
layout: post
title:  "Service or IntentService"
date:   2016-10-18 10:41:06 +00:00
categories: android
author: Maximiliano Ferraiuolo
comments: true
tags: service, android
image: /assets/article_images/2016-10-18-service-or-IntentService/mac-glasses.jpeg
---


I have faced with many situation where I had to do a long running operation in background where the better solution could be create a service for that, but as you probably should already know, any operation we make in a service class is going to run in our main thread, this can be one of the cases we should take a look to IntentServices.

## This is a quick overview comparing the use of a Service vs an IntentService:

![](https://dl.dropboxusercontent.com/u/37464129/max/Screen%20Shot%202016-10-18%20at%2014.09.06.png)

# Make use of Intent Service

The **IntentService** is an extension of Android’s Service class. As such, Intent Services need to be registered in the application manifest, can be invoked either by your application or (if you allow) can be invoked by other applications (if we set to be exported). Intent Services are also designed specifically to handle background (usually long-running) tasks and the `onHandleIntent` method is already invoked on a background thread for you.

## Let's create our Intent Service

First, you define a class within your application that extends IntentService and defines the onHandleIntent which describes the work to do when this intent is executed:



```java
public class MyTestService extends IntentService {
    // Must create a default constructor
    public MyTestService() {
        // Used to name the worker thread, important only for debugging.
        super("test-service");
    }

    @Override
    public void onCreate() {
        super.onCreate(); // if you override onCreate(), make sure to call super().
        // If a Context object is needed, call getApplicationContext() here.
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        // This describes what will happen when service is triggered
        // Here is where you execute your long task work.

    }
}


```

## Registering the Intent Service


```xml

<application
        android:icon="@drawable/icon"
        android:label="@string/app_name">

        <service
          android:name=".MyTestService"
          android:exported="false"/>
          
<application/>



```

Note: We specify this in the manifest file with the name and `exported` properties set. `exported` determines whether or not the service can be executed by other applications.


## Execute the Intent Service

Now we already have defined the service, let's take a look how to trigger it passing some input data that we would need to run our task.


```java

public class MainActivity extends Activity { 
    // Call `launchTestService()` in the activity
    // to startup the service
    public void launchTestService() {
        // Construct our Intent specifying the Service
        Intent i = new Intent(this, MyTestService.class);
        // Add extras to the bundle
        i.putExtra("iterations", 10);
        // Start the service
        startService(i);
    }
}
```

Once this done, we were able to execute our intent service.. but now, we would like to do something when our intentService finish his work, right?

I had seen diferent options to communicate our service with the UI thread like:

* **ResultReciver**: In many cases, an IntentService only needs to communicate with the activity or application that spawns it. If this is the case, where only the parent application needs to receive data, but as ResultReceiver has some issues that is not the best approach in most of the cases including the fact that if the app quits, then the receiver will not work when the app is relaunched. The receiver also requires each and every activity that wants to receive messages to have a reference to the receiver object passed into the service.

* **BrodcastReceiver**: Probably the most used approach, just because we might want one application to be able to pick up IntentService messages even after it has been fully relaunched or we want multiple applications to be able to receive the messages from the service.

Using a BrodcastReceiver we would have something like the following code:


```java

public class MyTestService extends IntentService {

  public MyTestService() {
    super("test-service");
  }

  @Override
    public void onCreate() {
        super.onCreate(); // if you override onCreate(), make sure to call super().
        // If a Context object is needed, call getApplicationContext() here.
    }

 @Override
    protected void onHandleIntent(Intent intent) {
        int iter = intent.getIntExtra("iterations", 0);
 
        for(int i=1; i<=iter; i++) {
            longTask();
            //This intent is to send the progress
            Intent bcIntent = new Intent();
            bcIntent.setAction(ACTION_PROGRESS);
            bcIntent.putExtra("progress", i*10);
            sendBroadcast(bcIntent);
        }
 
        Intent bcIntent = new Intent();
        bcIntent.setAction(ACTION_END);
        sendBroadcast(bcIntent);
    }

// Simulates a long task with a sleep.
 private void longTask() {
            try {
                Thread.sleep(1000);
            } catch(InterruptedException e) {}
        }

}

```

After that, we create or ProgressReceiver class which is going to extend from BrodcastReceiver:


```java
public class ProgressReceiver extends BroadcastReceiver {
 
    @Override
    public void onReceive(Context context, Intent intent) {
        if(intent.getAction().equals(MiIntentService.ACTION_PROGRESS)) {
            int prog = intent.getIntExtra("progress", 0);
            pbarProgress.setProgress(prog);  // Show a progress bar loading
        }
        else if(intent.getAction().equals(MiIntentService.ACTION_END)) {
            Toast.makeText(MainActivity.this, "Task finished!", Toast.LENGTH_SHORT).show();
        }
    }
}

```

We need to declare the BrodcastReceiver in the AndroidManifest file so that way, our BrodcastReceiver is going to be able to be called even when the app is killed.


```xml
        <receiver android:name="com.brodcastest.MyTestService">

        <intent-filter>
            <action android:name="ACTION_PROGRESS"></action>
            <action android:name="ACTION_END"></action>
        </intent-filter>

    </receiver>

```