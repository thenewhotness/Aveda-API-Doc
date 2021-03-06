USER TABLE STRUCTURE
---
- APP info and PII can be stored separately, or togther like this.
- We would need to relate them via ```fb_id``` or email if the information lived in different places

```sql
+ USER
(APP INFO)
- fb_id : integer
- giver_fb_id : integer
- eligible_play_date : date
- did_win_sample : boolean
- did_redeem_sample : boolean
- did_receive_coupon : boolean
- did_capture_email : boolean

(PII)
- first_name : varchar
- last_name : varchar
- email : varchar
- street : varchar
- street2 : varchar
- city: varchar
- state : varchar
- zip : integer
```

API CALLS
---

####```check_status_for_user```

  request params : ```fb_id```
  
  request format : ```/check_status_for_user?user=fb_id ```
  
  response params : ```fb_id, user_status, giver_fb_id``` 
  
  ```user_status``` values : ```'contest_eligible', 'contest_ineligible', 'contest_ineligible_winner', 'redeem_state'```
  
  response format :
```json
  {
      "status_obj": {
          "user" : fb_id,
          "status" : user_status,
          "giver_id": giver_fb_id
      }
  }
```
  
  - Upon invoking this call, the database is queried to determine user's eligibility to enter the contest

```javascript      
      if (user.did_win_sample == true && user.did_redeem_sample == false) {
        - "status" : "redeem_state" is returned
        - "giver_id" : user.giver_fb_id is returned
      }
      else if (user.did_win_sample == true && user.did_redeem_sample == true) {
        - "status" : "contest_ineligible_winner" is returned
      }
      else if ({current time} > user.eligible_play_date) {
        - "status" : "contest_eligible" is returned
        
        // OPTIONAL - This flag would allow people to win more samples//
        - user.did_win_sample = false;
        // END OPTIONAL
      }
      else { 
        - "status" : "contest_ineligible" is returned
      }
```    
___

####```run_contest_for_user```

  - Pass a facebook id as the user param to receive results for contest run

  request params : ```fb_id```
  
  request format : ```/run_contest_for_user?user=fb_id```
  
  response params : ```fb_id, win_state```
  
  ```win_state``` values : ```"user_win", "user_coupon", "gather_email", "user_did_not_win"```
  
  response format : 
```json
    {
        "contest_obj": {
            "user" : fb_id,
            "outcome" : win_state
        }
    }
```
  
  - Upon invoking this call, a random number (CONTEST_ENTRY) is generated between 0 and 1
  - win_range is mapped from 0 to an arbitrary value for % chance to win
  - Coupon range is mapped from win_range to an arbitrary value for % chance to win coupon
    - example with 5% win chance, 10% coupon chance

```javascript
      CONTEST_ENTRY = arc4_rand(1)
      win_range = (0.0, 0.05)
      coupon_range = (0.05, 0.15)
      CONTEST_ENTRY of 0.024 => win
      CONTEST_ENTRY of 0.095 => coupon
      CONTEST_ENTRY of 0.05  => lose
      
    if (CONTEST_ENTRY {within} win_range) {
      // this is the win state. we disable entering the contest for the next 24 hours and mark them as a winner
      - user.eligible_play_date += 24 hours
      - user.did_win_sample = true
      - "outcome" : "user_win" is returned
    }
    else if (CONTEST_ENTRY {within} coupon_range && user.did_receive_coupon == false) {
      // this is the secondary win state. we disable entering the contest for the next 24 hours and mark them as receiving the coupon
      - user.eligible_play_date += 24 hours
      - user.did_receive_coupon = true
      - "outcome" : "user_coupon" is returned  
    }
    else {
      if (user.did_capture_email == true) {
        // the user has given their email already and must wait 24 hours to re-enter the contest
        - user.eligible_play_date += 24 hours
        - "outcome" : "user_did_not_win" is returned
      }
      else  {
        // in this state, the user can give their email for another chance to enter the contest
        - "outcome" : "gather_email" is returned
      }
    }
```
___

