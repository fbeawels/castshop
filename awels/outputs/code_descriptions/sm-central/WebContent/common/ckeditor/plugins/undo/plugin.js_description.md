# plugin.js

## Review

## 1. Summary  
**Purpose** – The snippet implements CKEditor’s built‑in *undo/redo* functionality as a plugin (`undo`).  
**Key components**  
| Component | Role |
|-----------|------|
| `CKEDITOR.plugins.add('undo', …)` | Declares the plugin, registers commands and UI elements. |
| `a` (Snapshot) | Represents a snapshot of the editor content plus bookmark data. |
| `b` (UndoStack) | Maintains the stack of snapshots, handles typing heuristics, and performs undo/redo logic. |
| Commands `undo` / `redo` | Trigger the stack’s `undo()` / `redo()` methods. |
| UI buttons | Provide a toolbar button that fires the corresponding command. |
| Event listeners | Hook into editor lifecycle events (e.g., `beforeCommandExec`, `contentDom`, `mode`) to capture snapshots and maintain stack state. |

**Design patterns**  
* The plugin follows **Command** pattern – each undoable operation is a command.  
* **Memento** pattern – snapshots (`a`) store a *memento* of the editor state.  
* Observer pattern – events (`beforeCommandExec`, `afterCommandExec`, etc.) notify the stack to save or restore state.

## 2. Detailed Description  
### Initialization (`init` callback)  
1. Create an instance of the stack (`d = new b(c)`), passing the editor instance.  
2. Add two commands:  
   * **undo** – runs `d.undo()`, updates selection, fires `afterUndo`.  
   * **redo** – runs `d.redo()`, updates selection, fires `afterRedo`.  
3. The stack exposes an internal `onChange` callback that updates the command states (`CKEDITOR.TRISTATE_OFF` when possible, otherwise `CKEDITOR.TRISTATE_DISABLED`).  
4. `g(h)` – helper that saves a snapshot when a command that can undo is executed.  
5. Register event listeners:  
   * `beforeCommandExec`, `afterCommandExec` → `g` → `d.save()` if necessary.  
   * `saveSnapshot` → explicit stack save.  
   * `contentDom` → monitor keydown events to detect typing.  
   * `beforeModeUnload` → snapshot before leaving WYSIWYG mode.  
   * `mode` → enable/disable the stack based on the current mode.  
   * `ui.addButton` – adds Undo/Redo buttons to the toolbar.  
6. Expose `resetUndo` on the editor to clear the stack and trigger a fresh snapshot.

### Snapshot (`a`)  
* Stores the raw HTML snapshot (`this.contents`).  
* Saves selection bookmarks (`this.bookmarks`).  
* For IE, removes the CKEditor expando attributes (`_cke_expando`).  
* `equals` compares both content and bookmarks, with an optional “strict” flag that ignores bookmark comparison.

### UndoStack (`b`)  
* **Typing detection** – `type(c)` is invoked on keydown.  
  * Handles *keystroke* values, distinguishes between typing (characters), navigation keys, and modifiers (Ctrl/Alt).  
  * Implements a debounce logic: after 25 character keystrokes or 25 modifier keystrokes, a snapshot is automatically saved.  
  * Uses `CKEDITOR.tools.setTimeout(...,0)` to queue the actual snapshot after the event loop to allow the editor state to stabilize.  
* **State tracking** – `hasUndo`, `hasRedo`, `typing`, `index`, etc.  
* **Stack operations**  
  * `save(c,d,e)` – inserts a new snapshot (`d`) at the current index, discarding older items beyond the configured limit.  
  * `restoreImage(c)` – loads snapshot contents into the editor, restores bookmarks, and updates internal state.  
  * `getNextImage(c)` – fetches the previous/next snapshot relative to the current index that differs from the current snapshot.  
  * `undo()`, `redo()` – perform the stack navigation, invoking `save(true)` to push a “current” snapshot before moving.  
* **Limits** – `undoStackSize` is configurable (default 20).

### Flow of Execution  
1. **Startup** – plugin initializes stack, adds commands/buttons.  
2. **User interaction** – typing triggers `type()` → snapshots are queued based on heuristics.  
3. **Commands** – pressing Undo/Redo or invoking from code triggers stack navigation.  
4. **Mode change** – entering/exiting WYSIWYG mode automatically saves a snapshot.  
5. **Reset** – `resetUndo()` clears the stack and forces a fresh snapshot (useful after programmatic changes).

## 3. Functions/Methods  

