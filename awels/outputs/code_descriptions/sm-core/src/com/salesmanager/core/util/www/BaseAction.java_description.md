# BaseAction.java

## Review

## 1. Summary

`BaseAction` is a Struts 2 base controller used throughout the *salesmanager* web tier.  
It extends `ActionSupport` so that every concrete action inherits:

* **i18n helpers** (`LabelUtil`, `MessageUtil`) for localized messages.  
* **Request/response handling** (`HttpServletRequest`, `HttpServletResponse`) via the standard Struts 2 *Aware* interfaces.  
* **ServletContext** access for resources that need the web‑app context.  
* **User principal** support through `PrincipalProxy`.  

The class focuses on message‑generation helpers (`addFieldMessage`, `setMessage`, etc.) and provides convenient getters/setters for request‑level data.

## 2. Detailed Description

### 2.1 Core Components

| Component | Responsibility |
|-----------|----------------|
| `Logger log` | Logging (currently unused) |
| `HttpServletRequest request / HttpServletResponse response` | HTTP plumbing via `ServletRequestAware` / `ServletResponseAware` |
| `Locale locale` | Holds an overridden locale, if set |
| `PrincipalProxy principal` | Encapsulates authenticated user data |
| `String lastUrl` | Stores the URL the user last visited (useful for redirects) |
| `ServletContext servletContext` | Application context (accessed via `ServletContextAware`) |

### 2.2 Flow of Execution

1. **Instantiation** – Struts 2 creates a new instance for each request.  
2. **Dependency Injection** – Struts calls the `Aware` setters (`setServletRequest`, `setServletResponse`, `setServletContext`, `setPrincipalProxy`) before invoking the action’s `execute()`.  
3. **Locale Resolution** – `getLocale()` first returns the field if set; otherwise it pulls the locale from the session (`WW_TRANS_I18N_LOCALE`). If found, it updates the `ActionContext` locale.  
4. **Action Logic** – Concrete actions override `execute()` and may call the helper methods to populate messages.  
5. **Result Rendering** – After `execute()` finishes, Struts renders the result. The helper methods add messages to the request or the `ActionSupport` message collections, making them available to the view.  
6. **Cleanup** – No explicit cleanup is needed; the action instance is discarded after the request.

### 2.3 Assumptions & Constraints

* **Thread‑safety** – Struts 2 actions are request‑scoped, so the fields are not shared across threads.  
* **Locale** – Assumes the session contains a `Locale` under `WW_TRANS_I18N_LOCALE`.  
* **Message Utilities** – `LabelUtil`/`MessageUtil` are expected to be thread‑safe and stateless.  
* **Principal** – No validation that `principal` is non‑null; downstream code must handle a missing principal.

### 2.4 Architectural Choices

* **Centralized Message Handling** – By providing all message helpers in a single base class, the code avoids duplication across actions.  
* **Struts 2 Integration** – Leveraging the *Aware* interfaces keeps the action class tightly coupled to the framework but also makes the code testable with mocks.  
* **Custom `BaseActionAware`** – The code implements a user‑defined interface (not shown) probably to inject the principal; this pattern is common in large Struts 2 projects.

## 3. Functions / Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `setLastUrl(String)` | Store the URL the user last visited. | `url` | void | assigns to `lastUrl`. |
| `setPrincipalProxy(PrincipalProxy)` | Inject the authenticated user. | `principal` | void | assigns to `principal`. |
| `getPrincipal()` | Retrieve the principal. | – | `PrincipalProxy` | – |
| `getLocale()` | Resolve the locale: field → session → default. | – | `Locale` | may set `ActionContext` locale. |
| `addFieldMessage(String, String)` | Add a field‑specific validation error to the request. | `field`, `messageKey` | void | writes to `MessageUtil`. |
| `addErrorMessage(String)` | Add a generic error message to the request. | `messageKey` | void | writes to `MessageUtil`. |
| `setLocale(Locale)` | Override locale for the action. | `locale` | void | assigns to field. |
| `setMessage(String)` | Add a success message to the action and request. | `messageKey` | void | `ActionSupport` message collection + `MessageUtil`. |
| `setErrorMessage(String)` | Add an error message to the action and request. | `messageKey` | void | `ActionSupport` error collection + `MessageUtil`. |
| `setInputMessage(String)` | Add an input (validation) message to the action. | `messageKey` | void | `ActionSupport` error collection. |
| `setErrorMessage(Exception)` | Add an exception’s message as an error. | `e` | void | `MessageUtil`. |
| `setSuccessMessage()` | Add a generic success confirmation. | – | void | `ActionSupport` message collection. |
| `setTechnicalMessage()` | Add a generic technical error. | – | void | `ActionSupport` error collection. |
| `setServletRequest(HttpServletRequest)` | Struts injects the request. | `request` | void | assigns to field. |
| `getServletRequest()` | Retrieve the current request. | – | `HttpServletRequest` | – |
| `setServletResponse(HttpServletResponse)` | Struts injects the response. | `response` | void | assigns to field. |
| `getServletResponse()` | Retrieve the current response. | – | `HttpServletResponse` | – |
| `getLastUrl()` | Retrieve the stored last URL. | – | `String` | – |
| `setServletContext(ServletContext)` | Struts injects the context. | `context` | void | assigns to field. |
| `getServletContext()` | Retrieve the application context. | – | `ServletContext` | – |

