---
title: gradle-android插件
date: 2018-12-22 11:56:31
tags: [Gradle,Android]
keywords: gradle基础,gradle android,gradle插件,
---

Android Gradle插件可以分为三类，分别对应Android中的三类工程：

* App应用工程，它可以生成一个可运行的apk应用，对应插件id：com.android.application
* Library库工程，可以生成AAR包给其他工程使用，对用插件id:com.android.library
* test测试工程，对App工程或者Library库工程进行单元测试，对应插件id：com.android.test

通过配置不同的插件，配合AS，就可以进行编译测试发布等操作

<!-- more -->

#### 应用Android插件

Android Gradle插件属于第三方插件，托管在Jcenter上，所以在应用之前，我们需要先配置classpath，

``` groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.3'
    }
}
```

我们配置仓库为jcenter，这样当我们配置依赖的时候，Gradle就会去这个仓库里寻找我们的依赖，然后我们在dependencies{}中依赖Android插件。

buildscript{}这部分配置了可以写到根工程的build.gradle脚本文件中，这样所有的自工程就不用重复配置了，之后就可以应用Android Gradle插件了

``` groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 28
    defaultConfig {
        minSdkVersion 19
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
    }
}
```

android{}是Android插件提供的一个扩展类型，可以让我们自定义Android Gradle工程。compileSdkVersion是编译所依赖的Android SDK的版本

#### Android Gradle 工程示例

Android Gradle插件继承于Java插件，具有所有的Java插件的特性。对于一个Android应用 ，常见的build.gradle文件内容如下

``` groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 28
    buildToolsVersion "28.0.3"
    defaultConfig {
        applicationId "com.huangyuanlove.testandroid"
        minSdkVersion 19
        targetSdkVersion 28
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
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0-rc01'
    implementation 'com.android.support.constraint:constraint-layout:1.1.2'
    implementation 'com.android.support:design:28.0.0-rc01'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}

```

Android Gradle工程的配置，都是在android{}中，这是唯一的一个入口。通过它，可以对Android Gradle工程进行自定义的配置，其具体实现是`com.android.build.gradle.AppExtension`，所以很多Android配置可以从这个类里去找，

##### compileSdkVersion

compileSdkVersion 28是配置我们编译Android工程的SDK，这里的23是Android SDK的API Level，该配置的原形是一个compileSdkVersion方法：

``` java
/** @see #getCompileSdkVersion() */
    public void compileSdkVersion(String version) {
        checkWritability();
        this.target = version;
    }

    /** @see #getCompileSdkVersion() */
    public void compileSdkVersion(int apiLevel) {
        compileSdkVersion("android-" + apiLevel);
    }

    public void setCompileSdkVersion(int apiLevel) {
        compileSdkVersion(apiLevel);
    }

    public void setCompileSdkVersion(String target) {
        compileSdkVersion(target);
    }
```

在gradle中，方法的括号可以省略，同时我们也注意到了String类型的重载方法和两个`setCompileSdkVersion`方法，完全可以按照这四种方式来进行配置。

##### buildToolsVersion

`buildToolsVersion "28.0.3"`表示我们使用的Android构建工具的版本。它的原形也是一个方法：

``` java
public void buildToolsVersion(String version) {
        checkWritability();
        //The underlying Revision class has the maven artifact semantic,
        // so 20 is not the same as 20.0. For the build tools revision this
        // is not the desired behavior, so normalize e.g. to 20.0.0.
        buildToolsRevision = Revision.parseRevision(version, Revision.Precision.MICRO);
    }

    /** {@inheritDoc} */
    @Override
    public String getBuildToolsVersion() {
        return buildToolsRevision.toString();
    }

    public void setBuildToolsVersion(String version) {
        buildToolsVersion(version);
    }
```

我们可以通过buildToolsVersion方法赋值，也可以通过android.buildToolsVersion这个属性读写它的值。

##### defaultConfig

defaultConfig是默认的配置，它是一个ProductFlavor。ProductFlavor允许我们根据不同的情况同时生成多个不同的APK包。如果不针对我们自定义的ProductFlavor单独配置的话，会为这个ProductFlavor使用默认的defaultConfig的配置。里面的属性就不再一一解释。
##### buildTypes
buildTypes是一个NamedDomainObjectContainer类型，是一个域对象。我们可以在buildTypes{}里面新增任意多个我们需要构建的类型，Gradle会帮我们自动创建一个对应的BuildType，名字就是我们定义的名字。
* 上面的 release就是一个BuildType。
* minifyEnabled是否为该构建类型启用混淆，false表示不启用，true表示启用
* proguardFiles，当我们启用混淆时，所使用proguard的配置文件，我们可以通过它配置混淆规则。它对应BuildType的proguardFiles方法，接受一个可变参数，所以我们同时可以配置多个配置文件。
`getDefaultProguardFile`是Android扩展的一个方法，它可以读取你的AndroidSDK目录下默认的proguard配置文件。
#### Android Gradle任务
* connectedCheck 在所有连接的设备或者模拟器上运行check检查。

* devicesCheck 通过API连接远程设备运行checks。

* lint在所有的ProductFlavor上运行lint检查。

* install和uninstall类的任务可以直接在我们已连接的设备上安装或者卸载你的app。

* signingReport可以打印App的签名

* AndroidDependencies可以打印Android的依赖
  还有一些其他类似的任务，可以通过./gradlew tasks 来查看

----
以上


