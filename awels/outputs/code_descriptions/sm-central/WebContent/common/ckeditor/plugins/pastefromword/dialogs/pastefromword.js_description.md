# pastefromword.js

## Review

## 1. Summary  

The snippet is a **CKEditor dialog definition** for the “Paste from Word” feature.  
* **Purpose** – Provides a modal that lets users paste rich‑text copied from Microsoft Word (or similar word processors) into the editor.  
* **Key components**  
  * `CKEDITOR.dialog.add('pastefromword', …)` – registers the dialog with CKEditor.  
  * `htmlToLoad` – inline HTML/JS that creates an editable iframe where the user can paste the content.  
  * `cleanWord` – a large regex‑driven routine that strips Word‑specific markup and optional style/face tags.  
  * Lifecycle callbacks (`onShow`, `onOk`, `onHide`, `onLoad`) that initialise the iframe, handle the paste, clean the HTML, and insert it into the editor.  
  * The `contents` array defines the dialog UI (advice text, the editing area, and two check‑boxes for “ignore font face” and “remove style”).  
* **Notable design patterns / libraries**  
  * **Dialog API** – CKEditor’s built‑in dialog framework.  
  * **Environment checks** – `CKEDITOR.env` is used heavily to work around IE quirks, WebKit quirks, and custom‑domain scenarios.  
  * **DOM manipulation helpers** – `CKEDITOR.dom.element`, `CKEDITOR.tools.htmlEncode`.  

---

## 2. Detailed Description  

### 2.1 Overall Flow  
1. **Dialog registration** – `CKEDITOR.dialog.add` registers a dialog definition that CKEditor will instantiate on request.  
2. **Opening the dialog** – `onShow` is executed.  
   * Makes the main editor’s body non‑editable (`contentEditable='false'` for IE).  
   * Creates an `<iframe>` that hosts an editable area (`contentEditable` for non‑IE or `designMode='on'` for IE).  
   * Loads the `htmlToLoad` template into the iframe. The template contains an `onload` script that:  
     * Enables the iframe’s contentEditable area.  
     * Stores a reference back to the dialog (`iframe.getCustomData('dialog')`).  
     * Listens for the Escape key to hide the dialog.  
     * Fires `iframeAdded` event once the iframe is ready.  
3. **User pastes content** – The iframe behaves like a mini editor.  
4. **Confirming the paste** – `onOk` is called.  
   * Extracts the inner HTML of the iframe body.  
   * Calls `cleanWord` to strip Word‑specific artifacts.  
   * Inserts the cleaned HTML back into the parent editor using `d.insertHtml(e)` (executed via `setTimeout` to avoid synchronous DOM manipulation during dialog teardown).  
5. **Closing the dialog** – `onHide` restores the editor body to editable.

### 2.2 Core Components & Their Interaction  

| Component | Responsibility | Interaction |
|-----------|----------------|-------------|
| `htmlToLoad` | Provides the iframe’s initial markup & script | Loaded into iframe on `onShow`. |
| `cleanWord` | Cleans pasted Word HTML | Called by `onOk`. |
| `onShow` | Initializes dialog state & iframe | Prepares editing area, binds events. |
| `onOk` | Processes and inserts cleaned content | Invoked when user clicks OK. |
| `onHide` | Restores editor state | Re‑enables editing after dialog close. |
| `onLoad` | Handles quirks for RTL in IE6/7 | Adjusts overflow style. |
| `contents` | UI definition (advice text, editing area, options) | CKEditor builds the dialog layout from this. |

### 2.3 Dependencies & Assumptions  

* **CKEditor core** – Uses the Dialog API (`CKEDITOR.dialog.add`), DOM utilities (`CKEDITOR.dom.element`), environment flags (`CKEDITOR.env`), and language object (`a.lang`).  
* **Browser quirks** – Assumes that:  
  * IE requires `contentEditable` to be set on the `<body>` or `designMode='on'` on the document.  
  * WebKit (Safari/Chrome) needs `table-layout:fixed` on a containing `<table>`.  
  * Custom domain pages need a special loading sequence for the iframe.  
* **Paste source** – Designed for content coming from Microsoft Word; no generic HTML sanitizer.  

---

## 3. Functions/Methods  

