IntegraDoor adaptive AI platform for capability-aware computing


set -e

PROJECT="$HOME/integardoor"
rm -rf "$PROJECT"
mkdir -p "$PROJECT/app/src/main/java/com/integardoor/live"
mkdir -p "$PROJECT/app/src/main/res/drawable"
mkdir -p "$PROJECT/app/src/main/res/mipmap-hdpi"
mkdir -p "$PROJECT/app/src/main/res/mipmap-mdpi"
mkdir -p "$PROJECT/app/src/main/res/mipmap-xhdpi"
mkdir -p "$PROJECT/app/src/main/res/mipmap-xxhdpi"
mkdir -p "$PROJECT/app/src/main/res/mipmap-xxxhdpi"

cd "$PROJECT"

cat > settings.gradle <<'EOF'
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "IntegraDoor"
include(":app")
EOF

cat > build.gradle <<'EOF'
plugins {
    id 'com.android.application' version '8.7.3' apply false
    id 'org.jetbrains.kotlin.android' version '2.0.21' apply false
}
EOF

cat > gradle.properties <<'EOF'
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
EOF

cat > app/build.gradle <<'EOF'
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace 'com.integadoor.live'
    compileSdk 35

    defaultConfig {
        applicationId 'com.integadoor.live'
        minSdk 26
        targetSdk 35
        versionCode 1
        versionName '0.1.0'
    }
}

kotlin {
    jvmToolchain(17)
}
EOF

cat > app/src/main/AndroidManifest.xml <<'EOF'
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:allowBackup="false"
        android:label="IntegraDoor"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">

        <activity
            android:name=".MainActivity"
            android:exported="true">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

    </application>

</manifest>
EOF

mkdir -p app/src/main/res/values

cat > app/src/main/res/values/styles.xml <<'EOF'
<?xml version="1.0" encoding="utf-8"?>
<resources>

    <style name="AppTheme"
        parent="android:style/Theme.Material.Light.NoActionBar">

        <item name="android:fontFamily">sans</item>
        <item name="android:windowLightStatusBar">true</item>
        <item name="android:colorAccent">#000000</item>

    </style>

</resources>
EOF

cat > app/src/main/java/com/integardoor/live/MainActivity.kt <<'EOF'
package com.integadoor.live

import android.app.Activity
import android.os.*
import android.os.Debug
import android.content.Context
import android.graphics.Color
import android.view.Gravity
import android.view.View
import android.widget.*
import java.io.File
import java.text.SimpleDateFormat
import java.util.*
import kotlin.concurrent.thread
import kotlin.math.max
import kotlin.math.min

class MainActivity : Activity() {

    private lateinit var statusText: TextView
    private lateinit var telemetryText: TextView
    private lateinit var modeText: TextView
    private lateinit var startButton: Button
    private lateinit var stopButton: Button

    private var running = false
    private var worker: Thread? = null

    private val handler = Handler(Looper.getMainLooper())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        buildUi()

        startButton.setOnClickListener {
            startMonitor()
        }

