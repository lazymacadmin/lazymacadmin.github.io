---
layout: post
title:  "Patch-less Autopatching with Jamf"
comments: false
meta: JamfPro, API, jamf-upload, patch-management, automation, autopkg, autopkg processors, autopkg recipes
---
I'm not great at coming up with great ideas, but I'm pretty decent at taking other people's ideas and making them automated. Because there always has to be an easier way, and as lazy Mac admin, that appeals to me.

Today's automation is based on the works of 2 really smart and talented Mac admins: [John Mahlman](https://yearofthegeek.net/) and [Anthony Reimer](https://maclabs.jazzace.ca). 

The idea of being able to run scripts as part of a patching strategy - even as simple as unloading and reloading a launch agent - as well as setting a patch title's latest version (regardless of version) makes things appealing. After all, It's easy to have [autopkg](https://github.com/autopkg/autopkg) upload a package and create a policy in Jamf. That's already a part of a standard workflow. There's even strategies to [automatically update patch titles](https://lazymacadmin.github.io/2023/09/11/automated-jamf-patch-workflow.html) that may not work in all situations.

I remembered seeing Anthony mention he had written a [processor](https://github.com/jazzace/grahampugh-recipes/blob/main/JamfUploaderProcessors/JamfPatchTitleVersioner.py) to get the latest patch version from Jamf, and realized that combining John's [post](https://yearofthegeek.net/posts/Run-Policies-With-Jamf-Patch-Management/) with that processor could make patching simple. When I first ran it, it failed authentication, likely due to changes in [jamf-upload](https://github.com/grahampugh/jamf-upload)'s codebase to deal with Jamf's newer auth schemes. So I grabbed a copy and went to changing things up.

The result is easy. Simply run a [patch recipe](https://github.com/lazymacadmin/UpdateTitleEditor/tree/main/Recipes/JamfPatchTitleVersioner) and your fake patch package will be associated with the patch version. 

So how is this different from my previous auto-patching method (which I still use for my campus's on-prem Jamf server)? In that method, patch titles are only updated when there is a new version uploaded and that installer package is associated with the patch version. For the University's cloud Jamf instance, where each college has a site, this allows centralized packaging of the newest versions but each college can easily and with minimal setup run their own patch-only autopkg server - only 2 repositories required.

Because there has to be an easier way.