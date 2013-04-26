# Customing Bee's UI

The external systems which your plugin interacts with will often have different 
terminology to the default terms used within Bee's UI.

E.g. the terms 'Component' and 'Milestone' may be represented differently in 
your external system.

Bee provides a way to customise these terms by taking advantage of the Cocoa 
localization system.

1. In Xcode, create a new 'Strings' file inside your plugin bundle named 
   `Localizable.strings`.

   <a href="xcode_strings.jpg"><img src="xcode_strings.jpg" width="500px" 
   height="auto" /></a>

2. Select the file in Xcode and hit the 'Localize...' button the Utilities pane.

3. Select the language you wish to add the localization for.

   <a href="xcode_localize.jpg"><img src="xcode_localize.jpg" width="500px" 
   height="auto" /></a>

4. You can now start adding localizations your strings file.  You can add 
   alternative strings for the following:
   - Component
   - Components
   - Milestone
   - Milestones
   - Label
   - Labels
   - Type
   - Types
   - User
   - Users
   - Assigned User
   - Created By User
