# netmerge
#!/bin/bash
# Run this in an empty folder. It recreates the whole NetMerge Android project.
set -e
mkdir -p netmerge && cd netmerge

mkdir -p "$(dirname 'build.gradle.kts')"
cat > 'build.gradle.kts' <<'NETMERGE_EOF'
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "2.0.20" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.20" apply false
}
NETMERGE_EOF

mkdir -p "$(dirname 'gradle.properties')"
cat > 'gradle.properties' <<'NETMERGE_EOF'
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
android.nonTransitiveRClass=true
NETMERGE_EOF

mkdir -p "$(dirname 'settings.gradle.kts')"
cat > 'settings.gradle.kts' <<'NETMERGE_EOF'
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
rootProject.name = "NetMerge"
include(":app")
NETMERGE_EOF

mkdir -p "$(dirname '.github/workflows/build.yml')"
cat > '.github/workflows/build.yml' <<'NETMERGE_EOF'
name: Build APK

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - uses: android-actions/setup-android@v3

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.9

      - name: Build debug APK
        run: gradle assembleDebug --no-daemon

      - uses: actions/upload-artifact@v4
        with:
          name: NetMerge-debug-apk
          path: app/build/outputs/apk/debug/app-debug.apk
NETMERGE_EOF

mkdir -p "$(dirname 'app/build.gradle.kts')"
cat > 'app/build.gradle.kts' <<'NETMERGE_EOF'
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}

android {
    namespace = "com.example.netmerge"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.netmerge"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "0.1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
    buildFeatures {
        compose = true
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2024.09.00")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.activity:activity-compose:1.9.2")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.6")
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
}
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/AndroidManifest.xml')"
cat > 'app/src/main/AndroidManifest.xml' <<'NETMERGE_EOF'
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:allowBackup="true"
        android:label="NetMerge"
        android:supportsRtl="true"
        android:usesCleartextTraffic="true"
        android:theme="@style/Theme.NetMerge">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.NetMerge">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            <!-- Lets "Share" / "Open with" from the browser send links to the app -->
            <intent-filter>
                <action android:name="android.intent.action.SEND" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:mimeType="text/plain" />
            </intent-filter>
        </activity>

        <service
            android:name=".DownloadService"
            android:exported="false"
            android:foregroundServiceType="dataSync" />
    </application>
</manifest>
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/java/com/example/netmerge/DownloadEngine.kt')"
cat > 'app/src/main/java/com/example/netmerge/DownloadEngine.kt' <<'NETMERGE_EOF'
package com.example.netmerge

import android.net.Network
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.async
import kotlinx.coroutines.awaitAll
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.delay
import kotlinx.coroutines.ensureActive
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import okhttp3.Dns
import okhttp3.OkHttpClient
import okhttp3.Request
import java.io.File
import java.io.IOException
import java.io.RandomAccessFile
import java.net.InetAddress
import java.util.concurrent.TimeUnit
import java.util.concurrent.atomic.AtomicLong

/** User-chosen limits for one connection. 0 = unlimited. */
data class NetworkLimit(
    val enabled: Boolean = true,
    val speedCapBytesPerSec: Long = 0L,
    val dailyBudgetBytes: Long = 0L
)

/** One connection a download may use. */
class Lane(
    val transport: Transport,
    val network: Network,
    val limit: NetworkLimit,
    val usedToday: AtomicLong
) {
    val bytes = AtomicLong(0)
    internal val client: OkHttpClient = OkHttpClient.Builder()
        .socketFactory(network.socketFactory)
        .dns(object : Dns {
            override fun lookup(hostname: String): List<InetAddress> =
                network.getAllByName(hostname).toList()
        })
        .connectTimeout(15, TimeUnit.SECONDS)
        .readTimeout(20, TimeUnit.SECONDS)
        .build()

    fun overBudget() = limit.dailyBudgetBytes > 0 && usedToday.get() >= limit.dailyBudgetBytes
}

/**
 * Downloads one file by splitting it into chunks and letting every connection pull the next
 * free chunk. A faster connection naturally finishes more chunks, so the load balances itself.
 */
