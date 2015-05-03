---
layout: post
title:  "Deep Linking in Android"
date:   2015-04-18 18:05:35
---
Apps that are able to natively open external URLs make for a considerably better user experience. Unfortunately, end-to-end implementation in Android does not come without a handful of hurdles to jump. Here we'll drill through all the steps required to create a robust deep linking solution.

## Determine the Entry Point
You need to determine which of your app's Activities will be responsible for handling your app's links. Which Activity is chosen depends on how your Android app is setup. The [launcher][1] Activity is an ideal candidate, so let's take that route.

> **Note:** Your launcher activity will have an `<intent-filter>` with the `android.intent.category.LAUNCHER` category, like this:
> ```xml
> <activity
    android:name=".MainActivity"
    android:label="@string/app_name">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
> ```

If you have multiple Activities in your app, you may choose a different one to handle deep links, but keep in mind that users will then enter your app on a different path compared to the one they would enter on if they open the app via the app drawer or launcher.

By default your `Actvity` will launch a new task to handle the deep link. This results in multiple copies of your activity running at the same time. If you would like the existing activity to handle the link you need to change the launch mode. Setting the launch mode to `android:launchMode="singleTask"` is generally a good option for this. Your activity should now look something like this:

```xml
<activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

[Read more about activity launch modes here][4]

## Install the Intent Filter
In order for links to reach your app, something needs to tell Android that your links belong to your app. This can be done by declaring another `<intent-filter>` on your activity.

An intent filter is configurable with actions, categories, and data. In order for your deep link intent filter to work properly you will need the following:

**Action** `android.intent.action.VIEW` ([javadoc][6])
Action used for intents that handle displaying data to the user.

**Category** `android.intent.category.DEFAULT` ([javadoc][2])
Category allowing implicit intents to resolve to this filter.
> **Note:** In order to receive implicit intents, you must include the CATEGORY_DEFAULT category in the intent filter. The methods startActivity() and startActivityForResult() treat all intents as if they declared the CATEGORY_DEFAULT category. If you do not declare this category in your intent filter, no implicit intents will resolve to your activity.

**Category** `android.intent.category.BROWSABLE` ([javadoc][3]) 
Category allowing intents triggered in web browsers and email clients to resolve to this filter.

**Data** ([javadoc][7])
This is where you describe the *scheme*, *host*, and *path* for your links.
> **Note:** The attributes of the `<data>` element are optional, but linear dependencies require you to approach this a certain way:
> * If a scheme is not specified, the host is ignored.
> * If a host is not specified, the port is ignored.
> * If both the scheme and host are not specified, the path is ignored.

To build a robust deep linking mechanism you can provide a `<data>` element to handle links that appear on the web (Facebook, email, etc) and point to your website. You can also define a `<data>` element to handle links that point to your app. You do this by setting the `android:scheme` to the protocol of your choice. If you set the scheme to http or https, you should also set the `android:host` attribute so that your app will not be offered to handle all http links.

### Filter links pointing to your website
```xml
<data android:scheme="http" android:host="tannerperrien.com" />
```
*Pro:* gives your app the chance to handle a link in an email.
*Con:* the user is prompted with a choice: "do you want to open this link in **your-app** or **Chrome** or **every-other-app-that-handles-urls-like-this**?"

### Filter links pointing to your app
```xml
<data android:scheme="tannerperrien" />
```
*Pro:* the intent will open your app immediately without requiring any user interaction.
*Con:* these types of links are really only useful for situations where you are exposing app entry points to other apps, push notifications, and so on.

### The finished `<Activity>` and `<intent-filter>`
When done, your `<Activity>` might look like this:
```xml
<activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <!-- handle website links -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" android:host="tannerperrien.com" />
    </intent-filter>

    <!-- handle app links -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="tannerperrien" />
    </intent-filter>
</activity>
```

For a deep dive, [read more about intent filters][5]

## Match the URI
At this point you've given some thought as to what your web and app links are going to look like. Now we will look at the component that will be responsible for matching links coming into the app with links your app knows how to handle.

Let's begin by creating a `Link` enumeration that contains the data we need to properly match URIs:

```java
public static enum Link {
    HOME(null),
    PROFILE("profile"),
    PROFILE_OTHER("profile" + PATH_ID),
    SETTINGS("settings");

    final String path;

    private Link(String path) {
        this.path = path;
    }
}
```

Then we can create a `DeepLinker` class that uses a `UriMatcher` to match URIs:

```java
public class DeepLinker {

    private final UriMatcher mUriMatcher;

