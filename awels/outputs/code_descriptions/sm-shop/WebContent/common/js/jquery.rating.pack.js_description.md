# jquery.rating.pack.js

## Review

## 1. Summary  
**Purpose** – The script is a classic jQuery star‑rating widget (originally *jQuery Star Rating Plugin v3.12*).  
It converts one or more `<input>` fields (typically radios or a hidden text input) into an interactive star‑based UI that lets users rate items.  

**Key Components**  
| Component | Role |
|-----------|------|
| **Plugin initialiser** – `$.p.4` (the minified/obfuscated `$.fn.rating`) | Attaches the rating widget to selected elements and creates the star DOM. |
| **Event handlers** – `click`, `mouseover`, `mouseout`, `focus`, `blur`, `keydown`, `keyup` | Manage user interaction, hover previews, rating selection, keyboard accessibility and cancel option. |
| **State management** – internal data object `f` (kept in the element’s data store) | Keeps current rating, options, DOM references to stars, cancel button, etc. |
| **Drawing logic** – helper that appends `<li>`/`<span>` elements representing stars | Renders the visual rating and updates it on user input. |
| **Callbacks** – `onChange`, `onSubmit`, `onCancel` | Allow consumers to hook into rating changes. |
| **Accessibility helpers** – `role="slider"`, ARIA attributes, keyboard focus handling | Makes the widget accessible to screen‑reader users. |

**Design patterns / libraries**  
* Uses the jQuery plugin pattern (`$.fn`) wrapped in an immediately‑invoked function expression (IIFE).  
* Heavy minification (obfuscation) – all identifiers reduced to one‑letter names, and the whole body is wrapped in a custom `eval` routine.  
* The code assumes jQuery is loaded and that the page has the *BackgroundImageCache* plugin for IE support.

---

## 2. Detailed Description  

### 2.1. Initialisation  

1. **Selector** – The plugin is called on any jQuery collection: `$('.rating').rating(options);`  
2. **Options** – Default options are defined in `$.p.4.18` (minified). They include:  
   * `starWidth` – width of a single star in pixels (default 16).  
   * `rating` – starting rating value.  
   * `readOnly` – boolean to lock the widget.  
   * `cancel` – boolean to show a cancel button.  
   * `cancelValue` – rating value to apply when cancel is clicked (default 0).  
   * `theme`, `class`, `width`, `number` – visual styling options.  
3. **Element handling** – For each element in the collection:  
   * If the element is an `<input>` of type `radio`, the plugin groups all radios with the same name to form a single widget.  
   * For a hidden input the widget is built around it, and its value is updated directly.

### 2.2. Rendering the UI  

* The widget creates a `<div>` container with a class like `jStar`, inside which it appends `<li>` or `<span>` elements for each star.  
* When `cancel` is true, an additional list item is inserted before the first star, representing the cancel button.  
* Each star has a data attribute holding its numeric value (`data-rater-value`).  
* The width of the container is calculated from `number * starWidth`.  

### 2.3. User Interaction  

| Event | Behaviour | Notes |
|-------|-----------|-------|
| **hover / mouseover** | Highlights stars up to the hovered one; preview rating is displayed. | Uses CSS classes `over` and `current`. |
| **mouseout / mouseleave** | Restores the visual state to the last selected rating. | Prevents flicker by storing the current value. |
| **click** | Sets the rating to the clicked star, updates the underlying input, fires callbacks. | If the cancel button is clicked and `cancelValue` is set, the rating is reset to that value. |
| **focus / blur** | Adds/removes the `focus` class for keyboard navigation. | Accessibility. |
| **keydown / keyup** | Arrow keys adjust rating incrementally, Space/Enter confirm selection. | Full keyboard support. |
| **change** | Triggered when the underlying input changes (e.g., form reset). | Re‑draws the widget accordingly. |

### 2.4. Clean‑up  

The plugin exposes a `destroy` method (`$.p.4.24` / `25`) that removes all event handlers and DOM nodes, restoring the original `<input>` element.

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| **`$.p.4`** (plugin initialiser) | Binds rating UI to elements, processes options, groups radios. | jQuery collection, options object | DOM elements with event handlers | Modifies DOM, stores data in element via `$()._data()` |
| **`$.p.4.18`** | Default options. | None | Object | None |
| **`draw`** (internal) | Renders star list, cancel button, calculates widths. | options, element | DOM appended | Creates `<div>`, `<ul>`, `<li>` etc. |
| **`setRating(value)`** | Updates widget to reflect a new rating. | Numeric value | None | Updates DOM, changes input value |
| **`cancelRating()`** | Resets rating to cancel value. | None | None | Updates DOM, clears data |
| **`focus()` / `blur()`** | Keyboard focus handling. | None | None | Adds/removes `focus` class |
| **`hover(index)`** | Highlights stars up to index. | Index of star | None | Adds `over` class |
| **`click(index)`** | Handles click on star or cancel. | Index of star | None | Calls `setRating` or `cancelRating` |
| **`destroy()`** | Removes widget, event handlers, restores original element. | None | None | Removes DOM nodes, data |