class DownloadEngine(
    private val url: String,
    private val dest: File,
    private val lanes: List<Lane>,
    private val onProgress: (done: Long, total: Long) -> Unit
) {
    private val chunkSize = 2L * 1024 * 1024
    private val done = AtomicLong(0)

    private class Chunk(val start: Long, val end: Long, var attempts: Int = 0)

    suspend fun run(scope: CoroutineScope) = withContext(Dispatchers.IO) {
        val active = lanes.filter { it.limit.enabled && !it.overBudget() }
        if (active.isEmpty()) throw IOException("No connection available (all off or over budget)")

        val (total, rangeOk) = probe(active.first())
        dest.parentFile?.mkdirs()

        // Server can't do ranges, or size unknown: single stream on the first connection.
        if (total <= 0 || !rangeOk) {
            singleStream(active.first())
            return@withContext
        }

        RandomAccessFile(dest, "rw").use { it.setLength(total) }

        val queue = Channel<Chunk>(Channel.UNLIMITED)
        var pos = 0L
        var pending = 0
        while (pos < total) {
            val end = minOf(pos + chunkSize, total) - 1
            queue.trySend(Chunk(pos, end)); pending++
            pos = end + 1
        }
        val remaining = java.util.concurrent.atomic.AtomicInteger(pending)

        val workers = active.map { lane ->
            async {
                for (chunk in queue) {
                    ensureActive()
                    if (lane.overBudget() || !lane.limit.enabled) {
                        queue.trySend(chunk)       // hand the chunk to another connection
                        if (remaining.get() == 0) break
                        // If every lane is out of budget this would spin; bail after a pause.
                        if (active.all { it.overBudget() }) throw IOException("Data budget reached on all connections")
                        delay(500)
                        continue
                    }
                    try {
                        fetch(lane, chunk, total)
                        if (remaining.decrementAndGet() == 0) queue.close()
                    } catch (e: IOException) {
                        chunk.attempts++
                        if (chunk.attempts >= 4) throw e
                        queue.trySend(chunk)
                        delay(300L * chunk.attempts)
                    }
                }
            }
        }
        workers.awaitAll()
        onProgress(total, total)
    }

    private fun probe(lane: Lane): Pair<Long, Boolean> {
        val req = Request.Builder().url(url).header("Range", "bytes=0-0").build()
        lane.client.newCall(req).execute().use { r ->
            if (!r.isSuccessful) throw IOException("HTTP ${r.code}")
            if (r.code == 206) {
                val total = r.header("Content-Range")?.substringAfter('/')?.toLongOrNull() ?: -1L
                return total to (total > 0)
            }
            return (r.header("Content-Length")?.toLongOrNull() ?: -1L) to false
        }
    }

    private suspend fun singleStream(lane: Lane) {
        val req = Request.Builder().url(url).build()
        lane.client.newCall(req).execute().use { r ->
            if (!r.isSuccessful) throw IOException("HTTP ${r.code}")
            val total = r.body?.contentLength() ?: -1L
            val body = r.body ?: throw IOException("Empty response")
            RandomAccessFile(dest, "rw").use { raf ->
                raf.setLength(0)
                val buf = ByteArray(64 * 1024)
                val src = body.byteStream()
                val limiter = Limiter(lane.limit.speedCapBytesPerSec)
                while (true) {
                    val n = src.read(buf)
                    if (n < 0) break
                    raf.write(buf, 0, n)
                    account(lane, n)
                    onProgress(done.get(), total)
                    limiter.pace(n)
                    if (lane.overBudget()) throw IOException("Data budget reached")
                }
            }
        }
    }

    private suspend fun fetch(lane: Lane, chunk: Chunk, total: Long) {
        val req = Request.Builder().url(url)
            .header("Range", "bytes=${chunk.start}-${chunk.end}")
            .build()
        lane.client.newCall(req).execute().use { r ->
            if (r.code != 206) throw IOException("Server ignored range (HTTP ${r.code})")
            val src = (r.body ?: throw IOException("Empty body")).byteStream()
            val limiter = Limiter(lane.limit.speedCapBytesPerSec)
            val buf = ByteArray(32 * 1024)
            var offset = chunk.start
            RandomAccessFile(dest, "rw").use { raf ->
                while (offset <= chunk.end) {
                    val want = minOf(buf.size.toLong(), chunk.end - offset + 1).toInt()
                    val n = src.read(buf, 0, want)
                    if (n < 0) throw IOException("Connection closed early")
                    raf.seek(offset)
                    raf.write(buf, 0, n)
                    offset += n
                    account(lane, n)
                    onProgress(done.get(), total)
                    limiter.pace(n)
                }
            }
        }
    }

    private fun account(lane: Lane, n: Int) {
        lane.bytes.addAndGet(n.toLong())
        lane.usedToday.addAndGet(n.toLong())
        done.addAndGet(n.toLong())
    }

    /** Simple speed cap: after each read, sleep long enough to stay under the limit. */
    private class Limiter(private val bytesPerSec: Long) {
        private var windowStart = System.nanoTime()
        private var windowBytes = 0L

        suspend fun pace(n: Int) {
            if (bytesPerSec <= 0) return
            windowBytes += n
            val elapsedMs = (System.nanoTime() - windowStart) / 1_000_000
            val allowedMs = windowBytes * 1000 / bytesPerSec
            if (allowedMs > elapsedMs) delay(allowedMs - elapsedMs)
            if (elapsedMs > 1000) { windowStart = System.nanoTime(); windowBytes = 0 }
        }
    }
}
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/java/com/example/netmerge/DownloadRepository.kt')"
cat > 'app/src/main/java/com/example/netmerge/DownloadRepository.kt' <<'NETMERGE_EOF'
package com.example.netmerge

