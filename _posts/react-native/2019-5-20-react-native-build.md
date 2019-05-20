---
layout: post
title: "[RN] React Native Build"
tags: [react-native]
---

```
$ yarn install
$ react-native run-android
```

## debugging app export

```
$ react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle --assets-dest android/app/src/main/res

$ cd android
```

_window_

```
$ gradlew.bat assembleDebug
```

_Mac OS_

```
$ gradlew assembleDebug
```

# production

- 참고 http://wagunblog.com/wp/?p=2371

## create signkey

```
$ cd android/app
$ keytool -genkey -v -keystore kr.wise7034.RealTimeEmergencyDepartmentInfo.keystore -alias kr.wise7034.RealTimeEmergencyDepartmentInfo.remote -keyalg RSA -keysize 2048 -validity 10000 -storepass {YOUR_PASSWORD}
```

## set android properties

```
$ cd android

// open gradle.properties
// add code
MYAPP_RELEASE_STORE_FILE=kr.wise7034.RealTimeEmergencyDepartmentInfo.keystore
MYAPP_RELEASE_KEY_ALIAS=kr.wise7034.RealTimeEmergencyDepartmentInfo.remote
MYAPP_RELEASE_STORE_PASSWORD={keystore password}
MYAPP_RELEASE_KEY_PASSWORD={keystore password}

// open app/build.gradle
android {
    defaultConfig { ... }
    signingConfigs {
        release {
            storeFile file(MYAPP_RELEASE_STORE_FILE)
            storePassword MYAPP_RELEASE_STORE_PASSWORD
            keyAlias MYAPP_RELEASE_KEY_ALIAS
            keyPassword MYAPP_RELEASE_KEY_PASSWORD
        }
    }
    buildTypes {
        release {
            ...
            signingConfig signingConfigs.release
        }
    }
}
```

## build

```
$ react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle
$ cd android
$ ./gradlew app:assembleRelease --stacktrace (or window ./gradlew -> gradlew.bat)
```

## build test

```
$ react-native run-android --variant=release
```

## bug report

```
[drawable-hdpi-v4/node_modules_reactnavigationstack_src_views_assets_backicon] C:\workspace\image-vision-rn\android\app\src\main\res\drawable-hdpi\node_modules_reactnavigationstack_src_views_assets_backicon.png      [drawable-hdpi-v4/node_modules_reactnavigationstack_src_views_assets_backicon] C:\workspace\image-vision-rn\android\app\build\generated\res\react\release\drawable-hdpi\node_modules_reactnavigationstack_src_views_assets_backicon.png: Error: Duplicate resources
[drawable-mdpi-v4/node_modules_reactnativeratings_src_images_airbnbstar] C:\workspace\image-vision-rn\android\app\src\main\res\drawable-mdpi\node_modules_reactnativeratings_src_images_airbnbstar.png  [drawable-mdpi-v4/node_modules_reactnativeratings_src_images_airbnbstar] C:\workspace\image-vision-rn\android\app\build\generated\res\react\release\drawable-mdpi\node_modules_reactnativeratings_src_images_airbnbstar.png: Error: Duplicate resources
```

위와 같은 버그가 난다면,

`android/app/src/main/res/drawable-*` 폴더를 삭제하자

[참고 스택오버플로우](https://github.com/facebook/react-native/issues/22234#issuecomment-468545832)