| Function | Signature | Purpose | Notes |
|----------|-----------|---------|-------|
| `cleanWord(b, c, d, e)` | `(editor, rawHtml, ignoreFontFace, removeStyle)` | Strips Word‑specific tags, attributes, and optional style/font‑face data using a series of regex replacements. Returns cleaned HTML. | Very large regex chain – fragile against changes in Word’s output. |
| `onShow()` | `()` | Sets editor to non‑editable, creates and loads the iframe, configures its attributes and event listeners. | Uses `window.onload` inside the iframe to initialise the editable area. |
| `onOk()` | `()` | Gathers pasted HTML from iframe, cleans it, and schedules insertion into the editor. | Uses `setTimeout(…,0)` to avoid synchronous DOM changes during dialog closing. |
| `onHide()` | `()` | Restores editor’s body to editable (`contentEditable='true'`). | Only needed for IE. |
| `onLoad()` | `()` | When dialog is loaded in IE6/7 RTL, hides overflow to avoid layout issues. | Triggers only for specific environment conditions. |
| `contents` array | – | Defines the dialog UI: label, advice text, editing area (`<fieldset>` + iframe), and option checkboxes. | Each element’s `type`, `id`, `style`, `onShow`, `focus`, etc. |

---

## 4. Dependencies  

| Library / API | Type | Usage |
|---------------|------|-------|
| **CKEditor core** | Third‑party | Dialog registration, environment detection, DOM helpers, language packs. |
| **CKEDITOR.env** | CKEditor | Browser feature detection (IE, quirks, webkit, custom domain). |
| **CKEDITOR.dom.element** | CKEditor | Creates elements, sets styles, custom data, attaches events. |
| **CKEDITOR.tools** | CKEditor | HTML escaping (`htmlEncode`). |
| **JavaScript native APIs** | Standard | `setTimeout`, `document.write`, iframe manipulation. |

No additional external libraries or APIs are referenced.

---

## 5. Additional Notes  

### 5.1 Strengths  
* **Cross‑browser support** – Extensive environment checks ensure compatibility with older IE versions and custom‑domain scenarios.  
* **Separation of concerns** – The dialog UI is defined declaratively (`contents` array), while the heavy processing logic is encapsulated in `cleanWord`.  
* **Configurability** – Options to ignore font faces or remove styles are exposed via check‑boxes tied to configuration (`pasteFromWordIgnoreFontFace`, `pasteFromWordRemoveStyle`).

### 5.2 Weaknesses & Edge Cases  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Regex‑heavy `cleanWord`** | Brittle against new Word HTML patterns; may strip legitimate markup or leave remnants. | Use DOM parsing (e.g., create a temporary `div`, manipulate nodes) instead of string replacement; or integrate a dedicated HTML sanitizer library. |
| **Inline `javascript:void(0)` src** | Browser may warn or block scripts; cross‑domain restrictions can still bite. | Use `data:text/html,` or a separate HTML file served from the same origin. |
| **`setTimeout(…,0)` hack** | Can lead to race conditions if the editor is in a non‑standard state. | Consider `requestAnimationFrame` or queueing the insert in the editor’s command stack. |
| **No handling of large pasted content** | Might cause UI freeze during regex processing. | Break processing into chunks or use a Web Worker. |
| **Limited styling** | When `removeStyle` is unchecked, inline styles are retained, which can break editor themes. | Provide an optional sanitiser that normalises styles. |
| **Security** | Pasted content could contain scripts if not properly sanitized. | Ensure no `<script>` tags make it through `cleanWord`. |
| **No unit tests** | Hard to verify correctness after changes. | Add tests that feed typical Word‑pasted snippets and assert expected output. |

### 5.3 Future Enhancements  

1. **Modernise the cleaning logic** – Replace regex chain with a DOM‑based approach, making the code more maintainable and safer.  
2. **Accessibility improvements** – Add ARIA roles/labels for the iframe and check‑boxes; ensure keyboard navigation works.  
3. **Internationalisation** – Ensure that the dialog respects the editor’s `dir` and `lang` attributes more robustly.  
4. **Performance profiling** – Measure the time spent in `cleanWord` for large inputs and optimise if necessary.  
5. **Modularisation** – Extract `cleanWord` into a separate module or plugin so it can be reused or replaced.  
6. **Configuration extension** – Expose more options (e.g., allow tables, keep formatting, etc.) to give users finer control.  

---

### 6. Final Verdict  

