# jquery.prettyPhoto.js

## Review

## 1. Summary

**Purpose**  
`prettyPhoto` is a lightweight jQuery lightbox clone that can display images, Flash, QuickTime, iframes, inline content, YouTube and Vimeo videos. It overlays the page, provides navigation controls, captions, and optional auto‑play.

**Key Components**

| Component | Role |
|-----------|------|
| **$.prettyPhoto** | Global namespace holding configuration and public API (`open`, `close`, `changePage`) |
| **$.fn.prettyPhoto** | jQuery plugin entry point that sets up event handlers on the target set |
| **Markup strings** (`markup`, `image_markup`, …) | Template strings that are injected into the DOM |
| **Private helper functions** (`_buildOverlay`, `_centerOverlay`, …) | Encapsulated logic for layout, positioning, resizing, media type detection, etc. |
| **Event bindings** | Click on trigger elements, keyboard navigation, window scroll/resize, and overlay clicks |

**Design patterns & libraries**

* **Module pattern (IIFE)** – Wraps the whole library, exposing only the jQuery plugin API.
* **jQuery plugin convention** – `$.fn.prettyPhoto` returns `this` for chainability.
* **Event delegation** – Each trigger element gets its own click handler; overlay and navigation use `.bind()`.
* **Feature detection** – Basic browser checks (`$.browser`) for IE6/7 fallbacks and flash visibility.

---

## 2. Detailed Description

### Initialization

```javascript
$.fn.prettyPhoto = function(settings) {
    settings = $.extend(defaults, settings);
    if ($('.pp_overlay').size() == 0) _buildOverlay();

    // bind window events
    $(window).on('scroll', function(){ … });
    $(window).on('resize', function(){ … });
    $(document).on('keydown', function(e){ … });

    // bind trigger elements
    this.each(function(){
        $(this).on('click', function(){ … });
    });
};
```

* The plugin merges user options with defaults.
* An overlay container (`.pp_overlay`) is created once.
* Global events (scroll, resize, keyboard) are set up to keep the lightbox centered and responsive.
* Each trigger element receives a click handler that collects gallery information and calls `$.prettyPhoto.open`.

### Runtime Behavior

1. **Trigger click**  
   *Collects* all links that belong to the same gallery (based on `rel="[gallery]"`) and stores the array of `images`, `titles`, and `descriptions`.  
   Calls `$.prettyPhoto.open`.

2. **`$.prettyPhoto.open`**  
   * Sets up the overlay, shows the loader icon.  
   * Calculates image/flash dimensions (`grab_param` is used to pull width/height from URL query strings).  
   * Preloads the current image and (if an image) its neighbours.  
   * On load success → calls `_showContent`.  
   * On load error → alerts and closes.

3. **`_showContent`**  
   * Hides the loader.  
   * Calculates the projected position (centered within the viewport).  
   * Animates the container to its new size/position.  
   * Shows the navigation controls if necessary.

4. **Navigation** (`changePage`)  
   * Updates `setPosition`, preloads the next/prev media, then calls `open` again.

5. **Close** (`$.prettyPhoto.close`)  
   * Fades out the overlay and container, resets visibility, restores hidden form elements for IE6, calls the user callback.

### Cleanup

* The overlay is never removed from the DOM; it is simply hidden.  
* No explicit memory‑cleanup logic is present—images/embeds are kept in the DOM until the next open call.

---

## 3. Functions / Methods

