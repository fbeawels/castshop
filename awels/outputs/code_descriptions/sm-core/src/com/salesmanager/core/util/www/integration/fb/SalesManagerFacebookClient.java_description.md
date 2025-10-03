# SalesManagerFacebookClient.java

## Review

## 1. Summary
The `SalesManagerFacebookClient` class is a thin wrapper around **RestFB’s** `DefaultFacebookClient`.  
It mainly customises the URL‑parameter handling logic by overriding `toParameterString`.  
The goal is to ensure that every request sent to Facebook is correctly encoded, while still
adding the standard `access_token` and `format=json` parameters automatically.

**Key components**

| Class | Purpose |
|-------|---------|
| `SalesManagerFacebookClient` | Sub‑class of RestFB’s `DefaultFacebookClient` that re‑implements parameter encoding |
| `urlEncode(String)` | Utility helper that performs UTF‑8 URL‑encoding |
| `toParameterString(Parameter...)` | Builds the query‑string from RestFB `Parameter` objects, encoding names/values |

The code relies on RestFB (v1‑v3 at the time of writing) and Apache Commons‑Lang (`StringUtils`).

---

## 2. Detailed Description
1. **Construction**  
   The only public constructor accepts an `accessToken` string and delegates to the parent
   constructor with default `DefaultWebRequestor` and `DefaultJsonMapper`.  
   This gives the client the same network/JSON behaviour as RestFB’s default client.

2. **Parameter handling**  
   The overridden `toParameterString` method is invoked by RestFB when building a request URL.
   * It first appends the OAuth `access_token` (if present) and `format=json` to the supplied
     `Parameter...` array.  
   * It then iterates over every `Parameter` instance, performing two key tasks:
     1. URL‑encode the parameter **name** (twice – once unconditionally and once inside the
        `if (!parameter.name.equals(ACCESS_TOKEN_PARAM_NAME))` block; the double‑encoding is
        redundant but harmless).
     2. URL‑encode the parameter **value** only when it isn’t the access‑token, by calling
        the inherited `urlEncodedValueForParameterName` helper.
   * The resulting name/value pairs are concatenated with `&` into a single query string.

3. **Utility**  
   `urlEncode` simply wraps `URLEncoder.encode` with a hard‑coded UTF‑8 charset and turns
   `UnsupportedEncodingException` into an `IllegalStateException`.

4. **Assumptions & constraints**  
   * `accessToken` is inherited from `DefaultFacebookClient`; it is expected to be non‑null
     for authenticated requests.  
   * All `Parameter` objects are expected to expose public fields `name` and `value`.  
     In modern RestFB versions these are private, so the code will not compile unless the
     library has package‑private access or the wrapper is placed in the same package.  
   * The method presumes that `Parameter... parameters` is non‑null; passing `null` will
     throw a `NullPointerException` at the loop start.

5. **Architecture**  
   The design keeps the bulk of RestFB intact and only tweaks the low‑level URL‑encoding logic.
   This keeps the client behaviour largely identical to the default client, while allowing
   the developer to enforce consistent encoding for all requests.

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `SalesManagerFacebookClient(String accessToken)` | Creates a new client with the supplied OAuth token. | `String accessToken` | new instance of this class | Instantiates parent with default web requestor & JSON mapper |
| `toParameterString(Parameter... parameters)` | Builds a URL‑encoded query string for a Facebook request. | `Parameter... parameters` | `String` | Modifies the passed `Parameter` array by prepending the token and format params. |
| `static String urlEncode(String string)` | URL‑encodes a string using UTF‑8. | `String string` | encoded string or `null` | Throws unchecked `IllegalStateException` if UTF‑8 unsupported |

### Reusable / Utility Methods
* `urlEncode` is a small helper that could be extracted to a common util class.
* `toParameterString` could be refactored to avoid duplicated encoding logic and to guard
  against `null` arguments.

---

## 4. Dependencies
| Library | Version (approx.) | Nature |
|---------|-------------------|--------|
| **RestFB** (`com.restfb`) | 1‑3.x (any version exposing `DefaultFacebookClient` & `Parameter`) | Third‑party, Facebook Graph API wrapper |
| **Apache Commons‑Lang** (`org.apache.commons.lang.StringUtils`) | 2.x/3.x | Third‑party utility |
| **Java Standard Library** | – | `java.net.URLEncoder`, `UnsupportedEncodingException` |

No platform‑specific dependencies; the code should compile on any JVM 8+.

---

## 5. Additional Notes & Recommendations

