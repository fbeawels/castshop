# plugin.js

## Review

## 1. Summary  

**Purpose**  
The snippet implements a CKEditor plugin named **`pastetext`** that forces the paste operation to be plain‑text only (i.e., it strips all formatting). The plugin is designed for older IE browsers that expose the `window.clipboardData` API and falls back to a custom clipboard‑hook for other browsers.

**Key Components**  

| Component | Role |
|-----------|------|
| `a.exec` | Command executor – either opens a “Paste Text” dialog or pastes raw text directly from the clipboard. |
| `CKEDITOR.plugins.add('pastetext', …)` | Registers the plugin with CKEditor, adding a button, a command, and a dialog, and hooking into the `beforePaste` event when `forcePasteAsPlainText` is enabled. |
| `CKEDITOR.getClipboardData` | Utility that captures clipboard content in IE by inserting a hidden div and executing the native `Paste` command. |
| `CKEDITOR.editor.prototype.insertText` | Overwrites the default text‑insertion helper to HTML‑encode the string and convert newlines to `<br>` tags. |
| `CKEDITOR.config.forcePasteAsPlainText` | Global configuration flag that activates the plugin’s plain‑text enforcement. |

**Design Patterns / Libraries**  
- Uses the **Command** pattern (`c.addCommand`).
- Leverages CKEditor’s **plugin architecture** (add, requires, dialog registration).
- Minimal external dependencies; all code runs inside a closure to avoid polluting the global namespace.

---

## 2. Detailed Description  

### 2.1 Initialization Flow  

1. **Plugin registration** – `CKEDITOR.plugins.add('pastetext', …)` is executed immediately after the script loads.  
2. Inside `init`:
   * A command named `pastetext` is created (`c.addCommand`).
   * The command is attached to a toolbar button (`c.ui.addButton`).
   * A dialog file (`dialogs/pastetext.js`) is registered via `CKEDITOR.dialog.add`.
   * If the editor configuration flag `forcePasteAsPlainText` is set, an event listener is added to the editor’s `beforePaste` event.  
     * The listener schedules the `pastetext` command to run asynchronously (`setTimeout(..., 0)`), then cancels the original paste.  
3. The plugin depends on the **`clipboard`** plugin, which supplies the dialog UI.

### 2.2 Runtime Behavior  

- **Normal Paste** (`window.clipboardData` present):  
  * `exec` calls `c.insertText(window.clipboardData.getData('Text'))`.  
  * `insertText` is overridden to HTML‑encode the raw string and convert line breaks to `<br>`.  
  * The encoded string is inserted via `insertHtml`.

- **Non‑IE Browsers / No `clipboardData`:**  
  * The command opens the “Paste Text” dialog (`c.openDialog('pastetext')`).  
  * The dialog (not shown here) would provide a textarea for the user to paste into.

- **Force‑Plain‑Text Mode**:  
  * When `forcePasteAsPlainText` is true, any paste action is intercepted by the `beforePaste` listener, which immediately triggers the plain‑text command and cancels the default paste.

### 2.3 Clipboard Capture for IE  

`CKEDITOR.getClipboardData` creates a hidden div (`b`) inside the editor body.  
1. It attaches a `paste` listener to the body that simply flags `e = true`.  
2. It creates a text range that selects the hidden div, then calls `execCommand('Paste')` – this dumps the clipboard content into the hidden div.  
3. The function returns the inner HTML of the hidden div if a paste event occurred, otherwise `false`.  

This hack works only in IE due to its peculiar clipboard API.

### 2.4 Cleanup  

The plugin never removes the hidden div or detaches global listeners; the div remains part of the editor DOM for the editor’s lifetime.  
The `beforePaste` listener is attached with a priority of `20` and no explicit removal, so it persists as long as the editor instance exists.

### 2.5 Assumptions & Constraints  

| Assumption | Impact |
|------------|--------|
| Editor is instantiated with `config.forcePasteAsPlainText` if plain‑text paste is desired | Users must set this config flag or manually trigger the command |
| `window.clipboardData` exists in IE and `execCommand('Paste')` works as expected | Will fail in browsers that drop support for the legacy clipboard API |
| The dialog file `dialogs/pastetext.js` is present | The plugin will break if the dialog is missing |
| Overwriting `CKEDITOR.editor.prototype.insertText` is acceptable for the target application | This global override can interfere with other plugins that rely on the original behavior |

---

## 3. Functions / Methods  

| Function | Purpose | Inputs | Outputs | Side‑Effects |
|----------|---------|--------|---------|--------------|
| `a.exec(c)` | Executes the paste‑text command. | `c` – CKEditor instance | none (presents dialog or inserts text) | Calls `c.insertText` or `c.openDialog` |
| `CKEDITOR.plugins.add('pastetext', { init: … })` | Registers the plugin with CKEditor. | – | – | Adds command, button, dialog, event listeners |
| `CKEDITOR.getClipboardData()` | Retrieves clipboard contents in IE via hidden div. | – | Clipboard text (string) or `false` | Creates/uses hidden div, attaches/detaches event listener |
| `CKEDITOR.editor.prototype.insertText(a)` | Overridden text insertion helper. | `a` – string to insert | – | Encodes string, replaces line breaks, inserts via `insertHtml` |
| `setTimeout(function(){e.exec();}, 0)` | Defers command execution until after the current call stack. | – | – | Executes `pastetext` command asynchronously |