| Function | Purpose | Inputs | Outputs | Side‑Effects |
|----------|---------|--------|---------|--------------|
| **`$.prettyPhoto.open(gallery_images, gallery_titles, gallery_descriptions)`** | Public API – opens the lightbox for a gallery. | Arrays of image URLs, titles, descriptions. | None (manipulates DOM). | Shows overlay, preloads media, triggers `_showContent`. |
| **`$.prettyPhoto.changePage(direction)`** | Public API – navigates to next/previous media. | `"next"` or `"previous"`. | None. | Updates `setPosition`, calls `_hideContent` → `open`. |
| **`$.prettyPhoto.close()`** | Public API – closes the lightbox. | None. | None. | Hides overlay, restores hidden elements, triggers user callback. |
| **`_buildOverlay()`** | Creates the overlay DOM structure and binds core events. | None. | None. | Adds markup to body, sets up navigation & close handlers. |
| **`_centerOverlay()`** | Keeps the overlay centered during scroll/resize. | None. | None. | Positions `.pp_pic_holder` and `.ppt`. |
| **`_resizeOverlay()`** | Adjusts overlay height to match document height. | None. | None. | Sets `.pp_overlay` height. |
| **`_getScroll()`** | Retrieves the current scroll position in a cross‑browser way. | None. | `{scrollTop, scrollLeft}` | None. |
| **`_getFileType(itemSrc)`** | Detects the media type based on URL pattern. | `string` | `string` (e.g. `'image'`, `'youtube'`, etc.) | None. |
| **`_fitToViewport(width, height)`** | Calculates size adjustments to fit inside the viewport. | `number` `width`, `number` `height` | `{width, height, containerHeight, containerWidth, contentHeight, contentWidth, resized}` | May modify `hasBeenResized`. |
| **`_getDimensions(width, height)`** | Calculates container dimensions, accounting for title/footer. | `number` `width`, `number` `height` | None. | Sets global `pp_contentHeight`, `pp_containerHeight`, etc. |
| **`_checkPosition(setCount)`** | Enables/disables navigation controls based on current index. | `number` `setCount` | None. | Shows/hides next/prev buttons. |
| **`_showContent()`** | Animates the container to show the current media. | None. | None. | Animates `.pp_pic_holder` and its children. |
| **`_hideContent(callback)`** | Hides the current media before a page change. | `function` `callback` | None. | Fades out media, then calls `callback`. |
| **`grab_param(name, url)`** | Retrieves a query string value from a URL. | `string` `name`, `string` `url` | `string` value | None. |

### Reusable / Utility Methods

* `grab_param` – Generic query‑string parser used by many media types.
* `unescape` – Used to decode HTML entities from titles/descriptions (deprecated in modern browsers; `decodeURIComponent` is safer).

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| **jQuery 1.x** | Third‑party | Relies on `$.browser`, `$.extend`, `$.makeArray`, `.size()`. Older APIs (`$.browser`, `.size()`) are deprecated in newer jQuery releases. |
| **Flash / QuickTime** | Browser plug‑ins | Requires users to have the relevant plug‑ins installed. |
| **YouTube / Vimeo** | External services | Uses the Flash embed URL (`/v/` for YouTube, `moogaloop.swf` for Vimeo). |
| **HTML5 `<iframe>`** | Browser native | For arbitrary iframe content. |
| **Legacy IE 6/7** | Browser-specific | Contains specific CSS/JS hacks for opacity and visibility. |

No other third‑party libraries are required.

---

## 5. Additional Notes & Recommendations

### 5.1 Code‑Quality Observations

| Issue | Impact | Recommendation |
|-------|--------|----------------|
| **Use of deprecated jQuery APIs** (`$.browser`, `.size()`, `.bind()`, `unescape`) | Breaks on jQuery ≥3.0; `unescape` is unsafe | Replace with `$.isPlainObject`, `.length`, `.on()`, `decodeURIComponent`. |
| **Global variables** (`images`, `titles`, `descriptions`, `setPosition`, `doresize`, `percentBased`, etc.) | Hard to test, possible name clashes | Encapsulate inside an object or closure; use `const/let`. |
| **Hard‑coded selectors** (`$('.pp_overlay')`, `$('object,embed')`) | May break if markup changes | Cache selectors locally, use data attributes to find elements. |
| **String concatenation for markup** | Susceptible to injection if user supplies malicious titles/descriptions | Escape HTML (`$.text()` or custom escaping) instead of `unescape`. |
| **No support for modern browsers’ media APIs** | Flash is obsolete | Consider using the `<video>` element and HTML5 APIs for YouTube/Vimeo (e.g., iframe API). |
| **Lack of `aria` attributes** | Accessibility issues | Add ARIA roles and labels to the overlay and navigation. |
| **Event binding duplication** | Potential memory leaks if plugin is used multiple times | Use event delegation (`$(document).on(...)`) or `$.fn.prettyPhoto` cleanup. |
| **Error handling** | Only alerts on image load failure | Provide a fallback UI and a more graceful degradation. |

### 5.2 Performance & UX Enhancements