> **Note** – Because of minification the method names appear as `.1f`, `.G`, `.v`, `.w`, `.m`, etc. In a readable source they would be more descriptive (`set`, `get`, `destroy`, `reset`, `cancel`, `disable`, `enable`).

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery** (core) | Third‑party | Required. The plugin uses `$`, `$.fn`, and many jQuery utilities. |
| **BackgroundImageCache** | Third‑party (optional) | Used for IE background‑image handling; not strictly required but improves cross‑browser behaviour. |
| **IE-specific code** | Browser quirk | Detects `$.browser.msie` to apply special handling. |
| **Optional CSS** | None | The plugin expects external CSS classes (`jStar`, `over`, `current`, `cancel`, etc.) to style stars. |

All other functionality is contained within the plugin file; there are no network calls or external APIs.

---

## 5. Additional Notes  

### 5.1 Readability & Maintainability  
* The code is **heavily obfuscated** (single‑letter function names, packed `eval`).  
* This makes manual debugging, contribution, or extension extremely difficult.  
* Recommended improvement: **Unminify / de‑obfuscate** the source before adding new features. Tools like [jsnice.org](https://www.jsnice.org) or a custom script can aid this.

### 5.2 Edge Cases & Limitations  
| Scenario | Current Handling | Potential Issue |
|----------|------------------|-----------------|
| **Disabled input** | The plugin checks `this.disabled` and skips event binding. | Works fine. |
| **Read‑only mode** | Adds a `readOnly` flag; disables click/hover. | Good. |
| **Half‑star support** | The plugin originally had half‑rating logic, but it’s commented out in the obfuscated version (`f.x`). | If half‑rating is needed, the logic may be incomplete. |
| **Large number of stars (>20)** | CSS overflow may occur if container width is too large. | Could add a max‑width or wrap stars. |
| **Keyboard navigation on touch devices** | Focus handling may not be useful on touch screens. | Add touch‑specific events if needed. |

### 5.3 Performance  
* The plugin creates one DOM element per star and attaches individual handlers for each.  
* For typical use (≤10 stars) the overhead is negligible.  
* For large scales (hundreds of stars per page) consider event delegation to reduce handler count.

### 5.4 Future Enhancements  
1. **Modern API** – Wrap the plugin in a ES6 class, expose a Promise‑based `setRating` and `getRating`.  
2. **Internationalisation** – Allow localisation of the cancel tooltip and ARIA labels.  
3. **Accessibility improvements** – Implement live regions for screen‑readers.  
4. **Unit tests** – Add Jest/ Mocha tests covering all interaction paths.  
5. **Remove `eval`** – Rewrite the minification routine to a normal IIFE; `eval` hampers debugging and security.  

---

### Bottom‑Line  
The script is a functional, well‑documented jQuery star‑rating widget. The main drawback is the **extreme obfuscation**, which hampers readability, debugging, and future development. If you’re maintaining or extending this code, first **unobfuscate** it, then consider refactoring to a modern, modular structure.

## Code Critique



## Code Preview

```javascript
/*
 ### jQuery Star Rating Plugin v3.12 - 2009-04-16 ###
 * Home: http://www.fyneworks.com/jquery/star-rating/
 * Code: http://code.google.com/p/jquery-star-rating-plugin/
 *
	* Dual licensed under the MIT and GPL licenses:
 *   http://www.opensource.org/licenses/mit-license.php
 *   http://www.gnu.org/licenses/gpl.html
 ###
*/
eval(function(p,a,c,k,e,r){e=function(c){return(c<a?'':e(parseInt(c/a)))+((c=c%a)>35?String.fromCharCode(c+29):c.toString(36))};if(!''.replace(/^/,String)){while(c--)r[e(c)]=k[c]||e(c);k=[function(e){return r[e]}];e=function(){return'\\w+'};c=1};while(c--)if(k[c])p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c]);return p}(';5(1O.1t)(7($){5($.29.1x)1I{1m.23("1u",P,z)}1F(e){}$.p.4=7(j){5(3.K==0)l 3;5(E J[0]==\'1j\'){5(3.K>1){8 k=J;l 3.W(7(){$.p.4.H($(3),k)})};$.p.4[J[0]].H(3,$.1T(J).21(1)||[]);l 3};8 j=$.10({},$.p.4.18,j||{});3.1v(\'.9-4-1l\').n(\'9-4-1l\').W(7(){8 a=(3.1J||\'1K-4\').1L(/\\[|\\]+/g,"1S");8 b=$(3.1U||1m.1X);8 c=$(3);8 d=b.6(\'4\')||{y:0};8 e=d[a];8 f;5(e)f=e.6(\'4\');5(e&&f){f.y++}B{f=$.10({},j||{},($.1k?c.1k():($.1H?c.6():s))||{},{y:0,C:[],u:[]});f.t=d.y++;e=$(\'<1M 12="9-4-1Q"/>\');c.1R(e);e.n(\'4-T-13-S\');5(c.R(\'Q\'))f.m=z;e.1a(f.A=$(\'<O 12="4-A"><a 14="\'+f.A+\'">\'+f.15+\'</a></O>\').1d(7(){$(3).4(\'N\');$(3).n(\'9-4-M\')}).1b(7(){$(3).4(\'v\');$(3).D(\'9-4-M\')}).1h(7(){$(3).4(\'w\')}).6(\'4\',f))};8 g=$(\'<O 12="9-4 q-\'+f.t+\'"><a 14="\'+(3.14||3.1p)+\'">\'+3.1p+\'</a></O>\');e.1a(g);5(3.U)g.R(\'U\',3.U);5(3.17)g.n(3.17);5(f.1V)f.x=2;5(E f.x==\'19\'&&f.x>0){8 h=($.p.11?g.11():0)||f.1c;8 i=(f.y%f.x),V=1y.1z(h/f.x);g.11(V).1A(\'a\').1B({\'1C-1D\':\'-\'+(i*V)+\'1E\'})};5(f.m)g.n(\'9-4-1e\');B g.n(\'9-4-1G\').1d(7(){$(3).4(\'1f\');$(3).4(\'G\')}).1b(7(){$(3).4(\'v\');$(3).4(\'F\')}).1h(7(){$(3).4(\'w\')});5(3.L)f.o=g;c.1i();c.1N(7(){$(3).4(\'w\')});g.6(\'4.r\',c.6(\'4.9\',g));f.C[f.C.K]=g[0];f.u[f.u.K]=c[0];f.q=d[a]=e;f.1P=b;c.6(\'4\',f);e.6(\'4\',f);g.6(\'4\',f);b.6(\'4\',d)});$(\'.4-T-13-S\').4(\'v\').D(\'4-T-13-S\');l 3};$.10($.p.4,{G:7(){8 a=3.6(\'4\');5(!a)l 3;5(!a.G)l 3;8 b=$(3).6(\'4.r\')||$(3.Z==\'X\'?3:s);5(a.G)a.G.H(b[0],[b.I(),$(\'a\',b.6(\'4.9\'))[0]])},F:7(){8 a=3.6(\'4\');5(!a)l 3;5(!a.F)l 3;8 b=$(3).6(\'4.r\')||$(3.Z==\'X\'?3:s);5(a.F)a.F.H(b[0],[b.I(),$(\'a\',b.6(\'4.9\'))[0]])},1f:7(){8 a=3.6(\'4\');5(!a)l 3;5(a.m)l;3.4(\'N\');3.1n().1o().Y(\'.q-\'+a.t).n(\'9-4-M\')},N:7(){8 a=3.6(\'4\');5(!a)l 3;5(a.m)l;a.q.1W().Y(\'.q-\'+a.t).D(\'9-4-1q\').D(\'9-4-M\')},v:7(){8 a=3.6(\'4\');5(!a)l 3;3.4(\'N\');5(a.o){a.o.6(\'4.r\').R(\'L\',\'L\');a.o.1n().1o().Y(\'.q-\'+a.t).n(\'9-4-1q\')}B $(a.u).1r(\'L\');a.A[a.m||a.1Y?\'1i\':\'1Z\']();3.20()[a.m?\'n\':\'D\'](\'9-4-1e\')},w:7(a){8 b=3.6(\'4\');5(!b)l 3;5(b.m)l;b.o=s;5(E a!=\'1s\'){5(E a==\'19\')l $(b.C[a]).4(\'w\');5(E a==\'1j\')$.W(b.C,7(){5($(3).6(\'4.r\').I()==a)$(3).4(\'w\')})}B b.o=3[0].Z==\'X\'?3.6(\'4.9\'):(3.22(\'.q-\'+b.t)?3:s);3.6(\'4\',b);3.4(\'v\');8 c=$(b.o?b.o.6(\'4.r\'):s);5(b.1g)b.1g.H(c[0],[c.I(),$(\'a\',b.o)[0]])},m:7(a,b){8 c=3.6(\'4\');5(!c)l 3;c.m=a||a==1s?z:P;5(b)$(c.u).R("Q","Q");B $(c.u).1r("Q");3.6(\'4\',c);3.4(\'v\')},24:7(){3.4(\'m\',z,z)},25:7(){3.4(\'m\',P,P)}});$.p.4.18={A:\'26 27\',15:\'\',x:0,1c:16};$(7(){$(\'r[28=1w].9\').4()})})(1t);',62,134,'|||this|rating|if|data|function|var|star||||||||||||return|readOnly|addClass|current|fn|rater|input|null|serial|inputs|draw|select|split|count|true|cancel|else|stars|removeClass|typeof|blur|focus|apply|val|arguments|length|checked|hover|drain|div|false|disabled|attr|drawn|to|id|spw|each|INPUT|filter|tagName|extend|width|class|be|title|cancelValue||className|options|number|append|mouseout|starWidth|mouseover|readonly|fill|callback|click|hide|string|metadata|applied|document|prevAll|andSelf|value|on|removeAttr|undefined|jQuery|BackgroundImageCache|not|radio|msie|Math|floor|find|css|margin|left|px|catch|live|meta|try|name|unnamed|replace|span|change|window|context|control|before|_|makeArray|form|half|children|body|required|show|siblings|slice|is|execCommand|disable|enable|Cancel|Rating|type|browser'.split('|'),0,{}))


```
