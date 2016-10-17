# Blender Addon Updater

With this python module, developers can create auto-checking for updates with their blender addons as well as one-click version installs. Updates are retrieved using GitHubs code api, so the addon must have it's updated code available on GitHub and be making use of either GitHub tags or releases.

**This code is close but not yet production ready**
*This notice will change when ready for external use*

# Key Features
*From the user perspective*

- Uses GitHub repositories for source of versions and code
  - In the future, may have support for additional or custom code repositories
- One-click to check if update is available
- Auto-check: Ability to automatically check for updates in the background (user must enable)
- Ability to set the interval of time between background checks (if auto-check enabled)
- On a background check for update, contextual popup to tell user update available
- One-click button to install update
- Ability to install other (older) versions of the addon



# High level setup

This module works by utilizing git releases on a repository. When a [release](https://github.com/CGCookie/blender-addon-updater/releases) or [tag](https://github.com/CGCookie/blender-addon-updater/tags) is created on GitHub, the addon can check for and update to the code included in that tag. The local addon version number is checked against the versions on GitHub based on the name of the release or tag itself. 

![alt](/images/file_diagram.png)

This repository contains a fully working example of an addon with the updater code, but to integrate into another or existing addon, only the `addon_updater.py` and `addon_updater_ops.py` files are needed. 

`addon_updater.py` is an independent python module that is the brains of the updater. It is implemented as a singleton, so the module-level variables are the same wherever it is imported. This file should not need to be modified by a developer looking to integrate auto-updating into an addon. Local "private" variables starting with _ have corresponding @property interfaces for interacting with the singleton instance's variables.

`addon_updater_ops.py` links the states and settings of the `addon_updater.py` module and displays the according interface. This file is expected to be modified accordingly to be integrated with into another addon, and serves mostly as a working example of how to implement the updater code. 

In this documentation, `addon_updater.py` is referred to by "the Python Module" and `addon_updater.py` is referred to by "the Operator File".

# About the example addon

Included in this repository is an example addon which is integrates the auto-updater feature. It is currently linked to this repository and it's tags for testing. To use in your own addon, you only need the `addon_upder.py` and `addon_updater_ops.py` files. Then, you simply need to make the according function calls and create a release or tag on the corresponding GitHub repository.

# Step-by-step as-is integration with existing addons


0) Copy the Python Module (addon_updater.py) and the Operator File (addon_updater_ops.py) to the root folder of the existing addon folder

1) import the updater operator file in `__init__` file e.g. `from . import addon_updater_ops`

2) Run the register function on the updater module in the addon's def register() function, e.g. `addon_updater_ops.register(bl_info)`. Consider trying to place the updater register near the front of the addon along with any preferences function so that if the user updates/reverts to a non-working version of the addon, they can still use the updater to revert

3) Edit the according fields in the register function of the `addon_updater_ops.py` file

3) To get the updater UI in the preferences draw panel, add the line `addon_updater_ops.update_settings_ui(self,context)` to the end of the preferences class draw function (be sure to import the addon_updater_ops file if in a file other than the addon's `__init__` file where already imported)

5) Add the needed blender properties to make the sample updater preferences UI work by copying over the blender properties from the sample demo addon's `DemoPreferences` class, located in the `__init__` file

```
# addon updater preferences from `__init__`, be sure to copy all of them

    auto_check_update = bpy.props.BoolProperty(
        name = "Auto-check for Update",
        description = "If enabled, auto-check for updates using an interval",
        default = False,
        )
    
    ....

    updater_intrval_minutes = bpy.props.IntProperty(
        name='Minutes',
        description = "Number of minutes between checking for updates",
        default=0,
        min=0,
        max=59
        )

```

6) Add the draw call to an according panel to indicate there is an update by adding this line to the end of the panel or window: `addon_updater_ops.update_notice_box_ui()`, again making sure to import the addon_updater_ops module if this panel is defined in a file other than the addon's `__init__` file.

