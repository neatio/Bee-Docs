# Walkthrough of writing a Bee plugin

We're going to be building a simple plugin which will let Bee do most of the
heavy lifting.  It will be a local plugin with no network communication but
because all of the relevant API methods are asynchronous, the implementations
can easily be adapted for ones which talk to a remote server.
 
Create a new project in Xcode by following [these instructions](Xcode-Project-Setup).
Once you've done that and have a project ready to go, come back here so we can 
write some code!

## Fetch

1. Create a new file which inherits from `NSObject` and name it 'TasksFetch'.

2. In the interface declaration, import the `BeeKit` framework header and
   declare the class as conforming to the `BeePluginFetch` protocol.
   E.g.:

		#import <BeeKit/BeeKit.h>

		@interface TasksFetch : NSObject <BeePluginFetch>

		@end
	
   _Note: conforming to Bee plugin protocols is completely optional but does
   help to express the intent of your class to others._

3. Now register an instance of this plugin using [BeeKeeper registerPlugin:]
   inside of `+ load`.
   
   Bee loads plugin bundles lazily.  When a bundle is loaded in Cocoa, the
   `+ load` message is sent to all the classes within the bundle.
   
   This is the ideal place to register an instance of our plugin class so that
   Bee knows to use it.

   In your implementation, have the following:

		+ (void)load {
			[BeeKeeper registerPlugin:[self new]];
		}

   _Without this declaration Bee will not be able to send messages to the plugin 
   methods you implement in this class, so be sure to include this._

4. For an item to be created, it needs to be associated with an item type entity 
   at a minimum.  How you define an item type entity is up to the service or 
   type of workflow you are trying to implement.  You can read more about item 
   types in the [Entity Guide](Entity-Guide).

   Let's implement [BeePluginFetch allItemTypesForProject:callback:] to always 
   return a single item type:

		- (void)allItemTypesForProject:(BeeProject *)project callback:(BeePluginCallback)callback {
			BeeItemType *task = (BeeItemType *) [BeeModelHelper entityForName:@"BeeItemType"
																   identifier:@"task"
													  existingMutableSetProxy:project.typesMutableProxy
																	  account:project.account];
			task.name = NSLocalizedString(@"Task", nil);
			task.imageTemplateName = BeeItemTypeTemplateGeneric;

			callback(nil, [NSSet setWithObject:task]);
		}

   The first line looks for an existing `BeeItemType` with the identifier 
   'task'.  If one is not found, `BeeModelHelper` will create one.

   The next two lines are just assigning a name and an image template (a full 
		   list of which can be found in the `BeeItemTypes` header).

   The callback block expects an error (if one exists) and a set of item types,
   which we are providing.  You should always check the docs to see what objects 
   the plugin callback expects to receive.

   At this stage, you can 'Build & Run' and you should be able to create items 
   which have a default item type of 'Task'.

   <a href="bee-empty-details.jpg"><img src="bee-empty-details.jpg" 
   width="500px" height="auto" /></a>

   If you select one of the items, you'll notice that there is no UI being 
   displayed in the right-most pane.  We'll build this UI now.

## UI

1. Create a new file which inherits from `NSObject` and name it 'TasksUI'.

2. In the interface declaration, import the `BeeKit` framework header and
   declare the class as conforming to the `BeePluginUI` protocol.
   E.g.:

		#import <BeeKit/BeeKit.h>

		@interface TasksUI : NSObject <BeePluginUI>

		@end

3. Now register an instance of this plugin using [BeeKeeper registerPlugin:]
   inside of `+ load`.

		+ (void)load {
			[BeeKeeper registerPlugin:[self new]];
		}

