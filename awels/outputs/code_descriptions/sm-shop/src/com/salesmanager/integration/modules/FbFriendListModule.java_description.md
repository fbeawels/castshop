# FbFriendListModule.java

## Review

## 1. Summary  
The `FbFriendListModule` is a Spring‑managed component that implements the `PortletModule` interface, intended to display a Facebook friends list on a merchant store page.  
- **Core responsibilities**  
  - Determine if Facebook authentication is required (`requiresAuthorization`).  
  - Render the friends list (`display`).  
  - Handle form submissions (currently a stub).  
- **Key components**  
  - **Spring**: The class is annotated with `@Component("friendslist")`, enabling dependency injection.  
  - **restfb**: The code uses the RestFB library (`FacebookClient`, `Connection`, `User`, `CategorizedFacebookType`) to talk to the Facebook Graph API.  
  - **SalesManager utilities**: `SalesManagerFacebookClient`, `PageExecutionContext`, `PageRequestAction` etc. are part of the SalesManager code‑base.  
- **Design patterns**  
  - *PortletModule* acts as a strategy for rendering page modules.  
  - *Factory / Adapter* pattern is hinted at by `SalesManagerFacebookClient` wrapping a RestFB client.

## 2. Detailed Description  
### Execution Flow  
1. **Initialization** – The Spring container instantiates the bean; no constructor logic is present.  
2. **Authorization** – `requiresAuthorization()` always returns `false`. In practice, a friends list typically requires a user‑level access token; the module currently does not enforce this.  
3. **Display** – The `display()` method is invoked during page rendering.  
   - It contains only a large block of commented‑out example code that shows how to fetch the logged‑in user, friends, and likes.  
   - No actual logic runs; the method simply logs an exception if one occurs.  
4. **Submit** – The `submit()` method is a stub; it does nothing.  
5. **Cleanup** – None required; the method ends normally.

### Dependencies & Assumptions  
- Relies on a pre‑configured `FacebookUser` stored in the `PageExecutionContext`.  
- Assumes a valid OAuth token (`facebookUser.getOauth_token()`).  
- Expects the `SalesManagerFacebookClient` to expose a `FacebookClient` that can call the Graph API.  
- No error handling for pagination or API rate‑limits.

### Architecture Choices  
- **Separation of concerns**: UI logic is in the module, API logic is encapsulated in the client wrapper.  
- **Extensibility**: By implementing `PortletModule`, the component can be swapped or extended without touching the page rendering logic.

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `requiresAuthorization()` | Declares if this module needs Facebook auth. | – | `boolean` | None | Returns `false` – not aligned with typical friends‑list use‑case. |
| `display(MerchantStore store, HttpServletRequest request, Locale locale, PageRequestAction action, PageExecutionContext pageContext)` | Renders the module on a page. | * store – merchant context<br>* request – servlet request<br>* locale – locale for i18n<br>* action – page request action (e.g., edit, view)<br>* pageContext – shared execution context | `void` | May set request attributes (`fbFriends`). | Currently only contains commented example code; no runtime logic. |
| `submit(MerchantStore store, HttpServletRequest request, Locale locale, PageRequestAction action, PageExecutionContext pageContext)` | Handles form submissions. | Same as `display` | `void` | None | Stub; no implementation. |

### Utility Methods (not present in this class)  
The code references helper classes (`SalesManagerFacebookClient`, `FacebookUser`) that encapsulate API logic. Those would contain reusable methods like `fetchFriends()` or `getOAuthToken()`.

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `org.springframework.stereotype.Component` | Spring | Marks the class as a bean. |
| `org.apache.log4j.Logger` | Logging | Provides error logging. |
| `javax.servlet.http.HttpServletRequest` | Servlet API | Access to request data. |
| `java.util.Locale` | Java Standard | Locale handling. |
| `com.restfb.*` | RestFB (third‑party) | Facebook Graph API client. |
| `com.salesmanager.*` | SalesManager internal | Core domain entities (`MerchantStore`, `PortletModule`), utilities (`PageExecutionContext`, `PageRequestAction`), Facebook client wrapper. |
| `org.w3c.dom.*` | Java Standard | DOM parsing (unused in the current code). |

No platform‑specific dependencies beyond the Servlet API and Spring.

## 5. Additional Notes  