1. **Lazy‑load next/prev media** – Already preloads neighbours, but we can defer loading until needed.  
2. **Responsive design** – Currently uses hard‑coded pixel values (`-200` margin). Consider a CSS‑only solution using `max-width: 90vw; max-height: 90vh;`.  
3. **Touch support** – Swipe gestures for mobile devices.  
4. **Keyboard focus trap** – Keep focus inside the overlay for accessibility.  

### 5.3 Security Improvements

* Replace `unescape` with proper HTML entity encoding (`$.text()` or a dedicated escape function).  
* Sanitize `description` and `title` before injecting into the DOM to prevent XSS.  
* Validate URLs passed to `grab_param` to avoid injection in query strings.  

### 5.4 Testing & Documentation

| Area | Suggested Tests |
|------|-----------------|
| `_getFileType` | Pass all supported URL patterns and assert return values. |
| `_fitToViewport` | Test various viewport and media sizes (large, small, percentage based). |
| `open` / `changePage` | Verify correct media loads, navigation updates, and callback invocation. |
| Keyboard events | Simulate key codes and assert navigation/close behavior. |

Documentation should describe:

* How to initialize (`$('.gallery').prettyPhoto({options})`).  
* Supported media types and URL formats.  
* Customization of markup and themes.  
* Known limitations (IE6, Flash, etc.).  

### 5.5 Potential Future Enhancements

* **Modular build** – Split core logic and themes into separate files.  
* **Modern build tools** – Convert to ES6 modules and bundle with Webpack or Rollup.  
* **Internationalization** – Externalize text strings for easier localization.  
* **Unit‑testable API** – Expose helper functions for easier testing.  
* **Better plugin discovery** – Use `$.fn.prettyPhoto` to automatically initialize elements with `data-pretty-photo`.  

---

### Final Verdict

`prettyPhoto` is a mature, feature‑rich lightbox that served many sites in the early 2010s. Its architecture is clear, but the code relies on legacy jQuery features and browser plug‑ins that are no longer standard. For a modern codebase, a refactor is recommended:

1. **Remove deprecated APIs and polyfills** (switch to jQuery 3+ or vanilla JS).  
2. **Encapsulate state** in a class or factory pattern to avoid globals.  
3. **Adopt HTML5 media elements** instead of Flash/QuickTime.  
4. **Add accessibility and security hardening**.  

With these changes the library would stay functional while aligning with current web development best practices.

## Code Critique



## Code Preview

