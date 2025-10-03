# MessageUtil.java

## Review

## 1. Summary
`MessageUtil` is a small utility helper that centralises the creation, retrieval, formatting and display of user-facing messages (normal, error, notice, and form‑specific error messages) in a web application.  
It operates on a `HttpServletRequest` (and its `HttpSession`) to store messages in request or session attributes and then renders them into simple HTML snippets.  

Key components  
- **Session‑based message storage**: `SM-MESSAGE`, `SM-ERR-MESSAGE`, `SM-ERR-MESSAGES`, `SM-NOTICEMESSAGE`.  
- **Form‑error handling**: per‑field error maps (`errfrmmsg`) stored as request attributes.  
- **Display helpers**: `displayMessages()`, `displayFormErrorMessage()` and `displayFormErrorMessageNoFormating()`.  

Design pattern: a thin façade around HTTP attributes – essentially a *message context* pattern.  
The only external library is `javax.servlet.http.HttpServletRequest`, and it relies on a `LabelUtil` singleton for i18n of error strings.

---

## 2. Detailed Description
1. **Construction**  
   - The class has a private constructor and static methods only, so it is never instantiated.  
   - The private `messages` list is never used (bug or leftover).

2. **Adding messages**  
   - `addNoticeMessage`, `addMessage`, `addErrorMessage`, `addErrorMessages` all put a value into the current session.  
   - `addFormErrorMessage` creates/updates a `Map<String, String>` on the request for a particular form field.

3. **Retrieving messages**  
   - Simple getters (`getMessage`, `getErrorMessage`, `getFormErrorMessage`) pull from session/request attributes.  
   - `getFormErrorMessage` removes the entry after retrieval (single‑use semantics).

4. **Display logic**  
   - `displayMessages` builds an HTML `<div>` per message type.  
   - `displayFormErrorMessage` renders a table row with an error icon, using `LabelUtil.getText()` to resolve a localisation key.  
   - `displayFormErrorMessageNoFormating` just returns the localised string.

5. **Cleanup**  
   - After rendering, `displayMessages` removes all message attributes from the session.  
   - `resetMessages` can be called manually to clear state.  
   - `hasMessage` checks whether any message is present.

6. **Assumptions & Constraints**  
   - The caller must be inside a servlet/JSP environment with a valid `HttpServletRequest`.  
   - The application guarantees a single thread per session; concurrency is not considered.  
   - It assumes `LabelUtil.getInstance()` is thread‑safe and available.

7. **Architecture & Design Choices**  
   - The util is purely static, making it easy to use but hard to unit‑test without mocks.  
   - Storing error messages as session attributes couples the life‑cycle to the HTTP session, which may cause messages to persist across redirects if not cleared.  
   - Using raw `List` and `Map` without generics (pre‑Java 5 style) reduces type safety.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `addNoticeMessage(HttpServletRequest req, String message)` | Stores a notice message in session. | `req`, `message` | `void` | `req.getSession().setAttribute("SM-NOTICEMESSAGE", message)` |
| `addMessage(HttpServletRequest req, String message)` | Stores a normal message. | `req`, `message` | `void` | `req.getSession().setAttribute("SM-MESSAGE", message)` |
| `addErrorMessage(HttpServletRequest req, String message)` | Stores a single error message. | `req`, `message` | `void` | `req.getSession().setAttribute("SM-ERR-MESSAGE", message)` |
| `addErrorMessages(HttpServletRequest req, List<String> messages)` | Stores a list of error messages. | `req`, `messages` | `void` | `req.getSession().setAttribute("SM-ERR-MESSAGES", messages)` |
| `addFormErrorMessage(HttpServletRequest req, String field, String message)` | Associates a field‑specific error message in request scope. | `req`, `field`, `message` | `void` | `Map` stored under `"errfrmmsg"` |
| `getMessage(HttpServletRequest req)` | Retrieves normal message. | `req` | `String` | none |
| `getErrorMessage(HttpServletRequest req)` | Retrieves single error message. | `req` | `String` | none |
| `getFormErrorMessage(HttpServletRequest req, String field)` | Retrieves and removes the error message for a field. | `req`, `field` | `String` or `null` | removes entry from map |
| `displayMessages(HttpServletRequest req)` | Renders all session messages as HTML and clears them. | `req` | `String` (HTML) | removes message attributes from session |
| `displayFormErrorMessage(HttpServletRequest req, String field)` | Renders a formatted error row for a form field. | `req`, `field` | `String` (HTML) | removes error from map |
| `displayFormErrorMessageNoFormating(HttpServletRequest req, String field)` | Returns the raw localized message. | `req`, `field` | `String` | none |
| `resetMessages(HttpServletRequest req)` | Clears all message attributes from session. | `req` | `void` | removes session attributes |
| `hasMessage(HttpServletRequest req)` | Checks if any message exists. | `req` | `boolean` | none |

