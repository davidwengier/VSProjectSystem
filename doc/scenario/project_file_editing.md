Project File Editing Overview
=============================

A feature of CPS based projects is that their project files can be editing live, while the project is loaded in Visual Studio, and project changes through the Visual Studio UI or through the file system are processed correctly and merged into the editor.

This feature is enabled for any project with the `OpenProjectFile` capability.

## ProjectFileDocumentManager

The `ProjectFileDocumentManager` is the entry point to project file editing, as it handles requests from the shell to open a project file. It does two jobs:

* If a request to open the file is made without the caller specifying a particular editor to use, it forces the use of the XML editor, so syntax highlighting works as expected.
* If any editor is used to open the project file, with the exception of the App Designer (property pages) editor, it tells the `ProjectFileEditorPresenter` and `ProjectFrameOpenCloseListener` components, below, to activate and start tracking the editing operation.

### Why do we need it?

We activate services only when editing is actively being performed by the user to avoid any possible overhead and improve performance. Essentially the entire project file editing system gets out of the way if it's not needed.

## ProjectFileBufferManager

The `ProjectFileBufferManager` is responsible for providing the text buffer that is used for modifications to the project file, to the editor and other Visual Studio components. It manages handling changes between the three possible sources:

1. User changes via an editor window
1. Project changes made inside Visual Studio (eg, changing project properties, excluding files, adding references)
1. File changes made externally on disk

These changes are all managed for correct handling of buffer concerns like the undo stack, dirty flags, and saving the project to disk.

### Why do we need it?

This service has perhaps the most important job in the system, and is the main reason that a custom system is needed at all. When editing a project file by hand almost every keystroke made by the user will result in an invalid project, and it is important that the experience of the project inside Visual Studio is not adversely affected during this time, so the `ProjectFileBufferManager` prevents updating the UnconfiguredProject while editing is in progress.

When an external file change occurs, or changes are made inside Visual Studio however, they do get applied immediately to the UnconfiguredProject, so it's important that the buffer update in response to them so the user is never looking at an out-of-date version of the project.

There are also cases where the text buffer can change without the user directly editing it, like via Replace In Files, and in those cases when there is no text editor open we want to apply the changes to the UnconfiguredProject as it is the most logical thing to do from the user point of view.

Finally if the user does try to apply invalid edits to the project, by saving the document, we want to give them an opportunity to fix the mistake so we deliberately ignore those changes but from that point it's important that the buffer is _not_ updated lest the user loses their changes.

### Updating and merging the text buffer

There are a few scenarios for updating the text buffer:

* When external project changes are detected, if the UnconfiguredProject is unchanged and the edit buffer is unchanged, we reload the file from disk, and clear the undo stack
* When the UnconfiguredProject has changes, and the edit buffer is unchanged, we serialize the UnconfiguredProject to the buffer, and clear the undo stack
* When either the UnconfiguredProject changes, or the external file changes, and the buffer also has local changes we make a best effort attempt to merge the two sets of changes together

To merge the two project states together, one state being the text buffer, and one being the UnconfiguredProject which has already been reloaded from external file changes if applicable, we run through the following process. Before this process starts we will have captured a "base snapshot", which is the state of the text buffer at a time when it was the same as the project file and the UnconfiguredProject (ie, at project open).

1. Calculate the diff of changes between the base snapshot, and the project changes we nee to apply
1. Apply each change as an edit to the base snapshot, so that text movements can be tracked as though the project changes were made manually
1. Replay all of the edits that were applied to the text buffer since the base snapshot, taking into account the text movements
1. Reset the base snapshot, and undo history, to the point before step 3, above, as the new common point

## ProjectFileEditorPresenter

The `ProjectFileEditorPresenter` is responsible for tracking all of the text editor windows that are open for the project file, and updating their visual state as needed. Additionally it can track invisible editors like those used by Live Share when remote users open the project file for editing.

When the project file is dirty this class applies a visual indicator to any tabs that are open, and promotes said tabs from being provisional if they were. It also updates the caption of tabs if the project file is renamed.

This class also is used by the Edit Project File command, and double click action, so that we can switch to an existing editor tab if one is available, and only open a new tab is necessary.

### Why do we need it?

The main purpose of this service is to answer the question "Is anyone actively editing the project file?" so that the buffer manager knows whether to apply changes, or ignore them, as updates are made. Additionally this class handles asking the user to save changes as they close the last editor window.

In the Live Share case there aren't any physical tabs open on the host instance, but if a guest has an invisible editor open on their behalf then the answer to the above question still needs to be "yes".

## ProjectFrameOpenCloseListener

The `ProjectFrameOpenCloseListener` listens to all new document windows that are created, checks if they are text editors for the project file, and if so it tells the `ProjectFileEditorPresenter` to track them. Then document windows are closed it does the same check, and tells the editor presenter to stop tracking.

When the first window is opened it also removes the VirtualDocument flag from the Running Document Table so that the window participates in the check does when a "Close All" operation is performed. When the last window is closed that flag is restored. Just before the last tab is closed this class also tells the editor presenter to ask the user whether they want to save, if appropriate.

### Why do we need it?

There are many ways to open editor windows, and whilst the `ProjectFileEditorPresenter` could track any use of the Edit Project File menu item directly, for example, that wouldn't track duplicate windows opened with the Window | New Window command, or when the project file is open in the diff editor from the Git Changes window etc. Since all of the windows that are open use the same text buffer, changes are synchronized across them all, so they should be free to close any windows in any order, and still have everything work as expected.

### Identifying text windows

The criteria for identifying a text window for the project file is simply that its `VSFPROPID_pszMkDocument` is equal to the project file full path and that it has an `IVsTextView` that we can retrieve an instance of `IVsTextLines` from.

In order to support initialization of uninitialized restored document tabs it is not sufficient to just listen to frame open events, but rather every `OnFrameIsVisibleChanged` event needs to be processed if it causes a frame to become visible. To make this more efficient the `ProjectFrameOpenCloseListener` sets itself as the frames `VSFPROPID_ViewHelper` to quickly check if it is already aware of a frame.

## ProjectFileInvisibleEditorTracker

The `ProjectFileInvisibleEditorTracker` is used, remotely, by Live Share to tell the `ProjectFileEditorPresenter` when invisible editors have been opened so that the system knows to ignore text changes as appropriate. This service lives in the global scope and looks up the appropriate UnconfiguredProject based on the passed in project GUID in order to call the right `ProjectFileEditorPresenter` instance.

### Why do we need it?

Without this class every edit made by a Live Share guest would be immediately applied as Live Share keeps the text buffers in sync as there are no "real" project file editor windows open.