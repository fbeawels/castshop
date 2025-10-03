# jquery.easing-1.3.pack.js

## Review

## 1. Summary  
- **Purpose**: This file augments jQuery’s animation engine by providing a rich set of easing functions (e.g., `easeInOutQuart`, `easeOutBounce`, `easeInElastic`, etc.).  
- **Key components**:  
  - A single `eval` call that expands a heavily compressed source into readable function names and definitions.  
  - The plugin registers each easing function under `jQuery.easing` via `jQuery.extend`.  
  - The easing functions follow the classic *t, b, c, d* signature:  
    - `t` – current time (or frame)  
    - `b` – beginning value  
    - `c` – change in value (end‑beginning)  
    - `d` – total duration  
- **Design patterns**:  
  - *Module pattern* – the easing functions are attached to the global `jQuery` namespace, keeping a clean separation from the core jQuery code.  
  - *Functional composition* – each easing is a pure function; no side‑effects, making them reusable and easy to test.

## 2. Detailed Description  
1. **Initialization**  
   - When the script is loaded, the `eval` string executes.  
   - It decodes the compressed representation into a readable JavaScript object that contains many easing functions.  
   - The resulting object is merged into `jQuery.easing` via `jQuery.extend`.

2. **Runtime behavior**  
   - jQuery’s animation engine calls the appropriate easing function based on the user’s `easing` option.  
   - Each function receives the time/duration parameters and returns the interpolated value.  
   - No state is stored between calls; the calculation is deterministic.

3. **Cleanup**  
   - The plugin does not expose any global variables other than `jQuery.easing`.  
   - The only side effect is the augmentation of `jQuery.easing`; no timers or event listeners are created.

4. **Assumptions / Dependencies**  
   - Relies on **jQuery ≥ 1.1** (where `jQuery.extend` and the easing hook were added).  
   - Uses `Math` functions (`sin`, `cos`, `pow`, `abs`, `sqrt`) from the global `Math` object.  
   - The plugin expects that the consumer passes integer durations and time values in milliseconds.

5. **Architecture / Design choices**  
   - The entire implementation is compressed for distribution, which keeps the file size minimal but reduces readability.  
   - Each easing is written as a stateless pure function, enabling straightforward unit testing.  
   - By centralizing all easings in a single namespace, jQuery developers can add new easings or override existing ones with minimal effort.

## 3. Functions/Methods  
| Function | Purpose | Parameters | Returns | Side‑effects |
|---|---|---|---|---|
| `easeInQuad` | Quadratic easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutQuad` | Quadratic easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutQuad` | Quadratic easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInCubic` | Cubic easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutCubic` | Cubic easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutCubic` | Cubic easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInQuart` | Quartic easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutQuart` | Quartic easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutQuart` | Quartic easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInQuint` | Quintic easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutQuint` | Quintic easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutQuint` | Quintic easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInSine` | Sine easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutSine` | Sine easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutSine` | Sine easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInExpo` | Exponential easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutExpo` | Exponential easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutExpo` | Exponential easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInCirc` | Circular easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutCirc` | Circular easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutCirc` | Circular easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInElastic` | Elastic easing in | `t, b, c, d, s` | Interpolated value | None |
| `easeOutElastic` | Elastic easing out | `t, b, c, d, s` | Interpolated value | None |
| `easeInOutElastic` | Elastic easing in/out | `t, b, c, d, s` | Interpolated value | None |
| `easeInBack` | Back easing in | `t, b, c, d, s` | Interpolated value | None |
| `easeOutBack` | Back easing out | `t, b, c, d, s` | Interpolated value | None |
| `easeInOutBack` | Back easing in/out | `t, b, c, d, s` | Interpolated value | None |
| `easeInBounce` | Bounce easing in | `t, b, c, d` | Interpolated value | None |
| `easeOutBounce` | Bounce easing out | `t, b, c, d` | Interpolated value | None |
| `easeInOutBounce` | Bounce easing in/out | `t, b, c, d` | Interpolated value | None |
| `easeInOutCirc` (duplicate?) | ... | ... | ... | ... |

*All functions are pure – they never modify the DOM or capture state.*

## 4. Dependencies  
- **jQuery** (≥ 1.1): `jQuery.extend`, `jQuery.easing` namespace.  
- **Math** (standard JS global): `sin`, `cos`, `pow`, `abs`, `sqrt`, `PI`.  
- No other third‑party libraries.  
- The code is platform‑agnostic (runs in any JavaScript environment that provides the above).

## 5. Additional Notes  
### Strengths  
- **Comprehensive**: Provides a wide range of easing curves covering most animation use‑cases.  
- **Lightweight**: The minified source is tiny (≈3 KB), yet the runtime functions are uncompressed for readability.  
- **Extensible**: Users can override or add new easings simply by extending `jQuery.easing`.  

### Potential Issues / Edge Cases  
1. **Eval Usage**  
   - The plugin relies on `eval` to decompress the source. While this keeps the distribution small, it can trigger security concerns in strict CSP environments.  
2. **Compatibility**  
   - The code is written for early jQuery versions. In modern environments (ES6+), a more explicit module or UMD wrapper would be preferable.  
3. **Performance**  
   - Each easing function performs a handful of Math operations. For very high‑frequency animations (e.g., canvas or WebGL), a pre‑computed lookup table might be faster.  
4. **Readability**  
   - The minified `eval` payload makes debugging difficult. Developers are advised to use the uncompressed source when debugging or contributing.  

### Future Enhancements  
- **ES6 Module**: Provide a named export (`export const easing = {...}`) for modern build tools.  
- **TypeScript Definitions**: Add `.d.ts` files to improve IDE support.  
- **Testing Suite**: Unit tests for each easing curve against known reference values.  
- **Performance Profiling**: Benchmark against other easing libraries (e.g., GSAP, anime.js) to identify bottlenecks.  
- **Feature Flags**: Allow optional removal of rarely used easings to reduce bundle size for production.  

Overall, this plugin delivers a robust set of easing functions with minimal overhead, though modern JavaScript practices would favor a more explicit module system and removal of `eval`.

## Code Critique



## Code Preview

```javascript
/*
 * jQuery Easing v1.3 - http://gsgd.co.uk/sandbox/jquery/easing/
 *
 * Uses the built in easing capabilities added In jQuery 1.1
 * to offer multiple easing options
 *
 * TERMS OF USE - jQuery Easing
 * 
 * Open source under the BSD License. 
 * 
 * Copyright © 2008 George McGinley Smith
 * All rights reserved.
 * 
 * Redistribution and use in source and binary forms, with or without modification, 
 * are permitted provided that the following conditions are met:
 * 
 * Redistributions of source code must retain the above copyright notice, this list of 
 * conditions and the following disclaimer.
 * Redistributions in binary form must reproduce the above copyright notice, this list 
 * of conditions and the following disclaimer in the documentation and/or other materials 
 * provided with the distribution.
 * 
 * Neither the name of the author nor the names of contributors may be used to endorse 
 * or promote products derived from this software without specific prior written permission.
 * 
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY 
 * EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
 *  COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
 *  EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE
 *  GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED 
 * AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 *  NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED 
 * OF THE POSSIBILITY OF SUCH DAMAGE. 
 *
*/