4. Bee's UI is made up of the following classes:
	- `BeeForm`
		- Represents the entirety of a particular piece of UI.  All groups and 
		fields which will be displayed to the user ultimately live inside of a 
		BeeForm.
	- `BeeGroup`
		- A grouping of fields.  A form is made up of groups.  Groups have an 
		optional title (which represent different sections within the UI), and 
		they have contain a list of fields.
	- `BeeField`
		- A control which manipulates an entity or an attribute of an entity.

   The UI we want to build right now is the item details pane.  If you look in 
   the `BeePluginUI` header, you'll see the method we're about to implement: 
   [BeePluginUI itemDetailsFormForItemType:].

		- (BeeForm *)itemDetailsFormForItemType:(BeeItemType *)itemType {
			BeeForm *form = [BeeForm new];

			BeeFormGroup *topGroup = [BeeFormGroup new];
			BeeFormGroup *bottomGroup = [BeeFormGroup new];

			BeeFieldText *nameField = [BeeFieldText new];
			nameField.name = @"name";
			nameField.title = NSLocalizedString(@"Name", nil);
			[nameField configureFieldHandlersForKeyPath:@"name" defaultFieldValue:nil];

			BeeFieldAutocomplete *labelsField = [BeeFieldCatalog newLabelsField];

			BeeFieldText *descriptionField = [BeeFieldText new];
			descriptionField.name = @"description";
			descriptionField.placeholder = NSLocalizedString(@"Description", nil);
			[descriptionField configureFieldHandlersForKeyPath:@"detail" defaultFieldValue:nil];
			descriptionField.multiline = YES;

			topGroup.fields = @[ nameField, labelsField ];
			bottomGroup.fields = @[ descriptionField ];

			form.groups = @[ topGroup, bottomGroup ];

			return form;
		}

   - The first thing you'll notice is that this is a synchronous method which 
   expects a `BeeForm` object to be returned.  All UI methods are synchronous 
   and operate on the main thread, as opposed to all the entity mutation methods 
   which are asynchronous and operate in a background thread inside of a 
   separate process.
   - As we're configuring the `nameField`, you can see that we're using the 
   convenience method: [BeeField 
   configureFieldHandlersForKeyPath:defaultFieldValue:].
		- In the above example, we are binding the field to the `name` attribute 
		of a `BeeItem` entity.
   - To create the `labelsField`, we're using the convenience method provided by 
   `BeeFieldCatalog` which returns a configured `BeeFieldAutocomplete` field 
   ready and configured for a `BeeItem's` `labels` relationship.
   - We're ignoring the `itemType` parameter being passed in by Bee but that's 
   because we only have 1 item type.
		- If we had multiple item types, this would provide us with the ability 
		to provide a different form for each one.

5. 'Build & Run' and you'll see we now have a populated form in the details 
   pane, complete with fully functional and bound fields.

   You can even start typing and adding labels to the item and the `BeeLabel` 
   will be created and assigned to the `BeeItem` automatically!

   <a href="bee-details.jpg"><img src="bee-details.jpg" width="500px" 
   height="auto" /></a>

## Workflow

We have the ability to create items and edit item attributes, but what about 
closing items so we know that they're completed, or transitioning them to other 
states?

Let's implement item states and statuses to manage those states.

`BeeItem` has a `state` property which can take one of the following values
(as described in the header):

	typedef enum {
		BeeItemStateOpen = 0,
		BeeItemStateActive,
		BeeItemStateIntermediate,
		BeeItemStateClosed
	} BeeItemState;

When an item is created inside of Bee it has the `BeeItemStateOpen` state.  It 
is up to the plugin to determine what the correct state it should be, but 
`BeeItemStateOpen` is usually the correct state.

When an item is assigned to the current user, and has been started - it is 
assigned the `BeeItemStateActive` state.  The plugin can also customise this,
but this is usually the correct behaviour.

There is also the concept of a 'status'.  Statuses are represented by 
`BeeItemStatus` objects which are associated with `BeeItem`s.  These are a 
textual representation of what the item state is currently representing.

For example, an item could be in an intermediate state 
(`BeeItemStateIntermediate`) and have a `BeeItemStatus` which describes this 
state as 'Resolved'.  Once the `state` changes to `BeeItemStateClosed`, the item 
will be set to a status object with name: 'Closed'.

For our purposes we will be defining two statuses because we are only concerned 
with identifying two states: open and closed.

Note: the number of statuses you have doesn't have to reflect the number of 
states you choose to support.  There are a multitude of different situations 
which could call for different status objects representing the current status of 
an item, but each status sharing the same state.

1. Open `TasksFetch` again and implement [BeePluginFetch allItemStatusesForProject:callback:]:

		- (void)allItemStatusesForProject:(BeeProject *)project callback:(BeePluginCallback)callback {
			BeeItemStatus *openStatus = (BeeItemStatus *) [BeeModelHelper entityForName:@"BeeItemStatus"
																			 identifier:@"open"
																existingMutableSetProxy:project.statusesMutableProxy
																				account:project.account];
			openStatus.name = NSLocalizedString(@"Open", nil);

			BeeItemStatus *closedStatus = (BeeItemStatus *) [BeeModelHelper entityForName:@"BeeItemStatus"
																			   identifier:@"closed"
																  existingMutableSetProxy:project.statusesMutableProxy
																				  account:project.account];
			closedStatus.name = NSLocalizedString(@"Closed", nil);

			callback(nil, [NSSet setWithObjects:openStatus, closedStatus, nil]);
		}

   - Here we're just creating our status objects (if they don't already exist) 
   and then returning them in the callback block.

