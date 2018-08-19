---
title: gradle问题
permalink: gradle-issue
categories:
  - Android
tags:
  - gradle
  - 面试
date: 2017-01-15 22:54:34
---
## gradle 常见脚本

`编译出来的apk重命名`:
```xml
android.applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def outputFile = output.outputFile
        if (outputFile != null && outputFile.name.endsWith('.apk')) {
            def fileName = outputFile.name.replace(".apk", "-${defaultConfig.versionCode}.apk")
            output.outputFile = new File(outputFile.parent, fileName)
        }
    }
}
```

`productFlavors`:多版本编译apk(eg.大陆版/欧美版/台湾版)
```xml
productFlavors {
    generic {
        buildConfigField "boolean", "MOBILE", "false"
        buildConfigField "String", "UPDATE_URL", "/api/xxx"
    }
    china {
        buildConfigField "boolean", "MOBILE", "true"
        buildConfigField "String", "UPDATE_URL", "/api/yyy"
     
    }
    link {
        buildConfigField "boolean", "MOBILE", "false"
        buildConfigField "String", "UPDATE_URL", "/api/zzz"
    }
}
```

`signingConfigs`:
```xml
signingConfigs {
    debug_key {
        keyAlias 'androiddebugkey'
        keyPassword 'android'
        storePassword 'android'
        storeFile file('keys/debug.keystore')

    }
    HMS_release_key {
        keyAlias 'platform'
        keyPassword 'android'
        storePassword 'android'
        storeFile file('keys/release.keystore')
    }
}
```

`buildTypes`: depends: `signingConfigs`
```xml
buildTypes {
    release {
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        signingConfig signingConfigs.release_key
        buildConfigField "boolean", "DEBUG_MODE", "false"
    }
    debug {
        minifyEnabled = false
        signingConfig signingConfigs.debug_key
        buildConfigField "boolean", "DEBUG_MODE", "true"
    }
}
```
## 使用国内repositories镜像

vim /home/XXX/.gradle/init.gradle

```xml
allprojects{
    repositories {
        def ALIYUN_REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public'
        def ALIYUN_JCENTER_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_REPOSITORY_URL."
                    remove repo
                }
                if (url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_JCENTER_URL."
                    remove repo
                }
            }
        }
        maven {
                url ALIYUN_REPOSITORY_URL
            url ALIYUN_JCENTER_URL
        }
    }
}
```

## 项目打包发布

1. Android 项目打包

`gradlew assemble`

2. Desktop/HTML 项目打包

`gradlew dist`

Desktop: 

执行这个命令，将在项目的 `build/libs/` 文件夹下创建一个可运行的 jar 包。这个 jar 包中包含了所有需要的代码和资源，可以脱离整个项目，只要有 JDK 或 JRE 环境即可运行。运行这个 JAR 文件的命令： `java -jar jar-file-name.jar`。

```xml
task dist(type: FatCapsule) {
	applicationClass mainClass
	archiveName project.name + ".jar"
	capsuleManifest {
		systemProperties['log4j.configuration'] = 'log4j.xml'
		jvmArgs = ['-Xms768m', '-Xmx1g']
		minJavaVersion = '1.6.0'
	}
}
```

HTML: 

执行这个命令，将会把整个应用编译为一个静态的 WEB 应用，将在项目的 html/build/dist/ 文件夹下生产 JavaScript、HTML、asset 文件，这些文件组成的静态 WEB 应用可以脱离 Java 环境，部署在任何支持 HTML 和 JavaScript 的 WEB 服务器上，并通过任何支持 WebGL 的浏览器上访问并运行。

注意： 执行打包命令后，在 html/build/dist/ 目录下会生成一个 `WEB-INF` 文件夹，这个文件夹占用了较大的空间，对于一般的 WEB 服务器没有用，可以删掉。

3. library 编译Jar包

`gradlew makeJar`

```xml
task makeJar(type: Copy) {
    delete 'libs/libmedia.jar'
    from('build/intermediates/bundles/default/')
    into('libs/')
    include('classes.jar')
    rename ('classes.jar', 'libmedia.jar')
}
makeJar.dependsOn(build)
```