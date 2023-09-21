---
layout: post
title:  "Using networkQuality to Gather Data for Support Calls"
comments: false
---


Some of our frequent support calls involves choppy video or audio on Zoom or Teams. As more of our users work, teach, or learn from home it becomes harder to narrow down if the issue is local to the machine or from the user's internet connection. I decided to use Jamf's Self Service app to enable gathering of data about a user's internet connection to help our techs on those calls.

#### **Self Service Method**

Initially I used the Jamf Pro API to upload a report file to the user's machine record. One downside to this is that some form of credentials need to be present on the machine during execution. It could be: 
- username/password either embedded in the script or passed on the command line as a script parameter
- a lazily-obfuscated base64'ed auth string embedded in the script or passed on the command line as a script parameter
- a client_id and client_secret for an [api user](https://lazymacadmin.github.io/2023/09/04/obtaining-a-bearer-token-with-api-roles-with-jamf.html) with minimal permissions embedded in the script or passed on the command line as a script parameter:
    - `Computers: RU` 
    - `File Attachments: C`

Which option works best for you, or even is acceptable, will be a decision to be made by each organization. Originally I was fine with the base64-obfuscation method but have since updated the script to use API Roles for more limited access.

Since this script is user initiated, I use [swiftDialog](https://swiftdialog.app/) to show progress and the results to the user so they can know when the testing is done. The script displays "Ethernet" if a wired network is used, otherwise the active SSID is recorded and used.<br>
<img src="/assets/images/uploading_results.png" width="461" height="237" class="responsive" alt="Uploading results screenshot" ><br>
<img src="/assets/images/testing_done.png" width="230" height="118" class="responsive" alt="Testing complete screenshot">
<br>
<details><summary markdown="span"> networkQuality Script for Self Service </summary>{% gist 9c0d28ac7197582f915bb0742a68721c %}</details>
<details><summary markdown="span"> networkQuality Self Service Uploads </summary>{% gist 3d43134a00b5d89c27f309f91ce76b36 %}</details>
<br/>

---
#### **Teams Webhook Method**

We also use a variation to monitor bandwidth constraints on campus. I moved away from the file upload option for this when I created a Teams channel for speed tests and set up an [incoming webhook](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook?tabs=dotnet) to receive the results.  I also simplified the script to show no progress feedback. Now we have regular checks to ensure our campus's connection is sufficient for classroom, hybrid, and online instruction.<br>
<img src="/assets/images/webhook_results.png" height="401" width="614" class="responsive"><br>
<details><summary markdown="span"> networkQuality Script for Teams </summary>{% gist 7ed223e219836a39732263abf160d5c3 %}</details>

