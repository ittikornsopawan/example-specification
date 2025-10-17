# Create IAM Account

| Title           | Description                                      |
| --------------- | ------------------------------------------------ |
| API Name        | Create IAM Account                               |
| Endpoint        | /iam/v1/auth/register                            |
| Method          | POST                                             |
| Version         | 1                                                |
| Last Updated By | [Ittikorn Sopawan (Aof)](ittikorn.sopawan@gmail.com) |
| Last Updated At | 01/10/2025                                       |

## Domain

| Environment    | URL                                                                |
| -------------- | ------------------------------------------------------------------ |
| DEVELOPMENT    | [https://api-dev.domain.com](https://api-dev.domain.com)           |
| QA             | [https://api-qa.domain.com](https://api-qa.domain.com)             |
| UAT            | [https://api-uat.domain.com](https://api-uat.domain.com)           |
| PRE-PRODUCTION | [https://api-pre-prod.domain.com](https://api-pre-prod.domain.com) |
| PRODUCTION     | [https://api.domain.com](https://api.domain.com)                   |

## Description

This API creates a new user account with authentication, profile, consent, and default attributes. It follows these steps:

1. **Receive Request from Client**  
   - The client sends user authentication data to the API.

2. **Validation (Application Layer)**  
   - Verify user information (User Info).  
   - Verify authentication information (username, password).  
   - Verify profile information.  
   - Verify user consent information.  
   - If any data is invalid, the process stops and returns the appropriate error code.

3. **Database Operations (Infrastructure Layer)**  
   - **User:** Insert a new row into `t_users`.  
   - **Password:** Insert a new row into `t_user_authentications`.  
   - **Referral Mapping (if applicable):** Insert a new row into `t_user_referrer_mappings`.  
   - **User Profile:** Insert a new row into `m_user_profiles`.  
   - **User Consent:** Insert a new row into `t_user_consent_mappings`.  
   - **User Attribute:** Retrieve default attributes from `m_parameters` and insert them into `t_user_attribute_mappings`.

4. **Response Mapping (Application Layer)**  
   - Build the response object containing the created user, profile, consent, and attributes.

5. **Response**  
   - Return the response object to the client.

## Change

- 01/10/2025 : Ittikorn Sopawan
  - Initial Specification

## Sequence Diagram

![Sequence Diagram](../uml/create-iam.svg)

## Request Parameters

### Header Parameters

| Property Name | Data Type | Length | M/O         | Description      | Example   |
| ------------- | --------- | ------ | ----------- | ---------------- | --------- |
| Application   | `String`  | `128`  | `Mandatory` | Application Name | Pikul App |

### Body Parameters

| Property Name  | Data Type                                | Length | M/O         | Description         | Example |
| -------------- | ---------------------------------------- | ------ | ----------- | ------------------- | ------- |
| user           | [User](#user) Object                     |        | `Mandatory` | User Info           |         |
| authentication | [Authentication](#authentication) Object |        | `Mandatory` | Authentication Info |         |
| referrer       | [Referrer](#referrer) Object             |        | `Mandatory` | Referrer Info       |         |
| profile        | [Profile](#profile) Object               |        | `Mandatory` | Profile Info        |         |
| userConsent    | [User Consent](#user-consent) Array      |        | `Mandatory` | User Consent Info   |         |

#### User

| Property Name | Data Type        | Length | M/O         | Description | Example                 |
| ------------- | ---------------- | ------ | ----------- | ----------- | ----------------------- |
| username      | `String: Base64` | `128`  | `Mandatory` | Username    | username_example_base64 |

#### Authentication

| Property Name   | Data Type        | Length | M/O         | Description      | Example                 |
| --------------- | ---------------- | ------ | ----------- | ---------------- | ----------------------- |
| password        | `String: AES256` |        | `Mandatory` | Password         | password_example_aes256 |
| confirmPassword | `String: AES256` |        | `Mandatory` | Confirm Password | password_example_aes256 |

#### Referrer

| Property Name | Data Type | Length | M/O         | Description   | Example       |
| ------------- | --------- | ------ | ----------- | ------------- | ------------- |
| referrer      | `String`  |        | `Mandatory` | Referrer Code | referrer_code |

#### Profile

| Property Name | Data Type        | Length | M/O         | Description   | Example       |
| ------------- | ---------------- | ------ | ----------- | ------------- | ------------- |
| firstName     | `String: Base64` |        | `Mandatory` | First Name    | first_name    |
| lastName      | `String: Base64` |        | `Mandatory` | Last Name     | last_name     |
| middleName    | `String: Base64` |        | `Optional`  | Middle Name   | middle_name   |
| displayName   | `String`         |        | `Optional`  | Display Name  | display_name  |
| dateOfBirth   | `String: Date`   |        | `Optional`  | Date Of Birth | date_of_birth |

#### User Consent

| Property Name | Data Type      | Length | M/O         | Description              | Example         |
| ------------- | -------------- | ------ | ----------- | ------------------------ | --------------- |
| consentId     | `String: UUID` |        | `Mandatory` | Consent Id               | consent_id      |
| version       | `String`       |        | `Mandatory` | Consent Version          | consent_version |
| result        | `Boolean`      |        | `Mandatory` | Consent Acception Result | consent_result  |

### Example Request Parameters

```JSON
{
    "user": {
        "username": "String: Base64"
    },
    "authentication": {
        "password": "String: Base64",
        "confirmPassword": "String: Base64"
    },
    "referrer": {
        "referrer": "String"
    },
    "profile": {
        "firstName": "String: Base64",
        "lastName": "String: Base64",
        "middleName": "String: Base64",
        "displayName": "String: Base64",
        "dateOfBirth": "String:Date"
    },
    "userConsent": [
        {
            "consentId": "String: UUID",
            "version": "String",
            "result": "Boolean"
        }
    ]
}
```

### Example Request

#### cURL

```curl
curl -X POST "https://api-dev.domain.com/iam/v1/auth/register" \
  -H "Content-Type: application/json" \
  -H "Application: Pikul App" \
  -d '{
    "user": {
        "username": "String: Base64"
    },
    "authentication": {
        "password": "String: Base64",
        "confirmPassword": "String: Base64"
    },
    "referrer": {
        "referrer": "String"
    },
    "profile": {
        "firstName": "String: Base64",
        "lastName": "String: Base64",
        "middleName": "String: Base64",
        "displayName": "String: Base64",
        "dateOfBirth": "String:Date"
    },
    "userConsent": [
        {
            "consentId": "String: UUID",
            "version": "String",
            "result": "Boolean"
        }
    ]
}'
```

#### HTTP Request

```http
POST /iam/v1/auth/register HTTP/1.1
Host: api-dev.domain.com
Content-Type: application/json
Application: Pikul App

{
    "user": {
        "username": "String: Base64"
    },
    "authentication": {
        "password": "String: Base64",
        "confirmPassword": "String: Base64"
    },
    "referrer": {
        "referrer": "String"
    },
    "profile": {
        "firstName": "String: Base64",
        "lastName": "String: Base64",
        "middleName": "String: Base64",
        "displayName": "String: Base64",
        "dateOfBirth": "String:Date"
    },
    "userConsent": [
        {
            "consentId": "String: UUID",
            "version": "String",
            "result": "Boolean"
        }
    ]
}
```

## Response Parameters

### Response Parameters Example

| Property Name | Data Type         | Length | M/O         | Description     | Example |
| ------------- | ----------------- | ------ | ----------- | --------------- | ------- |
| status        | [Status](#status) |        | `Mandatory` | Response Status |         |
| data          | [Data](#data)     |        | `Optional`  | Response Data   |         |

#### Status

| Property Name   | Data Type | Length | M/O         | Description            | Example      |
| --------------- | --------- | ------ | ----------- | ---------------------- | ------------ |
| code            | `Int`     |        | `Mandatory` | Http Status Code       | 201          |
| message         | `String`  | `128`  | `Mandatory` | Http Status Message    | Created      |
| bizErrorCode    | `Int`     |        | `Optional`  | Business Error Code    | 30001        |
| bizErrorMessage | `String`  | `512`  | `Optional`  | Business Error Message | Server Error |

#### Data

| Property Name  | Data Type                                | Length | M/O         | Description         | Example |
| -------------- | ---------------------------------------- | ------ | ----------- | ------------------- | ------- |
| user           | [User](#user) Object                     |        | `Mandatory` | User Info           |         |
| authentication | [Authentication](#authentication) Object |        | `Mandatory` | Authentication Info |         |
| referrer       | [Referrer](#referrer) Object             |        | `Mandatory` | Referrer Info       |         |
| profile        | [Profile](#profile) Object               |        | `Mandatory` | Profile Info        |         |
| userConsent    | [User Consent](#user-consent) Array      |        | `Mandatory` | User Consent Info   |         |

##### User

| Property Name | Data Type        | Length | M/O         | Description | Example                 |
| ------------- | ---------------- | ------ | ----------- | ----------- | ----------------------- |
| username      | `String: Base64` | `128`  | `Mandatory` | Username    | username_example_base64 |

#### Referrer

| Property Name | Data Type | Length | M/O         | Description   | Example       |
| ------------- | --------- | ------ | ----------- | ------------- | ------------- |
| referrer      | `String`  |        | `Mandatory` | Referrer Code | referrer_code |

##### Profile

| Property Name | Data Type        | Length | M/O         | Description   | Example       |
| ------------- | ---------------- | ------ | ----------- | ------------- | ------------- |
| firstName     | `String: Base64` |        | `Mandatory` | First Name    | first_name    |
| lastName      | `String: Base64` |        | `Mandatory` | Last Name     | last_name     |
| middleName    | `String: Base64` |        | `Optional`  | Middle Name   | middle_name   |
| displayName   | `String`         |        | `Optional`  | Display Name  | display_name  |
| dateOfBirth   | `String: Date`   |        | `Optional`  | Date Of Birth | date_of_birth |

#### User Consent

| Property Name | Data Type      | Length | M/O         | Description              | Example         |
| ------------- | -------------- | ------ | ----------- | ------------------------ | --------------- |
| consentId     | `String: UUID` |        | `Mandatory` | Consent Id               | consent_id      |
| version       | `String`       |        | `Mandatory` | Consent Version          | consent_version |
| result        | `Boolean`      |        | `Mandatory` | Consent Acception Result | consent_result  |

### Example Request Parameters

#### Success

```JSON
{
    "status": {
        "code": 201,
        "message": "Created"
    },
    "data": {
        "user": {
            "id": "String: UUID",
            "username": "String: Base64"
        },
        "referrer": {
            "referrer": "String"
        },
        "profile": {
            "firstName": "String: Base64",
            "lastName": "String: Base64",
            "middleName": "String: Base64",
            "displayName": "String: Base64",
            "dateOfBirth": "String:Date"
        },
        "userConsent": [
            {
                "consentId": "String: UUID",
                "version": "String",
                "result": "Boolean"
            }
        ] 
    }
}
```

#### Fail

```JSON
{
    "status": {
        "code": 400,
        "message": "Bad Request",
        "bizErrorCode": 30001,
        "bizErrorMessage": "Field 'Username' is Required!"
    }
}
```

## Field Mapping

### t_users

| Source  | Field Name | Destination | Column Name | Remark                                                                           |
| ------- | ---------- | ----------- | ----------- | -------------------------------------------------------------------------------- |
|         |            | t_users     | id          | SET id = Generate UUID                                                           |
|         |            | t_users     | created_by  | id                                                                               |
|         |            | t_users     | created_at  | SET created_at = Current DateTime                                                |
|         |            | t_users     | code        | Generate by [00000000-1] <br />{00000000} = running<br />{1/0} = active/inactive |
|         |            | t_users     | is_active   | SET is_activate = `TRUE`                                                         |
| request | user       | t_users     | username    |                                                                                  |

### m_parameters - Get Default Hash

| Source | Field Name | Destination  | Column Name | Remark                   |
| ------ | ---------- | ------------ | ----------- | ------------------------ |
|        |            | m_parameters | key         | SET key = `DEFAULT_HASH` |

### t_user_authentications

| Source  | Field Name | Destination | Column Name | Remark                                                                           |
| ------- | ---------- | ----------- | ----------- | -------------------------------------------------------------------------------- |
|         |            | t_users     | id          | SET id = Generate UUID                                                           |
|         |            | t_users     | created_by  | id                                                                               |
|         |            | t_users     | created_at  | SET created_at = Current DateTime                                                |
|         |            | t_users     | code        | Generate by [00000000-1] <br />{00000000} = running<br />{1/0} = active/inactive |
|         |            | t_users     | is_active   | SET is_activate = `TRUE`                                                         |
| request | user       | t_users     | username    |                                                                                  |
