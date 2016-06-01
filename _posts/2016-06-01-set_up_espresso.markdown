---
layout: post
title:  "[Android] Espresso"
date:   2016-06-01 22:23:00 +0800
categories: Android
---

[Espress][espresso] is the official Android UI Automation solution.
Why does it call Espresso? Because Google wants the Android Developers finish writing the test cases easily then take their time to enjoy the espresso, in the meanwhile, watching the test cases running.
There is a [YouTube video][youtube_video] introducing the Espresso.

## Set up the Espresso

### 1. Create a project in the Android Studio

### 2. Modify the `build.gradle` of the App

#### 2.1 Add testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner" in the defaultConfig;

#### 2.2 Add packagingOptions to avoid Liscens conflict;

#### 2.3 Add the Espresso related dependencies;

You may encount some conflicts when building with Espresso dependencies. What you need to do is to use the `exclude group` to exclude the duplicate ones.
{% highlight gradle %}
apply plugin: 'com.android.application'

android {
    compileSdkVersion 22
    buildToolsVersion "22.0.1"

    defaultConfig {
        applicationId "com.example.yezhenrong.myapplication"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    packagingOptions {
        exclude 'LICENSE.txt'
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:22.1.1'

    // Espresso related dependencies
    compile 'com.android.support:support-annotations:22.1.1'
    androidTestCompile 'com.android.support:support-annotations:22.1.1'
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.1'){
        exclude group: 'javax.inject'
    }
    androidTestCompile 'com.android.support.test.espresso:espresso-intents:2.1'
    androidTestCompile 'com.android.support.test.espresso:espresso-contrib:2.1'
    androidTestCompile 'com.android.support.test:runner:0.2'
}
{% endhighlight %}

### 3. Add Espresso TestRunner

#### 3.1 Open the Edit Configurations in the Run menu;

#### 3.2 Press the `+` and select the `Android Tests`;
![Alt text](https://raw.githubusercontent.com/wingyippp/blog/gh-pages/images/edit_configuration.png)

#### 3.3 Change the name of the Configuration, select the App Module and fill in the Runner "android.support.test.runner.AndroidJUnitRunner". Press "OK";
![Alt text](https://raw.githubusercontent.com/wingyippp/blog/gh-pages/images/add_test_runner.jpg)

### 4 Create a test case
![Alt text](https://raw.githubusercontent.com/wingyippp/blog/gh-pages/images/create_test_case.jpg)

### 5 Write the test case

#### 5.1 Create a @Rule, `ActivityTestRule` to indicate your activity to be tested;

#### 5.2 Write your test method with the @Test annotation; You can also add the `setup()` and `tearDown()` methods;

#### 5.3 Implments the test method;
onView(withText("Hello world!")).check(ViewAssertions.matches(isDisplayed()));
This line of code is checking whether the "hello world!" text is displayed or not.
Espresso provides us a lot of UI Automation testing APIs. We can check them in the [official website][espresso].
{% highlight java%}
import static android.support.test.espresso.Espresso.onView;
import static android.support.test.espresso.matcher.ViewMatchers.isDisplayed;
import static android.support.test.espresso.matcher.ViewMatchers.withText;

@RunWith(AndroidJUnit4.class)
public class MainActivityTest {
    @Rule
    public ActivityTestRule mActivityRule = new ActivityTestRule(MainActivity.class);

    @Test
    public void testTextViewDisplay() {
        onView(withText("Hello world!")).check(ViewAssertions.matches(isDisplayed()));
    }
}
{% endhighlight %}

### 6 Run the test case and get the result
![Alt text](https://raw.githubusercontent.com/wingyippp/blog/gh-pages/images/test_result.png)

[espresso]: https://google.github.io/android-testing-support-library/docs/espresso/index.html
[youtube_video]: https://www.youtube.com/watch?v=TGU0B4qRlHY