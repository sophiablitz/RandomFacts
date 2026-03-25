# RandomFacts — RecyclerView Tutorial

This tutorial walks you through converting the RandomFacts starter app into a scrollable list of facts using a `RecyclerView`. By the end, the app will fetch 10 random facts on startup and display them in a scrollable list.

**Reference project:** `and101-fa25/And101RecyclerDemo`

---

## Overview of Changes

| What you'll add/change | Why |
|---|---|
| RecyclerView dependency in `build.gradle.kts` | Makes RecyclerView available |
| `Fact.kt` data class | Holds a single fact's data |
| `fact_item.xml` layout | Defines how one item looks in the list |
| `FactAdapter.kt` | Connects the list of facts to the RecyclerView |
| `activity_main.xml` | Replace Button + TextView with a RecyclerView |
| `MainActivity.kt` | Fetch 10 facts and wire up the adapter |

---

## Step 1 — Add the RecyclerView Dependency

Open `app/build.gradle.kts`. Inside the `dependencies { }` block, add:

```kotlin
implementation("androidx.recyclerview:recyclerview:1.3.2")
```

It should look something like this:

```kotlin
dependencies {
    implementation("androidx.recyclerview:recyclerview:1.3.2")
    implementation("com.codepath.libraries:asynchttpclient:2.2.0")
    // ... other existing dependencies
}
```

Click **Sync Now** in the banner that appears at the top of the editor.

---

## Step 2 — Create the `Fact` Data Class

Create a new Kotlin file: `app/src/main/java/com/example/randomfacts/Fact.kt`

```kotlin
package com.example.randomfacts

data class Fact(val text: String, val id: String)
```

This data class holds the two fields we care about from the API response: the fact text and its unique ID.

---

## Step 3 — Create the Item Layout

Create a new layout file: `app/src/main/res/layout/fact_item.xml`

This layout defines how a single row in the list looks.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/fact_id"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="#000000"
        android:textColor="#FFFFFF"
        android:textSize="18sp"
        android:padding="4dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent" />

    <TextView
        android:id="@+id/fact_text"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="30sp"
        android:padding="4dp"
        app:layout_constraintTop_toBottomOf="@id/fact_id"
        app:layout_constraintStart_toStartOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

- `fact_id` — displays the fact's ID on a black background
- `fact_text` — displays the fact text in large text below it

---

## Step 4 — Create the `FactAdapter`

Create a new Kotlin file: `app/src/main/java/com/example/randomfacts/FactAdapter.kt`

```kotlin
package com.example.randomfacts

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class FactAdapter(private val factList: List<Fact>) : RecyclerView.Adapter<FactAdapter.ViewHolder>() {

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val factId: TextView = itemView.findViewById(R.id.fact_id)
        val factText: TextView = itemView.findViewById(R.id.fact_text)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.fact_item, parent, false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val fact = factList[position]
        holder.factId.text = fact.id
        holder.factText.text = fact.text
    }

    override fun getItemCount() = factList.size
}
```

**How it works:**
- `ViewHolder` — caches references to the two TextViews so `findViewById` isn't called on every scroll
- `onCreateViewHolder` — inflates `fact_item.xml` once per visible row
- `onBindViewHolder` — fills in the data for a given position in the list
- `getItemCount` — tells the RecyclerView how many items exist

---

## Step 5 — Update `activity_main.xml`

Replace the entire contents of `app/src/main/res/layout/activity_main.xml` with:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/fact_recycler_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

This removes the Button and single-fact TextView, replacing them with a full-screen RecyclerView.

---

## Step 6 — Rewrite `MainActivity.kt`

Replace the entire contents of `app/src/main/java/com/example/randomfacts/MainActivity.kt` with:

```kotlin
package com.example.randomfacts

import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.codepath.asynchttpclient.AsyncHttpClient
import com.codepath.asynchttpclient.callback.JsonHttpResponseHandler
import okhttp3.Headers

class MainActivity : AppCompatActivity() {

    private val factList = mutableListOf<Fact>()
    private lateinit var factRecyclerView: RecyclerView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        factRecyclerView = findViewById(R.id.fact_recycler_view)
        factRecyclerView.layoutManager = LinearLayoutManager(this)
        factRecyclerView.adapter = FactAdapter(factList)

        // Fetch 10 facts when the app starts
        for (i in 1..10) {
            getRandomFactAndAddToList()
        }
    }

    private fun getRandomFactAndAddToList() {
        val client = AsyncHttpClient()
        client.get(
            "https://uselessfacts.jsph.pl/random.json",
            object : JsonHttpResponseHandler() {
                override fun onSuccess(statusCode: Int, headers: Headers, json: JSON) {
                    val text = json.jsonObject.getString("text")
                    val id = json.jsonObject.getString("id")
                    Log.d("BLITZ", "Got fact: $text")
                    factList.add(Fact(text, id))
                    factRecyclerView.adapter = FactAdapter(factList)
                }

                override fun onFailure(
                    statusCode: Int,
                    headers: Headers?,
                    response: String?,
                    throwable: Throwable?
                ) {
                    Log.e("BLITZ", "Failed to fetch fact: $response")
                }
            }
        )
    }
}
```

**Key changes from the starter:**
- `factList` — a `MutableList<Fact>` that grows as API responses arrive
- The RecyclerView is set up in `onCreate` with a `LinearLayoutManager` and `FactAdapter`
- `getRandomFactAndAddToList()` is called 10 times — each successful response adds a `Fact` to the list and refreshes the adapter
- The button click listener is gone; facts load automatically on startup

---

## Step 7 — Run the App

1. Connect a device or start an emulator
2. Click **Run** (the green play button) in Android Studio
3. The app should open and populate with 10 facts as the API responses come in

---

## Troubleshooting

**App crashes immediately**
- Make sure `INTERNET` permission is in `AndroidManifest.xml`:
  ```xml
  <uses-permission android:name="android.permission.INTERNET" />
  ```

**RecyclerView is empty / nothing shows up**
- Check Logcat for `BLITZ` tag to see if API calls are succeeding
- Verify the RecyclerView id in the layout (`fact_recycler_view`) matches `findViewById` in `MainActivity`

**Build error: unresolved reference RecyclerView**
- Make sure you synced Gradle after adding the RecyclerView dependency in Step 1

**Items appear one at a time slowly**
- That's expected — each of the 10 API calls is asynchronous and arrives independently