    public DeepLinker() {
        mUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        
        // Add all links to the matcher
        for (Link link : Link.values()) {
            mUriMatcher.addURI(AUTHORITY, link.path, link.ordinal());
        }
    }

    /**
     * Builds a bundle from the given URI
     */
    public Bundle buildBundle(Uri uri) {
        int code = mUriMatcher.match(uri);

        // Default to home
        Link link = Link.HOME;

        if (code == UriMatcher.NO_MATCH) {
            // Revert code to match default link
            code = link.ordinal();
        } else {
            // Update default link with the matched one
            link = Link.values()[code];
        }

        Bundle bundle = new Bundle();
        bundle.putInt(KEY_LINK, code);

        switch (link) {
            case HOME:
                break;
            case PROFILE:
                break;
            case PROFILE_OTHER:
                // Capture the profile ID
                bundle.putLong(KEY_ID, Long.valueOf(uri.getLastPathSegment()));
                break;
            case SETTINGS:
                break;
        }

        return bundle;
    }
}
```


## Handle the Intent
At this point your links are going to resolve to your app and you know how to parse them. Now it's time to tie it all together and handle the links inside your targeted activity. First off, every activity is launched with an `Intent`. Sometimes an activity may be given a new intent while it is running. This is the case when an activity's launch mode is `singleTop` or `singleTask`. In this case the activity will not be re-created, so the `onNewIntent()` method will be called in order to deliver the new intent.

For the purposes of deep linking, you probably don't care if your app is starting up fresh or resuming from the background. We can easily handle both cases.

### Setting up `onCreate()`
It is your responsibility to figure out exactly where the deep link should be handled inside your activity's `onCreate()`. Typically the end of this method is a good location.

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    handleIntent();
}
```

### Setting up `onNewIntent()`
When you receive a new intent, the easiest thing to do is replace the activity's original intent with the new one. This can be done with a call to `setIntent()`. 

```java
@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);

    // Override previous intent
    setIntent(intent);

    // Handle new intent
    handleIntent();
}
```

### Make a Decision
At this point all activity intents are going to be delivered to a the method `handleIntent()`. Inside this method we need to look at the intent's URI, match it to the appropriate location in the app, and finally send the user to that section of your app.

```java
private void handleIntent() {
    // Get the intent set on this activity
    Intent intent = getIntent();
    
    // Get the uri from the intent
    Uri uri = intent.getData();
    
    // Do not continue if the uri does not exist
    if (uri == null) {
        return;
    }
    
    // Let the deep linker do its job
    Bundle data = mDeepLinker.buildBundle(uri);
    if (data == null) {
        return;
    }

    // See if we have a valid link
    DeepLinker.Link link = DeepLinker.getLinkFromBundle(data);
    if (link == null) {
        return;
    }
    
    // Do something with the link
    switch (link) {
        ...
    }
}
```

## Testing with ADB
Now that everything is in place it's time to test a few links and ensure they work the way you are expecting. A tedious way to test would be to open a web page on your test device and click a link. Thankfully there is a better way: `adb`

Using `adb` you can send intents to your test device the same way a web page might. The command is drafted like this:

    adb shell am start -a android.intent.action.VIEW -d "<YOUR-URI>"

### Examples
A website link can be sent like this:

    adb shell am start -a android.intent.action.VIEW -d "http://tannerperrien.com/settings"

An app link can be sent like this:

    adb shell am start -a android.intent.action.VIEW -d "tannerperrien:///settings"

> **Note:** in the second example the third forward slash is used to start the `path` portion of the URI. Without this, *settings* would be treated as the host name rather than the path. The example `DeepLinker` class provided in this tutorial configures the `UriMatcher` to only look at paths. In other words, it does not give any consideration to the **host** or **authority**, as the `UriMatcher` calls it.

## Sample App
Want to see this tutorial in action? Clone the project here: https://github.com/TannerPerrien/android-deep-link-example

[1]:http://developer.android.com/training/basics/activity-lifecycle/starting.html#launching-activity
[2]:http://developer.android.com/reference/android/content/Intent.html#CATEGORY_DEFAULT
[3]:http://developer.android.com/reference/android/content/Intent.html#CATEGORY_BROWSABLE
[4]:http://developer.android.com/guide/components/tasks-and-back-stack.html#TaskLaunchModes
[5]:http://developer.android.com/guide/components/intents-filters.html
[6]:http://developer.android.com/reference/android/content/Intent.html#ACTION_VIEW
[7]:http://developer.android.com/guide/topics/manifest/data-element.html