### Utility Notes

* All message helpers rely on **`LabelUtil`** to fetch a localized string (`label.getText(locale, key)`).  
* **`MessageUtil`** is used to write messages directly into the request for potential custom rendering (e.g., via JSP tag libraries).  

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Logging framework; not used in the current code. |
| `org.apache.struts2.interceptor.*` | Struts 2 | Provides `Aware` interfaces for request/response/context injection. |
| `org.apache.struts2.util.ServletContextAware` | Struts 2 | For servlet context injection. |
| `com.opensymphony.xwork2.ActionContext` | Struts 2 | Session and locale handling. |
| `com.opensymphony.xwork2.ActionSupport` | Struts 2 | Base class for actions; supplies i18n methods (`getText`, `addActionError`, etc.). |
| `com.salesmanager.core.util.LabelUtil` | Custom | i18n label helper. |
| `com.salesmanager.core.util.MessageUtil` | Custom | Utility for writing messages to the request. |
| `javax.servlet.*` | Standard | Servlet API for request/response/context objects. |
| `javax.servlet.http.*` | Standard | HTTP specific request/response. |
| `java.util.*` | Standard | Locale, List, ArrayList. |

No database or persistence dependencies appear in this class.

## 5. Additional Notes & Recommendations

### 5.1 Code‑Quality Observations

1. **Raw types** – `List` and `ArrayList` are used without generics (`List msg = new ArrayList();`). Modern Java (since 5) should use `List<String>` and `new ArrayList<>()`.  
2. **Unnecessary field** – The class holds a `Logger log` but never uses it. Either remove it or add logging statements to aid debugging.  
3. **Locale shadowing** – In `getLocale()` a local variable `Locale locale` shadows the field. This is confusing; rename the local variable or refactor the method.  
4. **Null‑checks** – Methods like `setErrorMessage(Exception)` assume `e` is non‑null. Defensive coding (e.g., `if (e != null)`) would make the class more robust.  
5. **Principal handling** – No null‑check for `principal`; downstream code may throw `NullPointerException` if not set. Consider adding a guard or throwing a custom exception if the principal is required.  
6. **Thread safety** – The class is per‑request, so no concurrency issues, but documenting this assumption helps future maintainers.  

### 5.2 Feature Enhancements

| Area | Suggested Improvement |
|------|-----------------------|
| **Internationalization** | Use `ActionSupport`’s `addActionError` / `addActionMessage` directly instead of duplicating logic via `MessageUtil`. |
| **Message Constants** | Centralize message keys in an enum or constants class to avoid typos. |
| **Logging** | Log each message addition for audit trails, especially for errors. |
| **Exception Handling** | Provide a method `handleException(Exception e)` that logs, maps to user‑friendly messages, and returns a default error result. |
| **Unit Tests** | Add unit tests (e.g., using Mockito) to verify that message helpers correctly interact with `MessageUtil`. |
| **Documentation** | Javadoc for each public method would improve readability. |
| **Configuration** | Inject `LabelUtil` and `MessageUtil` via dependency injection (e.g., Spring) instead of static `getInstance()` calls. |

### 5.3 Edge Cases

