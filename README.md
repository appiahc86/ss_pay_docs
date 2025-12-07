
Base url :
``
https://server.smartservepay.com
``


## Auth

### Request OTP
````
const data = {
  "phone": "", //eg. 0244123456 must be a valid Ghana phone number
  "purpose": "registration",
  "user_type": "user'
}


fetch('/auth/request-otp', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(data)  // Convert to JSON string
})


````
#### Success response
````
{
    "success": true,
    "message": "OTP sent to +233***456",
    "expiresIn": 1800, //3o munites
    "canResendAfter": 50 //Can resend after 30 seconds
}
````
#### Error response. (error code 400)
````
{
    "success": false,
    "message": "Validation failed",
    "errors": [
        "Invalid Phone number"
    ]
}
````
#### Error response. (error code 409)
````
{
    "success": false,
    "message": "Phone number is already registered"
}
````
### Verify OTP
````
const data = {
  "phone": "", //eg. 0244123456
  "otp": "", //6 digits code recieved via sms
  "purpose": "registration"
}


fetch('/auth/verify-otp', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(data) 
})


````

#### Success response
````
{
    "success": true,
    "message": "OTP verified successfully",
    "otpId": 7
}
````

#### Error response (code 400)
````
{
    "success": false,
    "message": "", // May display this to user
    "error": ""
}
````

### Register
````
const data = {
  "name": "", //min 2, max 100 characters
  "phone": "", //eg. 0244123456
  "otpId": "", // Will get this after verifying otp
}


fetch('/auth/register', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(data) 
})


````

#### Success response
````
{
 
}
````

#### Error response (code 400)
````
{

}
````
