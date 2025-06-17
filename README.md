

# 🛎️ Flutter Local Notifications with its implementation

A singleton service class for scheduling and displaying local notifications in Flutter using `flutter_local_notifications`, `timezone`, and `permission_handler` packages.

---

## ✨ Features

- 📱 Foreground notifications
- ⏰ Scheduled daily notifications
- 🔁 Periodic notifications
- ⏳ Custom interval-based repeating notifications
- 🔕 Cancel individual or all notifications
- 🔐 Handles permissions for Android & iOS

---

## 🚀 Getting Started

### 🔧 Dependencies

Add the following dependencies to your `pubspec.yaml`:

```yaml
dependencies:
  flutter_local_notifications: ^16.3.2
  timezone: ^0.9.2
  permission_handler: ^11.3.1
  intl: ^0.19.0
```

### 📜 Permissions
#### ![Android](https://img.shields.io/badge/Android-green?logo=android&logoColor=white&style=for-the-badge) 
Add the following permissions in android/app/src/main/AndroidManifest.xml:

Inside `<manifest>` tag:
```
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" android:maxSdkVersion="32"/>
<uses-permission android:name="android.permission.USE_EXACT_ALARM"/>

```

Inside `<application>` tag:

```
<receiver android:exported="false" android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver"/>
<receiver android:exported="false" android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
  <intent-filter>
    <action android:name="android.intent.action.BOOT_COMPLETED"/>
    <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
    <action android:name="android.intent.action.QUICKBOOT_POWERON"/>
    <action android:name="com.htc.intent.action.QUICKBOOT_POWERON"/>
  </intent-filter>
</receiver>

```

#### ![iOS](https://img.shields.io/badge/iOS-black?logo=apple&logoColor=white&style=for-the-badge)
In your iOS project, open ios/Runner/Info.plist and add:
```
    <key>UIBackgroundModes</key>
    <array>
        <string>remote-notification</string>
    </array>

    <key>NSUserNotificationUsageDescription</key>
    <string>We use notifications to remind you about important events.</string>
```

### 📜 Create a file with name local_notifcaition_service.dart or whatever you want and paste this code

```
import 'dart:io';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:intl/intl.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

class NotificationService {
  static final NotificationService _instance = NotificationService._internal();

  factory NotificationService() {
    return _instance;
  }

  NotificationService._internal();

  final FlutterLocalNotificationsPlugin _notificationsPlugin = FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    await requestPermissions();
    tz.initializeTimeZones();
    final String currentTimeZone = tz.TZDateTime.local(DateTime.now().year).timeZoneName;
    tz.setLocalLocation(tz.getLocation(currentTimeZone));

    const AndroidInitializationSettings initializationSettingsAndroid =
        AndroidInitializationSettings('@mipmap/ic_launcher');
    final DarwinInitializationSettings initializationSettingsIOS = DarwinInitializationSettings(
      requestSoundPermission: false,
      requestBadgePermission: false,
      requestAlertPermission: false,
    );
    final InitializationSettings initializationSettings = InitializationSettings(
      android: initializationSettingsAndroid,
      iOS: initializationSettingsIOS,
    );

    await _notificationsPlugin.initialize(initializationSettings);
  }


  NotificationDetails _notificationDetails() {
    return NotificationDetails(
      android: AndroidNotificationDetails(
        'your_channel_id',
        'your_channel_name',
        channelDescription: 'your_channel_description',
        importance: Importance.max,
        priority: Priority.high,
        showWhen: true,
        playSound: true,
        enableVibration: true,
        ticker: 'ticker',
        visibility: NotificationVisibility.public,
      ),
      iOS: DarwinNotificationDetails(),
    );
  }

  Future<void> requestPermissions() async {
    if (Platform.isIOS) {
      await _notificationsPlugin
          .resolvePlatformSpecificImplementation<IOSFlutterLocalNotificationsPlugin>()
          ?.requestPermissions(alert: true, badge: true, sound: true);
    } else {
      await Permission.notification.isDenied.then((value) {
        if (value) {
          Permission.notification.request();
        }
      });
    }
  }

  Future<void> foregroundNotification({
    required int id,
    required String title,
    required String body,
    String? payload,
  }) async {
    await _notificationsPlugin.show(id, title, body, _notificationDetails(), payload: payload);
  }

  Future<void> scheduleNotification({
    required int id,
    required String title,
    required String body,
    required DateTime selectedTime,
  }) async {
    if (selectedTime.isBefore(DateTime.now())) {
      selectedTime = selectedTime.add(const Duration(days: 1));
    }
    final tz.TZDateTime scheduledTime = tz.TZDateTime.from(selectedTime, tz.local);
    try {
      await _notificationsPlugin.zonedSchedule(
        id,
        title,
        body,
        scheduledTime,
        _notificationDetails(),
        androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
        matchDateTimeComponents: DateTimeComponents.time,
      );

      print('Notification scheduled successfully');
    } catch (e) {
      print('Error scheduling notification: $e');
    }
  }

  Future<void> periodicNotification({
    required int id,
    required String title,
    required String body,
  }) async {
    try {
      await _notificationsPlugin.periodicallyShow(
        id,
        title,
        body,
        RepeatInterval.hourly,
        _notificationDetails(),
        androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      );

      print('Notification scheduled successfully');
    } catch (e) {
      print('Error scheduling notification: $e');
    }
  }

  Future<void> periodicNotificationWithCustomIntervalDuration({
    required int id,
    required String title,
    required String body,
    required Duration intervalDuration,
  }) async {
    try {
      await _notificationsPlugin.periodicallyShowWithDuration(
        id,
        title,
        body,
        intervalDuration,
        _notificationDetails(),
        androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      );

      print('Notification scheduled successfully');
    } catch (e) {
      print('Error scheduling notification: $e');
    }
  }


  Future<void> unsubscribeToAllNotification() async {
    await _notificationsPlugin.cancelAll();
  }

  Future<void> unsubscribeToSpecificNotification({required int id}) async {
    await _notificationsPlugin.cancel(id);
  }
}


```