* **Missing Locale** – If the session lacks a locale, `getLocale()` falls back to `super.getLocale()`. Ensure that the default locale is configured in `struts.xml`.  
* **Non‑UTF Requests** – If requests use a different character set, ensure `LabelUtil` and `MessageUtil` respect the request encoding.  
* **Multiple Calls to Message Methods** – The current implementation always adds a single message; if multiple messages need to be aggregated, consider exposing the underlying list.  

### 5.4 Overall Assessment

`BaseAction` provides a pragmatic, framework‑centric foundation for the web layer. Its responsibilities are clear, and the use of helper utilities keeps action classes focused on business logic. Minor refactoring (generics, null‑checks, logging) and a move towards more declarative message handling would raise the code quality and maintainability.

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
package com.salesmanager.core.util.www;

import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.log4j.Logger;
import org.apache.struts2.interceptor.PrincipalAware;
import org.apache.struts2.interceptor.PrincipalProxy;
import org.apache.struts2.interceptor.ServletRequestAware;
import org.apache.struts2.interceptor.ServletResponseAware;
import org.apache.struts2.util.ServletContextAware;

import com.opensymphony.xwork2.ActionContext;
import com.opensymphony.xwork2.ActionSupport;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

public class BaseAction extends ActionSupport implements ServletContextAware, ServletRequestAware,
		ServletResponseAware, BaseActionAware {

	private Logger log = Logger.getLogger(BaseAction.class);
	private HttpServletRequest request;
	private HttpServletResponse response;
	private Locale locale;
	protected PrincipalProxy principal;
	private String lastUrl;
	
	private ServletContext servletContext = null;

	public void setLastUrl(String url) {
		this.lastUrl = url;
	}

	public void setPrincipalProxy(PrincipalProxy principal) {
		this.principal = principal;
	}

	public PrincipalProxy getPrincipal() {
		return principal;
	}

	public Locale getLocale() {
		if (locale != null) {
			return locale;
		}
		Locale locale = (Locale) ActionContext.getContext().getSession().get(
				"WW_TRANS_I18N_LOCALE");
		if (locale != null && (locale instanceof Locale)) {
			ActionContext.getContext().setLocale(locale);
			return locale;
		} else {
			return super.getLocale();
		}
	}

	protected void addFieldMessage(String field, String messageKey) {

		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(super.getLocale());
		MessageUtil.addFormErrorMessage(getServletRequest(), field, label
				.getText(getLocale(), messageKey));
	}

	protected void addErrorMessage(String messageKey) {

		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(super.getLocale());

		MessageUtil.addErrorMessage(getServletRequest(), label.getText(
				getLocale(), messageKey));
	}

	public void setLocale(Locale locale) {
		this.locale = locale;
	}

	protected void setMessage(String messageKey) {
		List msg = new ArrayList();
		msg.add(getText(messageKey));
		super.setActionMessages(msg);

		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(super.getLocale());

		MessageUtil.addMessage(getServletRequest(), label.getText(messageKey));
	}

	protected void setErrorMessage(String messageKey) {
		List msg = new ArrayList();
		msg.add(getText(messageKey));
		super.setActionErrors(msg);

		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(super.getLocale());

		MessageUtil.addErrorMessage(getServletRequest(), label
				.getText(messageKey));
	}

	protected void setInputMessage(String messageKey) {
		List msg = new ArrayList();
		msg.add(getText(messageKey));
		super.setActionErrors(msg);

	}

	protected void setErrorMessage(Exception e) {

		MessageUtil.addErrorMessage(getServletRequest(), e.getMessage());
	}

	protected void setSuccessMessage() {
		List msg = new ArrayList();
		msg.add(getText("message.confirmation.success"));
		super.setActionMessages(msg);
	}

	protected void setTechnicalMessage() {
		List msg = new ArrayList();
		msg.add(getText("errors.technical"));
		super.setActionErrors(msg);
	}

	public void setServletRequest(HttpServletRequest request) {
		this.request = request;
	}

	public HttpServletRequest getServletRequest() {
		return request;
	}

	public void setServletResponse(HttpServletResponse response) {
		this.response = response;
	}

	public HttpServletResponse getServletResponse() {
		return response;
	}

	public String getLastUrl() {
		return lastUrl;
	}

	public void setServletContext(ServletContext context) {
		servletContext = context;
		
	}

	public ServletContext getServletContext() {
		return servletContext;
	}
}



```
