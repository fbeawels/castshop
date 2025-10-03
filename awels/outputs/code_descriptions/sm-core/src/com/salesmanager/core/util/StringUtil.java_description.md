# StringUtil.java

## Review

## 1. Summary
`StringUtil` is a small utility class that provides a handful of string‑related helpers:

| Feature | What it does |
|---------|--------------|
| `buildMultipleValueLine` | Joins a list of strings into a semicolon‑separated line |
| `parseTokenLine` | Parses a delimited string into a bidirectional map (value ↔ index) |
| `deformatUrlResponse` | Turns a URL‑encoded key/value query string into a `Map` (keys upper‑cased) |
| `unescape` | A manual implementation of URL‑decoding that also decodes UTF‑8 bytes |

The class relies on a few Apache‑Commons utilities (`StringUtils`, `TreeBidiMap`) and Java’s standard `URLDecoder`. No external framework is required, but the code uses several deprecated / raw‑type APIs and could benefit from modern Java idioms.

---

## 2. Detailed Description
### Overall flow
The class is entirely static – no state is kept. Each method accepts input parameters, performs a straightforward transformation, and returns the result or `null`.  
All methods assume the caller will provide non‑`null` values where indicated; the only guard against `null` is in `buildMultipleValueLine` and `parseTokenLine`.

### Core components
| Method | Key operations |
|--------|----------------|
| `buildMultipleValueLine` | Iterates over a `List`, concatenates each element with `;` as a separator, returns the final string or `null` if the list is empty/`null`. |
| `parseTokenLine` | Tokenises the input string by a delimiter, inserts each token as a key into a `TreeBidiMap` and maps it to the token’s index. |
| `deformatUrlResponse` | Splits a payload by `&`, then each token by `=`, URL‑decodes both parts using UTF‑8, puts the upper‑cased key and decoded value into a `HashMap`. |
| `unescape` | Implements percent‑encoding and `+`‑to‑space conversion and then manually reconstructs UTF‑8 characters byte by byte. |

### Design notes
* **Static utility class** – No instances are created.  
* **Raw types** – All collections are raw (`List`, `Map`) which disables compile‑time type safety.  
* **Deprecated APIs** – `StringTokenizer` is old‑fashioned; `StringUtils.isBlank` is fine but could be replaced with `String.isBlank()` in Java 11+.  
* **Manual UTF‑8 decoding** – The `unescape` method duplicates what `URLDecoder` already offers and is fragile (e.g., missing bounds checks).  

---

## 3. Functions/Methods
| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `buildMultipleValueLine` | `static String buildMultipleValueLine(List list)` | Concatenates list elements with `;` | `list` – any `List` of `String`s | The concatenated string, or `null` if `list` is `null` or empty | None |
| `parseTokenLine` | `static Map parseTokenLine(String line, String delimiter)` | Parses a delimited string into a bidirectional map (`value` ↔ `index`) | `line` – input string, `delimiter` – separator | `TreeBidiMap` containing tokens as keys and their position as values | None |
| `deformatUrlResponse` | `static Map deformatUrlResponse(String payload) throws Exception` | Converts a URL‑encoded query string into a map (keys upper‑cased) | `payload` – `%`/`=`‑encoded string | `HashMap` of decoded key/value pairs | None |
| `unescape` | `static String unescape(String s)` | Decodes percent‑encoded strings and reconstructs UTF‑8 characters manually | `s` – encoded string | Decoded string | None |

### Reusable / Utility methods
None beyond the four public helpers; all are single‑purpose.

---

## 4. Dependencies
| Dependency | Scope | Notes |
|------------|-------|-------|
| `java.net.URLDecoder` | Standard | Used in `deformatUrlResponse` |
| `org.apache.commons.collections.BidiMap`, `TreeBidiMap` | Third‑party (Apache Commons Collections) | Provides a bidirectional map for `parseTokenLine` |
| `org.apache.commons.lang.StringUtils` | Third‑party (Apache Commons Lang) | Used for `isBlank` checks |
| `java.util.*` | Standard | Collections, StringTokenizer, etc. |

All dependencies are straightforward; only the Apache‑Commons libraries are third‑party and are optional to replace with Java’s own collections.

---

## 5. Additional Notes & Recommendations

### 5.1 Coding style & safety
1. **Raw types** – Replace `List`, `Map`, `HashMap`, `TreeBidiMap` with generics:  
   ```java
   public static String buildMultipleValueLine(List<String> list) { … }
   public static Map<String, Integer> parseTokenLine(String line, String delimiter) { … }
   public static Map<String, String> deformatUrlResponse(String payload) { … }
   ```
   This eliminates unchecked‑conversion warnings and protects callers from `ClassCastException`.