The code effectively implements the classic “Paste from Word” dialog within CKEditor, covering many browser quirks and providing useful options. However, the heavy reliance on regular expressions for HTML cleaning makes the logic fragile and hard to maintain. Refactoring `cleanWord` to a DOM‑based approach, tightening security, and adding proper testing would greatly improve robustness and future‑proof the feature.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.dialog.add('pastefromword',function(a){return{title:a.lang.pastefromword.title,minWidth:CKEDITOR.env.ie&&CKEDITOR.env.quirks?370:350,minHeight:CKEDITOR.env.ie&&CKEDITOR.env.quirks?270:260,htmlToLoad:'<!doctype html><script type="text/javascript">window.onload = function(){if ( '+CKEDITOR.env.ie+' ) '+'document.body.contentEditable = "true";'+'else '+'document.designMode = "on";'+'var iframe = new window.parent.CKEDITOR.dom.element( frameElement );'+'var dialog = iframe.getCustomData( "dialog" );'+''+'iframe.getFrameDocument().on( "keydown", function( e )\t\t\t\t\t\t{\t\t\t\t\t\t\tif ( e.data.getKeystroke() == 27 )\t\t\t\t\t\t\t\tdialog.hide();\t\t\t\t\t\t});'+'dialog.fire( "iframeAdded", { iframe : iframe } );'+'};'+'</script><style>body { margin: 3px; height: 95%; } </style><body></body>',cleanWord:function(b,c,d,e){c=c.replace(/<\!--[\s\S]*?-->/g,'');c=c.replace(/<o:p>\s*<\/o:p>/g,'');c=c.replace(/<o:p>[\s\S]*?<\/o:p>/g,'&nbsp;');c=c.replace(/\s*mso-[^:]+:[^;"]+;?/gi,'');c=c.replace(/\s*MARGIN: 0(?:cm|in) 0(?:cm|in) 0pt\s*;/gi,'');c=c.replace(/\s*MARGIN: 0(?:cm|in) 0(?:cm|in) 0pt\s*"/gi,'"');c=c.replace(/\s*TEXT-INDENT: 0cm\s*;/gi,'');c=c.replace(/\s*TEXT-INDENT: 0cm\s*"/gi,'"');c=c.replace(/\s*TEXT-ALIGN: [^\s;]+;?"/gi,'"');c=c.replace(/\s*PAGE-BREAK-BEFORE: [^\s;]+;?"/gi,'"');c=c.replace(/\s*FONT-VARIANT: [^\s;]+;?"/gi,'"');c=c.replace(/\s*tab-stops:[^;"]*;?/gi,'');c=c.replace(/\s*tab-stops:[^"]*/gi,'');if(d){c=c.replace(/\s*face="[^"]*"/gi,'');c=c.replace(/\s*face=[^ >]*/gi,'');c=c.replace(/\s*FONT-FAMILY:[^;"]*;?/gi,'');}c=c.replace(/<(\w[^>]*) class=([^ |>]*)([^>]*)/gi,'<$1$3');if(e)c=c.replace(/<(\w[^>]*) style="([^\"]*)"([^>]*)/gi,'<$1$3');c=c.replace(/<STYLE[^>]*>[\s\S]*?<\/STYLE[^>]*>/gi,'');c=c.replace(/<(?:META|LINK)[^>]*>\s*/gi,'');c=c.replace(/\s*style="\s*"/gi,'');c=c.replace(/<SPAN\s*[^>]*>\s*&nbsp;\s*<\/SPAN>/gi,'&nbsp;');c=c.replace(/<SPAN\s*[^>]*><\/SPAN>/gi,'');c=c.replace(/<(\w[^>]*) lang=([^ |>]*)([^>]*)/gi,'<$1$3');c=c.replace(/<SPAN\s*>([\s\S]*?)<\/SPAN>/gi,'$1');c=c.replace(/<FONT\s*>([\s\S]*?)<\/FONT>/gi,'$1');c=c.replace(/<\\?\?xml[^>]*>/gi,'');c=c.replace(/<w:[^>]*>[\s\S]*?<\/w:[^>]*>/gi,'');c=c.replace(/<\/?\w+:[^>]*>/gi,'');c=c.replace(/<(U|I|STRIKE)>&nbsp;<\/\1>/g,'&nbsp;');c=c.replace(/<H\d>\s*<\/H\d>/gi,'');c=c.replace(/<(\w+)[^>]*\sstyle="[^"]*DISPLAY\s?:\s?none[\s\S]*?<\/\1>/ig,'');c=c.replace(/<(\w[^>]*) language=([^ |>]*)([^>]*)/gi,'<$1$3');c=c.replace(/<(\w[^>]*) onmouseover="([^\"]*)"([^>]*)/gi,'<$1$3');c=c.replace(/<(\w[^>]*) onmouseout="([^\"]*)"([^>]*)/gi,'<$1$3');
if(b.config.pasteFromWordKeepsStructure){c=c.replace(/<H(\d)([^>]*)>/gi,'<h$1>');c=c.replace(/<(H\d)><FONT[^>]*>([\s\S]*?)<\/FONT><\/\1>/gi,'<$1>$2</$1>');c=c.replace(/<(H\d)><EM>([\s\S]*?)<\/EM><\/\1>/gi,'<$1>$2</$1>');}else{c=c.replace(/<H1([^>]*)>/gi,'<div$1><b><font size="6">');c=c.replace(/<H2([^>]*)>/gi,'<div$1><b><font size="5">');c=c.replace(/<H3([^>]*)>/gi,'<div$1><b><font size="4">');c=c.replace(/<H4([^>]*)>/gi,'<div$1><b><font size="3">');c=c.replace(/<H5([^>]*)>/gi,'<div$1><b><font size="2">');c=c.replace(/<H6([^>]*)>/gi,'<div$1><b><font size="1">');c=c.replace(/<\/H\d>/gi,'</font></b></div>');var f=new RegExp('(<P)([^>]*>[\\s\\S]*?)(</P>)','gi');c=c.replace(f,'<div$2</div>');c=c.replace(/<([^\s>]+)(\s[^>]*)?>\s*<\/\1>/g,'');c=c.replace(/<([^\s>]+)(\s[^>]*)?>\s*<\/\1>/g,'');c=c.replace(/<([^\s>]+)(\s[^>]*)?>\s*<\/\1>/g,'');}return c;},onShow:function(){var g=this;if(CKEDITOR.env.ie)g.getParentEditor().document.getBody().$.contentEditable='false';g.parts.dialog.$.offsetHeight;var b=g.getContentElement('general','editing_area').getElement(),c=CKEDITOR.dom.element.createFromHtml('<iframe src="javascript:void(0)" frameborder="0" allowtransparency="1"></iframe>'),d=g.getParentEditor().lang;c.setStyles({width:'346px',height:'152px','background-color':'white',border:'1px solid black'});c.setCustomData('dialog',g);var e=d.editorTitle.replace('%1',d.pastefromword.title);if(CKEDITOR.env.ie)b.setHtml('<legend style="position:absolute;top:-1000000px;left:-1000000px;">'+CKEDITOR.tools.htmlEncode(e)+'</legend>');else{b.setHtml('');b.setAttributes({role:'region',title:e});c.setAttributes({role:'region',title:' '});}b.append(c);if(CKEDITOR.env.ie)b.setStyle('height',c.$.offsetHeight+2+'px');if(CKEDITOR.env.isCustomDomain()){CKEDITOR._cke_htmlToLoad=g.definition.htmlToLoad;c.setAttribute('src','javascript:void( (function(){document.open();document.domain="'+document.domain+'";'+'document.write( window.parent.CKEDITOR._cke_htmlToLoad );'+'delete window.parent.CKEDITOR._cke_htmlToLoad;'+'document.close();'+'})() )');}else{var f=c.$.contentWindow.document;f.open();f.write(g.definition.htmlToLoad);f.close();}},onOk:function(){var b=this.getContentElement('general','editing_area').getElement(),c=b.getElementsByTag('iframe').getItem(0),d=this.getParentEditor(),e=this.definition.cleanWord(d,c.$.contentWindow.document.body.innerHTML,this.getValueOf('general','ignoreFontFace'),this.getValueOf('general','removeStyle'));setTimeout(function(){d.insertHtml(e);},0);},onHide:function(){if(CKEDITOR.env.ie)this.getParentEditor().document.getBody().$.contentEditable='true';
},onLoad:function(){if((CKEDITOR.env.ie7Compat||CKEDITOR.env.ie6Compat)&&(a.lang.dir=='rtl'))this.parts.contents.setStyle('overflow','hidden');},contents:[{id:'general',label:a.lang.pastefromword.title,elements:[{type:'html',style:'white-space:normal;width:346px;display:block',onShow:function(){if(CKEDITOR.env.webkit)this.getElement().getAscendant('table').setStyle('table-layout','fixed');},html:a.lang.pastefromword.advice},{type:'html',id:'editing_area',style:'width: 100%; height: 100%;',html:'<fieldset></fieldset>',focus:function(){var b=this.getElement(),c=b.getElementsByTag('iframe');if(c.count()<1)return;c=c.getItem(0);setTimeout(function(){c.$.contentWindow.focus();},500);}},{type:'vbox',padding:0,children:[{type:'checkbox',id:'ignoreFontFace',label:a.lang.pastefromword.ignoreFontFace,'default':a.config.pasteFromWordIgnoreFontFace},{type:'checkbox',id:'removeStyle',label:a.lang.pastefromword.removeStyle,'default':a.config.pasteFromWordRemoveStyle}]}]}]};});



```