####```re_enter_contest_with_email```
  
  request params : ```fb_id, email```
  
  request format : ```/re_enter_contest_with_email?user=fb_id&user_email=email```
  
  response params : ```fb_id, win_state```
  
  ```win_state``` values : ```"user_win", "user_coupon", "user_did_not_win"```
  
  response format : 
```json
  {
      "contest": {
          "user" : fb_id,
          "outcome" : win_state
      }
  }
```
  
  - If a user has received the "gather_email" status in response to a call to "run_contest_for_user", they can use this call to re-enter the contest

```javascript
  - user.email = params(email)
  - user.did_capture_email = true
  - {RE-ENTER CONTEST FLOW}
```
___

####```give_and_get_from_product_win```
  
  request params : ```fb_id, recipient_fb_id```
  
  request format : ```/give_and_get_from_product_win?user=fb_id&recipient=recipient_fb_id```
  
  response params : ```fb_id, recipient_fb_id```
  
  response format :
```json
  {
      "give_and_get_win": {
          "user" : fb_id,
          "recipient" : recipient_fb_id
      }
  }
```
  
  - If a user has received the "user_win" status in response to a call to "run_contest_for_user", they can use this call to give their 'win' to another user
  
  _for clarity,
     ```user``` = facebook user with ID: ```fb_id```
     ```recipient``` = facebook user with ID: ```recipient_fb_id```
  
  Upon invoking this call,

```javascript
    - user.did_redeem_sample = true
    - recipient.did_win_sample = true;
    - recipient.did_redeem_sample = false; // this should be false already from init
```

___



####```redeem_sample_for_user```
  
  request params : ```fb_id, form_data```
  
  request format : ```/redeem_win_for_user?user=fb_id&formData=form_data```
  
  response params : ```fb_id, redeem_state, &error```
  
  ```redeem_state``` values : ```"success", "error"```
  response format : 
```json
  {
    "redeem": {
        "user": fb_id,
        "outcome": redeem_state,
        "error" : &error,
    }
  }
```

  Upon invoking this call,
```javascript
    if ({no problem with form data, successful submission}){
      - user.did_redeem_sample = true
    }
    else {
      "outcome" : "error"
      "error" : {ERROR FROM SERVER} is returned
    }
```

SUMMARIZED USE
---

####```check_status_for_user```

req:
```
/check_status_for_user?user=34203086
```

example responses:

(new user, not gifted)
```json
{
  "status_obj": {
      "user" : 34203086,
      "status" : "contest_eligible",
      "giver_id": null
  }
}
```

(user redeeming a gift)
```json
{
  "status_obj": {
      "user" : 34203086,
      "status" : "redeem_state",
      "giver_id": 09237623
  }
} 
```

(returning user, able to participate in contest)
```json
{
  "status_obj": {
      "user" : 34203086,
      "status" : "contest_eligible",
      "giver_id": null
  }
}
```
  
___
  
####```run_contest_for_user```

req:
```
/run_contest_for_user?user=34203086
```

example responses:

(winner)
```json
{
  "status_obj": {
      "user" : 34203086,
      "outcome" : "user_win",
  }
}
```

(coupon received)
```json
{
  "status_obj": {
      "user" : 34203086,
      "outcome" : "user_coupon",
  }
}
```
___
  
####```give_and_get_from_product_win```

req:
```
/give_and_get_from_product_win?user=34203086&recipient=34891543
```

example response:
```json
{
    "give_and_get": {
        "user" : 34203086,
        "recipient" : 34891543
    }
}
```
___
  
####```redeem_sample_for_user```

req:
```
/redeem_win_for_user?user=34203086&formData={'name':'T Swell', 'address':'77 Franklin St. \n New York, NY 10013'}
```

response:
```json
{
      "redeem": {
          "user": 34203086,
          "outcome": "success",
          "error" : null,
      }
}
```

bad req:
```
/redeem_win_for_user?user=34203086&formData={'name', 'address':'77 Franklin St. \n New York, NY 10013'}
```

