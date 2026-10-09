name: Build APK

on:
  workflow_dispatch:
  push:
    branches: [main, master]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.7"

      - name: Generate project
        run: |
          set -e
          mkdir -p quickroute/app/src/main/java/com/quickroute/app
          mkdir -p quickroute/app/src/main/res/xml
          mkdir -p quickroute/app/src/main/res/values
          cd quickroute

          cat > settings.gradle.kts <<'EOF_A'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "QuickRoute"
          include(":app")
          EOF_A

          cat > build.gradle.kts <<'EOF_B'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF_B

          cat > gradle.properties <<'EOF_C'
          org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
          android.nonTransitiveRClass=true
          EOF_C

          cat > app/build.gradle.kts <<'EOF_D'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }

          android {
              namespace = "com.quickroute.app"
              compileSdk = 34

              defaultConfig {
                  applicationId = "com.quickroute.app"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }

              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions {
                  jvmTarget = "17"
              }
          }
          EOF_D

          cat > app/src/main/AndroidManifest.xml <<'EOF_E'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
              <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
              <uses-permission android:name="android.permission.INTERNET" />

              <application
                  android:label="@string/app_name"
                  android:theme="@android:style/Theme.DeviceDefault.Light">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".AddrService"
                      android:exported="false"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifest>
          EOF_E

          cat > app/src/main/res/xml/accessibility_config.xml <<'EOF_F'
          <?xml version="1.0" encoding="utf-8"?>
          <accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
              android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
              android:accessibilityFeedbackType="feedbackGeneric"
              android:accessibilityFlags="flagIncludeNotImportantViews"
              android:canRetrieveWindowContent="true"
              android:notificationTimeout="300"
              android:description="@string/service_desc" />
          EOF_F

          cat > app/src/main/res/values/strings.xml <<'EOF_G'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <string name="app_name">퀵 경로</string>
              <string name="service_desc">인성앱 위치보기 화면의 주소를 읽어 퀵 경로 앱의 목록에 추가합니다.</string>
          </resources>
          EOF_G

          cat > app/src/main/java/com/quickroute/app/Store.kt <<'EOF_H'
          package com.quickroute.app

          import android.content.Context
          import android.content.SharedPreferences

          object Store {
              private const val CITY = "(?:(?:서울|부산|대구|인천|광주|대전|울산|세종|경기|강원|충북|충남|전북|전남|경북|경남|제주)[가-힣]*\\s+)?"
              private const val GU = "(?:[가-힣]{1,6}[시군구]\\s+){1,2}"
              private val ROAD = Regex(CITY + GU + "[가-힣0-9]+(?:대로|로|길)(?:\\s?\\d+번?길)?\\s?\\d+(?:-\\d+)?")
              private val JIBUN = Regex(CITY + GU + "[가-힣0-9]+(?:동|읍|면|리|가)\\s?\\d+(?:-\\d+)?")

              fun prefs(c: Context): SharedPreferences =
                  c.getSharedPreferences("qr", Context.MODE_PRIVATE)

              fun norm(s: String): String = s.replace(Regex("\\s+"), " ").trim()

              fun list(c: Context): MutableList<String> {
                  val raw = prefs(c).getString("stops", "") ?: ""
                  return raw.split("\n").filter { it.isNotBlank() }.toMutableList()
              }

              fun save(c: Context, l: List<String>) {
                  prefs(c).edit().putString("stops", l.joinToString("\n")).apply()
              }

              fun addAll(c: Context, items: List<String>): Boolean {
                  val l = list(c)
                  var changed = false
                  for (i in items) {
                      val n = norm(i)
                      if (n.isNotEmpty() && n !in l) {
                          l.add(n)
                          changed = true
                      }
                  }
                  if (changed) save(c, l)
                  return changed
              }

              fun extract(texts: List<String>): List<String> {
                  val res = LinkedHashSet<String>()
                  for (t in texts) scan(t, res)
                  if (res.isEmpty()) scan(texts.joinToString(" "), res)
                  return res.toList()
              }

              private fun scan(t: String, out: MutableSet<String>) {
                  val r = ROAD.findAll(t).map { norm(it.value) }.toList()
                  if (r.isNotEmpty()) {
                      out.addAll(r)
                  } else {
                      JIBUN.findAll(t).forEach { out.add(norm(it.value)) }
                  }
              }
          }
          EOF_H

          cat > app/src/main/java/com/quickroute/app/Router.kt <<'EOF_I'
          package com.quickroute.app

          data class Pt(val name: String, val lat: Double, val lng: Double)

          object Router {
              fun dist(a: Pt, b: Pt): Double {
                  val r = 6371.0
                  val dLat = Math.toRadians(b.lat - a.lat)
                  val dLng = Math.toRadians(b.lng - a.lng)
                  val x = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                          Math.cos(Math.toRadians(a.lat)) * Math.cos(Math.toRadians(b.lat)) *
                          Math.sin(dLng / 2) * Math.sin(dLng / 2)
                  return 2 * r * Math.asin(Math.sqrt(x))
              }

              // start가 있으면 start에서 출발, 없으면 첫 번째 주소에서 출발 (도착지는 자유)
              fun optimize(start: Pt?, pts: List<Pt>): List<Pt> {
                  if (pts.size <= 1) return pts
                  val nodes = ArrayList<Pt>()
                  val remaining = pts.toMutableList()
                  var cur: Pt
                  if (start != null) {
                      nodes.add(start)
                      cur = start
                  } else {
                      cur = remaining.removeAt(0)
                      nodes.add(cur)
                  }
                  while (remaining.isNotEmpty()) {
                      val from = cur
                      val nx = remaining.minByOrNull { dist(from, it) }!!
                      remaining.remove(nx)
                      nodes.add(nx)
                      cur = nx
                  }
                  var improved = true
                  var guard = 0
                  while (improved && guard < 200) {
                      improved = false
                      guard++
                      for (i in 1 until nodes.size - 1) {
                          for (j in i + 1 until nodes.size) {
                              val a = nodes[i - 1]
                              val b = nodes[i]
                              val c = nodes[j]
                              val d = if (j + 1 < nodes.size) nodes[j + 1] else null
                              val oldLen = dist(a, b) + (if (d != null) dist(c, d) else 0.0)
                              val newLen = dist(a, c) + (if (d != null) dist(b, d) else 0.0)
                              if (newLen + 1e-9 < oldLen) {
                                  nodes.subList(i, j + 1).reverse()
                                  improved = true
                              }
                          }
                      }
                  }
                  return if (start != null) nodes.drop(1) else nodes
              }
          }
          EOF_I

          cat > app/src/main/java/com/quickroute/app/AddrService.kt <<'EOF_J'
          package com.quickroute.app

          import android.accessibilityservice.AccessibilityService
          import android.os.Handler
          import android.os.Looper
          import android.view.accessibility.AccessibilityEvent
          import android.view.accessibility.AccessibilityNodeInfo

          class AddrService : AccessibilityService() {
              private val handler = Handler(Looper.getMainLooper())
              private val scan = Runnable { scanNow() }

              private fun allowed(pkg: String): Boolean {
                  if (pkg == packageName || pkg == "com.android.systemui") return false
                  val p = Store.prefs(this)
                  if (!p.getBoolean("collect", true)) return false
                  val f = (p.getString("filter", "") ?: "").trim()
                  return f.isEmpty() || pkg.contains(f, true)
              }

              override fun onAccessibilityEvent(e: AccessibilityEvent?) {
                  if (e == null) return
                  val pkg = e.packageName?.toString() ?: return
                  if (pkg == packageName || pkg == "com.android.systemui") return
                  Store.prefs(this).edit().putString("last_pkg", pkg).apply()
                  if (!allowed(pkg)) return
                  handler.removeCallbacks(scan)
                  handler.postDelayed(scan, 600)
              }

              private fun scanNow() {
                  val root = rootInActiveWindow ?: return
                  val pkg = root.packageName?.toString() ?: return
                  if (!allowed(pkg)) return
                  val texts = ArrayList<String>()
                  walk(root, texts, 0)
                  val found = Store.extract(texts)
                  if (found.isNotEmpty()) Store.addAll(this, found)
              }

              private fun walk(n: AccessibilityNodeInfo?, out: MutableList<String>, depth: Int) {
                  if (n == null || depth > 40) return
                  val t = n.text
                  if (t != null) out.add(t.toString())
                  val d = n.contentDescription
                  if (d != null) out.add(d.toString())
                  for (i in 0 until n.childCount) {
                      walk(n.getChild(i), out, depth + 1)
                  }
              }

              override fun onInterrupt() {}
          }
          EOF_J

          cat > app/src/main/java/com/quickroute/app/MainActivity.kt <<'EOF_K'
          package com.quickroute.app

          import android.Manifest
          import android.app.Activity
          import android.app.AlertDialog
          import android.content.ClipboardManager
          import android.content.Context
          import android.content.Intent
          import android.content.SharedPreferences
          import android.content.pm.PackageManager
          import android.location.Geocoder
          import android.location.Location
          import android.location.LocationManager
          import android.net.Uri
          import android.os.Bundle
          import android.provider.Settings
          import android.text.Editable
          import android.text.TextWatcher
          import android.view.Gravity
          import android.widget.Button
          import android.widget.EditText
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.Switch
          import android.widget.TextView
          import android.widget.Toast
          import java.util.Locale

          class MainActivity : Activity() {
              private lateinit var status: TextView
              private lateinit var listBox: LinearLayout
              private lateinit var resultBox: LinearLayout
              private lateinit var filterEdit: EditText

              private val prefListener = SharedPreferences.OnSharedPreferenceChangeListener { _, key ->
                  if (key == "stops" || key == "last_pkg") runOnUiThread { refresh() }
              }

              private fun dp(v: Int): Int = (v * resources.displayMetrics.density).toInt()

              private fun toast(s: String) {
                  Toast.makeText(this, s, Toast.LENGTH_SHORT).show()
              }

              private fun button(t: String, f: () -> Unit): Button {
                  val b = Button(this)
                  b.text = t
                  b.setOnClickListener { f() }
                  return b
              }

              private fun label(t: String): TextView {
                  val v = TextView(this)
                  v.text = t
                  v.setPadding(0, dp(14), 0, dp(4))
                  return v
              }

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val p = Store.prefs(this)
                  val sv = ScrollView(this)
                  val col = LinearLayout(this)
                  col.orientation = LinearLayout.VERTICAL
                  col.setPadding(dp(16), dp(32), dp(16), dp(32))
                  sv.addView(col)
                  setContentView(sv)

                  val title = TextView(this)
                  title.text = "퀵 경로"
                  title.textSize = 24f
                  col.addView(title)

                  status = TextView(this)
                  status.setPadding(0, dp(8), 0, dp(8))
                  col.addView(status)

                  col.addView(button("접근성 설정 열기") {
                      startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS))
                  })

                  val sw = Switch(this)
                  sw.text = "주소 자동 수집"
                  sw.isChecked = p.getBoolean("collect", true)
                  sw.setOnCheckedChangeListener { _, c -> p.edit().putBoolean("collect", c).apply() }
                  col.addView(sw)

                  col.addView(label("수집할 앱 패키지명 일부 (비우면 모든 앱)"))
                  filterEdit = EditText(this)
                  filterEdit.setSingleLine()
                  filterEdit.setText(p.getString("filter", "insung"))
                  filterEdit.addTextChangedListener(object : TextWatcher {
                      override fun beforeTextChanged(s: CharSequence?, a: Int, b: Int, c: Int) {}
                      override fun onTextChanged(s: CharSequence?, a: Int, b: Int, c: Int) {}
                      override fun afterTextChanged(s: Editable?) {
                          p.edit().putString("filter", s.toString()).apply()
                      }
                  })
                  col.addView(filterEdit)
                  col.addView(button("마지막 감지 앱을 수집 대상으로 설정") {
                      val last = p.getString("last_pkg", null)
                      if (last != null) {
                          filterEdit.setText(last)
                          refresh()
                      }
                  })

                  col.addView(label("주소 직접 추가"))
                  val addEdit = EditText(this)
                  addEdit.hint = "주소 입력"
                  col.addView(addEdit)
                  val addRow = LinearLayout(this)
                  addRow.orientation = LinearLayout.HORIZONTAL
                  addRow.addView(button("추가") {
                      val t = addEdit.text.toString()
                      if (t.isNotBlank()) {
                          Store.addAll(this, listOf(t))
                          addEdit.setText("")
                      }
                  })
                  addRow.addView(button("붙여넣기") { pasteClip() })
                  addRow.addView(button("전체 삭제") {
                      Store.save(this, emptyList())
                      refresh()
                  })
                  col.addView(addRow)

                  col.addView(label("주소 목록"))
                  listBox = LinearLayout(this)
                  listBox.orientation = LinearLayout.VERTICAL
                  col.addView(listBox)

                  col.addView(button("경로 최적화") { optimize() })
                  resultBox = LinearLayout(this)
                  resultBox.orientation = LinearLayout.VERTICAL
                  col.addView(resultBox)

                  if (checkSelfPermission(Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
                      requestPermissions(
                          arrayOf(
                              Manifest.permission.ACCESS_FINE_LOCATION,
                              Manifest.permission.ACCESS_COARSE_LOCATION
                          ), 1
                      )
                  }
              }

              override fun onResume() {
                  super.onResume()
                  Store.prefs(this).registerOnSharedPreferenceChangeListener(prefListener)
                  refresh()
              }

              override fun onPause() {
                  super.onPause()
                  Store.prefs(this).unregisterOnSharedPreferenceChangeListener(prefListener)
              }

              private fun isServiceOn(): Boolean {
                  val e = Settings.Secure.getString(contentResolver, Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES) ?: ""
                  return e.contains(packageName)
              }

              private fun refresh() {
                  val p = Store.prefs(this)
                  val last = p.getString("last_pkg", null)
                  val f = (p.getString("filter", "") ?: "").trim()
                  var s = "접근성: " + (if (isServiceOn()) "켜짐" else "꺼짐 (위 버튼 → 퀵 경로 켜기)")
                  s += "\n마지막 감지 앱: " + (last ?: "없음")
                  if (last != null && f.isNotEmpty() && !last.contains(f, true)) s += "  (수집 대상 아님)"
                  status.text = s

                  listBox.removeAllViews()
                  val l = Store.list(this)
                  if (l.isEmpty()) {
                      val t = TextView(this)
                      t.text = "아직 주소가 없어요"
                      listBox.addView(t)
                  }
                  for ((i, a) in l.withIndex()) {
                      val row = LinearLayout(this)
                      row.orientation = LinearLayout.HORIZONTAL
                      row.gravity = Gravity.CENTER_VERTICAL
                      val t = TextView(this)
                      t.text = "${i + 1}. $a"
                      row.addView(t, LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f))
                      row.addView(button("삭제") {
                          val cur = Store.list(this)
                          cur.remove(a)
                          Store.save(this, cur)
                          refresh()
                      })
                      listBox.addView(row)
                  }
              }

              private fun pasteClip() {
                  val cm = getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
                  var t = ""
                  if (cm.hasPrimaryClip() && (cm.primaryClip?.itemCount ?: 0) > 0) {
                      t = cm.primaryClip?.getItemAt(0)?.coerceToText(this)?.toString() ?: ""
                  }
                  if (t.isBlank()) {
                      toast("클립보드가 비어 있어요")
                      return
                  }
                  val found = Store.extract(listOf(t))
                  if (found.isNotEmpty()) {
                      Store.addAll(this, found)
                  } else {
                      Store.addAll(this, t.lines().filter { it.isNotBlank() })
                  }
              }

              private fun geocode(a: String): Pair<Double, Double>? {
                  val p = Store.prefs(this)
                  val k = "geo_" + a
                  val cached = p.getString(k, null)
                  if (cached != null) {
                      val s = cached.split(",")
                      return Pair(s[0].toDouble(), s[1].toDouble())
                  }
                  if (!Geocoder.isPresent()) return null
                  return try {
                      @Suppress("DEPRECATION")
                      val r = Geocoder(this, Locale.KOREA).getFromLocationName(a, 1)
                      if (r == null || r.isEmpty()) {
                          null
                      } else {
                          val x = r[0]
                          p.edit().putString(k, "${x.latitude},${x.longitude}").apply()
                          Pair(x.latitude, x.longitude)
                      }
                  } catch (e: Exception) {
                      null
                  }
              }

              private fun currentLocation(): Pair<Double, Double>? {
                  if (checkSelfPermission(Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) return null
                  val lm = getSystemService(Context.LOCATION_SERVICE) as LocationManager
                  var best: Location? = null
                  for (prov in lm.getProviders(true)) {
                      val l: Location? = try {
                          lm.getLastKnownLocation(prov)
                      } catch (e: SecurityException) {
                          null
                      }
                      val b = best
                      if (l != null && (b == null || l.time > b.time)) best = l
                  }
                  val r = best ?: return null
                  return Pair(r.latitude, r.longitude)
              }

              private fun optimize() {
                  resultBox.removeAllViews()
                  val msg = TextView(this)
                  msg.text = "계산 중... (주소를 좌표로 바꾸는 데 시간이 걸려요)"
                  resultBox.addView(msg)
                  val stops = Store.list(this)
                  Thread {
                      val pts = ArrayList<Pt>()
                      val failed = ArrayList<String>()
                      for (a in stops) {
                          val g = geocode(a)
                          if (g != null) pts.add(Pt(a, g.first, g.second)) else failed.add(a)
                      }
                      val cur = currentLocation()
                      val start = if (cur != null) Pt("현재 위치", cur.first, cur.second) else null
                      val ordered = Router.optimize(start, pts)
                      runOnUiThread { showResult(start, ordered, failed) }
                  }.start()
              }

              private fun showResult(start: Pt?, ordered: List<Pt>, failed: List<String>) {
                  resultBox.removeAllViews()
                  val head = TextView(this)
                  head.setPadding(0, dp(12), 0, dp(8))
                  resultBox.addView(head)
                  var prev: Pt? = start
                  var total = 0.0
                  for ((i, pt) in ordered.withIndex()) {
                      val pv = prev
                      val d = if (pv != null) Router.dist(pv, pt) else 0.0
                      total += d
                      prev = pt
                      val ds = "%.1f".format(d)
                      val row = LinearLayout(this)
                      row.orientation = LinearLayout.HORIZONTAL
                      row.gravity = Gravity.CENTER_VERTICAL
                      val t = TextView(this)
                      t.text = "${i + 1}. ${pt.name}" + (if (pv != null) "  (+${ds}km)" else "")
                      row.addView(t, LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f))
                      row.addView(button("길안내") { pickNav(pt) })
                      resultBox.addView(row)
                  }
                  val ts = "%.1f".format(total)
                  head.text = if (ordered.isEmpty()) {
                      "좌표를 찾은 주소가 없어요"
                  } else {
                      "최적 순서 (직선거리 기준 약 ${ts}km)" +
                              (if (start == null) "\n위치 권한이 없어서 첫 번째 주소를 출발지로 했어요" else "")
                  }
                  if (failed.isNotEmpty()) {
                      val f = TextView(this)
                      f.setPadding(0, dp(12), 0, 0)
                      f.text = "좌표를 못 찾은 주소 (순서에서 제외):\n" + failed.joinToString("\n")
                      resultBox.addView(f)
                  }
              }

              private fun pickNav(p: Pt) {
                  AlertDialog.Builder(this)
                      .setTitle("길안내 앱 선택")
                      .setItems(arrayOf("카카오맵", "T맵", "네이버지도")) { _, which -> navigate(p, which) }
                      .show()
              }

              private fun navigate(p: Pt, app: Int) {
                  val name = Uri.encode(p.name)
                  val uri = when (app) {
                      0 -> "kakaomap://route?ep=${p.lat},${p.lng}&by=CAR"
                      1 -> "tmap://route?goalname=$name&goalx=${p.lng}&goaly=${p.lat}"
                      else -> "nmap://route/car?dlat=${p.lat}&dlng=${p.lng}&dname=$name&appname=com.quickroute.app"
                  }
                  try {
                      startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(uri)))
                  } catch (e: Exception) {
                      toast("해당 앱이 설치되어 있지 않아요")
                  }
              }
          }
          EOF_K

      - name: Build debug APK
        working-directory: quickroute
        run: gradle assembleDebug --no-daemon

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: QuickRoute-APK
          path: quickroute/app/build/outputs/apk/debug/app-debug.apk
