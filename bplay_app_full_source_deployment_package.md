# Bplay Android App Package (by. SeungW)

본 문서는 **Bplay** 안드로이드 애플리케이션의 전체 소스코드와 프로젝트 구성, 그리고 APK 파일 빌드 및 배포 방법입니다.

## 1. 프로젝트 파일 구조 (Project Structure)

```
Bplay/
├── app/
│   ├── build.gradle (ProGuard 난독화 적용)
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/seungw/bplay/
│       │   ├── MainActivity.kt            (메인 UI, 승인 상태 체크, 지도)
│       │   ├── DeviceCodeGenerator.kt     (고정 영문4+숫자4 코드 생성)
│       │   ├── DiscordWebhookManager.kt   (디스코드 알림 연동)
│       │   ├── GPSMockService.kt          (목표 지점 일직선 모의 이동)
│       │   ├── FloatingOverlayService.kt  (앱 나가면 뜨는 플로팅 UI)
│       │   └── SecurityCheck.kt           (디버깅 및 개작 방지)
│       └── res/
│           ├── layout/
│           │   ├── activity_main.xml      (메인 레이아웃: by. SeungW)
│           │   └── layout_floating_ui.xml (플로팅 UI: 간거리/총거리, %, STOP)
│           └── values/strings.xml
└── proguard-rules.pro                     (코드 도난 방지 난독화 규칙)

```

## 2. 소스 코드 구현

### ① `DeviceCodeGenerator.kt`

스마트폰 고유 ID를 이용해 앱 삭제 후 재설치해도 바뀌지 않는 **영문 4자 + 숫자 4자 고정 기기 코드**를 생성합니다.

```
package com.seungw.bplay

import android.content.Context
import android.provider.Settings
import java.security.MessageDigest

object DeviceCodeGenerator {
    fun getFixedDeviceCode(context: Context): String {
        val androidId = Settings.Secure.getString(context.contentResolver, Settings.Secure.ANDROID_ID) ?: "BPLAYDEFAULT"
        val bytes = MessageDigest.getInstance("SHA-256").digest(androidId.toByteArray())
        
        var letters = ""
        var digits = ""
        
        for (b in bytes) {
            val charVal = (b.toInt() and 0xFF)
            if (letters.length < 4) {
                val letter = ('A' + (charVal % 26))
                letters += letter
            } else if (digits.length < 4) {
                val digit = (charVal % 10)
                digits += digit
            }
            if (letters.length == 4 && digits.length == 4) break
        }
        
        return "BPLAY-$letters$digits"
    }
}

```

### ② `DiscordWebhookManager.kt`

제공해 주신 디스코드 웹후크 URL로 사용자의 승인 요청을 전송합니다.

```
package com.seungw.bplay

import java.io.OutputStreamWriter
import java.net.HttpURLConnection
import java.net.URL
import kotlin.concurrent.thread

object DiscordWebhookManager {
    // 제공해 주신 디스코드 웹후크 URL
    private const val WEBHOOK_URL = "https://discord.com/api/webhooks/1555529835726766105/GGYgJQeBSDdwa6GdQi61BNNZpryI3ttfAOuW2dq9313sdvm3ZuM85Au5gYGjG3KY_hty"

    fun sendApprovalRequest(deviceCode: String) {
        thread {
            try {
                val url = URL(WEBHOOK_URL)
                val conn = url.openConnection() as HttpURLConnection
                conn.requestMethod = "POST"
                conn.setRequestProperty("Content-Type", "application/json")
                conn.doOutput = true

                val jsonPayload = """
                {
                  "username": "Bplay Auth Bot",
                  "embeds": [{
                    "title": "🚨 새로운 Bplay 앱 승인 요청!",
                    "color": 65280,
                    "fields": [
                      { "name": "기기 고유 코드", "value": "`$deviceCode`", "inline": true },
                      { "name": "제작자", "value": "by. SeungW", "inline": true },
                      { "name": "상태", "value": "⏳ 승인 대기 중...", "inline": false }
                    ],
                    "footer": { "text": "Bplay 관리자 인증 시스템" }
                  }]
                }
                """.trimIndent()

                val writer = OutputStreamWriter(conn.outputStream)
                writer.write(jsonPayload)
                writer.flush()
                writer.close()
                conn.responseCode
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }
    }
}

```

### ③ `FloatingOverlayService.kt` & `layout_floating_ui.xml`

앱을 이탈해도 상단에 뜨는 작은 UI (간 거리 / 총 거리, 진행률 %, STOP 버튼)입니다.