import android.content.Context
import android.os.Environment
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import kotlinx.coroutines.sync.Semaphore
import kotlinx.coroutines.sync.withPermit
import java.io.File
import java.net.URLDecoder
import java.util.concurrent.atomic.AtomicLong

enum class Status { QUEUED, DOWNLOADING, DONE, FAILED, CANCELLED }

data class DownloadItem(
    val id: Long,
    val url: String,
    val name: String,
    val status: Status = Status.QUEUED,
    val done: Long = 0,
    val total: Long = -1,
    val speedBps: Long = 0,
    val perNetworkBytes: Map<Transport, Long> = emptyMap(),
    val error: String? = null
)

/** App-wide singleton so the UI and the foreground service share one queue. */
object DownloadRepository {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val maxParallel = Semaphore(2)
    private var nextId = 1L
    private val jobs = mutableMapOf<Long, Job>()

    private lateinit var appContext: Context
    lateinit var networks: NetworkManager
        private set

    private val _items = MutableStateFlow<List<DownloadItem>>(emptyList())
    val items: StateFlow<List<DownloadItem>> = _items.asStateFlow()

    private val _limits = MutableStateFlow(Transport.values().associateWith { NetworkLimit() })
    val limits: StateFlow<Map<Transport, NetworkLimit>> = _limits.asStateFlow()

    private val usedToday = Transport.values().associateWith { AtomicLong(0) }
    private val _usedTodayView = MutableStateFlow<Map<Transport, Long>>(emptyMap())
    val usedTodayView: StateFlow<Map<Transport, Long>> = _usedTodayView.asStateFlow()

    fun init(context: Context) {
        if (::appContext.isInitialized) return
        appContext = context.applicationContext
        networks = NetworkManager(appContext).also { it.start() }
    }

    fun setLimit(t: Transport, limit: NetworkLimit) = _limits.update { it + (t to limit) }

    fun add(url: String) {
        val clean = url.trim()
        if (!clean.startsWith("http://") && !clean.startsWith("https://")) return
        val id = nextId++
        val item = DownloadItem(id, clean, fileNameFrom(clean))
        _items.update { it + item }
        DownloadService.start(appContext)
        jobs[id] = scope.launch { maxParallel.withPermit { run(item) } }
    }

    fun cancel(id: Long) {
        jobs[id]?.cancel()
        update(id) { it.copy(status = Status.CANCELLED, speedBps = 0) }
    }

