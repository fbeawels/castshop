# plugin.js

## Review

## 1. Summary

**Purpose**  
The code implements the **Remove Format** plugin for CKEditor 4.x.  
When the user invokes the *Remove Format* command (via the toolbar button or shortcut), the plugin strips away all formatting tags and selected attributes from the current selection.

**Key Components**  
| Component | Role |
|-----------|------|
| `CKEDITOR.plugins.add('removeformat', …)` | Registers the plugin, declares its dependency on the `selection` plugin, and exposes the UI button. |
| `CKEDITOR.plugins.removeformat.commands.removeformat.exec` | The core algorithm that performs the removal. |
| `CKEDITOR.config.removeFormatTags` & `CKEDITOR.config.removeFormatAttributes` | Configuration strings that list the tags/attributes to strip. |

**Notable Patterns / Libraries**  
- Uses CKEditor’s plugin API (`add`, `addCommand`, `ui.addButton`).  
- Leverages the editor’s internal DOM helpers (`CKEDITOR.dom.elementPath`, `getNextSourceNode`, etc.).  
- The logic is wrapped in a self‑executing module that attaches itself to `CKEDITOR.plugins.removeformat`.

---

## 2. Detailed Description

### Flow of Execution

1. **Initialization**  
   - When the editor loads, `CKEDITOR.plugins.add('removeformat', …)` is executed.  
   - The plugin requires the `selection` plugin (ensuring selection utilities are available).  
   - A new command `removeFormat` is added; it points to the `exec` function defined in the plugin.  
   - A toolbar button “RemoveFormat” is created, labeled by `a.lang.removeFormat` and wired to the command.

2. **Command Execution (`exec`)**  
   - **Regex & Attributes Cache**  
     - On first run, `a._.removeFormatRegex` is created from `config.removeFormatTags`.  
     - `a._.removeAttributes` is built from `config.removeFormatAttributes`.  
     - Subsequent calls reuse these cached values to avoid recomputing.
   - **Range Retrieval**  
     - The current selection ranges (`a.getSelection().getRanges()`) are fetched.  
   - **Per‑Range Processing**  
     - Each range that is not collapsed is enlarged to whole elements (`enlarge(CKEDITOR.ENLARGE_ELEMENT)`).  
     - A bookmark of the range start/end nodes is created to preserve the cursor after manipulation.  
     - The helper function `j` walks the element path from a node up to its block/blockLimit ancestor.  
       - If a node in the path matches the removal regex, `breakParent` is invoked so that the offending tag is lifted out of the path.  
     - `j` is called on both the start and end nodes to prune any formatting tags that encompass the selection.  
     - A forward iteration (`getNextSourceNode(true, CKEDITOR.NODE_ELEMENT)`) walks all source nodes inside the range.  
       - If a node is an `img` tag with `_cke_protected_html`, it is left untouched.  
       - If the node’s tag matches the regex, the whole element (including its children) is removed (`k.remove(true)`).  
       - Otherwise, the node’s attributes listed in `c` are stripped (`k.removeAttributes(c)`).  
     - The range is moved back to the bookmark, restoring the selection to the same visual position.  
   - **Restoring Selection**  
     - After all ranges are processed, the original ranges are re‑selected to maintain the user’s selection state.

3. **Cleanup**  
   - The plugin does not create global state; all temporary objects (regex, bookmarks, element paths) are scoped to the `exec` call.  
   - The only persistent state is the cached regex and attributes on the editor instance (`a._`).  
   - When the editor instance is destroyed, `a._` will be cleared automatically.

### Assumptions & Constraints

- **Configuration** – The plugin expects the `removeFormatTags` and `removeFormatAttributes` configuration properties to be comma‑separated strings.  
- **Selection** – The logic is designed for a text/element selection. It explicitly skips collapsed ranges.  
- **DOM Structure** – The algorithm relies on CKEditor’s DOM helpers. It assumes the editor’s data is represented as a flat tree of source nodes (tags, text nodes).  
- **Protected Images** – Images can be marked with a `_cke_protected_html` attribute to avoid deletion.  

---

## 3. Functions/Methods

| Function/Method | Purpose | Inputs | Outputs | Side Effects |
|-----------------|---------|--------|---------|--------------|
| `CKEDITOR.plugins.add('removeformat', …)` | Plugin registration | None (implicit plugin object) | Adds plugin to CKEditor | Adds UI button & command |
| `CKEDITOR.plugins.removeformat.commands.removeformat.exec` | Main removal routine | `a` – editor instance | None | Modifies the document, updates selection |
| `j(m)` (inner helper) | Walks element path upward, breaking tags that match the removal regex | `m` – DOM node | None | Calls `breakParent` on matching tags |
| `a.getSelection().getRanges()` | Fetches current selection ranges | None | Array of ranges | None |
| `f.enlarge(CKEDITOR.ENLARGE_ELEMENT)` | Expands range to cover whole elements | None | None | Modifies range boundaries |
| `f.createBookmark()` | Creates a bookmark to preserve cursor | None | Bookmark object with `startNode` & `endNode` | None |
| `k.remove(true)` | Deletes an element and its children | None | None | Removes node from DOM |
| `k.removeAttributes(c)` | Deletes specified attributes from a node | `c` – array of attribute names | None | Modifies node’s attributes |