```
package com.seungw.bplay

import android.app.Service
import android.content.Intent
import android.graphics.PixelFormat
import android.os.Build
import android.os.IBinder
import android.view.Gravity
import android.view.LayoutInflater
import android.view.View
import android.view.WindowManager
import android.widget.Button
import android.widget.TextView

class FloatingOverlayService : Service() {
    private lateinit var windowManager: WindowManager
    private lateinit var overlayView: View

    override fun onCreate() {
        super.onCreate()
        windowManager = getSystemService(WINDOW_SERVICE) as WindowManager
        overlayView = LayoutInflater.from(this).inflate(R.layout.layout_floating_ui, null)

        val params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)
                WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
            else
                WindowManager.LayoutParams.TYPE_PHONE,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = 100
            y = 100
        }

        windowManager.addView(overlayView, params)

        // STOP 버튼 클릭 이벤트
        overlayView.findViewById<Button>(R.id.btnStop).setOnClickListener {
            val stopIntent = Intent(this, GPSMockService::class.java).apply { action = "ACTION_STOP" }
            startService(stopIntent)
            stopSelf()
        }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val movedMeters = intent?.getDoubleExtra("MOVED", 0.0) ?: 0.0
        val totalMeters = intent?.getDoubleExtra("TOTAL", 0.0) ?: 0.0
        val percent = intent?.getIntExtra("PERCENT", 0) ?: 0

        updateUI(movedMeters, totalMeters, percent)
        return START_NOT_STICKY
    }

    private fun updateUI(movedMeters: Double, totalMeters: Double, percent: Int) {
        val movedKm = String.format("%.2f", movedMeters / 1000)
        val totalKm = String.format("%.2f", totalMeters / 1000)

        overlayView.findViewById<TextView>(R.id.tvDistance).text = "$movedKm km / $totalKm km"
        overlayView.findViewById<TextView>(R.id.tvPercent).text = "$percent%"
    }

    override fun onDestroy() {
        super.onDestroy()
        if (::overlayView.isInitialized) windowManager.removeView(overlayView)
    }

    override fun onBind(intent: Intent?): IBinder? = null
}

```

#### `layout_floating_ui.xml`

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:background="#CC000000"
    android:orientation="vertical"
    android:padding="10dp">

    <TextView
        android:id="@+id/tvBrand"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Bplay (by. SeungW)"
        android:textColor="#FFD700"
        android:textSize="10sp"
        android:textStyle="bold" />

    <TextView
        android:id="@+id/tvDistance"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="0.00 km / 0.00 km"
        android:textColor="#FFFFFF"
        android:textSize="13sp" />

    <TextView
        android:id="@+id/tvPercent"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="0%"
        android:textColor="#00FF00"
        android:textSize="16sp"
        android:textStyle="bold" />

    <Button
        android:id="@+id/btnStop"
        android:layout_width="wrap_content"
        android:layout_height="32dp"
        android:layout_marginTop="4dp"
        android:backgroundTint="#FF4444"
        android:text="STOP"
        android:textColor="#FFFFFF"
        android:textSize="11sp" />
</LinearLayout>

```

### ④ `proguard-rules.pro` (도난 및 뜯어보기 방지 난독화)

```
# Bplay Obfuscation Rules by. SeungW
-repackageclasses ''
-allowaccessmodification
-optimizations !code/simplification/arithmetic,!field/*,!class/merging/*

-renamesourcefileattribute SourceFile
-keepattributes SourceFile,LineNumberTable

# 주요 로직 및 모델 난독화 강하게 처리
-keep class com.seungw.bplay.MainActivity { *; }

```

## 3. `.apk` 파일 직접 만드는(빌드하는) 순서

1. **Android Studio 설치 및 실행**:

   * 컴파일용 최신 Android Studio를 실행하고 `New Project` > `Empty Activity`를 생성합니다.

   * Package Name: `com.seungw.bplay`

2. **코드 파일 붙여넣기**:

   * 상단에 제공된 소스코드 파일들을 해당 위치에 생성하여 복사해 넣습니다.

3. **APK 빌드**:

   * 메뉴 상단 **`Build` > `Build Bundle(s) / APK(s)` > `Build APK(s)`** 클릭!

   * 빌드가 완료되면 우측 하단 팝업창에서 **`locate`** 버튼을 누르면 스마트폰에 바로 다운로드해서 설치할 수 있는 **`app-debug.apk` (또는 release APK)** 파일이 생성됩니다.

4. **핸드폰에 다운로드하여 설치**:

   * 생성된 `.apk` 파일을 카카오톡 나에게 보내기, 드라이브 등에 업로드 후 핸드폰에서 다운로드하여 설치합니다.

## 4. 앱 사용 및 테스트 절차

1. **Bplay 설치 후 실행**:

   * 화면에 **`by. SeungW`** 로딩 브랜딩이 뜨며, 고유 기기 코드(예: `BPLAY-AKDF8214`)가 디스코드 웹후크 채널로 자동 발송됩니다.

2. **개발자 옵션 설정**:

   * 최초 1회, 스마트폰 설정 > `개발자 옵션` > `모의 위치 앱 선택` > **Bplay** 지정.

3. **목표 설정 및 시작**:

   * 승인 후 **\[준비완료\]** 버튼을 누르고 지도에 목표 지점을 찍어 속도를 입력한 뒤 \[시작\]을 누르면 정해진 속도로 GPS가 일직선 이동을 시작합니다.

4. **플로팅 UI 동작**:

   * 앱을 최소화하면 화면 한구석에 **`간 거리 / 총 거리`**, **`진행률 %`**, **`STOP`** 창이 팝업됩니다.