    fun remove(id: Long) {
        cancel(id)
        _items.update { l -> l.filterNot { it.id == id } }
    }

    fun hasActive() = _items.value.any { it.status == Status.QUEUED || it.status == Status.DOWNLOADING }

    private suspend fun run(item: DownloadItem) {
        val dir = appContext.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS) ?: appContext.filesDir
        val dest = uniqueFile(dir, item.name)

        val avail = networks.available.value
        val limits = _limits.value
        val lanes = avail.map { (t, net) -> Lane(t, net, limits.getValue(t), usedToday.getValue(t)) }
        if (lanes.isEmpty()) {
            update(item.id) { it.copy(status = Status.FAILED, error = "No network available") }
            return
        }

        update(item.id) { it.copy(status = Status.DOWNLOADING) }
        var lastBytes = 0L
        var lastTime = System.nanoTime()

        val engine = DownloadEngine(item.url, dest, lanes) { done, total ->
            val now = System.nanoTime()
            if (now - lastTime > 500_000_000L) {
                val speed = (done - lastBytes) * 1_000_000_000L / (now - lastTime)
                lastBytes = done; lastTime = now
                update(item.id) {
                    it.copy(
                        done = done, total = total, speedBps = speed,
                        perNetworkBytes = lanes.associate { l -> l.transport to l.bytes.get() }
                    )
                }
                _usedTodayView.value = usedToday.mapValues { it.value.get() }
            }
        }

        try {
            engine.run(scope)
            update(item.id) {
                it.copy(
                    status = Status.DONE, done = it.total.coerceAtLeast(it.done), speedBps = 0,
                    perNetworkBytes = lanes.associate { l -> l.transport to l.bytes.get() }
                )
            }
        } catch (e: kotlinx.coroutines.CancellationException) {
            dest.delete(); throw e
        } catch (e: Exception) {
            update(item.id) { it.copy(status = Status.FAILED, speedBps = 0, error = e.message ?: "Download failed") }
        }
    }

    private fun update(id: Long, f: (DownloadItem) -> DownloadItem) =
        _items.update { list -> list.map { if (it.id == id) f(it) else it } }

    private fun fileNameFrom(url: String): String {
        val raw = url.substringBefore('?').substringBefore('#').substringAfterLast('/')
        val decoded = runCatching { URLDecoder.decode(raw, "UTF-8") }.getOrDefault(raw)
        return decoded.ifBlank { "download_${System.currentTimeMillis()}" }
    }

    private fun uniqueFile(dir: File, name: String): File {
        var f = File(dir, name)
        var i = 1
        val base = name.substringBeforeLast('.', name)
        val ext = name.substringAfterLast('.', "").let { if (it.isEmpty()) "" else ".$it" }
        while (f.exists()) { f = File(dir, "$base ($i)$ext"); i++ }
        return f
    }
}
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/java/com/example/netmerge/DownloadService.kt')"
cat > 'app/src/main/java/com/example/netmerge/DownloadService.kt' <<'NETMERGE_EOF'
package com.example.netmerge

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.pm.ServiceInfo
import android.os.Build
import android.os.IBinder
import androidx.core.app.NotificationCompat
import androidx.core.content.ContextCompat
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.launch