### Potential Issues
| Issue | Why it matters | Suggested Fix |
|-------|----------------|---------------|
| **Field access to `Parameter.name` & `Parameter.value`** | In current RestFB releases these fields are `private`; direct access will not compile unless the wrapper is in the same package. | Use the public getters (`getName()`, `getValue()`) or rely on `Parameter.getValue()` to obtain the raw value. |
| **Redundant double‑encoding of the parameter name** | Unnecessary overhead; may lead to double‑encoded values if the method changes. | Remove the first unconditional `urlEncode` and keep only the one inside the `if` block, or vice‑versa. |
| **No null‑check for `parameters` array** | Passing `null` will cause `NullPointerException`. | Validate `parameters != null` and throw a clear exception or default to an empty array. |
| **Hard‑coded charset string** | Slightly less type‑safe than `StandardCharsets.UTF_8`. | Replace with `StandardCharsets.UTF_8` (Java 7+) and drop the string constant. |
| **Missing documentation** | Future maintainers may not understand the reason for overriding the method. | Add Javadoc explaining the encoding requirement and any deviations from the parent implementation. |
| **No tests** | No automated validation of the encoding logic. | Write unit tests covering normal, edge‑case (empty, null, special characters) parameters. |

### Possible Enhancements
1. **Parameter filtering** – allow callers to opt‑out of automatic token/format injection.
2. **Error handling** – wrap or propagate encoding errors more gracefully.
3. **Logging** – add optional debug logs for the generated query string (masked for the token).
4. **Builder pattern** – expose a fluent builder for creating `SalesManagerFacebookClient` instances with additional configuration (e.g., custom requestor, mapper).
5. **Compatibility shim** – provide conditional logic that compiles against both older and newer RestFB APIs.

### Summary
The class successfully adapts RestFB’s request‑building logic to ensure consistent UTF‑8 encoding and token handling. However, it relies on fragile assumptions about the `Parameter` class’s visibility, contains redundant encoding steps, and lacks defensive programming practices. Addressing these points will make the wrapper robust, maintainable, and future‑proof.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Nov 19, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util.www.integration.fb;

import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;
import java.util.List;
import java.util.Map;

import org.apache.commons.lang.StringUtils;

import com.restfb.DefaultFacebookClient;
import com.restfb.DefaultJsonMapper;
import com.restfb.DefaultWebRequestor;
import com.restfb.FacebookException;
import com.restfb.FacebookJsonMappingException;
import com.restfb.Parameter;



/**
 * Overwrites restfb.DefaultFacebookClient
 * @author Carl Samson
 *
 */
public class SalesManagerFacebookClient extends DefaultFacebookClient {
	
	  static final String ENCODING_CHARSET = "UTF-8";
	  
	  public SalesManagerFacebookClient(String accessToken) {
		    super(accessToken, new DefaultWebRequestor(), new DefaultJsonMapper());
	  }
	
	  protected String toParameterString(Parameter... parameters)
      throws FacebookJsonMappingException {
		  
		  
		  if (!StringUtils.isBlank(accessToken))
		      parameters =
		          parametersWithAdditionalParameter(
		            Parameter.with(ACCESS_TOKEN_PARAM_NAME, accessToken), parameters);

		    parameters =
		        parametersWithAdditionalParameter(
		          Parameter.with(FORMAT_PARAM_NAME, "json"), parameters);

		    StringBuilder parameterStringBuilder = new StringBuilder();
		    boolean first = true;

		    for (Parameter parameter : parameters) {
		      if (first)
		        first = false;
		      else
		        parameterStringBuilder.append("&");
		      
	    	  String name = urlEncode(parameter.name);
	    	  String value = parameter.value;
		      if(!parameter.name.equals(ACCESS_TOKEN_PARAM_NAME)) {
		    	  name = urlEncode(parameter.name);
		    	 value = urlEncodedValueForParameterName(parameter.name, parameter.value);
		      } 

		      parameterStringBuilder.append(name);
		      parameterStringBuilder.append("=");
		      parameterStringBuilder.append(value);
		    }

		    return parameterStringBuilder.toString();
	  }
	  
	  
	  
	  static String urlEncode(String string) {
		    if (string == null)
		      return null;
		    try {
		      return URLEncoder.encode(string, ENCODING_CHARSET);
		    } catch (UnsupportedEncodingException e) {
		      throw new IllegalStateException("Platform doesn't support "
		          + ENCODING_CHARSET, e);
		    }
	  }



}



```