response:
```json
{
      "redeem": {
          "user": 34203086,
          "outcome": "error",
          "error" : "INVALID JSON OBJECT",
      }
}
```

EXAMPLE USER FLOWS
---

####New user ```123456```, wins contest####

- A new user arrives at the app on facebook and is asked to authorize it to receiver her information
- The user authorizes the app and the applications grabs her ```fb_id``` of  ```123456```
- Application calls the server with request : ```/check_status_for_user?user=123456```
- The server returns : 

```json
{
  "status_obj": {
      "user" : 123456,
      "status" : "contest_eligible",
      "giver_id": null
  }
}
```

- User is presented with the visualization for entering the contest
- User interacts with the contest object
- The application calls the server with request :```/run_contest_for_user?user=123456```
- Server determines the user has won and returns :

```json
{
  "status_obj": {
      "user" : 123456,
      "outcome" : "user_win",
  }
}
```
- User is then brought to a screen where they can gift the app to a friend, and enter their PII
- She selects a friend from their list with ```fb_id``` of ```987654```
- Application calls server with request :```/give_and_get_from_product_win?user=123456&recipient=987654```
- Server completes the action of creating/updating user ```987654``` to reflect them having won, and having received the win from user ```123456```
- User is prompted to enter their information to receive their own sample
- Application calls server with request :

``` 
/redeem_win_for_user?user=123456&formData={
    "first_name": "T",
    "last_name": "SWELL",
    "street": "77FranklinSt.",
    "street2": "groundfloor",
    "city": "NewYork",
    "state": "NewYork",
    "zip": 10013
}
```
- If all data is correctly entered, server responds with :

```json
{
      "redeem": {
          "user": 123456,
          "outcome": "success",
          "error" : null,
      }
}
```
-The user is thanked for participating, and told to enjoy their sample
___

####Winning user ```123456```, returns to the site####

- The user returns to the app and the applications grabs her ```fb_id``` of  ```123456```
- Application calls the server with request : ```/check_status_for_user?user=123456```
- The server returns : 

```json
{
  "status_obj": {
      "user" : 123456,
      "status" : "contest_ineligible_winner",
      "giver_id": null
  }
}
```
___

####User ```987654```, enters to the site, having been gifted a sample####

- The user returns to the app and the applications grabs her ```fb_id``` of  ```987654```
- Application calls the server with request : ```/check_status_for_user?user=987654```
- The server returns : 

```json
{
  "status_obj": {
      "user" : 123456,
      "status" : "redeem_state",
      "giver_id": null
  }
}
```

- User is prompted to enter their information to receive their sample
- Application calls server with request :

``` 
/redeem_win_for_user?user=987654&formData={
    "first_name": "T",
    "last_name": "SWELL",
    "street": "77FranklinSt.",
    "street2": "groundfloor",
    "city": "NewYork",
    "state": "NewYork",
    "zip": 10013
}
```
- If all data is correctly entered, server responds with :

```json
{
      "redeem": {
          "user": 987654,
          "outcome": "success",
          "error" : null,
      }
}
```
-The user is thanked for participating, and told to enjoy their sample

___

####New user ```555555```, gets coupon####

- A new user arrives at the app on facebook and is asked to authorize it to receiver her information
- The user authorizes the app and the applications grabs her ```fb_id``` of  ```555555```
- Application calls the server with request : ```/check_status_for_user?user=555555```
- The server returns : 

```json
{
  "status_obj": {
      "user" : 555555,
      "status" : "contest_eligible",
      "giver_id": null
  }
}
```

- User is presented with the visualization for entering the contest
- User interacts with the contest object
- The application calls the server with request :```/run_contest_for_user?user=555555```
- Server determines the user has not won, but can receive a coupon and returns :

```json
{
  "status_obj": {
      "user" : 555555,
      "outcome" : "user_coupon",
  }
}
```
- User is then presented with a coupon, and told "thanks for playing, try again tomorrow!"

___