/** Keeps the process alive and shows a notification while downloads are running. */
class DownloadService : Service() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        createChannel()
        startForegroundCompat(build("Starting…"))

        scope.launch {
            DownloadRepository.items.collect { list ->
                val active = list.filter { it.status == Status.DOWNLOADING || it.status == Status.QUEUED }
                if (active.isEmpty()) {
                    stopForeground(STOP_FOREGROUND_REMOVE)
                    stopSelf()
                    return@collect
                }
                val speed = active.sumOf { it.speedBps }
                val text = "${active.size} download(s) · ${formatSpeed(speed)}"
                getSystemService(NotificationManager::class.java).notify(NOTIF_ID, build(text))
            }
        }
        return START_STICKY
    }

    override fun onDestroy() {
        scope.cancel()
        super.onDestroy()
    }

    private fun startForegroundCompat(n: Notification) {
        if (Build.VERSION.SDK_INT >= 29) {
            startForeground(NOTIF_ID, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC)
        } else {
            startForeground(NOTIF_ID, n)
        }
    }

    private fun build(text: String): Notification =
        NotificationCompat.Builder(this, CHANNEL)
            .setSmallIcon(android.R.drawable.stat_sys_download)
            .setContentTitle("NetMerge")
            .setContentText(text)
            .setOngoing(true)
            .setOnlyAlertOnce(true)
            .build()

    private fun createChannel() {
        if (Build.VERSION.SDK_INT >= 26) {
            val ch = NotificationChannel(CHANNEL, "Downloads", NotificationManager.IMPORTANCE_LOW)
            getSystemService(NotificationManager::class.java).createNotificationChannel(ch)
        }
    }

    companion object {
        private const val CHANNEL = "downloads"
        private const val NOTIF_ID = 1

        fun start(context: Context) {
            ContextCompat.startForegroundService(context, Intent(context, DownloadService::class.java))
        }
    }
}
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/java/com/example/netmerge/Format.kt')"
cat > 'app/src/main/java/com/example/netmerge/Format.kt' <<'NETMERGE_EOF'
package com.example.netmerge

import java.util.Locale

fun formatBytes(b: Long): String {
    if (b < 0) return "?"
    val units = arrayOf("B", "KB", "MB", "GB", "TB")
    var v = b.toDouble()
    var i = 0
    while (v >= 1024 && i < units.lastIndex) { v /= 1024; i++ }
    return if (i == 0) "${b} B" else String.format(Locale.US, "%.1f %s", v, units[i])
}

fun formatSpeed(bps: Long): String = formatBytes(bps) + "/s"
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/java/com/example/netmerge/MainActivity.kt')"
cat > 'app/src/main/java/com/example/netmerge/MainActivity.kt' <<'NETMERGE_EOF'
package com.example.netmerge

import android.Manifest
import android.content.Intent
import android.os.Build
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.statusBarsPadding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Slider
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.material3.darkColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

private val Accent = Color(0xFF4ADE80)
private val Bg = Color(0xFF0A0A0B)
private val Surface = Color(0xFF151517)

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        DownloadRepository.init(this)
        val sharedLink = intent?.takeIf { it.action == Intent.ACTION_SEND }
            ?.getStringExtra(Intent.EXTRA_TEXT)

        setContent {
            MaterialTheme(
                colorScheme = darkColorScheme(primary = Accent, background = Bg, surface = Surface)
            ) {
                val notifPermission = rememberLauncherForActivityResult(
                    ActivityResultContracts.RequestPermission()
                ) { }
                LaunchedEffect(Unit) {
                    if (Build.VERSION.SDK_INT >= 33) notifPermission.launch(Manifest.permission.POST_NOTIFICATIONS)
                    sharedLink?.let { DownloadRepository.add(it) }
                }
                HomeScreen()
            }
        }
    }
}