2. **Short‑circuit logic** – `list != null & list.size() > 0` uses bitwise `&`. It should be `&&`. The difference is negligible here, but it’s semantically wrong and can lead to unintended evaluation of the second operand.

3. **String concatenation** – Use `StringBuilder` (or `String.join` for Java 8+) instead of manual `StringBuffer`. `StringBuffer` is synchronized, which is unnecessary in single‑threaded utility methods.

4. **Tokenisation** – Replace `StringTokenizer` with `String.split` or `java.util.regex.Pattern`. `StringTokenizer` is legacy and less flexible.

5. **Upper‑casing keys** – The `deformatUrlResponse` method upper‑cases all keys (`key.toUpperCase()`). Consider whether case‑insensitive keys are really required; otherwise preserve the original case.

### 5.2 Error handling
* `deformatUrlResponse` declares `throws Exception` but never throws a checked exception. The only potential source is `UnsupportedEncodingException` from `URLDecoder`, which is a subclass of `IOException` and can be caught or declared as `UnsupportedEncodingException`. Declaring a generic `Exception` is unnecessary and obfuscates the API contract.

### 5.3 `unescape` implementation
* The method re‑implements `URLDecoder.decode()` (and adds UTF‑8 decoding). This is risky:
  * It lacks bounds checking for `%` sequences (e.g., `%` at the end of the string).
  * It fails silently on malformed UTF‑8.
  * It uses `StringBuffer` (synchronized) and `char`‑to‑byte conversions that can misbehave for non‑ASCII characters.
* Recommendation: Remove `unescape` and replace callers with `URLDecoder.decode(String, "UTF-8")`. If custom behaviour is needed (e.g., preserving `+` as literal), document it and use a well‑tested library.

### 5.4 Edge cases
* **Empty strings** – `buildMultipleValueLine(null)` returns `null`; `buildMultipleValueLine(Collections.emptyList())` also returns `null`. This inconsistency might surprise callers; consider returning an empty string for an empty list.
* **Delimiter conflicts** – `parseTokenLine` does not trim tokens; if the delimiter appears in the token itself (e.g., `value1;value;2` with delimiter `;`) the token will be split incorrectly.
* **Case‑sensitivity** – The `deformatUrlResponse` method normalises keys to uppercase, which may hide duplicates that differ only in case.

### 5.5 Future enhancements
1. **Introduce generics** for type safety.  
2. **Add Javadoc** to each method, describing the contract and edge‑case behaviour.  
3. **Provide overloaded variants** that accept delimiters as `Pattern` or `String` for flexibility.  
4. **Unit tests** – Add comprehensive test coverage for all methods, including edge cases (null, empty strings, malformed input).  
5. **Logging** – Optionally log decoding errors for debugging.  
6. **Migration** – Replace legacy APIs (`StringTokenizer`, `StringBuffer`) with modern equivalents.  
7. **Immutable collections** – Return `Collections.unmodifiableMap(...)` to prevent accidental modification of the result.

---

### 5.6 Quick refactor example

```java
public static String buildMultipleValueLine(List<String> list) {
    if (list == null || list.isEmpty()) {
        return null;   // or "" if preferred
    }
    return String.join(";", list);
}

public static Map<String, Integer> parseTokenLine(String line, String delimiter) {
    Map<String, Integer> result = new LinkedHashMap<>();
    if (StringUtils.isBlank(line) || StringUtils.isBlank(delimiter)) {
        return result;
    }
    String[] tokens = line.split(Pattern.quote(delimiter));
    for (int i = 0; i < tokens.length; i++) {
        result.put(tokens[i], i);
    }
    return result;
}

public static Map<String, String> deformatUrlResponse(String payload) throws UnsupportedEncodingException {
    Map<String, String> map = new HashMap<>();
    for (String pair : payload.split("&")) {
        int idx = pair.indexOf('=');
        if (idx == -1) continue; // malformed pair
        String key   = URLDecoder.decode(pair.substring(0, idx), StandardCharsets.UTF_8);
        String value = URLDecoder.decode(pair.substring(idx + 1), StandardCharsets.UTF_8);
        map.put(key.toUpperCase(Locale.ROOT), value);
    }
    return map;
}
```

These changes modernise the code, improve safety, and make the behaviour clearer.

---

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 24, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util;