### Edge Cases & Missing Features  
- **Authorization**: Returning `false` while the code later expects an authenticated user will cause runtime failures.  
- **Empty Implementation**: The module currently does nothing; it should at least handle the scenario where no `facebookUser` is present.  
- **Pagination**: The Graph API may return paginated results; the current logic (if it were enabled) ignores pagination.  
- **Error Handling**: A generic `catch (Exception)` suppresses stack traces; the log entry may not be actionable.  
- **Security**: The OAuth token is retrieved from the context but never validated or refreshed.  
- **Null Checks**: The code assumes `facebookUser` and `token` are non‑null; a null check would prevent `NullPointerException`.  

### Suggested Enhancements  
1. **Implement `requiresAuthorization()`**  
   ```java
   public boolean requiresAuthorization() { return true; }
   ```  
   or tie it to a configuration flag.

2. **Activate the display logic**  
   Move the commented example into production code, adding null checks, pagination handling, and request attribute population.

3. **Use Dependency Injection for FacebookClient**  
   ```java
   @Autowired
   private FacebookClient facebookClient;
   ```
   This decouples client creation from the module.

4. **Handle Pagination**  
   ```java
   List<User> friends = new ArrayList<>();
   Connection<User> connection = facebookClient.fetchConnection("me/friends", User.class);
   while (connection != null) {
       friends.addAll(connection.getData());
       connection = connection.getNextPage() != null ? facebookClient.fetchConnection(connection.getNextPage().getEndpoint(), User.class) : null;
   }
   ```

5. **Better Logging**  
   Log the exception’s stack trace and the request parameters that caused it.

6. **Unit Tests**  
   Mock `FacebookClient` and `PageExecutionContext` to test the module in isolation.

7. **Internationalization**  
   Use `locale` to format dates or other locale‑sensitive data.

### Future Extensions  
- **Like/Photo Retrieval**: Extend the module to show likes or photos, using `me/likes` and `me/photos`.  
- **Search**: Add a search field to filter friends by name.  
- **Caching**: Cache friend lists per user to reduce API calls.  
- **Asynchronous Loading**: Use AJAX to load the friends list after page render.  

---

**Conclusion**  
The skeleton of a Facebook friend‑list portlet is present, but the implementation is incomplete. To become production‑ready, the module needs real Facebook API integration, robust error handling, proper authorization checks, and test coverage. Once those aspects are addressed, the component will fit naturally into the SalesManager module framework and provide a useful social integration for merchants.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Jan 12, 2011 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.integration.modules;

import java.util.List;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;

import org.apache.log4j.Logger;
import org.springframework.stereotype.Component;
import org.w3c.dom.Document;
import org.w3c.dom.NodeList;


import com.restfb.Connection;
import com.restfb.DefaultFacebookClient;
import com.restfb.FacebookClient;
import com.restfb.types.CategorizedFacebookType;
import com.restfb.types.User;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.integration.PortletModule;
import com.salesmanager.core.util.www.PageExecutionContext;
import com.salesmanager.core.util.www.PageRequestAction;
import com.salesmanager.core.util.www.integration.fb.FacebookUser;
import com.salesmanager.core.util.www.integration.fb.SalesManagerFacebookClient;

@Component("friendslist")
public class FbFriendListModule implements PortletModule {
	
	private Logger log = Logger.getLogger(FbFriendListModule.class);
	
	public boolean requiresAuthorization() {
		return false;
	}

	public void display(MerchantStore store, HttpServletRequest request,
			Locale locale, PageRequestAction action,
			PageExecutionContext pageContext) {
		
		
		try {
			
			
/*			//example on how to use facebook java graph API

			
			FacebookUser user = (FacebookUser)pageContext.getFromExecutionContext("facebookUser");
			
			String token = user.getOauth_token();

			
			FacebookClient facebookClient = new SalesManagerFacebookClient(token);
			//default is from API
			//FacebookClient facebookClient = new DefaultFacebookClient(token);


			User u = facebookClient.fetchObject("me", User.class);
			
			Connection<User> myFriends = facebookClient.fetchConnection("me/friends", User.class);
					    
			if(myFriends!=null) {
				
				List<User> users = myFriends.getData();
				request.setAttribute("fbFriends", users);
				
			}
			
			Connection<CategorizedFacebookType> likes = 
				facebookClient.fetchConnection("me/likes", 
				CategorizedFacebookType.class);
			
			//fb_sig_profile_user
			
		    

			System.out.println(likes.hasNext());*/
			
		} catch (Exception e) {
			log.error(e);
		}
		
		


	}

	public void submit(MerchantStore store, HttpServletRequest request,
			Locale locale, PageRequestAction action,
			PageExecutionContext pageContext) {
		// TODO Auto-generated method stub

	}

}



```