@Composable
fun HomeScreen() {
    val items by DownloadRepository.items.collectAsState()
    val available by DownloadRepository.networks.available.collectAsState()
    val limits by DownloadRepository.limits.collectAsState()
    val usedToday by DownloadRepository.usedTodayView.collectAsState()
    var link by remember { mutableStateOf("") }

    LazyColumn(
        modifier = Modifier.fillMaxSize().background(Bg).statusBarsPadding().padding(horizontal = 16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        item {
            Spacer(Modifier.height(12.dp))
            Text("NetMerge", fontSize = 28.sp, fontWeight = FontWeight.Bold, color = Color.White)
            Text("Combine Wi-Fi and mobile data. Download faster.", color = Color(0xFF9CA3AF))
        }

        item {
            val totalSpeed = items.filter { it.status == Status.DOWNLOADING }.sumOf { it.speedBps }
            Card(colors = CardDefaults.cardColors(containerColor = Surface), shape = RoundedCornerShape(16.dp)) {
                Column(Modifier.padding(16.dp)) {
                    Text("TOTAL SPEED", fontSize = 11.sp, color = Color(0xFF9CA3AF))
                    Text(formatSpeed(totalSpeed), fontSize = 30.sp, fontWeight = FontWeight.Bold, color = Accent)
                }
            }
        }

        item { Text("Connections", fontWeight = FontWeight.SemiBold, color = Color.White) }
        items(Transport.values().toList()) { t ->
            NetworkCard(
                transport = t,
                connected = available.containsKey(t),
                limit = limits.getValue(t),
                usedToday = usedToday[t] ?: 0L,
                onChange = { DownloadRepository.setLimit(t, it) }
            )
        }

        item {
            Spacer(Modifier.height(4.dp))
            Text("Add download", fontWeight = FontWeight.SemiBold, color = Color.White)
            Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                OutlinedTextField(
                    value = link, onValueChange = { link = it },
                    placeholder = { Text("Paste a direct file link") },
                    singleLine = true, modifier = Modifier.weight(1f)
                )
                Button(onClick = { DownloadRepository.add(link); link = "" }, enabled = link.isNotBlank()) {
                    Text("Add")
                }
            }
        }

        item { Text("Queue (${items.size})", fontWeight = FontWeight.SemiBold, color = Color.White) }
        items(items.reversed(), key = { it.id }) { DownloadRow(it) }
        item { Spacer(Modifier.height(24.dp)) }
    }
}

@Composable
fun NetworkCard(
    transport: Transport,
    connected: Boolean,
    limit: NetworkLimit,
    usedToday: Long,
    onChange: (NetworkLimit) -> Unit
) {
    Card(colors = CardDefaults.cardColors(containerColor = Surface), shape = RoundedCornerShape(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            Row(Modifier.fillMaxWidth(), verticalAlignment = Alignment.CenterVertically) {
                Column(Modifier.weight(1f)) {
                    Text(transport.label, color = Color.White, fontWeight = FontWeight.Medium)
                    Text(
                        if (connected) "Connected · ${formatBytes(usedToday)} used today" else "Not connected",
                        fontSize = 12.sp, color = if (connected) Accent else Color(0xFF6B7280)
                    )
                }
                Switch(
                    checked = limit.enabled && connected,
                    enabled = connected,
                    onCheckedChange = { onChange(limit.copy(enabled = it)) }
                )
            }
            if (connected && limit.enabled) {
                val capMb = (limit.speedCapBytesPerSec / 1_000_000f)
                Text(
                    if (capMb <= 0f) "Speed cap: off" else "Speed cap: ${capMb.toInt()} MB/s",
                    fontSize = 12.sp, color = Color(0xFF9CA3AF)
                )
                Slider(
                    value = capMb, valueRange = 0f..20f,
                    onValueChange = { onChange(limit.copy(speedCapBytesPerSec = (it.toInt() * 1_000_000L))) }
                )
                val budgetMb = (limit.dailyBudgetBytes / 1_000_000f)
                Text(
                    if (budgetMb <= 0f) "Daily data budget: unlimited" else "Daily data budget: ${budgetMb.toInt()} MB",
                    fontSize = 12.sp, color = Color(0xFF9CA3AF)
                )
                Slider(
                    value = budgetMb, valueRange = 0f..5000f,
                    onValueChange = { onChange(limit.copy(dailyBudgetBytes = (it.toInt() / 50 * 50) * 1_000_000L)) }
                )
            }
        }
    }
}

