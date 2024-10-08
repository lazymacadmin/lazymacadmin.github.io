---
layout: post
title:  "Autopkg Run Reports Using Power Automate Workflows for Teams"
comments: true
meta: JamfPro, autopkg, teams, webhooks, automation, autopkg processors, autopkg recipes
---
Back when I was still doing the thing, sending reports from Autopkg was the most efficient way of sharing updated versions of software, as well as the status of autopkg runs. On July 3, 2024, Microsoft [announced](https://devblogs.microsoft.com/microsoft365dev/retirement-of-office-365-connectors-within-microsoft-teams/) that the old Teams webhooks connectors would be discontinued in October. While they have walked back that timeframe, to the end of 2025, the writing is on the wall - it's time to migrate to Power Automate workflows. There's just one catch. The old json payload is U-G-L-Y with the Power Automate workflows.
<br/><img src="/assets/images/teams-ugly.png" class="responsive">

Then, on Slack, [andrewvalentine](https://macadmins.slack.com/team/U060CFZV2) asked about posting to [Power Automate workflows](https://macadmins.slack.com/archives/C056155B4/p1727261116766689). I had been too ~~lazy~~ busy to do any more than a cursory look, so thankfully Adam Newhouse [stepped in to fill the gap](https://gist.github.com/anewhouse/9ad5cf751f63545cc511ab3b4d6efd52). Being the lazy (former) Macadmin that I am, I shamelessly stole it and started trying to make it work for an autopkg run using [jamf-upload](https://github.com/grahampugh/jamf-upload).

The first step is to run autopkg and create a report, using : `--report-plist=autopkg.plist`. From there, you can get a list of all the summary_result keys for your autopkg run. Here's mine:
```
% cat autopkg/1autopkg.plist | grep summary_result
<key>summary_results</key>
<key>jamfpackagecleaner_summary_result</key>
<key>jamfpackageuploader_summary_result</key>
<key>jamfpatchchecker_summary_result</key>
<key>jamfpatchuploader_summary_result</key>
<key>jamfpolicyuploader_summary_result</key>
<key>pkg_copier_summary_result</key>
<key>pkg_creator_summary_result</key>
<key>url_downloader_summary_result</key>
```
It seems, to me, that the most basic usage of jamf-upload would be to upload new versions of packages. In order to get the data we need from the summaries, we need to first get the keys offered, then select the individual values we want for our notification. This is relatively simple, and only needs to be done to set up additional sections in the script (assuming you like how this script works).
```
% plutil -extract summary_results.jamfpackageuploader_summary_result.data_rows.0 raw -o - autopkg.plist
category
name
pkg_display_name
pkg_name
pkg_path
version

% plutil -extract summary_results.jamfpackageuploader_summary_result.data_rows.0.name raw -o - autopkg.plist
VSCode

% plutil -extract summary_results.jamfpackageuploader_summary_result.data_rows.0.version raw -o - autopkg.plist
1.94.1

% plutil -extract summary_results.jamfpackageuploader_summary_result.data_rows.0.pkg_name raw -o - autopkg.plist
VSCode-1.94.1.pkg
```

I mostly only care about package uploads, policy uploads and patch policy uploads, but decided to add package clean stats, also, so I can ensure old versions are being deleted to save space. I tried to make it easy to add sections, if needed, and I'm sure someone will clean this up to make it better. But the best part is, this works with old-style Teams webhooks and the new Workflows.
<details><summary markdown="span"> autopkg-teams-workflows.zsh </summary>{% gist caf0c7181477269043aec183ef580921 %}</details>

And the results are vastly improved.
<br/><img src="/assets/images/teams-pretty.png" class="responsive">

This is a rough, non-production example script. Please, for the love of all that is good and proper, do not use this in production without a lot of testing. But it is my hope that this can help out the community. 

I'm still looking to leave my [autopkg children](https://github.com/lazymacadmin/UpdateTitleEditor) in someone's hands. In time, I'll archive the Github repository so that anyone who wants to can fork it, if I don't find someone who wants to take it over. 
