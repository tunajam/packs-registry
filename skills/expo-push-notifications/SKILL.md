# Expo Push Notifications

Complete guide for implementing push notifications in Expo apps.

## When to Apply

- Setting up push notifications
- Configuring FCM for Android
- Configuring APNs for iOS
- Handling notification interactions
- Scheduling local notifications

## Installation

```bash
npx expo install expo-notifications expo-device expo-constants
```

## Basic Setup

### Request Permissions
```tsx
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';

// Configure notification behavior
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

async function registerForPushNotifications() {
  if (!Device.isDevice) {
    alert('Push notifications require a physical device');
    return null;
  }

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') {
    alert('Failed to get push token');
    return null;
  }

  const projectId = Constants.expoConfig?.extra?.eas?.projectId;
  const token = (await Notifications.getExpoPushTokenAsync({ projectId })).data;
  
  return token;
}
```

### Listen for Notifications
```tsx
import { useEffect, useRef, useState } from 'react';
import * as Notifications from 'expo-notifications';

export function useNotifications() {
  const [expoPushToken, setExpoPushToken] = useState<string | null>(null);
  const notificationListener = useRef<Notifications.Subscription>();
  const responseListener = useRef<Notifications.Subscription>();

  useEffect(() => {
    registerForPushNotifications().then(setExpoPushToken);

    // Notification received while app is foregrounded
    notificationListener.current = Notifications.addNotificationReceivedListener(
      (notification) => {
        console.log('Notification received:', notification);
      }
    );

    // User tapped on notification
    responseListener.current = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const data = response.notification.request.content.data;
        console.log('Notification tapped:', data);
        // Navigate to relevant screen based on data
      }
    );

    return () => {
      if (notificationListener.current) {
        Notifications.removeNotificationSubscription(notificationListener.current);
      }
      if (responseListener.current) {
        Notifications.removeNotificationSubscription(responseListener.current);
      }
    };
  }, []);

  return { expoPushToken };
}
```

## Platform Configuration

### Android (FCM)

1. Create Firebase project at console.firebase.google.com
2. Add Android app with your package name
3. Download `google-services.json`
4. Place in project root (Expo handles placement)

```json
// app.json
{
  "expo": {
    "android": {
      "package": "com.yourcompany.yourapp",
      "googleServicesFile": "./google-services.json"
    }
  }
}
```

### iOS (APNs)

1. Enable Push Notifications capability in Apple Developer
2. Create APNs Key or Certificate
3. Configure in EAS:

```bash
eas credentials
# Select iOS > Production > Push Notifications
# Upload your .p8 key file
```

```json
// app.json
{
  "expo": {
    "ios": {
      "bundleIdentifier": "com.yourcompany.yourapp"
    }
  }
}
```

## Sending Notifications

### Expo Push API (Server-side)
```js
// Node.js server
async function sendPushNotification(expoPushToken, title, body, data = {}) {
  const message = {
    to: expoPushToken,
    sound: 'default',
    title,
    body,
    data,
  };

  const response = await fetch('https://exp.host/--/api/v2/push/send', {
    method: 'POST',
    headers: {
      Accept: 'application/json',
      'Accept-encoding': 'gzip, deflate',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(message),
  });

  return response.json();
}

// Send to multiple tokens
async function sendBatchNotifications(tokens, title, body, data = {}) {
  const messages = tokens.map(token => ({
    to: token,
    sound: 'default',
    title,
    body,
    data,
  }));

  const response = await fetch('https://exp.host/--/api/v2/push/send', {
    method: 'POST',
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(messages),
  });

  return response.json();
}
```

### Using expo-server-sdk
```js
import { Expo } from 'expo-server-sdk';

const expo = new Expo();

async function sendNotifications(pushTokens, title, body, data = {}) {
  const messages = [];
  
  for (const pushToken of pushTokens) {
    if (!Expo.isExpoPushToken(pushToken)) {
      console.error(`Invalid token: ${pushToken}`);
      continue;
    }

    messages.push({
      to: pushToken,
      sound: 'default',
      title,
      body,
      data,
    });
  }

  const chunks = expo.chunkPushNotifications(messages);
  const tickets = [];

  for (const chunk of chunks) {
    const ticketChunk = await expo.sendPushNotificationsAsync(chunk);
    tickets.push(...ticketChunk);
  }

  return tickets;
}
```

## Local Notifications

### Schedule Notification
```tsx
import * as Notifications from 'expo-notifications';

// Schedule immediately
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Reminder',
    body: "Don't forget to check the app!",
    data: { screen: 'Home' },
  },
  trigger: null, // Immediate
});

// Schedule with delay
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Scheduled',
    body: 'This fires in 5 seconds',
  },
  trigger: {
    seconds: 5,
  },
});

// Schedule at specific time
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Daily Reminder',
    body: 'Time to check in!',
  },
  trigger: {
    hour: 9,
    minute: 0,
    repeats: true,
  },
});

// Cancel all scheduled
await Notifications.cancelAllScheduledNotificationsAsync();
```

## Notification Categories (iOS)

```tsx
await Notifications.setNotificationCategoryAsync('message', [
  {
    identifier: 'reply',
    buttonTitle: 'Reply',
    options: {
      opensAppToForeground: true,
    },
    textInput: {
      submitButtonTitle: 'Send',
      placeholder: 'Type your reply...',
    },
  },
  {
    identifier: 'archive',
    buttonTitle: 'Archive',
    options: {
      isDestructive: true,
    },
  },
]);

// Send notification with category
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'New Message',
    body: 'Hey, how are you?',
    categoryIdentifier: 'message',
  },
  trigger: null,
});
```

## Badge Management

```tsx
// Set badge count
await Notifications.setBadgeCountAsync(5);

// Get badge count
const count = await Notifications.getBadgeCountAsync();

// Clear badge
await Notifications.setBadgeCountAsync(0);
```

## Best Practices

1. **Always check Device.isDevice** - Simulators don't support push
2. **Handle permission denial gracefully** - Explain why notifications are useful
3. **Store tokens server-side** - Associate with user accounts
4. **Handle token refresh** - Tokens can change; update server when they do
5. **Use notification categories** - Add actionable buttons
6. **Deep link from notifications** - Use data field to navigate
7. **Test on both platforms** - Behavior differs between iOS and Android
8. **Batch send** - Use chunks for large recipient lists
9. **Handle errors** - Check for invalid tokens and remove them