Reusable / utility methods  
- All methods are stateless, so they can be invoked anywhere with a `HttpServletRequest`.  
- The class could expose a helper object (e.g., `MessageContext`) instead of static methods for better testability.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard | Core servlet API. |
| `javax.servlet.http.HttpSession` | Standard | Obtained via request. |
| `java.util.*` | Standard | Lists, Maps, etc. |
| `com.salesmanager.core.util.LabelUtil` | Third‑party (project internal) | Provides internationalised text; assumed thread‑safe. |
| `java.io.Serializable` | Standard | Implemented but unused (no state). |

No external frameworks (e.g., Spring) are referenced, but the design presumes a typical Java EE servlet environment.

---

## 5. Additional Notes & Recommendations

### Strengths
- **Simplicity**: Clear, self‑contained methods for common messaging tasks.  
- **Internationalisation**: Delegates text lookup to `LabelUtil`, enabling locale‑aware messages.  
- **Single‑use semantics**: Removing form errors after reading prevents accidental duplication.

### Weaknesses & Edge Cases
1. **Unused fields** – `messages` and `errormessages` are declared but never used.  
2. **Type safety** – raw collections (`List`, `Map`) should use generics.  
3. **Thread safety** – Session attributes are not synchronized; concurrent requests from the same user could corrupt data.  
4. **HTML hard‑coding** – The class concatenates HTML strings directly, making it brittle and hard to maintain. Using a templating engine or JSP tag would be cleaner.  
5. **Redirect scenarios** – If a redirect occurs before `displayMessages` is called, messages may linger in the session until manually cleared.  
6. **No removal after `displayFormErrorMessageNoFormating`** – This method returns the string but does not remove the message from the map, potentially causing duplicate display.  
7. **Exception handling** – No checks for `NullPointerException` if attributes are missing or of unexpected type.

### Suggested Improvements
- **Introduce generics**: `private List<String> messages;` etc.  
- **Replace raw HTML with JSP/Thymeleaf fragments**: Keep rendering logic separate from business logic.  
- **Encapsulate session handling**: Provide a `MessageContext` bean that can be injected, improving testability and allowing unit tests without a servlet container.  
- **Add unit tests**: Mock `HttpServletRequest` and `HttpSession` to verify behavior.  
- **Remove unused fields** or repurpose them for future enhancements.  
- **Handle locale correctly**: Pass `Locale` explicitly instead of reading it inside every method.  
- **Document lifecycle**: Clarify when messages should be displayed or cleared, especially in redirect chains.  
- **Use a proper i18n framework**: Replace `LabelUtil` with standard Java `ResourceBundle` or Spring's `MessageSource`.  

Overall, `MessageUtil` serves its purpose but would benefit from modernizing its API, improving type safety, and separating concerns between message storage and presentation.

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

import java.io.Serializable;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

public class MessageUtil implements Serializable {

	private List messages;
	private List errormessages;

	private MessageUtil() {
		messages = new ArrayList();
	}

	public static void addNoticeMessage(HttpServletRequest req, String message) {
		req.getSession().setAttribute("SM-NOTICEMESSAGE", message);
	}

	public static void addMessage(HttpServletRequest req, String message) {
		req.getSession().setAttribute("SM-MESSAGE", message);
	}

	public static String getMessage(HttpServletRequest req) {
		return (String) req.getSession().getAttribute("SM-MESSAGE");
	}

	public static String getFormErrorMessage(HttpServletRequest req,
			String field) {

		Map errmsgs = (Map) req.getAttribute("errfrmmsg");

		if (errmsgs != null && errmsgs.containsKey(field)) {
			String msg = (String) errmsgs.get(field);
			errmsgs.remove(field);
			return msg;
		} else {
			return null;
		}

	}

