# Firebase push notifications

We used to send push notifications directly to APNS or GCM from our code. We were maintaining device_ids, OS types and GCM/APNS certificates. This approach was complicated and hard to maintain.

We knew that an easier solution exist and then we discovered Firebase.

Firebase is a service which does all the hard work for us. There is no need to store certificates on the server side because Firebase handles them for us. Furthermore, we don't need to maintain OS types, because Firebase knows which notification goes to the APNS and which to the GCM. Basically, Firebase handles the routing and delivery of push notifications to targeted devices. Firebase also has a dashboard with the statistics and mobile developers can use this dashboard for testing purposes.

There are many push notification services (like Amazon SNS) which provide us with push notifications, but we are using Firebase because of its simplicity and reliability.

## General
In this chapter we will explain a few approaches for push notifications implementations and theirs PROS and CONS.

Before development, read chapters about [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/), and [app server development](https://firebase.google.com/docs/cloud-messaging/server).

## Development Approaches
There are few approaches for sending push notifications using Firebase which we know of:
  1. sending directly to device_ids
    * storing one device id per user
    * storing multiple device ids per user
  2. sending to topics
    * with persisted topics
    * with non persisted topics
  3. sending to device groups

### Approach based on sending notifications directly to device_ids
  #### Having one device id per user
 * Using this approach means adding a new column `device_id` directly to a User model.
 * On each login this column will be overwritten.
 * On each logout this column should be deleted.

 * PROS
    * Easy to build.

 * CONS
    * User can get notification only on last logged in device.
    * Mobile developers need to be reminded to do a proper logout API call when user logs out. They are known to forget to do this.

 * Usually a user has multiple devices, so this approach is not recommended.

#### Having multiple device ids per user

  * A new table to store user device_ids needs to be implemented.
  * Index on device_id is mandatory

  * PROS
    * User can get push notifications at multiple devices.

  * CONS
    * We need to maintain user device_ids(create, delete).

    #### API sessions controller example
    ```ruby
      class SessionsController < ApiController

        def create
          # login logic goes here
          add_user_device
          # render user
        end

        def destroy
          # logout logic goes here
          device.destroy if device.present?
          # render user
        end

        private

        def add_user_device
          if device.exists?
            device.update(user: current_user)
          else
            current_user.devices.create(device_id: params[:device_id])
          end
        end

        def device
          @device ||= Device.find_by(device_id: params[:device_id])
        end
      end
    ```

  #### Sender service example
  * This example works for both approaches which are using device_ids.
  ```ruby
  module FirebaseCloudMessaging
    class UserNotificationSender

      attr_reader :message, :user_device_ids

      # Firebase works with up to 1000 device_ids per call
      MAX_USER_IDS_PER_CALL = 1000

      def initialize(user_device_ids, message)
        @user_device_ids = user_device_ids
        @message = message
      end

      def call
        user_device_ids.each_slice(MAX_USER_IDS_PER_CALL) do |device_ids|
          fcm_client.send(device_ids, options)
        end
      end

      private

      def options
        {
          priority: 'high',
          data: {
            message: message
          },
          notification: {
            body: message,
            sound: 'default'
          }
        }
      end

      def fcm_client
        @fcm_client ||= FCM.new(Rails.application.secrets.fcm['server_api_key'])
      end
    end
  end
  ```

### Approach based on sending notifications to topics

  * Topic messaging is best suited for content which is often sent to a group of users. E.g. all users subscribed to weather information or some subreddits etc...

  * Based on the publish/subscribe model, FCM topic messaging allows you to send a message to multiple devices that have opted in to a particular topic. You compose topic messages as needed, and FCM handles routing and delivering the message reliably to the right devices.

  * Topic messaging supports unlimited topics and subscriptions for each app.

  * Persisted topics
    * In this approach we need a way to create topic names and expose them through the API.
    * If Firebase doesn't have a topic, it will be created on users first subscription.

    * CONS
      * Topic names are being duplicated in Firebase as well as our backend.

  * Non Persisted topics
    * We can agree on a convention for topic names with mobile developers.
    * Every topic name is parameterized. (Firebase Push Notifications -> firebase-push-notifications)

    * PROS:
      * With this approach we don't need to store topic names in backend.
      * Easy to build.

    * CONS:
      * It doesn't seems that Firebase has limits on topics, but maybe in future they will restrict this like Amazon SNS does.

  * Sender service example
    ```ruby
    module FirebaseCloudMessaging
      class UserNotificationSender
        attr_reader :message, :topic

        def initialize(topic, message)
          @topic = topic
          @message = message
        end

        def call
          fcm_client.send_to_topic(topic, options)
        end

        private

        def options
          {
            priority: 'high',
            data: {
              message: message
            },
            notification: {
              body: message,
              sound: 'default'
            }
          }
        end

        def fcm_client
          @fcm_client ||= FCM.new(Rails.application.secrets.fcm['server_api_key'])
        end
      end
    end
    ```

  * Firebase recently added [support](https://firebase.google.com/docs/cloud-messaging/admin/manage-topic-subscriptions)
  for managing topic subscriptions through server. With this feature, we can have full control over subscriptions at the server if this is required.

### Approach based on sending notifications to device groups
  * With device group messaging, you can send a single message to multiple instances of an app running on devices belonging to a group. Typically, "group" refers a set of different devices that belong to a single user. All devices in a group share a common notification key, which is the token that FCM uses to fan out messages to all devices in the group.

  * If you need to send messages to multiple devices per user, consider device group messaging for those use cases.

  * The maximum number of members allowed for a notification key is 20.
  * We haven't used device groups, but you can read more about this [approach]((https://firebase.google.com/docs/cloud-messaging/android/device-group)).

## Choosing right development approach
Since we are rather new at using Firebase please consult with somebody from the team before choosing a development approach.

## Setup
1. DevOps has to create new Firebase project.

2. Server key from Firebase project settings should be copied to Vault. Server key authorizes your app server on Google services.

3. Add [fcm gem](https://github.com/spacialdb/fcm) to your project. The Fcm gem serves as an SDK for communicating with the [Firebase](https://firebase.google.com/docs/cloud-messaging/send-message).