**Reusable / Utility**  
- The regex caching (`a._.removeFormatRegex`) and attribute list caching (`a._.removeAttributes`) are reusable across invocations.  
- The inner helper `j` could be extracted as a separate utility if used elsewhere.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` core | Third‑party | The plugin is a part of the CKEditor 4.x plugin architecture. |
| `selection` plugin | CKEditor plugin | Required to guarantee selection helpers. |
| `CKEDITOR.dom.elementPath`, `CKEDITOR.NODE_ELEMENT`, `CKEDITOR.ENLARGE_ELEMENT` | CKEditor internal DOM API | No external libraries. |
| `config.removeFormatTags`, `config.removeFormatAttributes` | CKEditor configuration | Must be set by the user or editor config file. |
| None other |  |  |

---

## 5. Additional Notes

### Strengths
- **Performance** – Regex and attribute list are cached, avoiding repetitive string operations.  
- **Simplicity** – The code is concise and follows CKEditor conventions.  
- **Configurable** – Users can tweak which tags/attributes are removed via the config.

### Potential Issues / Edge Cases
1. **Nested Formatting**  
   - The algorithm uses `breakParent` to lift offending tags. In deeply nested situations, this could create malformed HTML if not carefully handled (e.g., `<b><i>text</i></b>` may become `<b>text</b>` or `<i>text</i>` depending on traversal).  
   - A more robust approach might use a tree walk that removes only the exact tags in the selection path without altering sibling nodes.

2. **Mixed Selections**  
   - If a range spans multiple block elements, the `enlarge` call will expand each block separately. The loop logic may inadvertently remove whole elements that contain only partially selected content. CKEditor 5’s “remove format” works at a word‑level granularity; this plugin may be more aggressive.

3. **Attribute Removal**  
   - Only attributes listed in `removeFormatAttributes` are stripped. If an element contains additional attributes that the developer wants to preserve (e.g., `data-*`), the current config will not remove them.  
   - There is no safeguard against removing necessary attributes for certain plugins (e.g., `src` for `<img>` if not protected).

4. **Protected Images**  
   - The code checks for `_cke_protected_html` to skip `<img>` tags. However, if an image is protected but contains removable attributes, those attributes will still be stripped.

5. **Editor Modes**  
   - The plugin assumes the editor is in “wysiwyg” mode. In source editing mode (`config.readOnly` or `sourcearea`), `getSelection()` may behave differently, potentially causing errors.

6. **Memory Leak**  
   - While the regex and attributes are cached, they are attached to `a._`. If the editor instance is reused (e.g., multiple editors on the same page), each will maintain its own cache, which is fine, but could be cleaned up in `destroy` if necessary.

### Recommendations for Enhancement
- **Granular Removal**: Implement a token‑level removal that only deletes formatting on the selected text rather than whole elements.
- **Config Validation**: Validate the `removeFormatTags` and `removeFormatAttributes` on plugin init to catch typos or empty values.
- **Accessibility**: Add a confirmation dialog for destructive actions or provide an “Undo” shortcut immediately after removal.
- **Testing**: Unit‑test with a variety of nested structures and selections to ensure no unintended DOM manipulation.
- **Compatibility**: Add fallbacks for older browsers or CKEditor versions that lack certain DOM helpers.

---

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.plugins.add('removeformat',{requires:['selection'],init:function(a){a.addCommand('removeFormat',CKEDITOR.plugins.removeformat.commands.removeformat);a.ui.addButton('RemoveFormat',{label:a.lang.removeFormat,command:'removeFormat'});}});CKEDITOR.plugins.removeformat={commands:{removeformat:{exec:function(a){var b=a._.removeFormatRegex||(a._.removeFormatRegex=new RegExp('^(?:'+a.config.removeFormatTags.replace(/,/g,'|')+')$','i')),c=a._.removeAttributes||(a._.removeAttributes=a.config.removeFormatAttributes.split(',')),d=a.getSelection().getRanges();for(var e=0,f;f=d[e];e++){if(f.collapsed)continue;f.enlarge(CKEDITOR.ENLARGE_ELEMENT);var g=f.createBookmark(),h=g.startNode,i=g.endNode,j=function(m){var n=new CKEDITOR.dom.elementPath(m),o=n.elements;for(var p=1,q;q=o[p];p++){if(q.equals(n.block)||q.equals(n.blockLimit))break;if(b.test(q.getName()))m.breakParent(q);}};j(h);j(i);var k=h.getNextSourceNode(true,CKEDITOR.NODE_ELEMENT);while(k){if(k.equals(i))break;var l=k.getNextSourceNode(false,CKEDITOR.NODE_ELEMENT);if(k.getName()!='img'||!k.getAttribute('_cke_protected_html'))if(b.test(k.getName()))k.remove(true);else k.removeAttributes(c);k=l;}f.moveToBookmark(g);}a.getSelection().selectRanges(d);}}}};CKEDITOR.config.removeFormatTags='b,big,code,del,dfn,em,font,i,ins,kbd,q,samp,small,span,strike,strong,sub,sup,tt,u,var';CKEDITOR.config.removeFormatAttributes='class,style,lang,width,height,align,hspace,valign';



```
