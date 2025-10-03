# SetCompanyModal.java

## Review

## 1. Summary
- **Purpose**: This class (`SetCompanyModal`) is a tiny helper used by the Direct Web Remoting (DWR) framework to persist the name of a customer company selected in a modal dialog.  
- **Key components**:
  - Two private methods: `setCompanyName(String)` and `getCompanyName()`.  
  - Both methods interact with the current HTTP session via the `WebContextFactory` provided by DWR.
- **Design notes**: The class implements a very narrow responsibility—storing and retrieving a single session attribute. It leverages the DWR `WebContextFactory` to obtain the servlet request instead of injecting a `HttpServletRequest` or session directly.

## 2. Detailed Description
1. **Execution Flow**  
   - **setCompanyName(String)**  
     1. Obtains the current `HttpServletRequest` using `WebContextFactory.get().getHttpServletRequest()`.  
     2. Stores the provided company name under the session attribute key `"customercompany"`.  
   - **getCompanyName()**  
     1. Retrieves the same request.  
     2. Checks if the session contains the `"customercompany"` attribute.  
     3. Returns its value cast to `String`, or an empty string if the attribute is missing.  

2. **Assumptions & Constraints**  
   - The class assumes that the code is executed in a servlet container where a request and session are available (i.e., within a DWR callback).  
   - It assumes that the session attribute value is always a `String`.  
   - The session key `"customercompany"` is hard‑coded, meaning any change to the key must be reflected in all calling code.

3. **Architecture & Design Choices**  
   - **Encapsulation**: The session handling is encapsulated within the class, but the methods are private, suggesting that the class may be used through DWR’s introspection or a higher‑level wrapper.  
   - **No public API**: With all methods private, the class would not be callable from ordinary Java code unless DWR uses reflection.  
   - **No error handling**: The code silently defaults to an empty string when the session attribute is missing; no logging or exceptions are raised.

## 3. Functions/Methods
| Method | Visibility | Purpose | Parameters | Return | Side Effects |
|--------|------------|---------|------------|--------|--------------|
| `setCompanyName(String name)` | private | Stores the supplied company name in the current HTTP session. | `name` – the company name to persist | void | Sets session attribute `"customercompany"` |
| `getCompanyName()` | private | Retrieves the company name stored in the session, or `""` if none. | none | `String` – company name or empty string | Reads session attribute `"customercompany"` |

> **Note**: Both methods use `WebContextFactory.get().getHttpServletRequest()` to obtain the request. This is a static call to the DWR context, which couples the code tightly to DWR.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard Java EE API | Required for session handling. |
| `uk.ltd.getahead.dwr.WebContextFactory` | Third‑party (DWR) | Provides access to the servlet request and session. |
| `javax.servlet.http.HttpSession` (indirectly via `HttpServletRequest`) | Standard Java EE API | For storing the attribute. |

No other external libraries are referenced. The class is platform‑agnostic within any servlet container that supports DWR.

## 5. Additional Notes
### Edge Cases & Limitations
- **Session Expiration**: If the session expires between the set and get calls, `getCompanyName()` will return an empty string, potentially masking a real error.
- **Concurrency**: The code does not synchronize access to the session; however, the servlet container typically handles per‑session synchronization.
- **Attribute Overwrite**: Re‑calling `setCompanyName` will silently overwrite any existing value; no versioning or conflict detection.

### Suggested Improvements
1. **Expose Public API**: Make the two methods public or provide a wrapper so that they can be invoked from Java code or other frameworks.  
2. **Parameterize Session Key**: Use a constant or configuration for the session attribute key to avoid hard‑coding.  
3. **Error Handling / Logging**: Log when the session attribute is missing or when an exception occurs during storage.  
4. **Validation**: Validate the `name` parameter (e.g., non‑null, length limits) before storing.  
5. **Unit Tests**: Add tests that mock the `WebContextFactory` to verify correct session interactions.  
6. **Decouple from DWR**: Inject `HttpServletRequest` or `HttpSession` via constructor or setter to make the class easier to test and less tightly coupled to DWR.

Overall, the class is straightforward and accomplishes a very narrow task, but its tight coupling to DWR and lack of a public API limit its reusability and testability.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.customer;

import javax.servlet.http.HttpServletRequest;

import uk.ltd.getahead.dwr.WebContextFactory;

/**
 * Class used by DWR for storing selection from modal form
 * 
 * @author Administrator
 * 
 */
public class SetCompanyModal {

	private void setCompanyName(String name) {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		req.getSession().setAttribute("customercompany", name);

	}

	private String getCompanyName() {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();

		if (req.getSession().getAttribute("customercompany") != null) {
			return (String) req.getSession().getAttribute("customercompany");
		} else {
			return "";
		}

	}

}



```