| Function/Method | Purpose | Inputs | Outputs / Side‑Effects |
|-----------------|---------|--------|------------------------|
| **a(c)** (constructor) | Create snapshot of editor content & selection. | `c`: editor instance | `this.contents`, `this.bookmarks` |
| **a.prototype.equals(c,d)** | Compare two snapshots. | `c`: snapshot to compare, `d`: strict flag | `true/false` |
| **b(c)** (constructor) | Initialise undo stack. | `c`: editor instance | `this.editor`, internal state |
| **b.prototype.type(c)** | Handle keydown events, decide when to snapshot. | `c`: keydown event | Queue snapshot, update typing counters |
| **b.prototype.reset()** | Reset stack to initial state. | – | Clears snapshots, indices, counters |
| **b.prototype.resetType()** | Reset typing heuristics. | – | Resets counters, flags |
| **b.prototype.fireChange()** | Update `hasUndo`/`hasRedo` and call `onChange`. | – | Sets flags, triggers UI update |
| **b.prototype.save(c,d,e)** | Push new snapshot onto stack. | `c`: bool (current image), `d`: snapshot object, `e`: flag to fire change | Returns `true/false` |
| **b.prototype.restoreImage(c)** | Load snapshot contents into editor. | `c`: snapshot | Editor content set, selection restored |
| **b.prototype.getNextImage(c)** | Get previous/next differing snapshot. | `c`: boolean (`true`=undo) | Snapshot or `null` |
| **b.prototype.redoable()** | Test if redo is possible. | – | Boolean |
| **b.prototype.undoable()** | Test if undo is possible. | – | Boolean |
| **b.prototype.undo()** | Perform an undo operation. | – | Boolean |
| **b.prototype.redo()** | Perform a redo operation. | – | Boolean |

