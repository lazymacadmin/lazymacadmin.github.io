---
layout: post
title:  "Minor Functionality Update for UpdateTitleEditor.py"
comments: true
meta: JamfPro, UpdateTitleEditor, jamf-upload, patch-management, automation, autopkg, autopkg processors, autopkg recipes
---

Today I pushed a minor change to [UpdateTitleEditor](https://github.com/lazymacadmin/UpdateTitleEditor), in an attempt to make recipes incorporating it more flexible.  I've noticed that Grahama Pugh uses true/false values in a autopkg environment variables (i.e. `pkg_uploaded`) to work with `StopProcessingIf` processors, and have really found myself liking the flexibility, so I decided to work `title_updated` in to be used as a predicate. Now a `.jamf` recipe can look this to:
- Update Title Editor
- If the version is not new, exit, If it is:
   - Upload the package.
   - If that does not succeed, exit. If it does:
      - Update a policy

```yaml
Process:
-   Processor: really.cool.Processors/UpdateTitleEditor
    Arguments:
        title_id: '1001'
        
-   Arguments:
        predicate: title_updated == False
    Processor: StopProcessingIf

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
```

```xml
<key>Process</key>
<array>
    <dict>
        <key>Processor</key>
        <string>really.cool.Processors/UpdateTitleEditor</string>
        <key>Arguments</key>
        <dict>
            <key>title_id</key>
            <integer>1001</integer>
        </dict>
    </dict>
    <dict>
        <key>Arguments</key>
        <dict>
            <key>predicate</key>
            <string>title_updated == False</string>
        </dict>
        <key>Processor</key>
        <string>StopProcessingIf</string>
    </dict>
    <dict>
        <key>Processor</key>
        <string>com.github.grahampugh.jamf-upload.processors/JamfPackageUploader</string>
        <key>Arguments</key>
        <dict>
            <key>pkg_name</key>
            <string>%NAME%-%version%.pkg</string>
            <key>pkg_category</key>
            <string>%CATEGORY%</string>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>StopProcessingIf</string>
        <key>Arguments</key>
        <dict>
            <key>predicate</key>
            <string>pkg_uploaded == False</string>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>com.github.grahampugh.jamf-upload.processors/JamfPolicyUploader</string>
        <key>Arguments</key>
        <dict>
            <key>policy_name</key>
            <string>%POLICY_NAME%</string>
            <key>policy_template</key>
            <string>%POLICY_TEMPLATE%</string>
            <key>policy_category</key>
            <string>%CATEGORY%</string>
            <key>icon</key>
            <string>%SELF_SERVICE_ICON%</string>
            <key>replace_icon</key>
            <string>True</string>
            <key>replace_policy</key>
            <string>True</string>
        </dict>
    </dict>
</array>
```

It can speed up recipe runs by exiting if the version hasn't changed, but won't affect your use case if you won't want to. Most importantly, it's easy. 

Because there has to be an easier way.