// t: current time, b: begInnIng value, c: change In value, d: duration
eval(function(p,a,c,k,e,r){e=function(c){return(c<a?'':e(parseInt(c/a)))+((c=c%a)>35?String.fromCharCode(c+29):c.toString(36))};if(!''.replace(/^/,String)){while(c--)r[e(c)]=k[c]||e(c);k=[function(e){return r[e]}];e=function(){return'\\w+'};c=1};while(c--)if(k[c])p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c]);return p}('h.i[\'1a\']=h.i[\'z\'];h.O(h.i,{y:\'D\',z:9(x,t,b,c,d){6 h.i[h.i.y](x,t,b,c,d)},17:9(x,t,b,c,d){6 c*(t/=d)*t+b},D:9(x,t,b,c,d){6-c*(t/=d)*(t-2)+b},13:9(x,t,b,c,d){e((t/=d/2)<1)6 c/2*t*t+b;6-c/2*((--t)*(t-2)-1)+b},X:9(x,t,b,c,d){6 c*(t/=d)*t*t+b},U:9(x,t,b,c,d){6 c*((t=t/d-1)*t*t+1)+b},R:9(x,t,b,c,d){e((t/=d/2)<1)6 c/2*t*t*t+b;6 c/2*((t-=2)*t*t+2)+b},N:9(x,t,b,c,d){6 c*(t/=d)*t*t*t+b},M:9(x,t,b,c,d){6-c*((t=t/d-1)*t*t*t-1)+b},L:9(x,t,b,c,d){e((t/=d/2)<1)6 c/2*t*t*t*t+b;6-c/2*((t-=2)*t*t*t-2)+b},K:9(x,t,b,c,d){6 c*(t/=d)*t*t*t*t+b},J:9(x,t,b,c,d){6 c*((t=t/d-1)*t*t*t*t+1)+b},I:9(x,t,b,c,d){e((t/=d/2)<1)6 c/2*t*t*t*t*t+b;6 c/2*((t-=2)*t*t*t*t+2)+b},G:9(x,t,b,c,d){6-c*8.C(t/d*(8.g/2))+c+b},15:9(x,t,b,c,d){6 c*8.n(t/d*(8.g/2))+b},12:9(x,t,b,c,d){6-c/2*(8.C(8.g*t/d)-1)+b},Z:9(x,t,b,c,d){6(t==0)?b:c*8.j(2,10*(t/d-1))+b},Y:9(x,t,b,c,d){6(t==d)?b+c:c*(-8.j(2,-10*t/d)+1)+b},W:9(x,t,b,c,d){e(t==0)6 b;e(t==d)6 b+c;e((t/=d/2)<1)6 c/2*8.j(2,10*(t-1))+b;6 c/2*(-8.j(2,-10*--t)+2)+b},V:9(x,t,b,c,d){6-c*(8.o(1-(t/=d)*t)-1)+b},S:9(x,t,b,c,d){6 c*8.o(1-(t=t/d-1)*t)+b},Q:9(x,t,b,c,d){e((t/=d/2)<1)6-c/2*(8.o(1-t*t)-1)+b;6 c/2*(8.o(1-(t-=2)*t)+1)+b},P:9(x,t,b,c,d){f s=1.l;f p=0;f a=c;e(t==0)6 b;e((t/=d)==1)6 b+c;e(!p)p=d*.3;e(a<8.w(c)){a=c;f s=p/4}m f s=p/(2*8.g)*8.r(c/a);6-(a*8.j(2,10*(t-=1))*8.n((t*d-s)*(2*8.g)/p))+b},H:9(x,t,b,c,d){f s=1.l;f p=0;f a=c;e(t==0)6 b;e((t/=d)==1)6 b+c;e(!p)p=d*.3;e(a<8.w(c)){a=c;f s=p/4}m f s=p/(2*8.g)*8.r(c/a);6 a*8.j(2,-10*t)*8.n((t*d-s)*(2*8.g)/p)+c+b},T:9(x,t,b,c,d){f s=1.l;f p=0;f a=c;e(t==0)6 b;e((t/=d/2)==2)6 b+c;e(!p)p=d*(.3*1.5);e(a<8.w(c)){a=c;f s=p/4}m f s=p/(2*8.g)*8.r(c/a);e(t<1)6-.5*(a*8.j(2,10*(t-=1))*8.n((t*d-s)*(2*8.g)/p))+b;6 a*8.j(2,-10*(t-=1))*8.n((t*d-s)*(2*8.g)/p)*.5+c+b},F:9(x,t,b,c,d,s){e(s==u)s=1.l;6 c*(t/=d)*t*((s+1)*t-s)+b},E:9(x,t,b,c,d,s){e(s==u)s=1.l;6 c*((t=t/d-1)*t*((s+1)*t+s)+1)+b},16:9(x,t,b,c,d,s){e(s==u)s=1.l;e((t/=d/2)<1)6 c/2*(t*t*(((s*=(1.B))+1)*t-s))+b;6 c/2*((t-=2)*t*(((s*=(1.B))+1)*t+s)+2)+b},A:9(x,t,b,c,d){6 c-h.i.v(x,d-t,0,c,d)+b},v:9(x,t,b,c,d){e((t/=d)<(1/2.k)){6 c*(7.q*t*t)+b}m e(t<(2/2.k)){6 c*(7.q*(t-=(1.5/2.k))*t+.k)+b}m e(t<(2.5/2.k)){6 c*(7.q*(t-=(2.14/2.k))*t+.11)+b}m{6 c*(7.q*(t-=(2.18/2.k))*t+.19)+b}},1b:9(x,t,b,c,d){e(t<d/2)6 h.i.A(x,t*2,0,c,d)*.5+b;6 h.i.v(x,t*2-d,0,c,d)*.5+c*.5+b}});',62,74,'||||||return||Math|function|||||if|var|PI|jQuery|easing|pow|75|70158|else|sin|sqrt||5625|asin|||undefined|easeOutBounce|abs||def|swing|easeInBounce|525|cos|easeOutQuad|easeOutBack|easeInBack|easeInSine|easeOutElastic|easeInOutQuint|easeOutQuint|easeInQuint|easeInOutQuart|easeOutQuart|easeInQuart|extend|easeInElastic|easeInOutCirc|easeInOutCubic|easeOutCirc|easeInOutElastic|easeOutCubic|easeInCirc|easeInOutExpo|easeInCubic|easeOutExpo|easeInExpo||9375|easeInOutSine|easeInOutQuad|25|easeOutSine|easeInOutBack|easeInQuad|625|984375|jswing|easeInOutBounce'.split('|'),0,{}))

/*
 *
 * TERMS OF USE - EASING EQUATIONS
 * 
 * Open source under the BSD License. 
 * 
 * Copyright © 2001 Robert Penner
 * All rights reserved.
 * 
 * Redistribution and use in source and binary forms, with or without modification, 
 * are permitted provided that the following conditions are met:
 * 
 * Redistributions of source code must retain the above copyright notice, this list of 
 * conditions and the following disclaimer.
 * Redistributions in binary form must reproduce the above copyright notice, this list 
 * of conditions and the following disclaimer in the documentation and/or other materials 
 * provided with the distribution.
 * 
 * Neither the name of the author nor the names of contributors may be used to endorse 
 * or promote products derived from this software without specific prior written permission.
 * 
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY 
 * EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
 *  COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
 *  EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE
 *  GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED 
 * AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 *  NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED 
 * OF THE POSSIBILITY OF SUCH DAMAGE. 
 *
 */



```