The plugin also defines event handlers (`g`, `c.on(...)`) and UI registration, but those are not part of a reusable API.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` | Third‑party | Core CKEditor namespace. |
| `CKEDITOR.plugins` | Third‑party | Plugin registration system. |
| `CKEDITOR.env` | Third‑party | Browser environment checks (IE detection). |
| `CKEDITOR.tools` | Third‑party | Utility functions (`setTimeout`, `arrayCompare`). |
| `CKEDITOR.selection` | Third‑party | Selection/bookmark handling. |
| `CKEDITOR.wysiwygarea` | Third‑party | Required to ensure the editor is in WYSIWYG mode before undo works. |
| `CKEDITOR.lang` | Third‑party | Localization for button labels. |
| `CKEDITOR.config.undoStackSize` | Third‑party | User‑configurable limit. |

All dependencies are part of CKEditor itself, no external libraries or platform‑specific code beyond the IE branch.

## 5. Additional Notes  

### Strengths  
* **Robust snapshot mechanism** – preserves both content and selection, critical for rich‑text editing.  
* **Typing heuristics** – reduces snapshot churn by grouping fast typing into single undo steps.  
* **Event‑driven** – automatically captures state before/after commands and mode changes.  
* **Configurability** – `undoStackSize` can be tuned per‑editor.  
* **Graceful IE handling** – strips expando attributes to avoid snapshot noise.  

### Potential Issues / Edge Cases  
1. **Performance** – The snapshot logic runs on every keydown and command execution. For very large documents, the `getSnapshot()` call may be expensive, especially with the IE expando replacement.  
2. **Large undo history** – The default limit is 20 snapshots; if users type a lot before pressing Undo, memory usage could grow. Although `limit` is enforced, the logic to remove older snapshots may still consume time when the stack is full.  
3. **Modifier key logic** – The heuristic for modifiers (`modifiersCount>25`) is somewhat arbitrary and might lead to unexpected snapshot boundaries on keyboard‑heavy interactions (e.g., copy‑paste with many shortcuts).  
4. **Undo after programmatic changes** – If external code changes the editor’s HTML directly (bypassing `editor.setData`), the stack might not capture that state, leading to inconsistent undo/redo.  
5. **Snapshot equality** – The `equals` method ignores bookmark differences when `d` (strict) is false. In complex selection scenarios this could mask real differences.  
6. **Cross‑browser inconsistencies** – While IE is handled specially, other browsers’ quirks (e.g., Firefox’s `createTextRange` usage) are not covered.  

### Future Enhancements  
* **Configurable heuristics** – Expose thresholds (`typingLimit`, `modifierLimit`) via config so developers can tune undo granularity.  
* **Lazy snapshotting** – Only capture snapshots on content changes, not on every keystroke, using a change detection diff.  
* **History persistence** – Persist undo stack across sessions (e.g., via localStorage) for longer‑lasting editing.  
* **More granular undo** – Allow per‑operation undo (e.g., per button press) rather than grouping by keystrokes.  
* **Unit tests** – Add automated tests for `b` and `a` to ensure snapshot comparison and stack navigation work across browsers.  
* **Accessibility** – Expose `undoable/redoable` state to ARIA attributes or custom events for assistive technologies.  

Overall, the code is a solid implementation of a standard undo/redo system within CKEditor, leveraging the editor’s event bus and snapshot capabilities effectively.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){CKEDITOR.plugins.add('undo',{requires:['selection','wysiwygarea'],init:function(c){var d=new b(c),e=c.addCommand('undo',{exec:function(){if(d.undo()){c.selectionChange();this.fire('afterUndo');}},state:CKEDITOR.TRISTATE_DISABLED,canUndo:false}),f=c.addCommand('redo',{exec:function(){if(d.redo()){c.selectionChange();this.fire('afterRedo');}},state:CKEDITOR.TRISTATE_DISABLED,canUndo:false});d.onChange=function(){e.setState(d.undoable()?CKEDITOR.TRISTATE_OFF:CKEDITOR.TRISTATE_DISABLED);f.setState(d.redoable()?CKEDITOR.TRISTATE_OFF:CKEDITOR.TRISTATE_DISABLED);};function g(h){if(d.enabled&&h.data.command.canUndo!==false)d.save();};c.on('beforeCommandExec',g);c.on('afterCommandExec',g);c.on('saveSnapshot',function(){d.save();});c.on('contentDom',function(){c.document.on('keydown',function(h){if(!h.data.$.ctrlKey&&!h.data.$.metaKey)d.type(h);});});c.on('beforeModeUnload',function(){c.mode=='wysiwyg'&&d.save(true);});c.on('mode',function(){d.enabled=c.mode=='wysiwyg';d.onChange();});c.ui.addButton('Undo',{label:c.lang.undo,command:'undo'});c.ui.addButton('Redo',{label:c.lang.redo,command:'redo'});c.resetUndo=function(){d.reset();c.fire('saveSnapshot');};}});function a(c){var e=this;var d=c.getSelection();e.contents=c.getSnapshot();e.bookmarks=d&&d.createBookmarks2(true);if(CKEDITOR.env.ie)e.contents=e.contents.replace(/\s+_cke_expando=".*?"/g,'');};a.prototype={equals:function(c,d){if(this.contents!=c.contents)return false;if(d)return true;var e=this.bookmarks,f=c.bookmarks;if(e||f){if(!e||!f||e.length!=f.length)return false;for(var g=0;g<e.length;g++){var h=e[g],i=f[g];if(h.startOffset!=i.startOffset||h.endOffset!=i.endOffset||!CKEDITOR.tools.arrayCompare(h.start,i.start)||!CKEDITOR.tools.arrayCompare(h.end,i.end))return false;}}return true;}};function b(c){this.editor=c;this.reset();};b.prototype={type:function(c){var d=c&&c.data.getKeystroke(),e={8:1,46:1},f=d in e,g=this.lastKeystroke in e,h=f&&d==this.lastKeystroke,i={37:1,38:1,39:1,40:1},j=d in i,k=this.lastKeystroke in i,l=!f&&!j,m=f&&!h,n=!this.typing||l&&(g||k);if(n||m){var o=new a(this.editor);CKEDITOR.tools.setTimeout(function(){var q=this;var p=q.editor.getSnapshot();if(CKEDITOR.env.ie)p=p.replace(/\s+_cke_expando=".*?"/g,'');if(o.contents!=p){if(!q.save(false,o,false))q.snapshots.splice(q.index+1,q.snapshots.length-q.index-1);q.hasUndo=true;q.hasRedo=false;q.typesCount=1;q.modifiersCount=1;q.onChange();}},0,this);}this.lastKeystroke=d;if(f){this.typesCount=0;this.modifiersCount++;if(this.modifiersCount>25){this.save();
this.modifiersCount=1;}}else if(!j){this.modifiersCount=0;this.typesCount++;if(this.typesCount>25){this.save();this.typesCount=1;}}this.typing=true;},reset:function(){var c=this;c.lastKeystroke=0;c.snapshots=[];c.index=-1;c.limit=c.editor.config.undoStackSize;c.currentImage=null;c.hasUndo=false;c.hasRedo=false;c.resetType();},resetType:function(){var c=this;c.typing=false;delete c.lastKeystroke;c.typesCount=0;c.modifiersCount=0;},fireChange:function(){var c=this;c.hasUndo=!!c.getNextImage(true);c.hasRedo=!!c.getNextImage(false);c.resetType();c.onChange();},save:function(c,d,e){var g=this;var f=g.snapshots;if(!d)d=new a(g.editor);if(g.currentImage&&d.equals(g.currentImage,c))return false;f.splice(g.index+1,f.length-g.index-1);if(f.length==g.limit)f.shift();g.index=f.push(d)-1;g.currentImage=d;if(e!==false)g.fireChange();return true;},restoreImage:function(c){var e=this;e.editor.loadSnapshot(c.contents);if(c.bookmarks)e.editor.getSelection().selectBookmarks(c.bookmarks);else if(CKEDITOR.env.ie){var d=e.editor.document.getBody().$.createTextRange();d.collapse(true);d.select();}e.index=c.index;e.currentImage=c;e.fireChange();},getNextImage:function(c){var h=this;var d=h.snapshots,e=h.currentImage,f,g;if(e)if(c)for(g=h.index-1;g>=0;g--){f=d[g];if(!e.equals(f,true)){f.index=g;return f;}}else for(g=h.index+1;g<d.length;g++){f=d[g];if(!e.equals(f,true)){f.index=g;return f;}}return null;},redoable:function(){return this.enabled&&this.hasRedo;},undoable:function(){return this.enabled&&this.hasUndo;},undo:function(){var d=this;if(d.undoable()){d.save(true);var c=d.getNextImage(true);if(c)return d.restoreImage(c),true;}return false;},redo:function(){var d=this;if(d.redoable()){d.save(true);if(d.redoable()){var c=d.getNextImage(false);if(c)return d.restoreImage(c),true;}}return false;}};})();CKEDITOR.config.undoStackSize=20;



```