7) Ensure at least one [release or tag](https://help.github.com/articles/creating-releases/) exists on the GitHub repository


# Minimal example setup / use cases

If interested in implemented a purely customized UI implementation of this code, it is also possible to not use the included Operator File. This section covers the typical steps required to accomplish the main tasks and what needs to be connected to an interface. This also exposes the underlying ideas implemented in the provided files.

**Required settings** *Attributes to define before any other use case, to be defined in the registration of the addon*

```
from .addon_updater import Updater as updater # for example
updater.user = "cgcookie"
updater.repo = "blender-addon-updater"
updater.current_version = bl_info["version"]
```

**Check for update** *(foreground using/blocking the main thread, after pressing an explicit "check for update button")*

```
updater.check_for_update_now()
(update_ready, version, link) = updater.check_for_update()
	
```

**Check for update** *(foreground using background thread, after pressing an explicit "check for update button")*

```
updater.check_for_update_now(callback=None)
```

**Check for update** *(background using background thread, triggered without notifying user - eg via auto-check after interval of time passed)*

```
updater.check_for_update_async(background_update_callback)
# callback could be the code to trigger a popup if result has updater.update_ready == True
```

**Update to newest addon**

```
res = updater.run_update(force=False, revert_tag=None, callback=function_obj)
if res == 0:
	print("Update ran successfully, restart blender")
else:
	print("Updater returned "+str(res)+", error occurred")
```

**Update to a target version of the addon**

```
tag_version = addon.tags[2] # or otherwise select a valid tag
res = updater.run_update(force=False,revert_tag=None, callback=function_obj)
if res == 0:
	print("Update ran successfully, restart blender")
else:
	print("Updater returned "+str(res)+", error occurred")
```



# addon_updater module settings

This section provides documentation for all of the addon_updater module settings available and required. These are the settings applied directly to the addon_updater module itself, imported into any other python file. 

**Example changing or applying a setting:**

```
from .addon_updater import Updater as updater
updater.addon = "addon_name"
```

*Required settings*

- **current_version:** The current version of the installed addon
  - Type: Tuple, e.g. (1,1,0) or (1,1)
- **repo:** The name of the repository as found in the GitHub link
  - Type: String, e.g. "blender-addon-updater"
- **user:** The name of the user the repository belongs to
  - Type: String, e.g. "cgcookie"

*Optional settings*

- **addon:**
  - Type: String, e.g. "demo_addon_updater"
  - Default: derived from the `__package__` global variable, but recommended to change to explicit string as `__package__` can differ based on how the user installs the addon
- **auto_reload_post_update:** If True, attempt to auto disable, refresh, and then re-enable the addon without needing to close blender
  - Type: Bool, e.g. False
  - Default: False
  - Notes: Depending on the addon and class setup, it may still be necessary or more stable to restart blender to fully load. In some cases, this may lead to instability and thus it is advised to keep as false and accordingly inform the user to restart blender unless confident otherwise. 
    - If this is set to True, a common error is thinking that the update completed because the version number in the preferences panel appears to be updated, but it is very possible the actual python modules have not fully reloaded or restored to an initial startup state. 
    - If it is set to True, a popup will appear just before it tries to reload, and then immediately after it reloads to confirm it worked. 
- **fake_install:** Used for debugging, to simulate in the user interface installing an update without actually modifying any files
  - Type: Bool, e.g. False
  - Default: False
  - Notes: Should be only used for debugging, and always set to false for production
- **updater_path:** Path location of stored json state file, backups, and staging of installing a new version
  - Type: String, absolute path location
  - Default: "{path to blender files}/addons/{addon name}/{addon name}_updater/"
- **verbose:** A debugging setting that prints additional information to the console
  - Type: Bool, e.g. False
  - Default: False
  - Notes: Messages will still be printed if errors occur, but verbose is helpful to keep enabled while developing or debugging this code. It may even be worthwhile to expose this option to the user through a blender interface property
- **website:** Website for this addon, specifically for manually downloading the addon
  - Type: String, valid url
  - Default: None
  - Notes: Used for no purpose other than allowing a user to manually install an addon and its update. It should be very clear from this webpage where to get the download, and thus may not be a typical landing page.
  - **backup_current** Create a backup of the current code when performing an update or reversion.

*User preference defined (ie optional but good to expose to user)*

- **check_interval_enable:** Allow for background checking.
- **check_interval_minutes:** Set the interval of minutes between the previous check for update and the next
- **check_interval_hours:** Set the interval of hours between the previous check for update and the next
- **check_interval_days:** Set the interval of days between the previous check for update and the next
- **check_interval_months:** Set the interval of months between the previous check for update and the next

*Internal values (read only)*

- **addon_package:** The package name of the addon, used for enabling or disabling the addon
  - Type: String
  - Default: `__package__`
  - Must use the provided default value of `__package__` , automatically assigned
- **addon_root:** The location of the root of the updater file
  - Type: String, path
  - Default: os.path.dirname(__file__)
- **api_url:** The GitHub API url
  - Type: String
  - Default: "https://api.github.com"
  - Notes: Should not be changed, but in future may be possible to select other API's and pass in custom retrieval functions
- **async_checking:** If a background thread is currently active checking for an update, this flag is set to True and prevents additional checks for updates. Otherwise, it is set to false
  - Type: Bool
  - Default: False
  - Notes:
    - This may be used as a flag for conditional drawing, e.g. to draw a "checking for update" button while checking in the background
    - However, even if the user were to still press a "check for update" button, the module would still prevent an additional thread being created until the existing one finishes by checking against this internal boolean
- **json:** Contains important state information about the updater
  - Type: Dictionary with string keys
  - Default: {}
  - Notes: This is used by both the module and the operator file to store saved state information, such as when the last update is and caching update links / versions to prevent the need to check the internet more than necessary. The contents of this dictionary object are directly saved to a json file in the addon updater folder
- **source_zip:** Once a version of the addon is downloaded directly from the server, this variable is set to the absolute path to the zip file created.
  - Type: String, OS path
  - Default: None
  - Notes: Path to the zip file named source.zip already downloaded
- **tag_latest** Returns the most recent tag or version of the addon online
  - Type: String, URL
  - Default: None
- **tag_names** Returns a list of the names (versions) for each tag of the addon online
  - Type: list
  - Default: []
  - Note: this is analogous to reading tags from outside the Python Module.
- **tags:** Contains a list of the tags (version numbers) of the addon
  - Type: list
  - Default: []
  - Notes: Can be used if the user wants to download and install a version other than the most recent. Can be used to draw a dropdown of the available versions.
- **update_link:** After check for update has completed and a version is found, this will be set to the direct download link of the new code zip file. 
- **update_ready:** Indicates if an update is ready
  - Type: Bool
  - Default: None
  - Notes:
    - Set to be True if a tag of a higher version number is found after checking for updates
    - Set to be False if a tag of a higher version number is not found after checking for updates
    - Set to be None before a check has been performed or cached
    - Using `updater.update_ready == None` is a good check for use in draw functions, e.g. to show different options if an update is ready or not or needs to be checked for still
- **update_version:** The version of the update downloaded or targeted
  - Type: String
  - Default: None
  - Notes: This is set to the new addon version string, e.g. `(1,0,1)` and is used to compare against the installed addon version
- **error:** If an error occurs, such as no internet or if the repo has no tags, this will be a string with the name of the error; otherwise, it is `None`
  - Type: String
  - Default: None
  - It may be useful for user interfaces to check e.g. `updater.error != None` to draw a label with an error message e.g. `layout.label(updater.error_msg)`
- **error_msg:** If an error occurs, such as no internet or if the repo has no tags, this will be a string with the description of the error; otherwise, it is `None`
  - Type: String
  - Default: None
  - It may be useful for user interfaces to check e.g. `updater.error != None` to draw a label with an error message e.g. `layout.label(updater.error_msg)`



# About addon_updater_ops

This is the code which acts as a bridge between the pure python addon_updater.py module and blender itself. It is safe and even advised to modify the addon_updater_ops file to fit the UI/UX wishes. You should not need to modify the addon_updater.py file to make a customized updater experience.

### User preferences UI

![Alt](/images/updater_preferences.png)

Most of the key settings for the user are available in the user preferences of the addon, including the ability to restore the addon, force check for an update now, and allowing the addon to check for an update in the background

### Integrated panel UI

![Alt](/images/integrated_panel.png)

*If a check has been performed and an update is ready, this panel is displayed in the panel otherwise just dedicated to the addon's tools itself. The draw function can be appended to any panel.*

### Popup notice after new update found

![Alt](/images/popup_update.png)

*After a check for update has occurred, either by the user interface or automatically in the background (with auto-check enabled and the interval passed), a popup is set to appear when the draw panel is first triggered. It will not re-trigger until blender is restarted. Pressing ignore on the integrate panel UI box will prevent popups in the future.*


### Install different addon versions

![Alt](/images/install_versions.png)

*In addition to grabbing the code for the most recent release or tag of a GitHub repository, this updater can also install other target versions of the addon through the popup interface.* 



# Issues or help

If you are attempting to integrate this code into your addon and run into problems, please open a new issue. As the module improves, it will be easier for more developers to integrate updating and improve blender's user experience overall! 
