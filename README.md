xUnique
=======

_xUnique_, is a pure Python script to regenerate `project.pbxproj`, a.k.a the Xcode project file, and make it unique and same on any machine.
 
As you may know, the UUID generated by Xcode (a.k.a [rfc4122](http://www.ietf.org/rfc/rfc4122.txt)) in the file is not unique for the same added file( or other entries like groups,build phases,etc.) on different machines, which makes it a developer's nightmare to merge and resolve conflicts in `project.pbxproj`.

_xUnique_ convert all the 96bits `UUID`(24 hex chars) to MD5 hex digest(32 hex chars), and Xcode do recognize these MD5 digests.

### What it does & How it works
----------------
1. convert `project.pbxproj` to JSON format
2. Iterate all `objects` in JSON and give every UUID an absolute path, and create a new UUID using MD5 hex digest of the path
	* All elements in this json object is actually connected as a tree
	* We give a path attribute to every node of the tree using its unique attribute; this path is the absolute path to the root node, 
	* Apply MD5 hex digest to the path for the node
3. Replace all old UUIDs with the MD5 hex digest and also remove unused UUIDs if any
4. Sort the project file inlcuding `children`, `files`, `PBXFileReference` and `PBXBuildFile` list and remove all duplicated entries in these lists 
	* see `sort_pbxproj` method in xUnique.py if you want to know the implementation;
	* It's ported from my modified [`sort-Xcode-project-file`](https://github.com/truebit/webkit/commits/master/Tools/Scripts/sort-Xcode-project-file), with some differences in ordering `PBXFileReference` and `PBXBuildFile` 
5. With different [options](#supported-argument-options), you can use _xUnique_ with more flexibility


### How to use
--------------
There are many ways to use this script. I will introduce two types:

##### Xcode "build post-action" (Recommended)
  1. open `Edit Scheme` in Xcode (shortcut: `⌘`+`Shift`+`,`)
  2. choose the sheme you use to run your project
  3. expand `Build`, select `Post-actions`
  4. click symbol `+` on the left bottom corner of the right pane
  5. choose `New Run Script Action`
  6. choose your selected sheme name in `Provide build settings from`
  7. input commands below (assume that `xUnique.py` is under directory of `/project/dir/Scripts`):
  
     ```bash
     python "${PROJECT_DIR}/Scripts/xUnique.py" "${PROJECT_FILE_PATH}/project.pbxproj"
     ```
  8. click `Close` and it's all done.
  9. Next time when you Build or Run the project, xUnique would be triggered after build success. If the build works, you could commit all files.
  10. Demo gif animation is [here](#add-xunique-to-xcode-post-action)
     
##### Git hook
  1. Put **xUnique.py** file in your project repository somewhere and add it as track file via `git add path/to/xUnique.py`, so all members could use the same script
  2. create a git hook like `{ echo '#!/bin/sh'; echo 'python2 path/to/xUnique.py path/to/MyProject.xcodeproj'; } > .git/hooks/pre-commit`
  3. Add permission `chmod 755 .git/hooks/pre-commit` 
  4. xUnique will be triggered when you trying to commit:
  	* Using option `-c` in command would fail the commit operation if project file is modified. Then you can add the modified project file and commit all the files again.
  	* Option `-c` is not activated by default. The commit operation will proceed successfully even if the project file is modified by xUnique. So do not push the commit unless you add the modified project file again and do another commit.

#### Supported argument options
* `-v`: print verbose output, and generate `debug_result.json` file for debug.
* `-u`: uniquify project file, that is, replace UUID to MD5 digest.
* `-s`: sort project file inlcuding `children`, `files`,`PBXFileReference` and `PBXBuildFile` list and remove all duplicated entries in these lists. Supports both original and uniquified project file. 
* `-p`: sort `PBXFileReference` and `PBXBuildFile` sections in project file ordered by file names. Only works with `-s`. Befor v4.0.0, this was hard-coded in `-s` option and cannot be turned off. Starting from v4.0.0, without this option along with `-s`, xUnique will sort these two types by MD5 digests, the same as Xcode does.
* `-c`: When project file was modified, xUnique quit with non-zero status. Without this option, the status code would be zero if so. This option is usually used in Git hook to submit xUnique result combined with your original new commit.
* if neither `-u` nor `-s` exists, `-u -s` will be appended to existing option list.

    
#### Git merge strategy
* There are still conflicts after using xUnique. That's because the incorrect merge strategy used by Git against project file.
* use [Git Attributes](http://git-scm.com/book/en/Customizing-Git-Git-Attributes) to make git know how to deal with `project.pbxproj`:
  * create a file named `.gitattributes` in git repo root directory (a.k.a the same level with `.git` directory) if it does not exist
  * add line `*.pbxproj text -crlf -diff -merge=union` in the file
  * save file and add it to git repo
      

### Examples
------------
* [APNS Pusher](https://github.com/blommegard/APNS-Pusher) is a Xcode project which contains a subproject named "Fragaria" as git submodule. Use _xUnique_ to convert it. You can clone [my forked repo](https://github.com/truebit/APNS-Pusher) and try to open and build it in Xcode. You will find that `xUnique` does not affect the project at all.
  * The initial diff result could be found [here](https://github.com/truebit/APNS-Pusher/commit/fb27af54627ca0836aa5eb847766441b991220bf).
  * The diff result with my modified [`sort-Xcode-project-file`](https://github.com/truebit/webkit/blob/7afa105d20fccdec68d8bd778b649409f17cbdc0/Tools/Scripts/sort-Xcode-project-file) with `PBXBuildFile` and `PBXFileReference`sort support could be found [here](https://github.com/truebit/APNS-Pusher/commit/d5ff3dc053c4be96d6c209cc9ced890faad263c9). 
  * Pure python sort result could be found [here](https://github.com/truebit/APNS-Pusher/commit/f79d182b0b5892cbb889b67242845807689bd5e4)

###### add xUnique to Xcode post action
![xUnique_Build_Post_Action](https://raw.github.com/truebit/xUnique/gif/xUnique_Build_Post_Action.gif)

### NOTICE
----------
* All project members must add the build post-action or git hook. Thus the project file would be consistent in the repository.
* Tested supported `isa` types:
    * `PBXProject`
    * `XCConfigurationList`
    * `PBXNativeTarget`
    * `PBXTargetDependency`
    * `PBXContainerItemProxy`
    * `XCBuildConfiguration`
    * `PBXSourcesBuildPhase`
    * `PBXFrameworksBuildPhase`
    * `PBXResourcesBuildPhase`
    * `PBXFrameworksBuildPhase`
    * `PBXCopyFilesBuildPhase`
    * `PBXBuildFile`
    * `PBXReferenceProxy`
    * `PBXFileReference`
    * `PBXGroup`
    * `PBXVariantGroup`


### Authors
-----------
* Xiao Wang ([seganw](http://fclef.wordpress.com/about))

### Contributions
-----------------
* I only tested on one single project and one project with a subproject, so maybe there should be more unconsidered conditions. 
If you get any problem, feel free to fire a Pull Request or Issue

* You can also buy me a cup of tea: [![Donate to xUnique](https://www.paypalobjects.com/en_US/i/btn/btn_donate_SM.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=QQNATFYESVT76&item_name=xUnique)



### License
-----------
Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy of
the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.
