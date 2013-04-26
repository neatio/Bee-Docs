# Xcode Project Setup

1. Create a new project in Xcode of type 'Bundle', found by first selecting
   'Framework & Library' underneath the 'OS X' section in the left-hand pane.

2. Fill out the name and other details of your project, and link against the
   `Core Foundation` framework.

3. Go to the 'Build Settings' of your target and change the following:
	- 'Wrapper Extension' (`WRAPPER_EXTENSION`) should be set to `beeplugin`.
		- This is so the OS and Bee recognises that this is a plugin that Bee
		should load.
	- 'Runpath Search Paths' (`LD_RUNPATH_SEARCH_PATHS`) should be set to
	`@executable_path/../Frameworks`.
		- Your plugin links against the `BeeKit` framework (which we'll get to
		in a sec.).  This specifies that your plugin should use the
		`BeeKit` which is shipped inside of Bee.
	- 'Per-configuration Build Products Path' (`CONFIGURATION_BUILD_DIR`), the
	_'Debug'_ value should be set to
	`$(USER_LIBRARY_DIR)/Containers/io.neat.Bee/Data/Library/Application
	Support/io.neat.Bee/PlugIns`
		- This places your debug builds in a location that Bee can find, and
		helps with debugging.

4. Now let's link against `BeeKit`.  Download a copy of `[BeeKit](https://github.com/NeatIO/BeeKit-build)`
   and place it somewhere in your project's directory.

   Then drag the `BeeKit.framework` folder into your project hierarchy in Xcode
   somewhere.  Don't 'Copy items into destination group's folder' because its
   already in your folder.  And make sure it is added to your project's target.
 
   <a href="link-beekit.jpg"><img src="link-beekit.jpg" width="500px" 
   height="auto" /></a>

   Xcode should have configured 'Framework Search Paths'
   (`FRAMEWORK_SEARCH_PATHS`) for you automatically to include
   `"$(SRCROOT)/<BeeKit location relative to your project root>"`

   > We're not including the `BeeKit` framework in any other build phases.
   > Typically you might include a 3rd party framework inside of a 'Copy Files'
   > build phase but in this case, the framework is provided by Bee at runtime
   > so this is unnecessary.  It also means we'll always be dynamically linked
   > to the latest version of `BeeKit`.

   Below is an example of all the configured options.

   <a href="final-project-settings.jpg"><img src="final-project-settings.jpg" 
   width="500px" height="auto" /></a>

5. The last step is to configure the scheme so that you can debug your plugin
   running within the context of Bee.

   Navigate to 'Product' > 'Scheme' > 'Edit Scheme'.

   Ensure 'Run' is selected in the left-hand pane and then in the 'Executable'
   drop-down, navigate to and select `Bee.app`.

   <a href="final-scheme-settings.jpg"><img src="final-scheme-settings.jpg" 
   width="500px" height="auto" /></a>

   This allows us to just hit 'Build & Run' and Xcode will then launch Bee
   whilst LLDB is attached to our plugin, meaning we can configure breakpoints
   and see console output whilst our plugin is in use.

# ReactiveCocoa

`BeeKit` ships with a copy of the 
`[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)` framework 
included.

To link your project to `ReactiveCocoa`, do the following:

1. In Finder, navigate to your `BeeKit.framework` directory and go into the 
   following directory path:
	- `BeeKit.framework/Versions/Current/Frameworks/ReactiveCocoa.framework`
	
2. That's right, we've got a framework inside of a framework!  Once you've found 
   the `ReactiveCocoa.framework` directory, drag it into the Xcode project 
   window, similar to what you did with the `BeeKit.framework` directory.

3. Now whenever you want to use anything from `ReactiveCocoa`-land, just import 
   it:

		#import <ReactiveCocoa/ReactiveCocoa.h>

## Next steps

If you like, you can hit 'Build & Run' now to see your plugin within Bee.  You
can even create a new project right now.  You just won't be able to create items
yet.

Check out the [Plugin Writing Guide](Plugin-Writing-Guide) for what to do next!