        stopButton.setOnClickListener {
            stopMonitor()
        }
    }

    private fun buildUi() {

        val root = LinearLayout(this)
        root.orientation = LinearLayout.VERTICAL
        root.setPadding(32, 32, 32, 32)

        val title = TextView(this)
        title.text = "INTEGRADOOR"
        title.textSize = 30f
        title.setTextColor(Color.BLACK)
        title.gravity = Gravity.CENTER

        val subtitle = TextView(this)
        subtitle.text =
            "Capability-Aware Computing Engine\nLive Test 001"
        subtitle.textSize = 16f
        subtitle.gravity = Gravity.CENTER
        subtitle.setPadding(0, 8, 0, 30)

        statusText = TextView(this)
        statusText.text = "SYSTEM READY"
        statusText.textSize = 20f
        statusText.gravity = Gravity.CENTER
        statusText.setPadding(0, 10, 0, 20)

        telemetryText = TextView(this)
        telemetryText.textSize = 16f
        telemetryText.setTypeface(null, android.graphics.Typeface.MONOSPACE)

        modeText = TextView(this)
        modeText.textSize = 19f
        modeText.setPadding(0, 20, 0, 20)

        startButton = Button(this)
        startButton.text = "START LIVE TEST"

        stopButton = Button(this)
        stopButton.text = "STOP"
        stopButton.isEnabled = false

        root.addView(title)
        root.addView(subtitle)
        root.addView(statusText)
        root.addView(telemetryText)
        root.addView(modeText)
        root.addView(startButton)
        root.addView(stopButton)

        setContentView(root)
    }

    private fun startMonitor() {

        if (running) return

        running = true

        startButton.isEnabled = false
        stopButton.isEnabled = true

        statusText.text = "LIVE MONITOR ACTIVE"

        worker = thread(start = true) {

            while (running) {

                val snapshot = collectTelemetry()

                writeLog(snapshot)

                handler.post {

                    telemetryText.text =
                        formatTelemetry(snapshot)

                    modeText.text =
                        "ADAPTIVE MODE: ${snapshot.mode}"
                }

                Thread.sleep(3000)
            }
        }
    }

    private fun stopMonitor() {

        running = false

        worker?.interrupt()
        worker = null

        startButton.isEnabled = true
        stopButton.isEnabled = false

        statusText.text = "MONITOR STOPPED"
        modeText.text = "ADAPTIVE MODE: IDLE"
    }

    override fun onDestroy() {
        stopMonitor()
        super.onDestroy()
    }

    data class Telemetry(
        val time: String,
        val cpu: Double,
        val ramUsedMb: Long,
        val ramTotalMb: Long,
        val storageFreeGb: Double,
        val battery: Int,
        val temperature: Float,
        val cores: Int,
        val memoryClassMb: Int,
        val mode: String
    )

    private fun collectTelemetry(): Telemetry {

        val time =
            SimpleDateFormat(
                "yyyy-MM-dd HH:mm:ss",
                Locale.US
            ).format(Date())

        val cpu = readCpuUsage()

        val activityManager =
            getSystemService(Context.ACTIVITY_SERVICE)
                    as android.app.ActivityManager

        val memInfo =
            android.app.ActivityManager.MemoryInfo()

        activityManager.getMemoryInfo(memInfo)

        val totalRam =
            memInfo.totalMem / 1024 / 1024

        val availableRam =
            memInfo.availMem / 1024 / 1024

        val usedRam =
            max(0, totalRam - availableRam)

        val storage =
            File(filesDir.absolutePath).freeSpace /
                    1024.0 / 1024.0 / 1024.0

        val batteryManager =
            getSystemService(
                Context.BATTERY_SERVICE
            ) as android.os.BatteryManager

        val battery =
            batteryManager.getIntProperty(
                android.os.BatteryManager.BATTERY_PROPERTY_CAPACITY
            )

        val temperature =
            readBatteryTemperature()

        val cores =
            Runtime.getRuntime().availableProcessors()

        val memoryClass =
            activityManager.memoryClass

        val mode =
            calculateAdaptiveMode(
                cpu,
                usedRam.toDouble() /
                        max(1, totalRam) * 100.0,
                battery,
                temperature
            )

        return Telemetry(
            time,
            cpu,
            usedRam,
            totalRam,
            storage,
            battery,
            temperature,
            cores,
            memoryClass,
            mode
        )
    }

    private fun calculateAdaptiveMode(
        cpu: Double,
        ramPercent: Double,
        battery: Int,
        temperature: Float
    ): String {

        if (temperature >= 43f)
            return "THERMAL_PROTECTION"

        if (battery in 0..14)
            return "POWER_SAVE"

        if (ramPercent >= 90)
            return "MEMORY_PROTECTION"

        if (cpu >= 90)
            return "CPU_PROTECTION"

        if (ramPercent >= 80)
            return "MEMORY_BALANCED"

        if (cpu >= 75)
            return "CPU_BALANCED"

        return "MAX_AVAILABLE"
    }

    private fun formatTelemetry(t: Telemetry): String {

        val ramPercent =
            t.ramUsedMb.toDouble() /
                    max(1, t.ramTotalMb) * 100.0

        return """
TIME
${t.time}

CPU
${"%.1f".format(t.cpu)} %

RAM
${t.ramUsedMb} / ${t.ramTotalMb} MB
${"%.1f".format(ramPercent)} %

STORAGE FREE
${"%.2f".format(t.storageFreeGb)} GB

BATTERY
${t.battery} %

TEMPERATURE
${"%.1f".format(t.temperature)} °C

CPU CORES
${t.cores}

ANDROID MEMORY CLASS
${t.memoryClassMb} MB

--------------------------------

INTEGRADOOR DECISION
${t.mode}
        """.trimIndent()
    }

    private fun writeLog(t: Telemetry) {

        try {

            val logDir =
                File(filesDir, "integardoor_logs")

            if (!logDir.exists())
                logDir.mkdirs()

            val file =
                File(
                    logDir,
                    "live_test_001.jsonl"
                )

            val json = """
{
"time":"${t.time}",
"cpu_percent":${t.cpu},
"ram_used_mb":${t.ramUsedMb},
"ram_total_mb":${t.ramTotalMb},
"storage_free_gb":${t.storageFreeGb},
"battery_percent":${t.battery},
"temperature_c":${t.temperature},
"cpu_cores":${t.cores},
"memory_class_mb":${t.memoryClassMb},
"adaptive_mode":"${t.mode}"
}
""".trimIndent()

            file.appendText(json + "\n")

        } catch (_: Exception) {
        }
    }

    private fun readCpuUsage(): Double {

        return try {

            val first =
                File("/proc/stat")
                    .readLines()
                    .first { it.startsWith("cpu ") }

            val a =
                first.trim()
                    .split(Regex("\\s+"))
                    .drop(1)
                    .map { it.toLong() }

            val idleA =
                a.getOrElse(3) { 0L } +
                        a.getOrElse(4) { 0L }

            val totalA =
                a.sum()

            Thread.sleep(250)

            val second =
                File("/proc/stat")
                    .readLines()
                    .first { it.startsWith("cpu ") }

            val b =
                second.trim()
                    .split(Regex("\\s+"))
                    .drop(1)
                    .map { it.toLong() }

            val idleB =
                b.getOrElse(3) { 0L } +
                        b.getOrElse(4) { 0L }

            val totalB =
                b.sum()

            val totalDelta =
                totalB - totalA

            val idleDelta =
                idleB - idleA

            if (totalDelta <= 0)
                0.0
            else
                ((totalDelta - idleDelta)
                        .toDouble() /
                        totalDelta.toDouble()) * 100.0

        } catch (_: Exception) {

            0.0
        }
    }

    private fun readBatteryTemperature(): Float {

        return try {

            val intent =
                registerReceiver(
                    null,
                    android.content.IntentFilter(
                        android.content.Intent.ACTION_BATTERY_CHANGED
                    )
                )

            val temp =
                intent?.getIntExtra(
                    android.os.BatteryManager.EXTRA_TEMPERATURE,
                    0
                ) ?: 0

            temp / 10f

        } catch (_: Exception) {

            0f
        }
    }
}
EOF

echo
echo "=========================================="
echo "INTEGRADOOR PROJECT CREATED"
echo "=========================================="
echo
echo "Project:"
echo "$PROJECT"
echo
echo "Next:"
echo "cd $PROJECT"
echo "./gradlew assembleDebug"
echo
