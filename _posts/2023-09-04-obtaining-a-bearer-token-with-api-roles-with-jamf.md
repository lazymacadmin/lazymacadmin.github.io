---
layout: post
title:  "Obtaining a Bearer Token Using API Role Authentication with Jamf"
comments: true
---
Beginning with Jamf Pro 10.49, it became possible to create [api-only clients and roles](https://learn.jamf.com/bundle/jamf-pro-documentation-current/page/API_Roles_and_Clients.html). From my perspective, there is no immediate reason to change all of my scripts over but if you have ever wanted your scripts to not require a username and password capable of GUI access to you Jamf server, this will certainly be welcome news.

But that's all I'm going to say about the why of API roles and clients in Jamf Pro. This is about how to get a bearer token in your scripts. I focused on my 2 most common use cases in oulling this together. There may be better or more efficient ways to do this, but these work and can be a jumping off point for building pr modifying existing scripts.

In zsh:
```zsh
#!/bin/zsh

jss_url=$(defaults read /Library/Preferences/com.jamfsoftware.jamf jss_url)
client_id="your_client_id"
client_secret="your_client_secret"

output=$(curl --location --request POST "${jss_url}/api/oauth/token" \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=${client_id}" \
--data-urlencode "grant_type=client_credentials" \
--data-urlencode "client_secret=${client_secret}"
)

token=$(echo "$output" | plutil -extract access_token raw -)
token_expires=$(echo "$output" | plutil -extract expires_in raw -)
```

For python, I focused on using the system `curl` to get the token because that's how I tend to do it. Mostly because most of my python work is done on systems with autopkg and its python, which doesn't include the `requests` module (though I do use that some in my processors), but better to keep it simple for a first attempt.
```python
#!/usr/bin/env python3

import subprocess
import json

jamf_url = "https://jamf.fq.dn"
client_id = "your_client_id"
client_secret = "your_client_secret"

url = jamf_url + "/api/oauth/token"

curl_cli = [
                "/usr/bin/curl",
                "--request",
                "POST",
                url,
            ]
curl_cli.extend(["--header", "Content-Type: application/x-www-form-urlencoded"])
curl_cli.extend(["--data-urlencode", "client_id={}".format(client_id)])
curl_cli.extend(["--data-urlencode", "grant_type=client_credentials"])
curl_cli.extend(["--data-urlencode", "client_secret={}".format(client_secret)])

proc = subprocess.Popen(curl_cli, stdout=subprocess.PIPE,
                        stderr=subprocess.PIPE)
(out, err) = proc.communicate()
auth_json_raw = json.loads(out.decode())
token = auth_json_raw["access_token"]
token_expires = auth_json_raw["expires_in"]
```
