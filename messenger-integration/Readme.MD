# Facebook Messenger Integration with Home Assistant

This guide helps you set up a Messenger integration, so that you can let HA send notifications through an existing chat app.

There are other tools I checked out, but had some problems with all of them. You might be in a different situation, so the below are only my own reasons.

- Whatsapp: There is a [whatsapp notifier](https://github.com/t0mer/wapi-custom-notifier) and an addon for this, but it mainly uses a concrete account to send messages from. So if using my own account, I can send messages to myself, but then I won't get any notifications. Create a [business app](https://developers.facebook.com/docs/whatsapp) is not realistic, since it needs approval and that is tailored for medium/large corporates. (Note: a Whatsapp business app might still be possible to be used with all household members being "testers" of the app, but I didn't try that method yet.) You can get around this by setting up an account with another phone number, but I don't have any extra phone numbers for this purpose.
- Telegram: This is a perfectly fine solution with an existing integration, but my other household members don't use it, and I don't want to force them using it.
- Callmebot: This seems a nice easy to use, out of the box solution without the need for any manual setup, but it is using a 3rd party cloud tool (which I prefer not to use), and some people report slowness with it.

Messenger is the common platform used by everyone here at home, so it seems the perfect choice, but I also had trouble setting it up based on the guides available, so I wanted to document the complete procedure.

## Setting up a Page, App and Invite People

### Set up a Facebook "Page"

- Log in to Facebook main page
- On the left side, click Pages
- Click "Create new Page"
- The page and app name should be the same, so enter something like "Home"/"Jarvis"
- Select any category you like (doesn't matter) and click create

There is no need for any special setting on this page, though you might want to customize visibility of things, and restrict others not to post content directly to your page (as it will be "public").

Some extra things to do:

- Send an initial test message to this page (any message) - probably you will get this in your phone right away
- It's nice to upload a profile picture, as this will be shown in the messenger app.

The new page will be available later in the Pages menu, and you can switch between your profile and the "page profile" in the top right menu.

### Set up Developer Account

You need to create a "developer account" (with your normal facebook account) for this step.

- Sign in at the [developer portal](https://developers.facebook.com/async/registration) with your facebook login
- Enter some additional info needed (e.g. phone number)
- Done. No credit card or other stuff is needed.

### Create a Facebook "App"

#### Create App

Once your developer account is set up, let's create the app.

- Go to [my apps](https://developers.facebook.com/apps/)
- Click Create App
- Click Next, pick "other" and click Next, pick "business" and click Next, then give it a name (same as above for the page)
- Finally, click Create App

#### Configure App

This is an empty app, it needs some configuration:

- In the "Add products to your app" section select "Messenger"
- There is no need for webhooks and such
- In the 2nd step, we need to generate an access token: click "Add page", select your page created above
- Finally, Generate a token - this will be your "page token", copy and save this token
- You don't need to complete the app review - that would be quite challenging for this purpose

### Invite People

Now that the page and the app is created, we're almost done. However, since this is not an approved app, it will only be able to send messages to people who are explicitly added as "developers" or "testers" to the app. This shouldn't be trouble for household members. The below you need to do with all the accounts you want to send messages to. It's only a few minutes, and this is the only thing they need to be involved with (or you can quickly do it for them on their phone/laptop - of course with their approval).

- GUEST: Activate a developer account (just as described above)
- GUEST: As that person, open the page with its link, and send an initial message (any message)
- YOU: Go to App roles in your app on the developer portal, click Add People, pick e.g. "tester" and invite the person simply by typing in their name, they will get an email
- GUEST: Accept the invitation by clicking a button in an email

We're almost done, but there is one more step. Sending messages from a page we need a so called PSID of the recipients, which is a page specific ID. We need to retrieve that ID. Some guides use a webhook and some code for it, but we can just simply use the [graph explorer](https://developers.facebook.com/tools/explorer/).

- Open graph explorer, and select your previously created app in the "meta app" dropdown, then select get page token under "user or page" to use your previously created page token
- Use `me` in the URL box on the top, and click Submit -> this will tell you the graph ID for the page
- Type `me/conversations` in the URL box and hit Submit -> this will show all conversations with people who sent a message before to the app, including you, and your other guest users
- For every `t_xxxxxxxxxxxx` formatted id, copy `t_xxxxxxxxxxxx?fields=id,name,participants` in the URL box and hit Submit -> this will show who is behind that conversation, save the `id` of the people (this is the PSID)

## Using in Home Assistant

Once the page and the app is created, people got access to the app and we have their PSID ids, we can send our notifications.

Add the facebook built-in integration to your `configuration.yaml` (not possible to set up from the UI as of now):

```yaml
notify:
  - name: messenger
    platform: facebook
    page_access_token: !secret fb_page_token
```

(In this case, I placed my `fb_page_token` in my secrets.yaml file - this is the page token from above.)

After restarting HA you can try out your new notification service (action). Go to Developer tools, Actions, pick `notify.messenger` from the dropdown, and enter a message and one of the PSID to the target. Alternatively, use this yaml config:

```yaml
action: notify.messenger
data:
  message: Test message
  target: '[PSID]'
```

Hitting Perform action should send the message, which should arrive quickly to the phone of the target person.

Using this as a notification for certain things is not included in this guide, as from here on this service/action can be used as other notify actions. The [HA Messenger integration page](https://www.home-assistant.io/integrations/facebook/) also describes details on how to use the service, how to attach images, etc.

Have fun!

### Simplifying PSID Handling

There is a Facebook Messenger HACS integration, which can be used to associate names to the PSIDs, so that notify action calls (and automations) look nicer. Once the PSID/name pairs are defined, names can be used in actions. I won't go into details here, as the description of the integration is straightforward.

### Sending images

The official way to send an image is by sending an attachment. This is how it SHOULD work (for simplicity, showing as JSON):

```json
 {
    "message": "attachment",
    "target": "26428272146817880",
    "data": {
        "attachment": {
            "type": "image",
            "payload": {
                "url": [url of image]
            }
        }
    }
}
```

There are three problems: 1) this doesn't work with the above new service (with PSID/name support), 2) the API is sassy and will reject images for various reasons (like not having mime type specified properly, or coming from an unknown server, etc.) and 3) serving a local file needs some workaround.

I don't want to present examples for all the cases, just document how I did it, in node-red. The same can be done in an automation (e.g. see [this](https://community.home-assistant.io/t/facebook-messenger-no-longer-able-to-send-local-images/719956)), but I'm using node-red, so here is the code of a function node getting a camera image and sending it via Messenger:

```javascript
const imageResponse = await axios(msg.url, { responseType: 'arraybuffer' });

const form = new FormData();
form.append('recipient', `{ "id": "${msg.recipient}" }`);
form.append('message', '{ "attachment": { "type": "image", "payload": {} } }');
form.append(
  'filedata',
  Buffer.from(imageResponse.data, 'binary'),
  `filename.${msg.extension ?? 'jpg'}`
);

const url = `https://graph.facebook.com/v2.6/me/messages?access_token=${msg.accessToken}`;
try {
  msg.payload = await axios.post(url, form, {
    headers: {
      ...form.getHeaders(),
    },
  });
} catch (e) {
  msg.payload = e;
}

return msg;
```

If you have problems with sending the image, not sending a proper mime type for the file is a typical problem. The mime type can be explicitly defined like this:

```javascript
form.append('filedata', Buffer.from(imageResponse.data, 'binary'), {
  filename: `filename.${msg.extension ?? 'jpg'}`,
  contentType: 'image/jpeg',
});
```

Use a camera or image entity to get a temporary image url for the above code: `msg.[camera/image].attributes.entity_picture`.

Messenger supports these attachment formats: `audio`, `file`, `image`, `template` and `video`.

## Getting a long-lived token

Your token might expire, resulting in errors sending messages/files. There is a way to get a long-lived page token.

- Go to [my apps](https://developers.facebook.com/apps/)
- Open your app, go to Settings, Basic, copy "app id", and "app secret"
- Open [graph explorer](https://developers.facebook.com/tools/explorer/)
- Send this request: `oauth/access_token?grant_type=fb_exchange_token&client_id={app-id}&client_secret={app-secret}&fb_exchange_token={your-access-token}` - you will get a long-lived user token in the response.
- Send this request: `{app-scoped-user-id}/accounts?access_token={long-lived-user-token}` - you will get your long-lived page access token in the response.

To investigate a token's type and validity, use the [access token debugger](https://developers.facebook.com/tools/debug/accesstoken/).
