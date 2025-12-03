# Amazon Advertising API Integration Guide (Direct Advertiser)

This document describes how to integrate with the **Amazon Advertising API** as a **Direct Advertiser** using **Login with Amazon (LwA)** and obtain the required credentials to call the Advertising API.

---

## Overview

To access the Amazon Advertising API, you must:

1. Create and configure a Login with Amazon (LwA) application.
2. Obtain an **authorization code** via the Amazon OAuth flow.
3. Exchange the authorization code for **access** and **refresh tokens**.
4. Use the access token and client ID to retrieve your **Advertising Profile ID**.
5. Use the **profileId** for subsequent Advertising API calls.

---

## Prerequisites

* An Amazon account with access to Amazon Ads.
* Access to the **Amazon Developer Console** to create an LwA security profile.
* Basic understanding of HTTP requests (e.g., using `curl`, Postman, or any HTTP client).

---

## Step 1: Create LwA Application & Get Credentials

1. **Login with Amazon**

   Sign in to your Amazon Developer account and navigate to the **Login with Amazon** / **Security Profile** section.

2. **Create an Application**

   * Create a new **Security Profile / Application** for your Advertising integration.
   * Configure the allowed/redirect URLs (e.g., your app’s callback URL).

3. **Get `client_id` and `client_secret`**

   Once the application is created, note down:

   * **Client ID** (`YOUR_CLIENT_ID`)
   * **Client Secret** (`YOUR_CLIENT_SECRET`)

You will use these in later steps.

---

## Step 2: Obtain Authorization Code

To obtain an authorization code, redirect the user (or yourself during setup) to the following URL in a browser:

```text
https://www.amazon.com/ap/oa?client_id=YOUR_CLIENT_ID&scope=advertising::campaign_management&response_type=code&redirect_uri=YOUR_RETURN_URL
```

* `client_id`: The **LwA client ID** from Step 1.
* `scope`: Use `advertising::campaign_management` for access to Campaign Management.
* `response_type`: Must be `code`.
* `redirect_uri`: Must match exactly one of the redirect URLs configured in your LwA app (e.g., `https://amazon.com` or your own URL).

### Flow

1. Open the URL in your browser.

2. Login and grant permissions (if prompted).

3. After consent, Amazon redirects to:

   ```text
   YOUR_RETURN_URL?code=AUTH_CODE&state=...
   ```

4. Extract and **save** the `code` value (this is your **AUTH_CODE**).

> **Endpoint used**:
> `https://www.amazon.com/ap/oa`

---

## Step 3: Exchange Authorization Code for Tokens

Use the **authorization code** from Step 2 and your LwA credentials from Step 1 to request tokens:

```bash
curl \
  -X POST \
  --data "grant_type=authorization_code&code=AUTH_CODE&redirect_uri=YOUR_RETURN_URL&client_id=YOUR_CLIENT_ID&client_secret=YOUR_SECRET_KEY" \
  https://api.amazon.com/auth/o2/token
```

* `grant_type`: `authorization_code`
* `code`: The authorization code from Step 2 (`AUTH_CODE`).
* `redirect_uri`: Must match the same URL used in Step 2.
* `client_id`: Your LwA client ID from Step 1.
* `client_secret`: Your LwA client secret from Step 1.

### Sample Response

```json
{
  "access_token": "Atza|IQEBLjAsAhRmHjNgHpi0U-Dme37rR6CuUpSR...",
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "Atzr|IQEBLzAtAhRPpMJxdwVz2Nn6f2y-tpJX2DeX..."
}
```

* `access_token`: Short-lived token used to call the Amazon Advertising API.
* `refresh_token`: Long-lived token used to obtain new access tokens without re-authorizing the user.
* `expires_in`: Lifetime of the access token in seconds (typically 3600 = 1 hour).

> **Important**:
> Store the `refresh_token` securely. You will use it to refresh the access token in production flows.

---

## Step 4: Retrieve Advertising Profile ID

Once you have a valid **access token**, you can retrieve the associated Advertising profiles. The **profileId** is required for most Advertising API requests.

Use the following request:

```bash
curl \
  -H "Amazon-Advertising-API-ClientId: YOUR_CLIENT_ID" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  https://advertising-api.amazon.com/v2/profiles
```

* `Amazon-Advertising-API-ClientId`: Your LwA client ID from Step 1.
* `Authorization`: `Bearer` followed by the `access_token` from Step 3.

### Sample Response

```json
[
  {
    "profileId": 888888888,
    "countryCode": "MX",
    "currencyCode": "MXN",
    "timezone": "America/Los_Angeles",
    "accountInfo": {
      "marketplaceStringId": "A1AM78C64UM0Y8",
      "id": "ENTITY2Ihjasdjkeru",
      "type": "vendor",
      "name": "Name of the Account",
      "validPaymentMethod": false
    }
  }
]
```

Note the `profileId` value. You will pass this in the `Amazon-Advertising-API-Scope` header for subsequent calls to the Advertising API.

---

## Headers Summary for Advertising API Calls

For most Advertising API endpoints, you will use headers similar to:

```http
Authorization: Bearer YOUR_ACCESS_TOKEN
Amazon-Advertising-API-ClientId: YOUR_CLIENT_ID
Amazon-Advertising-API-Scope: YOUR_PROFILE_ID
Content-Type: application/json
```

* `Authorization`: From Step 3.
* `Amazon-Advertising-API-ClientId`: From Step 1.
* `Amazon-Advertising-API-Scope`: `profileId` from Step 4.

---

## Token Refresh (Optional but Recommended)

After the initial setup, you should use the **refresh token** to obtain new access tokens without repeating the user authorization step:

```bash
curl \
  -X POST \
  --data "grant_type=refresh_token&refresh_token=YOUR_REFRESH_TOKEN&client_id=YOUR_CLIENT_ID&client_secret=YOUR_SECRET_KEY" \
  https://api.amazon.com/auth/o2/token
```

The response structure will be similar to Step 3, returning a new `access_token`.

---

## Example Sequence

1. **Create LwA app** → Get `client_id`, `client_secret`.
2. **Get authorization code** via browser `https://www.amazon.com/ap/oa?...`.
3. **Exchange code** for `access_token` and `refresh_token` using `https://api.amazon.com/auth/o2/token`.
4. **Get profiles** using `https://advertising-api.amazon.com/v2/profiles` and retrieve `profileId`.
5. Use `access_token`, `client_id`, and `profileId` to call other Amazon Advertising API endpoints.

---

## Notes & Best Practices

* Always store **client_secret** and **refresh_token** in a secure location (e.g., environment variables, secrets manager).
* Implement automatic token refresh using the `refresh_token` before the `access_token` expires.
* Use different environments (sandbox vs production) if available, and keep credentials separated.
* Add proper error handling and logging for all HTTP requests.

---

## License

This documentation is intended as integration guidance for internal or project-specific use. Adjust and extend as needed for your own application or team.