import java.net.URLDecoder;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.commons.collections.BidiMap;
import org.apache.commons.collections.bidimap.TreeBidiMap;
import org.apache.commons.lang.StringUtils;

public class StringUtil {

	/**
	 * Build a ; delimited line with the values contained in the list
	 * 
	 * @param list
	 * @return
	 */
	public static String buildMultipleValueLine(List list) {

		if (list != null & list.size() > 0) {

			Iterator i = list.iterator();
			StringBuffer linebuffer = new StringBuffer();
			int icount = 0;
			while (i.hasNext()) {
				String value = (String) i.next();
				linebuffer.append(value);
				if (icount < list.size() - 1)
					linebuffer.append(";");
				icount++;
			}
			return linebuffer.toString();
		} else {
			return null;
		}
	}

	public static Map parseTokenLine(String line, String delimiter) {

		BidiMap returnMap = new TreeBidiMap();

		if (StringUtils.isBlank(line) || StringUtils.isBlank(delimiter)) {
			return returnMap;
		}

		StringTokenizer st = new StringTokenizer(line, delimiter);

		int count = 0;
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
			returnMap.put(value, count);
			count++;
		}

		return returnMap;

	}
	
	
	public static Map deformatUrlResponse(String payload) throws Exception {
		HashMap nvp = new HashMap();
		StringTokenizer stTok = new StringTokenizer(payload, "&");
		while (stTok.hasMoreTokens()) {
			StringTokenizer stInternalTokenizer = new StringTokenizer(stTok
					.nextToken(), "=");
			if (stInternalTokenizer.countTokens() == 2) {
				String key = URLDecoder.decode(stInternalTokenizer.nextToken(),
						"UTF-8");
				String value = URLDecoder.decode(stInternalTokenizer
						.nextToken(), "UTF-8");
				nvp.put(key.toUpperCase(), value);
			}
		}
		return nvp;
	}
	
	/**
	 * Can be used to decode URL 
	 * @param s
	 * @return
	 */
	public static String unescape(String s) {
		    StringBuffer sbuf = new StringBuffer () ;
		    int l  = s.length() ;
		    int ch = -1 ;
		    int b, sumb = 0;
		    for (int i = 0, more = -1 ; i < l ; i++) {
		      /* Get next byte b from URL segment s */
		      switch (ch = s.charAt(i)) {
			case '%':
			  ch = s.charAt (++i) ;
			  int hb = (Character.isDigit ((char) ch) 
				    ? ch - '0'
				    : 10+Character.toLowerCase((char) ch) - 'a') & 0xF ;
			  ch = s.charAt (++i) ;
			  int lb = (Character.isDigit ((char) ch)
				    ? ch - '0'
				    : 10+Character.toLowerCase ((char) ch)-'a') & 0xF ;
			  b = (hb << 4) | lb ;
			  break ;
			case '+':
			  b = ' ' ;
			  break ;
			default:
			  b = ch ;
		      }
		      /* Decode byte b as UTF-8, sumb collects incomplete chars */
		      if ((b & 0xc0) == 0x80) {			// 10xxxxxx (continuation byte)
			sumb = (sumb << 6) | (b & 0x3f) ;	// Add 6 bits to sumb
			if (--more == 0) sbuf.append((char) sumb) ; // Add char to sbuf
		      } else if ((b & 0x80) == 0x00) {		// 0xxxxxxx (yields 7 bits)
			sbuf.append((char) b) ;			// Store in sbuf
		      } else if ((b & 0xe0) == 0xc0) {		// 110xxxxx (yields 5 bits)
			sumb = b & 0x1f;
			more = 1;				// Expect 1 more byte
		      } else if ((b & 0xf0) == 0xe0) {		// 1110xxxx (yields 4 bits)
			sumb = b & 0x0f;
			more = 2;				// Expect 2 more bytes
		      } else if ((b & 0xf8) == 0xf0) {		// 11110xxx (yields 3 bits)
			sumb = b & 0x07;
			more = 3;				// Expect 3 more bytes
		      } else if ((b & 0xfc) == 0xf8) {		// 111110xx (yields 2 bits)
			sumb = b & 0x03;
			more = 4;				// Expect 4 more bytes
		      } else /*if ((b & 0xfe) == 0xfc)*/ {	// 1111110x (yields 1 bit)
			sumb = b & 0x01;
			more = 5;				// Expect 5 more bytes
		      }
		      /* We don't test if the UTF-8 encoding is well-formed */
		    }
		    return sbuf.toString() ;
	}


}



```