	public static void addFormErrorMessage(HttpServletRequest req,
			String field, String message) {

		Map errmsgs = (Map) req.getAttribute("errfrmmsg");
		if (errmsgs == null) {
			errmsgs = new HashMap();
			req.setAttribute("errfrmmsg", errmsgs);
		}
		errmsgs.put(field, message);

	}

	public static void addErrorMessage(HttpServletRequest req, String message) {
		req.getSession().setAttribute("SM-ERR-MESSAGE", message);
	}

	public static String getErrorMessage(HttpServletRequest req) {
		return (String) req.getSession().getAttribute("SM-ERR-MESSAGE");
	}

	public static void addErrorMessages(HttpServletRequest req,
			List<String> messages) {
		req.getSession().setAttribute("SM-ERR-MESSAGES", messages);
	}

	public static String displayFormErrorMessage(HttpServletRequest req,
			String field) {
		String message = getFormErrorMessage(req, field);

		String printmessage = LabelUtil.getInstance().getText(req.getLocale(),
				message);

		if (message != null) {
			StringBuffer sb = new StringBuffer();
			sb
					.append("<tr errorFor='")
					.append(field)
					.append("'>")
					.append(
							"<td align='center' valign='top' colspan='2'><span class='errorMessage'>")
					.append(printmessage).append("</span></td></tr>");
			return sb.toString();
		} else {
			return "";
		}
	}

	public static String displayFormErrorMessageNoFormating(
			HttpServletRequest req, String field) {
		String message = getFormErrorMessage(req, field);

		String printmessage = LabelUtil.getInstance().getText(req.getLocale(),
				message);

		if (message != null) {

			return printmessage;

		} else {
			return "";
		}
	}

	public static String displayMessages(HttpServletRequest req) {

		StringBuffer buffer = new StringBuffer();

		String errmessage = (String) req.getSession().getAttribute(
				"SM-ERR-MESSAGE");
		String message = (String) req.getSession().getAttribute("SM-MESSAGE");
		String noticemessage = (String) req.getSession().getAttribute(
				"SM-NOTICEMESSAGE");

		List<String> errorList = (List<String>) req.getSession().getAttribute(
				"SM-ERR-MESSAGES");

		if (noticemessage != null) {

			buffer.append("<div id=\"message\" class=\"clean-yellow\">")
					.append(noticemessage).append("</div>");

		}

		if (message != null) {

			buffer.append("<div id=\"message\" class=\"icon-ok\">").append(
					message).append("</div>");

		}

		if (errmessage != null) {

			buffer.append("<div id=\"message\" class=\"icon-error\">").append(
					errmessage).append("</div>");

		}

		if (errorList != null && !errorList.isEmpty()) {
			buffer.append("<div id=\"message\" class=\"icon-error\">");
			for (String error : errorList) {
				buffer.append(error);
				buffer.append("<br>");
			}
			buffer.append("</div>");
		}

		req.getSession().removeAttribute("SM-MESSAGE");
		req.getSession().removeAttribute("SM-ERR-MESSAGE");
		req.getSession().removeAttribute("SM-ERR-MESSAGES");
		req.getSession().removeAttribute("SM-NOTICEMESSAGE");
		return buffer.toString();

	}

	public static void resetMessages(HttpServletRequest req) {
		req.getSession().removeAttribute("SM-MESSAGE");
		req.getSession().removeAttribute("SM-ERR-MESSAGE");
		req.getSession().removeAttribute("SM-ERR-MESSAGES");
		req.getSession().removeAttribute("SM-NOTICEMESSAGE");
	}
	
	public static boolean hasMessage(HttpServletRequest req) {
		
		boolean msg = false;
		String errmessage = (String) req.getSession().getAttribute(
			"SM-ERR-MESSAGE");
		String message = (String) req.getSession().getAttribute("SM-MESSAGE");
		String noticemessage = (String) req.getSession().getAttribute(
			"SM-NOTICEMESSAGE");

		List<String> errorList = (List<String>) req.getSession().getAttribute(
			"SM-ERR-MESSAGES");
		
		if (noticemessage != null || message != null || errmessage != null || (errorList != null && !errorList.isEmpty())) {
			msg = true;
		}
		
		return msg;
		
	}
	
}



```
