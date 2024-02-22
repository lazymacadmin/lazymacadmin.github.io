---
layout: post
title:  "Manually Syncing JCDS2 to On-Premise Distribution Points"
comments: false
meta: JCDS JCDS2 On-premise Jamf Distribution Point
---
## Why? Just...why?

With the release of Jamf Pro 11.2, Jamf has exposed API endpoits to programatically retrieve download URLs for packages in your JCDS2 distribution point. While I'm sure that most people are fine with only using their cloud distribution point, there are times when a on-premise distribution point may be preferable. In edu, there are times where a remote campus may have slower or limited capacity to pull down packages during peak hours, so the ability to sync during off-hours can reduce the load on the network.

As our org has DPs on at least one branch campus, and since Jamf has announced the [impending demise of the Jamf Admin app](https://learn.jamf.com/bundle/jamf-pro-release-notes-current/page/Deprecations_and_Removals.html), I felt it was important to look into how to manage that workflow. While Jamf says they are "committed to finding alternative solutions for key workflows from Jamf Admin", they have shown in the past that they are not opposed to letting a loss of functionality - with no replacement - stand in the way of their product roadmap. For example, the delay between the [removal of Jamf Remote](https://learn.jamf.com/bundle/jamf-pro-release-notes-10.40.0/page/Deprecations_and_Removals.html) and the [introduction of Remote Assist](https://learn.jamf.com/bundle/jamf-pro-release-notes-11.1.0/page/New_Features_and_Enhancements.html) functionality was approximately 18 months, and the ability to install a package on demand still hasn't been replaced.

## Bits and Bobs
To that end, I began Monday morning (immediately after the weekend upgrade to 11.2.1) looking at Graham Pugh's [post](https://grahamrpugh.com/2023/08/21/introducing-jcds2.html) about uploading packages to the JCDS2 endpoints. After borrowing parts of my script from his (we are going the opposite direction, after all), I had a poorly-working concept script. The important part is getting a list of all packages and then figuring out which ones to sync. This was simply the reverse of the method Graham used - get the MD5 of the packages in the JCDS.
```zsh
% curl -X 'GET' \
  'https://yourjss.jamfcloud.com/api/v1/jcds/files' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer [token]]'

{"fileName":"Public-GoogleChrome-121.0.6167.160.pkg","length":276659119,"md5":"da325c5965a0bd3e9cea603bf4d5f05d","region":"us-east-1","sha3":""},
{"fileName":"Public-JASP-Arm-0.18.2.0.pkg","length":1175542018,"md5":"4ff5d6fb8cb771e5767590f4b18f21e6","region":"us-east-1","sha3":""},
{"fileName":"Public-Kaltura-Capture-4.2.141.pkg","length":76875008,"md5":"9a112279541638a2d10235a80cfb0618","region":"us-east-1","sha3":""},
{"fileName":"Public-R-4.1.0-arm64.pkg","length":91387296,"md5":"e8cf7cb40a735af9e54dac92346637c3","region":"us-east-1","sha3":""}]
```

Then, calculate the MD5 of the local packages as you look at syncing them. Download if necessary.
```zsh
        for ((i=1; i<=pkg_count; i++)); do
            jcds_pkg=$(/usr/libexec/PlistBuddy -c "Print :$i:fileName" "$output_file_list")
            jcds_pkg_md5=$(/usr/libexec/PlistBuddy -c "Print :$i:md5" "$output_file_list")
            local_pkg="${rsync_path}/${jcds_pkg}"
            if [ -f "${local_pkg}" ]; then
                local_pkg_md5=$(md5 -q "${local_pkg}")
            fi

            if [[ "$local_pkg_md5" == "$jcds_pkg_md5" ]]; then
                echo "$jcds_pkg is the same"
                logStatement "$jcds_pkg is the same"
            else
                getToken
                # retrieve the download URL from the API endpoint
                get_download_url=$(curl --request GET \
                    --silent \
                    --location \
                    --header "authorization: Bearer $token" \
                    --header 'Accept: application/json' \
                    "$jss_url/api/v1/jcds/files/${download_pkg}" )
                download_url=$(echo "$get_download_url" | plutil -extract uri raw -)
                # Get the file, please
                echo "Downloading $jcds_pkg ..."
                logStatement "Downloading $jcds_pkg ..."
                curl "${download_url}" -o "${rsync_path}/${jcds_pkg}"
            fi
        done
```

One thing I noticed immediately is that any filename with a space in it resulted in a failure getting the download URL. Working interactively with the API showed that spaces needed to be url-encoded, so I shamelessly stole a function for that.

## The Script in Action
So here's how it works:
```
% sync_jcds_local_dp.zsh 
Usage: sync_jcds_local_dp.zsh --id /path/to/identity_file --jss your.jamf.server --localpath /path/to/dp/share --log /path/to/log/file
(don't include https:// but do include the port if needed)
```
1. Get `client_id`, `client_secret`, and `grant_type` from a json file. It's easiest to just copy that data when you create the api client in Jamf Pro.
<br/><img src="/assets/images/api_client.png" class="responsive">
<br/>Clicking "Copy client credentials to clipboard" gives you a perfect identity file:
> {"client_name":"test_for_screenshot","client_id":"a7682e67-a276-448b-83af-9c83be5e02f4","client_secret":"ubPrXqRRyc0jRLFOY9iJYGpCjHiJc1M-tRLlCFb9aWZhtqf7iC9UpQG_dO1ZkvNF","grant_type":"client_credentials"}

1. Use those credentials to obtain a bearer token.
1. Download a list of all packages from the `/api/v1/jcds/files` endpoint
1. Count the files then iterate through that list to
   - Get the MD5 for the file
   - Check for a matching file in your mounted on-premise distribution point
     - Check the MD5 of the local file, if it exists
     - If it exists, and MD5 matches, skip.
     - If it doesn't exist, or if it exists and MD5 doesn't match, get the download URL and download the newer copy
1. Write all actions to a log file specified on the command line

I, as usual, let Graham do a lot of the heavy lifting on this and just cobbled together the pieces I needed to make it work. Hopefully this will help keep required functionality alive once Jamf Admin goes away.

## The Complete Script
[The zsh script](https://gist.github.com/lazymacadmin/d4be46c2a782f34e1443e7714bfd22b4)