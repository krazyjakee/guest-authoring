---
title: 'Passwordless Authentication in Directus Flows with Plivo'
description: 'Set up passwordless authentication using phone SMS codes in Directus with Plivo API.'
author:
  name: 'Jacob Cattrall'
  avatar_file_name: 'avatar.png'
---

In this tutorial, we will be setting up Passwordless Authentication using Directus Automate and the [Plivo Verify API](https://www.plivo.com/verify/).

This will allow users to sign in to your Directus project by sending a one-time password (OTP) to their phone number, and then validating that it is correct before returning a token for further requests.

This solution can be used once a user already exists in your project and has a unique, correctly-formatted, phone number. Formatting numbers can be hard and the [Plivo Lookup API](https://www.plivo.com/lookup/) can be used to validate a phone number.

## Before You Start

You will need...

- A Directus project - follow our [quickstart guide](https://docs.directus.io/getting-started/quickstart) if you don't already have one.
- A [Plivo account](https://console.plivo.com/accounts/request-trial/).
- A phone number that you can use to test receiving SMS messages (such as your own mobile phone number).
- The `directus_users` table should have a `phone` field that is a `String`.
- The `directus_users` table should have a `otp_session_uuid` field that is a `String`.
- A Directus user account with a valid mobile phone number added to the `phone` field.

## Setting Up Plivo

In the [Plivo Verify Overview](https://console.plivo.com/verify/overview/), take note of your Auth ID and Auth Token. Create a new application and also take note of it's UUID.

Plivo uses BasicAuth to authenticate requests, but Directus Automate does not support this natively. Fortunately, you can use BasicAuth through headers by encoding the username and password as a Base64 string. 

The encoded value must be in the format `auth_id:auth_token`. You can use [this web tool to encode the string](https://www.base64encode.org/). Take note of the string.

## The Login Flow

Using Directus Flows, we will accept a phone number and country code from the user, clean up the number, create a Plivo session saving session ID against the Directus user account. We will then return the session ID to the user ready for the verification flow.

Create a new Flow from your project settings with a Webhook trigger and caching disabled. Your application will make a request to this URL when starting a login.

### Number Cleanup

Numbers must be formatted in E.164 format to be accepted by Plivo. That means a format such as `+447123456789` (a `+`, a country code, and a subscriber number). 

Create a **Run Script** operation with the following code:

```javascript
module.exports = async function(data) {
  const countryCode = `+${data.$trigger.query.country_code}`;
  const phoneNumber = data.$trigger.query.phone_number;

  let fullPhoneNumber = phoneNumber.replace(/[^\d+]/g, ''); // remove all non-numeric characters except "+"
  
  if (phoneNumber.startsWith('00')) {
    fullPhoneNumber = phoneNumber.replace(/^00/g, '+')
  } else if (!phoneNumber.startsWith('+')) {
    fullPhoneNumber = `${countryCode}${phoneNumber.replace(/^[0-9]/g, '')}`;
  }
  
  return {
    phone_number: fullPhoneNumber
  };
}
```

Save the flow, open your browser and navigate to your Trigger URL appended with `?phone_number=07123456789&country_code=44` to the end. This will trigger the flow, clean up the phone number and return the following JSON: `{"phone_number":"+447123456789"}`

If this is your response, then the Number Cleanup operation works.

### Creating the Plivo OTP Session

Plivo's Verify API has two stages - creating a session will send the user a OTP and return a Session UUID. To verify the OTP later, a Session UUID and OTP must be sent to Plivo who will validate whether it was correct.

To create a session, create a **Webhook / Request URL** operation with the following options:

- Set the method to POST
- In the URL field, enter `https://api.plivo.com/v1/Account/{PLIVO_AUTH_ID}/Verify/Session/`. Replace `{PLIVO_AUTH_ID}` with your Plivo Auth ID.
- In the headers section, add the following header replacing `{AUTH_HEADER}` with the encoded Authentication Header we created earlier:
  - Header: Authorization
  - Value: Basic {AUTH_HEADER}
- In the body section, add the following JSON replacing `{PLIVO_APP_UUID}` with your Plivo App UUID:

```json
{
  "app_uuid": "{PLIVO_APP_UUID}",
  "recipient": "{{$last.phone_number}}",
	"channel": "sms",
	"method": "POST"
}
```

A typical response from Plivo will look like this:

```json
{
	"api_id": "8ad839d3-34ab-4549-945d-33ed4f350ef5",
	"message": "Session initiated",
	"session_uuid": "b3d06b0c-d1cb-47cd-ab4b-64b579f72f93"
}
```

### Saving the Session UUID

Create a new **Update Data** operation. In the collection field, edit the raw value and set the value to `directus_users`. Set full access permissions and the following payload:

```json
{
    "otp_session_uuid": "{{$last.session_uuid}}"
}
```

Set the following query:

```json
{
    "filter": {
        "phone": {
            "_eq": "{{number_cleanup.phone_number}}"
        }
    }
}
```

### Returning the OTP Session

Your application will need the Session UUID. Create a **Run Script** operation:

```javascript
module.exports = async function(data) {
  return {
    otp_session_uuid: data.create_otp_session.data.session_uuid
  };
}
```

### Testing the Flow

Open your browser to your trigger URL appended with `?phone_number={YOUR_NUMBER}&country_code={YOUR_COUNTRY_CODE}`, replacing the values with your real number.

Directus will respond with a `otp_session_uuid`. This UUID is the OTP session ID that we will use to verify the OTP code. You should also receive an OTP code via SMS to the phone number you provided. Make a note of the OTP code. The `otp_session_uuid` has also been stored against the user account.

![Passwordless Login Flow Screenshot](login_flow.png)

## The Verification Flow

The second flow will accept a Session UUID and a One Time Password. If correct, it will generate and save a new static token against the user and deliver it. The token can then be used to authenticate requests.

Create a new Flow from your project settings with a Webhook trigger and caching disabled. Your application will make a request to this URL when verifying a OTP.

To create a session, create a **Webhook / Request URL** operation with the following options:

- Set the method to POST
- In the URL field, enter `https://api.plivo.com/v1/Account/{PLIVO_AUTH_ID}/Verify/Session/{{$trigger.query.session_uuid}}/`. Replace `{PLIVO_AUTH_ID}` with your Plivo Auth ID.
- In the headers section, add the following header replacing `{AUTH_HEADER}` with the encoded Authentication Header we created earlier:
  - Header: Authorization
  - Value: Basic {AUTH_HEADER}
- In the body section, add the following JSON replacing `{PLIVO_APP_UUID}` with your Plivo App UUID:

```json
{
    "otp": "{{$trigger.query.otp}}"
}

### Generating a Directus Static Token

Each user can have one static token that does not expire. It is stored in plaintext and can either be generated by Directus or created and manually set.

Create a **Run Script** operation:

```javascript
module.exports = async function(data) {
    const message = data.$last.message;
    const rand = () => Math.random().toString(36).substr(2);
    const token = () => rand() + rand();
    
    if (message == "session validated successfully.") {
        return { message, token: token() }
    } else {
        return { message }
    }
}
```

### Saving and Sending the Static Token

Create a new **Update Data** operation. In the collection field, edit the raw value and set the value to `directus_users`. Set full access permissions and the following payload:

```json
{
    "token": "{{$last.token}}"
}
```

Set the following query:

```json
{
    "filter": {
        "otp_session_uuid": {
            "_eq": "{{$trigger.query.session_uuid}}"
        }
    }
}
```

Your application will need the new static token. Create a **Run Script** operation:

```javascript
module.exports = async function(data) {
	return data.generate_session_token;
}
```

### Testing the Flow

Open your browser to your trigger URL appended with `?otp={YOUR_OTP_CODE}&session_uuid={YOUR_OTP_SESSION}`, replacing the values from the first flow run. If it works, Directus will respond with a `token`.

![Verify Flow Screenshot](verify_flow.png)

## Summary

You have now successfully set up passwordless authentication in Directus using the Plivo Verify API. This will allow users to sign in to your Directus project by sending a one time password (OTP) to their phone number.

This same general workflow could be used for emails and magic links could also work the same way. It's important to note that static tokens do not expire and are stored in plaintext, so you should consider a strategy for invalidating tokens.