## ⚙️ Initialization
Call the following in your main.dart file:
```
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NotificationService().init();
  runApp(MyApp());
}
```

## 🧪 Usage
### ✅ Foreground Notification
```
NotificationService().foregroundNotification(
  id: 1,
  title: "Reminder",
  body: "Take your medication now!",
);
```

### 🗓️ Schedule Daily Notification
```
NotificationService().scheduleNotification(
  id: 2,
  title: "Water Reminder",
  body: "Time to drink water!",
  selectedTime: DateTime.now().add(Duration(hours: 1)),
);
```

### 🔁 Periodic Notification (Hourly)
```
NotificationService().periodicNotification(
  id: 3,
  title: "Stretch!",
  body: "Take a quick break and stretch your body.",
);

```

### ⏳ Periodic Notification with Custom Duration
```
NotificationService().periodicNotificationWithIntervalDuration(
  id: 4,
  title: "Posture Check",
  body: "Sit up straight!",
  intervalDuration: Duration(minutes: 30),
);
```

### ❌ Cancel All Notifications
```
NotificationService().unsubscribeToAllNotification();
```

### ❌ Cancel Specific Notification
```
NotificationService().unsubscribeToSpecificNotification(id: 2);
```

## 🛠️ Internal Methods Overview
| Method                                       | Description                                             |
| -------------------------------------------- | ------------------------------------------------------- |
| `init()`                                     | Initializes plugin, requests permissions, sets timezone |
| `requestPermissions()`                       | Requests notification permissions (Android/iOS)         |
| `foregroundNotification()`                   | Triggers an immediate local notification                |
| `scheduleNotification()`                     | Schedules a notification at a specific time             |
| `periodicNotification()`                     | Schedules hourly periodic notification                  |
| `periodicNotificationWithIntervalDuration()` | Repeats notification with a custom interval             |
| `unsubscribeToAllNotification()`             | Cancels all active notifications                        |
| `unsubscribeToSpecificNotification({id})`    | Cancels a notification with a given ID                  |

## 🌐 Timezone Support
The current timezone is fetched and set using:
```
final String currentTimeZone = tz.TZDateTime.local(DateTime.now().year).timeZoneName;
tz.setLocalLocation(tz.getLocation(currentTimeZone));
```

For better accuracy, replace this with:

```
import 'package:flutter_native_timezone/flutter_native_timezone.dart';

final locationName = await FlutterNativeTimezone.getLocalTimezone();
tz.setLocalLocation(tz.getLocation(locationName));
```

#### Add flutter_native_timezone to pubspec.yaml if needed. 

### 📌 Best Practices
- Always initialize NotificationService in main() before using.
- Use unique id for each notification to avoid overwriting.
- Set permissions correctly for Android 13+.

### 🧽 Clean Architecture Tip
Place NotificationService inside your core/services/ or core/notifications/ folder to keep your codebase modular and well-structured.


### Snapshots

![Snapshot1](https://github.com/user-attachments/assets/543e73c9-8e41-4fce-b7e1-9d5bd5d79556)
![Snapshot2](https://github.com/user-attachments/assets/602ad593-ab36-4531-b373-4a15a3f25737)

