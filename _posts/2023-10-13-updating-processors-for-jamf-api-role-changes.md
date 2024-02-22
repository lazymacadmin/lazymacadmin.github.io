---
layout: post
title:  "Attempting to Update My Autopkg Processor for Jamf API Roles/Clients"
comments: true
meta: JamfPro, API, client, secret, autopkg, processors, JN-I-27780, feature-request
---
Part of my autopatchg workflow involves dismissing the patch notification when a patch policy is properly updated. In addition to Teams notifications, it's a visual reminder if an autopkg recipe fails. If there is a notification, I need to manually visit the patch and update the policy. To that end, I wrote [JamfClearPatchNotifications.py](https://github.com/lazymacadmin/UpdateTitleEditor/blob/main/Processor/JamfClearPatchNotifications.py) to handle clearing notifications at the end of the `.patch` recipe.

So with all of the changes Jamf is making to API authentication, I decided to update it to use the Jamf api client/role authentication method if anyone chooses to. My thought process was that if a `client_id` and `client_secret` were specified in a recipe, the processor should prefer that method of authentication. If those keys aren't given, everything should function as before.

So I started with this (modified snippet of already working code), fully intending to pretty it up and do functions and make it look all professional:
```python
        jamf_url = self.env.get("JSS_URL")
        not_url = jamf_url + "/api/v1/notifications"

        if self.env.get("client_id") and self.env.get("client_secret"):
            auth_url = jamf_url + "/api/v1/auth/token"
            headers = {"client_id": self.env.get("client_id"), \
                       "grant_type": "client_credentials", \
                        "client_secret": self.env.get("client_secret")}
            tokenreq = requests.post(auth_url,data=headers)
            token = tokenreq.json()["access_token"]
        elif self.env.get("API_USERNAME") and self.env.get("API_PASSWORD"):
            auth_url = jamf_url + "/api/v1/auth/token"
            username = self.env.get("API_USERNAME")
            password = self.env.get("API_PASSWORD")
            headers = {"Content-Type": "application/json"}
            tokenreq = requests.post(auth_url, \
                        auth=(username,password), headers=headers)
            token = tokenreq.json()["token"]
        else:
            self.output("Jamf API Credentials are not in prefs")
            raise ProcessorError("No Authentication credentials supplied")
```
But then I ran it and...nothing happened. I knew I was getting the token, so I started whipped up a quick script to test with
```zsh
#!/bin/zsh

jss_url=$(defaults read /Library/Preferences/com.jamfsoftware.jamf jss_url)

api_user="[username]"
api_pass="[password]"
client_id="[client_id]"
client_secret="[client_secret]"

## User
token=$(curl --silent --location --request 'POST' --url "$jss_url/api/v1/auth/token" \
  -H 'accept: application/json' -u "${api_user}:${api_pass}" -d '' \
  | plutil -extract token raw -)
echo "Trying user and password: "
curl -X 'GET' "$jss_url/api/v1/notifications" -H 'accept: application/json' -H "Authorization: Bearer ${token}"

## Api client
api_token=$(curl --silent --location --request POST "${jss_url}/api/oauth/token" \
  --header "Content-Type: application/x-www-form-urlencoded" --data-urlencode "client_id=${client_id}" \
  --data-urlencode "grant_type=client_credentials" --data-urlencode "client_secret=${client_secret}" \
| plutil -extract access_token raw -)
echo "\n\nTrying api client/role: "
curl -X 'GET' "$jss_url/api/v1/notifications" -H 'accept: application/json' -H "Authorization: Bearer ${api_token}"
echo "\n"
``` 
and found that api clients can't dismiss patch notifications. Because api clients don't see patch notifications.
```
% zsh notifications_test.zsh
Trying user and password:
[ {
  "id": "3154",
  "type": "PATCH_UPDATE",
  "params": {
    "softwareTitleName": "Arc",
    "id": 150,
    "publishedDate": "2023-10-13T16:07:37.473Z",
    "latestVersion": "1.12.1"
  }
}
]

Trying api client/role:
[ ]
```
So I asked about it in Slack and it was suggested I file a support ticket where I detailed what I had found and how I thought it should work, along with a description of my use case. After a few weeks of back and forth emailing about logs, sample scripts,and jamf permissions, we reached the conclusion that because api roles/clients cant check this box they can't see user-context patch notifications. 
<br><img src="/assets/images/patch-notification-checkbox.png" width="1051" height="51" class="responsive" alt="Checkbox to enable patch notifications">

The case was closed after a Zoom call between myself and 2 Jamf support persons with this note:<br>
**Cause**: Beta Feature not yet implemented<br>
**Resolution Summary**: Feature Request <br>

So if you are a Jamf customer and have a minute, please feel free to upvote this [Feature Request](https://ideas.jamf.com/ideas/JN-I-27780). Maybe we'll get lucky and I'll have an update to this post at some point in the future.