2. Now let's create a new file, inheriting from `NSObject` named 'TasksMutate'.  
   Follow the same drill as our two other files above and adhere to the 
   `BeePluginMutate` protocol.  Inside `+ load` be sure to register an instance 
   of the plugin with `BeeKeeper`.

3. Inside `TasksMutate`, implement [BeePluginMutate insertItem:formValues:callback:]
   and [BeePluginMutate liveInsertItem:formValues:callback:]:

		- (void)insertItem:(BeeItem *)item formValues:(NSDictionary *)formValues callback:(BeePluginCallback)callback {
			BeeProject *project = item.project;
			item.status = (BeeItemStatus *) [BeeModelHelper entityForName:@"BeeItemStatus"
															   identifier:@"open"
												  existingMutableSetProxy:project.statusesMutableProxy
																  account:project.account];

			callback(nil, nil);
		}

		- (void)liveInsertItem:(BeeItem *)item formValues:(NSDictionary *)formValues callback:(BeePluginCallback)callback {
			[self insertItem:item formValues:formValues callback:callback];
		}

   - Anytime a new item is created within Bee, these two methods are called 
   depending on the method in which the item was inserted.

   - Live-inserts occur when the user hit 'return' inside of a project and 
   inserted a new item inline.
   
   - A regular insert occurs when the new item window is used.

   - In this case, we want both implementations to have the same behaviour so we 
   write our method body inside of one and just call it from the other.

   - In our implementation of [BeePluginMutate insertItem:formValues:callback:] 
   you can see we're just assigning one of the statuses we made sure to create 
   earlier in our fetch method.  All new items created will have the 'Open' 
   status.

   - If you 'Build & Run' your plugin now, be sure to right click on the project 
   in the left-hand pane and select 'Synchronize Project' beforehand, so it is 
   able to create the item statuses you defined.  Then insert a new item and you 
   should see the item being created with the 'Open' status.

4. When you click on the 'Open' status, a menu appears telling you that there 
   are no transitions yet.  Bee needs a way of determining how to transition 
   from one status to the next.  We'll define those now.

   Open `TasksUI` again and let's implement [BeePluginUI transitionStatusMenuForItem:]:

		- (BeeMenu *)transitionStatusMenuForItem:(BeeItem *)item {
			BeeMenu *menu = [BeeMenu new];
			if (item.isClosed) {
				BeeMenuItem *item = [BeeMenuItem new];
				item.title = NSLocalizedString(@"Reopen", nil);
				[menu addItem:item];
			} else {
				BeeMenuItem *item = [BeeMenuItem new];
				item.title = NSLocalizedString(@"Close", nil);
				[menu addItem:item];
			}

			return menu;
		}

   - A `BeeMenu` can hold other `BeeMenu` instances (which are represented as 
		   sub-menus) and they can also hold `BeeMenuItem` instances which are 
   the individual menu items.

   - Here we're determining whether the item is currently closed, and if it is 
   we want to provide the user with the ability to reopen it.

   - If it isn't closed, then offer the user the ability to close it.

5. We need to determine what to do when the user selects a transition option, so 
   let's implement `BeePluginWorkflow`.

   You know the drill!  Create a new file inheriting from `NSObject` named 
   'TasksWorkflow' which adheres to `BeePluginWorkflow`.  Inside `+ load` be 
   sure to register an instance of the plugin with `BeeKeeper`.

6. The method we're going to implement is [BeePluginWorkflow itemStatusWasTransitionedForItem:formValues:menuItem:callback:]:

		- (void)itemStatusWasTransitionedForItem:(BeeItem *)item formValues:(NSDictionary *)formValues menuItem:(BeeMenuItem *)menuItem callback:(BeePluginCallback)callback {
			BeeProject *project = item.project;
			NSString *nextStatusIdentifier = item.isClosed ? @"open" : @"closed";
			BeeItemState nextState = item.isClosed ? BeeItemStateOpen : BeeItemStateClosed;
			item.status = (BeeItemStatus *) [BeeModelHelper entityForName:@"BeeItemStatus"
															   identifier:nextStatusIdentifier
												  existingMutableSetProxy:project.statusesMutableProxy
																  account:project.account];
			item.state = @(nextState);

			callback(nil, nil);
		}

   - As we only have two statuses to worry about, the logic is straightforward.  
   If the item is closed, the user intends to reopen the item.  If the item is 
   open, the user intends to close it.

   - All we then need to do is update the item's status and state properties 
   with the correct values.

   - 'Build & Run' now and you will be able to close and reopen the items that 
   you create!

## All Done!

