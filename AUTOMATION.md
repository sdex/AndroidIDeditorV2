# Changing IDs from other apps

Tasker and other automation apps, scripts over `adb`, and your own apps can change an app's ID by
sending Device ID Changer a broadcast. A request has the same options as the edit dialog, and the
answer says whether the change worked.

## Before you start

- Instant Apply must be activated and set up, and the **Instant apply** page must show **Active**.
  Requests apply IDs through it.
- Requests are off by default. To turn them on, open the **⋮** menu on the **Instant apply** page and
  check **Allow other apps to change IDs**. Uncheck it to turn them off again.
- Device ID Changer needs root, as it does for everything else.

While requests are on, any app on the phone can change any app's ID. Leave them off unless you use
them.

## Sending a request

Send a broadcast with:

- Action: `dev.sdex.idv2.action.CHANGE_ID`
- Package: `dev.sdex.idv2`. Android delivers the broadcast only when it names the package.

Only `package_name` is required:

| Extra | Type | Default | What it does |
|---|---|---|---|
| `package_name` | String | required | The app whose ID changes. |
| `id` | String | random | The new ID. Leave it out or empty for a random ID, as **Set Random ID** makes. |
| `instant_apply` | Boolean | `true` | Apply the ID right away, like the **Instant apply** switch. With `false`, the ID takes effect after the next reboot. |
| `clear_app_data` | Boolean | `false` | Clear the app's data after applying the ID, like **Clear the app's data**. Works only with `instant_apply`. |
| `user_id` | Integer | `0` | The Android user whose app changes, for a second user or a work profile. |
| `request_id` | String | none | Any text. It comes back with the answer, so you can match answers to requests. |
| `callback` | PendingIntent | none | Also sends the answer here. See [From your own app](#from-your-own-app). |

Automation apps often send every extra as text, so booleans may also be `"true"` or `"false"` and
`user_id` a number in text, such as `"10"`.

An ID must pass the same checks as in the app: 16 characters (64 for `android`, the system's own
entry), only digits and the letters A to F, and different from the app's current ID.

### From a computer (adb)

```sh
# A random ID, applied now
adb shell am broadcast -a dev.sdex.idv2.action.CHANGE_ID -p dev.sdex.idv2 \
  --es package_name com.example.app

# A given ID, applied now, with the app's data cleared
adb shell am broadcast -a dev.sdex.idv2.action.CHANGE_ID -p dev.sdex.idv2 \
  --es package_name com.example.app --es id 0123456789abcdef --ez clear_app_data true
```

`am` waits for the answer and prints the result code and the status:

```
Broadcast completed: result=-1, data="APPLIED", extras: Bundle[...]
```

The same commands work in a root shell on the phone: `su -c 'am broadcast ...'`.

### From Tasker

Device ID Changer is a Tasker plugin, so Tasker doesn't need an intent or extras:

1. In a task, add an action and choose **Plugin** → **Device ID Changer** → **Change an app's ID**.
2. Tap the pencil next to **Configuration**. If the screen says **Allow other apps to change IDs**,
   tap **Allow**.
3. Pick the app with the list button, or type its package name. Leave **New ID** empty for a new
   random ID on every run, or type an ID. Both fields also take Tasker variables, such as `%pkg`
   or `%myid`.
4. Set **Instant apply** and **Clear the app's data** as in the edit dialog, and tap **Save**.

Tasker waits for the change to finish, up to the action's timeout, and then sets these variables
for the following actions:

| Variable | Value |
|---|---|
| `%new_id` | The ID that was saved, including a random one. Empty when nothing was saved. |
| `%id_status` | `APPLIED`, `SAVED_REBOOT_REQUIRED`, `APPLY_FAILED` or `INVALID_REQUEST`, as in [The answer](#the-answer). |
| `%id_error` | The `error` code from [The answer](#the-answer), such as `REQUESTS_DISABLED`. Empty on success. |
| `%errmsg` | What went wrong, in words, when the action failed. |

The action fails with `APPLY_FAILED` and `INVALID_REQUEST`. Tasker then stops the task and shows
`%errmsg`, unless you check **Continue Task After Error** in the action.

MacroDroid and Automate can use the same plugin, through their action for Tasker and Locale
plugins.

### With a send-intent action

Without the plugin, use the automation app's send-intent or send-broadcast action with:

- Action: `dev.sdex.idv2.action.CHANGE_ID`
- Package: `dev.sdex.idv2`
- Target: broadcast receiver
- Extras: `package_name` and any others from the table. In Tasker's **Send Intent**, write each
  one as `name:value`, for example `package_name:com.example.app`.

### From your own app

```kotlin
val intent = Intent("dev.sdex.idv2.action.CHANGE_ID")
    .setPackage("dev.sdex.idv2")
    .putExtra("package_name", "com.example.app")
context.sendOrderedBroadcast(intent, null, object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val status = resultData                               // "APPLIED"
        val error = getResultExtras(true).getString("error")  // null on success
        val newId = getResultExtras(true).getString("id")
    }
}, null, Activity.RESULT_CANCELED, null, null)
```

On Android 11 and later, your app can send the broadcast only if it can see Device ID Changer. Add
this to your manifest, unless your app holds `QUERY_ALL_PACKAGES` or targets Android 10 or lower:

```xml
<queries>
    <intent>
        <action android:name="dev.sdex.idv2.action.CHANGE_ID" />
    </intent>
</queries>
```

If your app can't send an ordered broadcast, pass a `callback` PendingIntent. Device ID Changer
sends it with the answer's extras. Create it with `FLAG_MUTABLE` around an explicit intent for a
receiver or service in your app, because an immutable PendingIntent drops the extras.

Don't send the request with `FLAG_RECEIVER_FOREGROUND`. Android gives a foreground broadcast only
10 seconds, and a request can take longer.

## The answer

- Result code: `-1` for `APPLIED` and `SAVED_REBOOT_REQUIRED`, `1` for `APPLY_FAILED`, and `2` for
  `INVALID_REQUEST`. If the request never reaches Device ID Changer, the code stays at the value you
  sent it with.
- Result data: the status.
- Extras: `status`, `reboot_required`, `error` when something went wrong, and `id` with the ID that
  was saved, including a random one. Your `request_id` comes back too, and so do `package_name` and
  `user_id` when the request could be read.

| Status | ID saved | Meaning |
|---|---|---|
| `APPLIED` | yes | The app now runs with its new ID. |
| `SAVED_REBOOT_REQUIRED` | yes | `instant_apply` was `false`. The ID takes effect after a reboot. |
| `APPLY_FAILED` | yes | Applying the ID failed, see `error`. A reboot applies it. |
| `INVALID_REQUEST` | no | Nothing changed, see `error`. |

| `error` | What to do |
|---|---|
| `REQUESTS_DISABLED` | Check **Allow other apps to change IDs** in the **⋮** menu on the **Instant apply** page. |
| `INVALID_INPUT` | Check the request. `package_name` must be there, and each extra must have the type shown in the table above. The ID must be 16 characters (64 for `android`) of digits and the letters A to F, and differ from the current one. The app must already have an ID, so open it once and try again. `user_id` must be a user on the phone. |
| `ANDROID_NOT_SUPPORTED` | Instant Apply doesn't support the Android version on this phone. |
| `PLUGIN_UNAVAILABLE` | Instant Apply can't run right now. Open Device ID Changer with an internet connection and go to the **Instant apply** page. Check that it's activated and shows **Active**, and reboot if it asks you to. Also check that the app has superuser access. |
| `INTERNAL_ERROR` | Try again. If it keeps failing, tap **Contact us** in the app to send a report. With `APPLY_FAILED`, the ID is already saved and a reboot applies it. |

## Good to know

- Requests run one at a time, and never while the app is saving an ID. A request usually takes a few
  seconds, and the answer arrives within a minute.
- If Device ID Changer is open, its list shows the new ID right away.
- Some apps keep their own copy of the ID. For those, add `clear_app_data`, as you would turn on
  **Clear the app's data** in the app.