### Utility Methods  

* `CKEDITOR.tools.htmlEncode` – built‑in utility used to escape HTML characters.  
* `CKEDITOR.getUrl` – resolves the full path to the dialog file.  
* `c.addCommand`, `c.ui.addButton`, `c.dialog.add` – CKEditor’s standard API for plugin integration.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **CKEditor core** | Third‑party | Provides `CKEDITOR` namespace, `getClipboardData`, `editor` prototype, UI helpers. |
| **`clipboard` plugin** | CKEditor plugin | Required for the dialog; not included in the snippet. |
| **IE’s `window.clipboardData` API** | Platform‑specific | Only available in legacy IE. |
| **`window.execCommand('Paste')`** | Browser API | Works only in IE; not supported in modern browsers. |
| **`setTimeout`** | Standard JS | Used to defer execution. |
| **`CKEDITOR.tools.htmlEncode`** | CKEditor utility | Encodes special characters. |

No external libraries beyond CKEditor itself are required.

---

## 5. Additional Notes  

### 5.1 Edge Cases & Limitations  

1. **Modern browsers** – The fallback to the “Paste Text” dialog will be used, but the dialog file must exist. If it’s missing, users will see a blank dialog.  
2. **Clipboard permissions** – In many browsers, programmatic paste is disallowed unless triggered by a user action. The `exec` method may silently fail in such contexts.  
3. **Hidden div persistence** – The hidden div (`b`) remains in the DOM for the editor’s lifetime, potentially affecting layout or CSS if not carefully scoped.  
4. **Overwriting `insertText` globally** – Other plugins that expect the original behavior of `insertText` will break. A more robust approach would be to override the method on a per‑instance basis or wrap it rather than replace it entirely.  

### 5.2 Potential Enhancements  

- **Remove the hidden div on editor destruction** to prevent DOM clutter.  
- **Feature‑detect** `execCommand('Paste')` rather than hard‑coding for IE; consider newer clipboard APIs (`navigator.clipboard.readText()`), with graceful fallback.  
- **Make the plain‑text command optional per‑instance** rather than relying on a global config flag.  
- **Encapsulate the `insertText` override** inside the plugin to avoid affecting unrelated editors.  
- **Add unit tests** for the clipboard extraction logic, especially the IE hack, using a headless environment that simulates the old API.  
- **Improve user feedback** – provide a message if the paste fails due to permissions or missing dialog.  

### 5.3 Security Considerations  

* The plugin only accepts plain text and encodes it, mitigating XSS risks that might arise from unescaped content.  
* However, if the overridden `insertText` is later used by other code that expects raw insertion, unintended escaping could occur.  

---

**Overall Assessment**  
The code fulfills its primary goal of providing a plain‑text paste option for CKEditor, leveraging the plugin system and a creative IE clipboard hack. The implementation is concise but could benefit from clearer separation of concerns, better cleanup, and modern browser support. With the suggested refinements, the plugin would be more robust, maintainable, and future‑proof.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

(function(){var a={exec:function(c){if(CKEDITOR.getClipboardData()===false||!window.clipboardData){c.openDialog('pastetext');return;}c.insertText(window.clipboardData.getData('Text'));}};CKEDITOR.plugins.add('pastetext',{init:function(c){var d='pastetext',e=c.addCommand(d,a);c.ui.addButton('PasteText',{label:c.lang.pasteText.button,command:d});CKEDITOR.dialog.add(d,CKEDITOR.getUrl(this.path+'dialogs/pastetext.js'));if(c.config.forcePasteAsPlainText)c.on('beforePaste',function(f){if(c.mode=='wysiwyg'){setTimeout(function(){e.exec();},0);f.cancel();}},null,null,20);},requires:['clipboard']});var b;CKEDITOR.getClipboardData=function(){if(!CKEDITOR.env.ie)return false;var c=CKEDITOR.document,d=c.getBody();if(!b){b=c.createElement('div',{attributes:{id:'cke_hiddenDiv'},styles:{position:'absolute',visibility:'hidden',overflow:'hidden',width:'1px',height:'1px'}});b.setHtml('');b.appendTo(d);}var e=false,f=function(){e=true;};d.on('paste',f);var g=d.$.createTextRange();g.moveToElementText(b.$);g.execCommand('Paste');var h=b.getHtml();b.setHtml('');d.removeListener('paste',f);return e&&h;};})();CKEDITOR.editor.prototype.insertText=function(a){a=CKEDITOR.tools.htmlEncode(a);a=a.replace(/(?:\r\n)|\n|\r/g,'<br>');this.insertHtml(a);};CKEDITOR.config.forcePasteAsPlainText=false;



```