Congratulations, you've built your first Bee plugin!  There are a tonne of other 
hooks that you can use to enhance the functionality of your plugin which you can 
discover by exploring the `BeePlugin*` protocol headers.

You can explore this plugin online [here](https://github.com/NeatIO/Bee-Tasks). 
Its more fleshed out but includes the code discussed above.

## Networking

This plugin works locally but the real benefit of using Bee is that it allows 
you to connect to your online services.  `BeeKit` ships with a copy of 
`[AFNetworking](https://github.com/AFNetworking/AFNetworking)` included which is 
a great library for consuming APIs over HTTP.

However because `BeeKit` plugins operate in a special environment, there are a 
couple of very thin wrappers around `AFNetworking` included which you should use 
instead.  Namely:

- `BeeHTTPClient` instead of `AFHTTPClient`
- `BeeXMLRequestOperation` instead of `AFHTTPClient`
- `BeeJSONRequestOperation` instead of `AFJSONRequestOperation`
- `BeePropertyListRequestOperation` instead of `AFPropertyListRequestOperation`
- `BeeImageRequestOperation` instead of `AFImageRequestOperation`

`BeeHTTPClient` is already configured to use the `Bee*RequestOperation`
classes, so all you really need to do is use `BeeHTTPClient`.

A good example of a connected plugin is the GitHub Issues plugin located 
[here](https://github.com/NeatIO/Bee-GitHub).

## Tips

- A plugin should have an associated icon in retina and non-retina formats.  The 
icon size is 48x48px and should be located in the `Resources` directory of the 
bundle with the name `Icon.tiff`.  The `.tiff` image should include both 
representations of the image.  In Xcode, this is done for you automatically if 
'Combine High Resolution Artwork' (`COMBINE_HIDPI_IMAGES`) is true.

- A plugin should contain a short 1-sentence description of itself (namely what 
		it is or which versions of a particular API it supports).  This 
description should be included in the `BeePluginDescription` key of the bundle's 
`Info.plist` file.

- Don't ever modify entities in a different queue other than the one your plugin 
method was called on.  Entities are Core Data objects and they must only be 
mutated within the queue and the confined context in which they were created.  
It is in most cases unnecessary but if you must perform some processing on a 
separate queue, be sure to always dispatch your entity mutation calls to 
`bee_plugin_queue()`.

- `AFNetworking` classes are already configured to callback on the 
`bee_plugin_queue()`.

- Plugins found in the sandbox Application Support directory (
	`$(USER_LIBRARY_DIR)/Containers/io.neat.Bee/Data/Library/Application
	Support/io.neat.Bee/PlugIns`) will take precendence over any plugins shipped 
within Bee's app bundle.

- Check out `BeeFieldCatalog` for preconfigured fields to accomplish common 
tasks for Bee forms.

- See the [Customising Bee UI guide](Customising-Bee-UI) for the ability to 
customise further aspects of the Bee interface.

- Don't use state within your plugin instances.  There is no guarantee that your 
instance will be kept around at all.  Your plugin instance is run inside of an 
XPC process which can terminate at any time.

- All dates stored in Bee entities are in UTC.  In your plugin, please do the 
same.

- To support item reordering within groups, implement the methods inside of 
`BeePluginItemRearranging`.

- To support item drag and drop between groups, implement the methods inside of 
`BeePluginItemAssociations`.

- If you want to talk to another plugin instance within the same bundle, you can 
use the `BeeKeeper` methods:
	- [BeeKeeper performPluginSelector:arguments:account:]
	- [BeeKeeper performAsyncPluginSelector:arguments:account:]

- You don't have to split your plugin up into several files.  Bee doesn't care 
where your plugin functionality lives as long as it can find it (via [BeeKeeper registerPlugin:]),
but each plugin protocol is organised into a set of related operations already 
which correspond comfortably to a project structure.

- When creating the item details form ([BeePluginUI itemDetailsFormForItemType:]),
if you include a field which allows the user to select the item type, it must be 
of type `BeeFieldAutocomplete` and that field must have its `isItemTypeField` 
property set to `YES`.  This allows the item details form to know which field to 
observe so it can switch forms based on the user's selection.

- Objects adhering to the `BeeEntity` protocol support custom attributes, 
	meaning you can call [BeeEntity setCustomAttributeValue:forKey:] and 
	[BeeEntity customAttributeValueForKey:] on them.  You can associate the 
	plist types as well as `NSSet`.

	Objects supported are:
	- `NSArray`
	- `NSDictionary`
	- `NSData`
	- `NSSet`
	- `NSNumber`
	- `NSString`
	- `NSDate`
