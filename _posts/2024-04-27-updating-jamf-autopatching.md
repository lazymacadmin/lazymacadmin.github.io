---
layout: post
title:  "Updating how I (would) set up Jamf Autopatching today"
comments: true
meta: JamfPro, API, jamf-upload, patch-management, automation, autopkg, autopkg processors, autopkg recipes
---

Recently, Graham Pugh released a new autopkg processor, [JamfPatchChecker](https://github.com/grahampugh/jamf-upload/blob/main/JamfUploaderProcessors/READMEs/JamfPatchChecker.md). It works by comparing the version found by autopkg and comparing it to Jamf's Patch Management titles. With it, you can exit out of the recipe run with a StopProcessingIf processor until the latest version is available, potentially speeding up your autopackage runs by not uploading a package until the patch version is available.

In the past, I described how I [autopatched](https://lazymacadmin.github.io/2023/09/11/automated-jamf-patch-workflow.html) my on-prem Jamf Pro server. I also blogged about how I configured [autopatching](https://lazymacadmin.github.io/2024/02/12/patchless-autopatching-with-jamf.html) for a multi-site cloud server. Those ended up being, in some ways, very cobbled-together and rough around the edges. This most recent attempt addresses some of the very real shortcomings of both of those methods.

### Why Change What Is Working?
My original autopatch required 2 sets of recipes. That is 2 sets to maintain: .jamf recipes run first, to handle package creation, upload, and policy regeneration; .patch recipes run later to handle linking patch versions and policies to previously uploaded packages. There were a lot of moving parts involved, and sometimes an error (or a timeout) in a .patch run would prevent the patch version from being linked, and since a new package wouldn't be uploaded on the next run, new patch versions weren't linked. This required manual intervention to correct, and there has to be an easier way.

My second version had a different potential shortcoming: occasionally a patch version could be reported but we hadn't uploaded a new package associated with it. In those cases, the patch version would actually install the wrong version. I attempted to reduce that likelihood by only running .patch recipes twice a day, but the potential is still there. Thankfully, PI111395 (updating the patch policies via the API has a different timing for when the patch policies recalculate their scope to then deploy) works in our favor there because the patch policy scope updates slower, giving time for things to catch up.

### How Is This Process different?
With the new processor in play, we can actually streamline everything, and only upload the package if a patch version is associated. With Jamf's patch management definitions, this can be a little slower, but if you run your own Title Editor instance, as soon as you find a new version, you can continue (subject to Jamf's 5 minute delay polling for new patch titles). To handle that, I have written a modified version of [StopProcessingIf](https://github.com/autopkg/autopkg/blob/master/Code/autopkglib/StopProcessingIf.py) as [SleepIf](https://github.com/lazymacadmin/UpdateTitleEditor/blob/main/Processor/SleepIf.py). 

In this run, if the patch version doesn't exist yet, the first StopProcessingIf exits the run. There is no need to continue (in this case), because the patch version can't be updated. If the patch version does exist, and the version matches, the recipe attempts to upload the package. If the package upload isn't completed, either due to an error or because it already exists, the second StopProcessingIf ends the run. If the package is new (or `-k replace_pkg=True`), a Jamf policy is updated, along with the patch title and patch policy. 

If the patch version doesn't exist for the version found, the SleepIf processor will pause the run for a specified time (default is 5 seconds, my recipes are set for 5 minutes). This should give Jamf time to catch up, then we can upload the package, and link the patch version and update our patch policy.

### What does this look like, practically?

This is the `Process` section of one of my new format recipes. If you don't use Title Editor, you can simply leave out the first processor:
```yaml
Process:
-   Processor: really.cool.Processors/UpdateTitleEditor
    Arguments:
        title_id: '1001'

-   Processor: com.github.grahampugh.jamf-upload.processors/JamfPatchChecker
    Arguments:
      patch_softwaretitle: '%PATCH_SOFTWARE_TITLE%'
      pkg_name: '%NAME%-%version%.pkg'

-   Processor: StopProcessingIf
    Arguments:
      predicate: patch_version_found == False

-   Processor: really.cool.Processors/SleepIf
    Arguments:
      predicate: patch_version_found == False
      sleep_time: 300

-   Processor: com.github.grahampugh.jamf-upload.processors/JamfPackageUploader
    Arguments:
      pkg_name: '%NAME%-%version%.pkg'
      pkg_category: '%CATEGORY%'

-   Processor: StopProcessingIf
    Arguments:
      predicate: pkg_uploaded == False

-   Processor: com.github.grahampugh.jamf-upload.processors/JamfPolicyUploader
    Arguments:
      policy_name: '%POLICY_NAME%'
      policy_template: '%POLICY_TEMPLATE%'
      policy_category: '%CATEGORY%'
      icon: '%SELF_SERVICE_ICON%'
      replace_icon: 'True'
      replace_policy: 'True'

-   Processor: com.github.grahampugh.jamf-upload.processors/JamfPatchUploader
    Arguments:
      grace_period: 120
      patch_name: '%PATCH_POLICY_NAME%'
      patch_softwaretitle: '%PATCH_SOFTWARE_TITLE%'
      patch_template: '%PATCH_POLICY_TEMPLATE%'
      patch_icon_policy_name: '%POLICY_NAME%'
      replace_patch: 'True'
``` 

### What does this all look like at runtime?
Glad you asked. I have taken different run options and compiled them into a helpful [gist](https://gist.github.com/lazymacadmin/b75f2e6d89b2b5a4ce8a92d93a9e2333).

Happy patching!

Oh, and let me know if you think there has to be an easier way.