@Composable
fun DownloadRow(item: DownloadItem) {
    Card(colors = CardDefaults.cardColors(containerColor = Surface), shape = RoundedCornerShape(16.dp)) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(6.dp)) {
            Text(item.name, color = Color.White, fontWeight = FontWeight.Medium, maxLines = 1)
            if (item.status == Status.DOWNLOADING || item.status == Status.DONE) {
                val p = if (item.total > 0) (item.done.toFloat() / item.total).coerceIn(0f, 1f) else 0f
                LinearProgressIndicator(progress = { p }, modifier = Modifier.fillMaxWidth(), color = Accent)
                Text(
                    "${formatBytes(item.done)} of ${formatBytes(item.total)} · ${formatSpeed(item.speedBps)}",
                    fontSize = 12.sp, color = Color(0xFF9CA3AF)
                )
                if (item.perNetworkBytes.isNotEmpty()) {
                    val sum = item.perNetworkBytes.values.sum().coerceAtLeast(1L)
                    Text(
                        item.perNetworkBytes.entries.joinToString("  ") { (t, b) ->
                            "${t.label} ${(b * 100 / sum)}%"
                        },
                        fontSize = 12.sp, color = Accent
                    )
                }
            } else {
                Text(
                    when (item.status) {
                        Status.QUEUED -> "Queued"
                        Status.FAILED -> "Failed: ${item.error}"
                        Status.CANCELLED -> "Cancelled"
                        else -> ""
                    },
                    fontSize = 12.sp, color = Color(0xFF9CA3AF)
                )
            }
            Row {
                if (item.status == Status.DOWNLOADING || item.status == Status.QUEUED) {
                    TextButton(onClick = { DownloadRepository.cancel(item.id) }) { Text("Cancel") }
                } else {
                    TextButton(onClick = { DownloadRepository.remove(item.id) }) { Text("Remove") }
                }
            }
        }
    }
}
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/java/com/example/netmerge/NetworkManager.kt')"
cat > 'app/src/main/java/com/example/netmerge/NetworkManager.kt' <<'NETMERGE_EOF'
package com.example.netmerge

import android.content.Context
import android.net.ConnectivityManager
import android.net.Network
import android.net.NetworkCapabilities
import android.net.NetworkRequest
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

/** The kinds of connection the app can use side by side. */
enum class Transport(val label: String, val capability: Int) {
    WIFI("Wi-Fi", NetworkCapabilities.TRANSPORT_WIFI),
    CELLULAR("Mobile data", NetworkCapabilities.TRANSPORT_CELLULAR),
    ETHERNET("Ethernet / USB", NetworkCapabilities.TRANSPORT_ETHERNET)
}

/**
 * Asks Android to keep every transport alive at the same time.
 *
 * By default Android only routes app traffic over one network (usually Wi-Fi). Calling
 * requestNetwork() for each transport makes the system bring up mobile data even while
 * Wi-Fi is connected, and gives us a Network object we can bind sockets to.
 */
class NetworkManager(context: Context) {
    private val cm = context.applicationContext
        .getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    private val _available = MutableStateFlow<Map<Transport, Network>>(emptyMap())
    val available: StateFlow<Map<Transport, Network>> = _available.asStateFlow()

    private val callbacks = mutableMapOf<Transport, ConnectivityManager.NetworkCallback>()

    fun start() {
        if (callbacks.isNotEmpty()) return
        for (t in Transport.values()) {
            val request = NetworkRequest.Builder()
                .addTransportType(t.capability)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_VPN)
                .build()
            val cb = object : ConnectivityManager.NetworkCallback() {
                override fun onAvailable(network: Network) {
                    _available.update { it + (t to network) }
                }

                override fun onLost(network: Network) {
                    _available.update { m -> if (m[t] == network) m - t else m }
                }
            }
            callbacks[t] = cb
            try {
                cm.requestNetwork(request, cb)
            } catch (e: Exception) {
                // Some devices refuse a transport they don't have; just skip it.
                callbacks.remove(t)
            }
        }
    }

    fun stop() {
        callbacks.values.forEach { runCatching { cm.unregisterNetworkCallback(it) } }
        callbacks.clear()
        _available.value = emptyMap()
    }
}
NETMERGE_EOF

mkdir -p "$(dirname 'app/src/main/res/values/themes.xml')"
cat > 'app/src/main/res/values/themes.xml' <<'NETMERGE_EOF'
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="Theme.NetMerge" parent="android:Theme.Material.NoActionBar">
        <item name="android:statusBarColor">#0A0A0B</item>
        <item name="android:navigationBarColor">#0A0A0B</item>
        <item name="android:windowBackground">#0A0A0B</item>
    </style>
</resources>
NETMERGE_EOF

echo "Done. Open the netmerge folder as an Android project, sync Gradle, then Run."
