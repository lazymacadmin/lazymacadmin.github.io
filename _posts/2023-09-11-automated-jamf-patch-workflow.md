---
layout: post
title:  "Automated Jamf Patch Workflows"
comments: true
meta: JamfPro, API, jamf-upload, patch-management, automation, autopkg, autopkg processors, autopkg recipes
---

* [Jamf Patch Management Overview](#overview-of-jamf-patch-management)
* [Fixing Patch With Automation](#automating-patch)
* [TL;DR](#tldr)
* [Links](#links)
{% if page.comments %} * [Comments](#Comments) {% endif %}
<br>
<hr>

> _Disclaimer: I am not a Jamf Cloud customer, so I am not touching on whether or not App Installers is better, worse, or even comparable.<br>Until Jamf allows on-premise customers to use all aspects of Jamf Pro I can't give any consideration to App Installers._


#### **Overview of Jamf Patch Management**

Jamf's Patch Management is, on paper, a dream for an admin. Curated definitions of all versions for a given software. The ability to have one or more patch policies to distribute a given version to a given group. The ability to pop up a notification, then close the app and update it. When it's described to you, it comes across as the perfect solution to keeping all of the apps in your fleet up to date.

And it can be, but out of the box here is the typical workflow.
- Download an app or installer in .dmg or .pkg format
  - If a .dmg, use Composer or your favorite packaging tool to create a .pkg installer
- Upload the installer package to your Jamf distribution point
- Go to Patch Management and select the patch title
- Go to the Definition tab, edit, and manually select the package that coincides with each or any version(s)
- Save the Patch title, go to the Patch Policies tab and create or select the Patch Policy
- Enter a patch policy name, select the version to deploy, decide if you want to prompt users to manually install the patch, add machines on the Scope tab, and customize the user interaction to suit your tastes
- Move on to the next app on your list

That doesn't sound too bad, if patching is your only job. But for the rest of us, you could be looking at 10-15 minutes of manual work per app. And it will need to be done for. every. version.<br>
![picture alt](/assets/images/not_doing_that_meme.png 'A meme showing 2 frames - in the first a woman looks, wide-eyed. In the second she says "Yeah, I'm not doing all that."')

#### **Automating Patch**

The key to improving Patch is automating everything we can. Package creation? Automate with [Autopkg](https://github.com/autopkg/autopkg). There a ton of people who use and [describe](https://amsys.co.uk/introduction-autopkg-2) how to use autopkg, so I'm not going to get too far into the weeds on that. There are still more ways to automatically run autopkg [from a script](https://derflounder.wordpress.com/category/autopkg-conductor/) or [from a repository](https://grahamrpugh.com/2020/07/10/gitlab-runner-and-autopkg.html). And if you use some obscure piece of software that isn't Jamf's patch catalog or you want to maintain your own definitions, well, there's an [autopkg processor](https://github.com/lazymacadmin/UpdateTitleEditor/blob/main/Processor/UpdateTitleEditor.py) for that too.

And that gets us to an advanced, autopatching workflow. Using autopkg, Graham Pugh's [JamfUploader](https://github.com/autopkg/grahampugh-recipes/tree/main/JamfUploaderProcessors) processors to extend autopkg by [uploading built packages](https://github.com/autopkg/grahampugh-recipes/blob/main/JamfUploaderProcessors/JamfPackageUploader.py) automatically and [updating patch policies](https://github.com/autopkg/grahampugh-recipes/blob/main/JamfUploaderProcessors/JamfPatchUploader.py), we can set ourselves up to automate the whole process.

Here is what those steps will look like:
- Use `autopkg search` to find recipes
  - Write any recipes needed (i.e. `.jamf` recipes)
- `autopkg make-override` for each .recipe
- Modify the overrides with the variables we need for our own setup
- Create templates for any policies/patch policies, and harvest any needed icon files
- Run autopkg and watch your workload drop

Yes it takes time to set this all up, but the important part is this: Only the final step needs to be done repeatedly, freeing up your time for other tasks because you ~~never have to touch Jamf Patch Management again~~ only have to touch Patch in specific circumstances.

This is where PI111395 (updating the patch policies via the API has a different timing for when the patch policies recalculate their scope to then deploy) rears it's head. So how is the timing changed? 
> Currently it can vary depending on various factors like the number of scoped devices or other patch policies running as well but in general, we've seen that the Patch Policies recalculate their scope the Macs update inventory AND THEN check-in. We could maybe speed that up by running  2 `sudo jamf recon` commands on the Macs."[^1]

So when do we need to manually touch patch policies? Well, for most patches, a slight delay in deployment isn’t the end of the world. For patches with CVEs or where there is a business need for the latest and greatest, there is a workaround: manually edit and save those patch policies in the gui.

#### **TL;DR**

Autopkg and the referenced processors can make your automated Patch workflow as simple as:
- autopkg run [SampleApp.jamf](https://github.com/lazymacadmin/UpdateTitleEditor/blob/main/AutopatchSampleRecipes/SampleApp.jamf.recipe.yaml)
- autopkg run [Sleep](https://github.com/onecheapgeek/UpdateTitleEditor/blob/main/AutopatchSampleRecipes/Sleep.recipe.yaml) (if using Title Editor, so your Jamf server catches the new version)
- autopkg run [SampleApp.patch](https://github.com/lazymacadmin/UpdateTitleEditor/blob/main/AutopatchSampleRecipes/SampleApp.patch.recipe.yaml)

For multiple recipes, create a recipe list file with one per line and your automation can be as simple as a LaunchAgent running `autopkg run -l recipeList.txt` on a schedule. That's it - autopatching with autopkg and Jamf's Patch Management. I'm cheesy, so I call it **_autopatchg_**.

And here's how it looks when complete:
```zsh
% autopkg run Edge.jamf Sleep Edge.patch 

The following packages were copied:
    Pkg Path                                                                                      
    --------                                                                                      
    /Users/autopkg/autopkg/cache/local.jamf.edge/Microsoft_Edge_116.1938.23090676.pkg  

The following packages were uploaded to or updated in Jamf Pro:
    Pkg Path                                                                           Pkg Name                              Version    
    --------                                                                           --------                              -------    
    /Users/autopkg/autopkg/cache/local.jamf.edge/Microsoft_Edge_116.1938.23090676.pkg  Microsoft_Edge_116.1938.23090676.pkg  116.1938.23090676            

The following policies were created or updated in Jamf Pro:
    Policy                  Template                                                                   Icon      
    ------                  --------                                                                   ----      
    Install Microsoft Edge  /Users/autopkg/Library/AutoPkg/RecipeRepos/recipes/JamfPolicyTemplate.xml  edge.png  

The following patch policies were created or updated in Jamf Pro:
    Patch Id  Patch Policy Name  Patch Softwaretitle  Patch Version      
    --------  -----------------  -------------------  -------------      
    189       Edge ASAP          Edge                 116.1938.23090676  
```
#### **Links**
- [Autopkg](https://github.com/autopkg/autopkg)
- [Graham Pugh's autopkg recipe repository for JamfUploader processors](https://github.com/autopkg/grahampugh-recipes/tree/main)
- [UpdateTitleEditor repository](https://github.com/lazymacadmin/UpdateTitleEditor)

----
[^1]: This is literally all the guidance I got from Jamf to my support request asking about what I was seeing. If this affects you, please reach out to support and reference this PI.
