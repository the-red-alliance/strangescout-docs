disqus:

These API routes are used for user authentication (creating accounts, logging in, and deleting accounts)

## Create User

Creates a new user account

### Properties

**Restricted endpoint?:** No

**URL**

`/api/users`

**Method**

`POST`

**URL Params**

None

**Data Params**

|Name|Type|Required|Description|Example|
|----|----|--------|-----------|-------|
|`email`|string|required|email of the new user|bobjones@mydomain.com|
|`password`|string|required|password of the new user|supersecret|
|`code`|string|required|8 character alphanumeric invite code|A1b2C3d4|

### Example Payload

``` json
{
	"email": "bobjones@mydomain.com",
	"password": "supersecret",
	"code": "A1b2C3d4"
}
```

### Successful Response

**Response Code**

`200`

**Response Body**

|Name|Type|Description|Example|
|----|----|-----------|-------|
|`_id`|string|MongoDB object id|`507f191e810c19729de860ea`|
|`email`|string|email of the user|bobjones@mydomain.com|
|`admin`|boolean|is the user an admin|`true`|
|`invite`|boolean|can the user invite new users|`true`|
|`token`|string|the user's authorization token (JWT)|...|

**Example:**

``` json
{
	"_id": "507f191e810c19729de860ea",
	"email": "bobjones@mydomain.com",
	"admin": true,
	"invite": true,
	"token": "eyJhbGciOiJIUzI1NiJ9.eyJkYXRhIjp0cnVlfQ.v9y-_MKlFV_Dq4j7cDpyJMwNU6cJbd9bbdMbxwWs-GI"
}
```

### Failed Responses

**Response Code**

`403`

**Reason**

The invite code cannot be used for this email

---

**Response Code**

`409`

**Reason**

Username already exists

---

**Response Code**

`422`

**Reason**

Email, password, or code is either not given or invalid

---

**Response Code**

`440`

**Reason**

The invite code is expired

!!! note
    expired invite codes are deleted after attempted use, so this response will only appear the first time a code is attempted to be used after it expires

---

## Login User

Logs in to an account

### Properties

**Restricted endpoint?:** No

**URL**

`/api/users/session`

**Method**

`POST`

**URL Params**

None

**Data Params**

|Name|Type|Required|Description|Example|
|----|----|--------|-----------|-------|
|`email`|string|required|email of the user|bobjones@mydomain.com|
|`password`|string|required|password of the user|supersecret|

### Example Payload

``` json
{
	"email": "bobjones@mydomain.com",
	"password": "supersecret"
}
```

### Successful Response

**Response Code**

`200`

**Response Body**

|Name|Type|Description|Example|
|----|----|-----------|-------|
|`_id`|string|MongoDB object id|`507f191e810c19729de860ea`|
|`email`|string|email of the user|bobjones@mydomain.com|
|`admin`|boolean|is the user an admin|`true`|
|`invite`|boolean|can the user invite new users|`true`|
|`token`|string|the user's authorization token (JWT)|...|

**Example:**

``` json
{
	"_id": "507f191e810c19729de860ea",
	"email": "bobjones@mydomain.com",
	"admin": true,
	"invite": true,
	"token": "eyJhbGciOiJIUzI1NiJ9.eyJkYXRhIjp0cnVlfQ.v9y-_MKlFV_Dq4j7cDpyJMwNU6cJbd9bbdMbxwWs-GI"
}
```

### Failed Responses

**Response Code**

`422`

**Reason**

Valid email or password not given

---

**Response Code**

`404`

**Reason**

User not found

---

## Verify User Token

Verifies a session token

### Properties

**Restricted endpoint?:** Yes

**URL**

`/api/users/session`

**Method**

`GET`

**Headers**

- *Authorization*: `Token <user_token_here>`

**URL Params**

None

**Data Params**

None

### Successful Response

**Response Code**

`200`

**Response Body**

|Name|Type|Description|Example|
|----|----|-----------|-------|
|`_id`|string|MongoDB object id|`507f191e810c19729de860ea`|
|`email`|string|email of the user|bobjones@mydomain.com|
|`admin`|boolean|is the user an admin|`true`|
|`invite`|boolean|can the user invite new users|`true`|
|`token`|string|the user's authorization token (JWT)|...|

**Example:**

``` json
{
	"_id": "507f191e810c19729de860ea",
	"email": "bobjones@mydomain.com",
	"admin": true,
	"invite": true,
	"token": "eyJhbGciOiJIUzI1NiJ9.eyJkYXRhIjp0cnVlfQ.v9y-_MKlFV_Dq4j7cDpyJMwNU6cJbd9bbdMbxwWs-GI"
}
```

### Failed Responses

**Response Code**

`404`

**Reason**

User not found