```javascript
/* ------------------------------------------------------------------------
 * 	Class: prettyPhoto
 * 	Use: Lightbox clone for jQuery
 * 	Author: Stephane Caron (http://www.no-margin-for-errors.com)
 * 	Version: 2.5.6
 ------------------------------------------------------------------------- */

(function($){$.prettyPhoto={version:'2.5.6'};$.fn.prettyPhoto=function(settings){settings=jQuery.extend({animationSpeed:'normal',opacity:0.80,showTitle:true,allowresize:true,default_width:500,default_height:344,counter_separator_label:'/',theme:'light_rounded',hideflash:false,wmode:'opaque',autoplay:true,modal:false,changepicturecallback:function(){},callback:function(){},markup:'<div class="pp_pic_holder"> \
      <div class="pp_top"> \
       <div class="pp_left"></div> \
       <div class="pp_middle"></div> \
       <div class="pp_right"></div> \
      </div> \
      <div class="pp_content_container"> \
       <div class="pp_left"> \
       <div class="pp_right"> \
        <div class="pp_content"> \
         <div class="pp_loaderIcon"></div> \
         <div class="pp_fade"> \
          <a href="#" class="pp_expand" title="Expand the image">Expand</a> \
          <div class="pp_hoverContainer"> \
           <a class="pp_next" href="#">next</a> \
           <a class="pp_previous" href="#">previous</a> \
          </div> \
          <div id="pp_full_res"></div> \
          <div class="pp_details clearfix"> \
           <a class="pp_close" href="#">Close</a> \
           <p class="pp_description"></p> \
           <div class="pp_nav"> \
            <a href="#" class="pp_arrow_previous">Previous</a> \
            <p class="currentTextHolder">0/0</p> \
            <a href="#" class="pp_arrow_next">Next</a> \
           </div> \
          </div> \
         </div> \
        </div> \
       </div> \
       </div> \
      </div> \
      <div class="pp_bottom"> \
       <div class="pp_left"></div> \
       <div class="pp_middle"></div> \
       <div class="pp_right"></div> \
      </div> \
     </div> \
     <div class="pp_overlay"></div> \
     <div class="ppt"></div>',image_markup:'<img id="fullResImage" src="" />',flash_markup:'<object classid="clsid:D27CDB6E-AE6D-11cf-96B8-444553540000" width="{width}" height="{height}"><param name="wmode" value="{wmode}" /><param name="allowfullscreen" value="true" /><param name="allowscriptaccess" value="always" /><param name="movie" value="{path}" /><embed src="{path}" type="application/x-shockwave-flash" allowfullscreen="true" allowscriptaccess="always" width="{width}" height="{height}" wmode="{wmode}"></embed></object>',quicktime_markup:'<object classid="clsid:02BF25D5-8C17-4B23-BC80-D3488ABDDC6B" codebase="http://www.apple.com/qtactivex/qtplugin.cab" height="{height}" width="{width}"><param name="src" value="{path}"><param name="autoplay" value="{autoplay}"><param name="type" value="video/quicktime"><embed src="{path}" height="{height}" width="{width}" autoplay="{autoplay}" type="video/quicktime" pluginspage="http://www.apple.com/quicktime/download/"></embed></object>',iframe_markup:'<iframe src ="{path}" width="{width}" height="{height}" frameborder="no"></iframe>',inline_markup:'<div class="pp_inline clearfix">{content}</div>'},settings);if($.browser.msie&&parseInt($.browser.version)==6){settings.theme="light_square";}
if($('.pp_overlay').size()==0)_buildOverlay();var doresize=true,percentBased=false,correctSizes,$pp_pic_holder,$ppt,$pp_overlay,pp_contentHeight,pp_contentWidth,pp_containerHeight,pp_containerWidth,windowHeight=$(window).height(),windowWidth=$(window).width(),setPosition=0,scrollPos=_getScroll();$(window).scroll(function(){scrollPos=_getScroll();_centerOverlay();_resizeOverlay();});$(window).resize(function(){_centerOverlay();_resizeOverlay();});$(document).keydown(function(e){if($pp_pic_holder.is(':visible'))
switch(e.keyCode){case 37:$.prettyPhoto.changePage('previous');break;case 39:$.prettyPhoto.changePage('next');break;case 27:if(!settings.modal)
$.prettyPhoto.close();break;};});$(this).each(function(){$(this).bind('click',function(){_self=this;theRel=$(this).attr('rel');galleryRegExp=/\[(?:.*)\]/;theGallery=galleryRegExp.exec(theRel);var images=new Array(),titles=new Array(),descriptions=new Array();if(theGallery){$('a[rel*='+theGallery+']').each(function(i){if($(this)[0]===$(_self)[0])setPosition=i;images.push($(this).attr('href'));titles.push($(this).find('img').attr('alt'));descriptions.push($(this).attr('title'));});}else{images=$(this).attr('href');titles=($(this).find('img').attr('alt'))?$(this).find('img').attr('alt'):'';descriptions=($(this).attr('title'))?$(this).attr('title'):'';}
$.prettyPhoto.open(images,titles,descriptions);return false;});});$.prettyPhoto.open=function(gallery_images,gallery_titles,gallery_descriptions){if($.browser.msie&&$.browser.version==6){$('select').css('visibility','hidden');};if(settings.hideflash)$('object,embed').css('visibility','hidden');images=$.makeArray(gallery_images);titles=$.makeArray(gallery_titles);descriptions=$.makeArray(gallery_descriptions);image_set=($(images).size()>0)?true:false;_checkPosition($(images).size());$('.pp_loaderIcon').show();$pp_overlay.show().fadeTo(settings.animationSpeed,settings.opacity);$pp_pic_holder.find('.currentTextHolder').text((setPosition+1)+settings.counter_separator_label+$(images).size());if(descriptions[setPosition]){$pp_pic_holder.find('.pp_description').show().html(unescape(descriptions[setPosition]));}else{$pp_pic_holder.find('.pp_description').hide().text('');};if(titles[setPosition]&&settings.showTitle){hasTitle=true;$ppt.html(unescape(titles[setPosition]));}else{hasTitle=false;};movie_width=(parseFloat(grab_param('width',images[setPosition])))?grab_param('width',images[setPosition]):settings.default_width.toString();movie_height=(parseFloat(grab_param('height',images[setPosition])))?grab_param('height',images[setPosition]):settings.default_height.toString();if(movie_width.indexOf('%')!=-1||movie_height.indexOf('%')!=-1){movie_height=parseFloat(($(window).height()*parseFloat(movie_height)/100)-100);movie_width=parseFloat(($(window).width()*parseFloat(movie_width)/100)-100);percentBased=true;}
$pp_pic_holder.fadeIn(function(){imgPreloader="";switch(_getFileType(images[setPosition])){case'image':imgPreloader=new Image();nextImage=new Image();if(image_set&&setPosition>$(images).size())nextImage.src=images[setPosition+1];prevImage=new Image();if(image_set&&images[setPosition-1])prevImage.src=images[setPosition-1];$pp_pic_holder.find('#pp_full_res')[0].innerHTML=settings.image_markup;$pp_pic_holder.find('#fullResImage').attr('src',images[setPosition]);imgPreloader.onload=function(){correctSizes=_fitToViewport(imgPreloader.width,imgPreloader.height);_showContent();};imgPreloader.onerror=function(){alert('Image cannot be loaded. Make sure the path is correct and image exist.');$.prettyPhoto.close();};imgPreloader.src=images[setPosition];break;case'youtube':correctSizes=_fitToViewport(movie_width,movie_height);movie='http://www.youtube.com/v/'+grab_param('v',images[setPosition]);if(settings.autoplay)movie+="&autoplay=1";toInject=settings.flash_markup.replace(/{width}/g,correctSizes['width']).replace(/{height}/g,correctSizes['height']).replace(/{wmode}/g,settings.wmode).replace(/{path}/g,movie);break;case'vimeo':correctSizes=_fitToViewport(movie_width,movie_height);movie_id=images[setPosition];movie='http://vimeo.com/moogaloop.swf?clip_id='+movie_id.replace('http://vimeo.com/','');if(settings.autoplay)movie+="&autoplay=1";toInject=settings.flash_markup.replace(/{width}/g,correctSizes['width']).replace(/{height}/g,correctSizes['height']).replace(/{wmode}/g,settings.wmode).replace(/{path}/g,movie);break;case'quicktime':correctSizes=_fitToViewport(movie_width,movie_height);correctSizes['height']+=15;correctSizes['contentHeight']+=15;correctSizes['containerHeight']+=15;toInject=settings.quicktime_markup.replace(/{width}/g,correctSizes['width']).replace(/{height}/g,correctSizes['height']).replace(/{wmode}/g,settings.wmode).replace(/{path}/g,images[setPosition]).replace(/{autoplay}/g,settings.autoplay);break;case'flash':correctSizes=_fitToViewport(movie_width,movie_height);flash_vars=images[setPosition];flash_vars=flash_vars.substring(images[setPosition].indexOf('flashvars')+10,images[setPosition].length);filename=images[setPosition];filename=filename.substring(0,filename.indexOf('?'));toInject=settings.flash_markup.replace(/{width}/g,correctSizes['width']).replace(/{height}/g,correctSizes['height']).replace(/{wmode}/g,settings.wmode).replace(/{path}/g,filename+'?'+flash_vars);break;case'iframe':correctSizes=_fitToViewport(movie_width,movie_height);frame_url=images[setPosition];frame_url=frame_url.substr(0,frame_url.indexOf('iframe')-1);toInject=settings.iframe_markup.replace(/{width}/g,correctSizes['width']).replace(/{height}/g,correctSizes['height']).replace(/{path}/g,frame_url);break;case'inline':myClone=$(images[setPosition]).clone().css({'width':settings.default_width}).wrapInner('<div id="pp_full_res"><div class="pp_inline clearfix"></div></div>').appendTo($('body'));correctSizes=_fitToViewport($(myClone).width(),$(myClone).height());$(myClone).remove();toInject=settings.inline_markup.replace(/{content}/g,$(images[setPosition]).html());break;};if(!imgPreloader){$pp_pic_holder.find('#pp_full_res')[0].innerHTML=toInject;_showContent();};});};$.prettyPhoto.changePage=function(direction){if(direction=='previous'){setPosition--;if(setPosition<0){setPosition=0;return;};}else{if($('.pp_arrow_next').is('.disabled'))return;setPosition++;};if(!doresize)doresize=true;_hideContent(function(){$.prettyPhoto.open(images,titles,descriptions)});$('a.pp_expand,a.pp_contract').fadeOut(settings.animationSpeed);};$.prettyPhoto.close=function(){$pp_pic_holder.find('object,embed').css('visibility','hidden');$('div.pp_pic_holder,div.ppt,.pp_fade').fadeOut(settings.animationSpeed);$pp_overlay.fadeOut(settings.animationSpeed,function(){$('#pp_full_res').html('');$pp_pic_holder.attr('style','').find('div:not(.pp_hoverContainer)').attr('style','');_centerOverlay();if($.browser.msie&&$.browser.version==6){$('select').css('visibility','visible');};if(settings.hideflash)$('object,embed').css('visibility','visible');setPosition=0;settings.callback();});doresize=true;};_showContent=function(){$('.pp_loaderIcon').hide();projectedTop=scrollPos['scrollTop']+((windowHeight/2)-(correctSizes['containerHeight']/2));if(projectedTop<0)projectedTop=0+$ppt.height();$pp_pic_holder.find('.pp_content').animate({'height':correctSizes['contentHeight']},settings.animationSpeed);$pp_pic_holder.animate({'top':projectedTop,'left':(windowWidth/2)-(correctSizes['containerWidth']/2),'width':correctSizes['containerWidth']},settings.animationSpeed,function(){$pp_pic_holder.find('.pp_hoverContainer,#fullResImage').height(correctSizes['height']).width(correctSizes['width']);$pp_pic_holder.find('.pp_fade').fadeIn(settings.animationSpeed);if(image_set&&_getFileType(images[setPosition])=="image"){$pp_pic_holder.find('.pp_hoverContainer').show();}else{$pp_pic_holder.find('.pp_hoverContainer').hide();}
if(settings.showTitle&&hasTitle){$ppt.css({'top':$pp_pic_holder.offset().top-25,'left':$pp_pic_holder.offset().left+20,'display':'none'});$ppt.fadeIn(settings.animationSpeed);};if(correctSizes['resized'])$('a.pp_expand,a.pp_contract').fadeIn(settings.animationSpeed);settings.changepicturecallback();});};function _hideContent(callback){$pp_pic_holder.find('#pp_full_res object,#pp_full_res embed').css('visibility','hidden');$pp_pic_holder.find('.pp_fade').fadeOut(settings.animationSpeed,function(){$('.pp_loaderIcon').show();if(callback)callback();});$ppt.fadeOut(settings.animationSpeed);}
function _checkPosition(setCount){if(setPosition==setCount-1){$pp_pic_holder.find('a.pp_next').css('visibility','hidden');$pp_pic_holder.find('a.pp_arrow_next').addClass('disabled').unbind('click');}else{$pp_pic_holder.find('a.pp_next').css('visibility','visible');$pp_pic_holder.find('a.pp_arrow_next.disabled').removeClass('disabled').bind('click',function(){$.prettyPhoto.changePage('next');return false;});};if(setPosition==0){$pp_pic_holder.find('a.pp_previous').css('visibility','hidden');$pp_pic_holder.find('a.pp_arrow_previous').addClass('disabled').unbind('click');}else{$pp_pic_holder.find('a.pp_previous').css('visibility','visible');$pp_pic_holder.find('a.pp_arrow_previous.disabled').removeClass('disabled').bind('click',function(){$.prettyPhoto.changePage('previous');return false;});};if(setCount>1){$('.pp_nav').show();}else{$('.pp_nav').hide();}};function _fitToViewport(width,height){hasBeenResized=false;_getDimensions(width,height);imageWidth=width;imageHeight=height;if(((pp_containerWidth>windowWidth)||(pp_containerHeight>windowHeight))&&doresize&&settings.allowresize&&!percentBased){hasBeenResized=true;notFitting=true;while(notFitting){if((pp_containerWidth>windowWidth)){imageWidth=(windowWidth-200);imageHeight=(height/width)*imageWidth;}else if((pp_containerHeight>windowHeight)){imageHeight=(windowHeight-200);imageWidth=(width/height)*imageHeight;}else{notFitting=false;};pp_containerHeight=imageHeight;pp_containerWidth=imageWidth;};_getDimensions(imageWidth,imageHeight);};return{width:Math.floor(imageWidth),height:Math.floor(imageHeight),containerHeight:Math.floor(pp_containerHeight),containerWidth:Math.floor(pp_containerWidth)+40,contentHeight:Math.floor(pp_contentHeight),contentWidth:Math.floor(pp_contentWidth),resized:hasBeenResized};};function _getDimensions(width,height){width=parseFloat(width);height=parseFloat(height);$pp_details=$pp_pic_holder.find('.pp_details');$pp_details.width(width);detailsHeight=parseFloat($pp_details.css('marginTop'))+parseFloat($pp_details.css('marginBottom'));$pp_details=$pp_details.clone().appendTo($('body')).css({'position':'absolute','top':-10000});detailsHeight+=$pp_details.height();detailsHeight=(detailsHeight<=34)?36:detailsHeight;if($.browser.msie&&$.browser.version==7)detailsHeight+=8;$pp_details.remove();pp_contentHeight=height+detailsHeight;pp_contentWidth=width;pp_containerHeight=pp_contentHeight+$ppt.height()+$pp_pic_holder.find('.pp_top').height()+$pp_pic_holder.find('.pp_bottom').height();pp_containerWidth=width;}
function _getFileType(itemSrc){if(itemSrc.match(/youtube\.com\/watch/i)){return'youtube';}else if(itemSrc.match(/vimeo\.com/i)){return'vimeo';}else if(itemSrc.indexOf('.mov')!=-1){return'quicktime';}else if(itemSrc.indexOf('.swf')!=-1){return'flash';}else if(itemSrc.indexOf('iframe')!=-1){return'iframe'}else if(itemSrc.substr(0,1)=='#'){return'inline';}else{return'image';};};function _centerOverlay(){if(doresize){titleHeight=$ppt.height();contentHeight=$pp_pic_holder.height();contentwidth=$pp_pic_holder.width();projectedTop=(windowHeight/2)+scrollPos['scrollTop']-((contentHeight+titleHeight)/2);$pp_pic_holder.css({'top':projectedTop,'left':(windowWidth/2)+scrollPos['scrollLeft']-(contentwidth/2)});$ppt.css({'top':projectedTop-titleHeight,'left':(windowWidth/2)+scrollPos['scrollLeft']-(contentwidth/2)+20});};};function _getScroll(){if(self.pageYOffset){return{scrollTop:self.pageYOffset,scrollLeft:self.pageXOffset};}else if(document.documentElement&&document.documentElement.scrollTop){return{scrollTop:document.documentElement.scrollTop,scrollLeft:document.documentElement.scrollLeft};}else if(document.body){return{scrollTop:document.body.scrollTop,scrollLeft:document.body.scrollLeft};};};function _resizeOverlay(){windowHeight=$(window).height();windowWidth=$(window).width();$pp_overlay.css({'height':$(document).height()});};function _buildOverlay(){$('body').append(settings.markup);$pp_pic_holder=$('.pp_pic_holder');$ppt=$('.ppt');$pp_overlay=$('div.pp_overlay');$pp_pic_holder.attr('class','pp_pic_holder '+settings.theme);$pp_overlay.css({'opacity':0,'height':$(document).height()}).bind('click',function(){if(!settings.modal)
$.prettyPhoto.close();});$('a.pp_close').bind('click',function(){$.prettyPhoto.close();return false;});$('a.pp_expand').bind('click',function(){$this=$(this);if($this.hasClass('pp_expand')){$this.removeClass('pp_expand').addClass('pp_contract');doresize=false;}else{$this.removeClass('pp_contract').addClass('pp_expand');doresize=true;};_hideContent(function(){$.prettyPhoto.open(images,titles,descriptions)});$pp_pic_holder.find('.pp_fade').fadeOut(settings.animationSpeed);return false;});$pp_pic_holder.find('.pp_previous, .pp_arrow_previous').bind('click',function(){$.prettyPhoto.changePage('previous');return false;});$pp_pic_holder.find('.pp_next, .pp_arrow_next').bind('click',function(){$.prettyPhoto.changePage('next');return false;});};_centerOverlay();};function grab_param(name,url){name=name.replace(/[\[]/,"\\\[").replace(/[\]]/,"\\\]");var regexS="[\\?&]"+name+"=([^&#]*)";var regex=new RegExp(regexS);var results=regex.exec(url);if(results==null)
return"";else
return results[1];}})(jQuery);



```
