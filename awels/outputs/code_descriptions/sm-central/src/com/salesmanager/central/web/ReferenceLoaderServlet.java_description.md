# ReferenceLoaderServlet.java

## Review

## 1. Summary  

The **ReferenceLoaderServlet** is a bootstrap servlet that runs once at application startup (typically defined with `loadOnStartup` in `web.xml`). Its primary responsibility is to pre‑load reference data, system groups, registration associations, functions, and catalogue categories into in‑memory caches so that the rest of the application can serve requests quickly.  

Key components that the servlet orchestrates:  

| Component | Role |
|-----------|------|
| `ReferenceService` | Provides reference data for the UI (e.g., lookup tables). |
| `SystemService` | Supplies system‑wide configuration such as groups, functions, and associations. |
| `MenuFactory` | Singleton that holds menu configuration; is populated with groups, functions, and associations. |
| `RefCache` | Core reference cache used across the system. |
| `CatalogService` | Loads category hierarchies into cache. |

The servlet does **not** handle any HTTP requests (`doGet`/`doPost` are stubs). Its only job is initialization.

## 2. Detailed Description  

### Execution Flow  

1. **Construction** – The servlet’s no‑arg constructor simply calls `super()`.  
2. **Container‑managed `init(ServletConfig)`** – The container calls this method. The implementation delegates to `super.init(config)`.  
3. **Custom `init()`** – The servlet also implements the no‑arg `init()` (not invoked by the servlet container). Inside this method the servlet:  
   * Logs the start of initialization.  
   * Retrieves services from a static `ServiceFactory`.  
   * Calls `SystemService` to fetch groups, associations, and functions.  
   * Updates a singleton `MenuFactory` instance with this data.  
   * Instantiates (or obtains) the core `RefCache`.  
   * Instantiates `CatalogService` and loads the category cache.  
   * Logs completion.  
   * Any exception is caught, logged, but **not** re‑thrown.  
4. **HTTP Methods** – `doGet` and `doPost` are empty stubs; the servlet is not meant to process client requests.

### Assumptions & Constraints  

| Assumption | Impact |
|------------|--------|
| The container will call the no‑arg `init()` **or** the custom `init(ServletConfig)` will trigger the custom logic. | In practice, only `init(ServletConfig)` is called by the servlet container; the no‑arg `init()` will *not* execute unless invoked manually. This means the bootstrap logic may never run. |
| `ServiceFactory.getService(...)` returns fully initialized singleton services. | The code relies on a static factory; if the factory is mis‑configured the servlet will silently log errors. |
| `MenuFactory` is thread‑safe and designed as a singleton. | No synchronization is shown; if multiple threads modify it concurrently, race conditions may occur. |
| `CatalogService.loadCategoriesCache()` completes quickly and does not block the request thread for long. | If the method is slow, the servlet container will delay startup. |
| No external configuration or dependency injection frameworks are used. | Hard‑coded service lookups make unit testing difficult. |

### Architecture & Design Choices  

* **Service Locator** – `ServiceFactory` is used instead of dependency injection (DI). This is a classic *service locator* pattern, which makes the code tightly coupled to a static factory.  
* **Singletons** – `MenuFactory` and `RefCache` are singletons. The servlet only configures them once.  
* **No‑arg `init()`** – The use of a no‑arg `init()` that is not part of the servlet contract is a design flaw; it can cause the initialization block to be ignored.  

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `ReferenceLoaderServlet()` | `public ReferenceLoaderServlet()` | Constructor. Calls `super()`; does nothing else. | None | None | Instantiates the servlet object. |
| `init(ServletConfig)` | `public void init(ServletConfig config)` | Standard servlet init method called by the container. Delegates to `super.init(config)`. | `ServletConfig` | None | Initializes servlet configuration. |
| `init()` | `public void init()` | Custom initialization routine that loads reference data, configures `MenuFactory`, and pre‑loads caches. | None | None | Populates caches, logs debug messages. |
| `doGet(HttpServletRequest, HttpServletResponse)` | `protected void doGet(HttpServletRequest request, HttpServletResponse response)` | Stub for HTTP GET requests. | Request & response objects | None | None. |
| `doPost(HttpServletRequest, HttpServletResponse)` | `protected void doPost(HttpServletRequest request, HttpServletResponse response)` | Stub for HTTP POST requests. | Request & response objects | None | None. |

### Reusable/Utility Methods  

* The class contains no reusable methods beyond standard servlet lifecycle hooks. All logic is tightly coupled to the initialization routine.

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `javax.servlet` | Standard | Provides servlet API. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.salesmanager.core.service.*` | Third‑party | Domain services (`ServiceFactory`, `ReferenceService`, `SystemService`, `CatalogService`). |
| `com.salesmanager.core.service.cache.RefCache` | Third‑party | Cache implementation. |
| `com.salesmanager.central.web.MenuFactory` | Custom | Singleton that holds menu configuration. |

All dependencies are internal to the SalesManager application or standard Java EE.

## 5. Additional Notes  

### Issues & Edge Cases  

1. **Lifecycle Mis‑implementation** – The servlet overrides the no‑arg `init()` instead of extending the container‑invoked `init(ServletConfig)`. Consequently, the critical initialization code may never execute unless the servlet is manually instantiated.  
2. **Exception Handling** – All exceptions during initialization are swallowed after logging. The servlet should propagate the exception to signal startup failure.  
3. **Thread Safety** – `MenuFactory` and `RefCache` are singletons but the code does not enforce thread safety when writing to them.  
4. **Raw Types** – `Collection groups`, `associations`, `functions` use raw types. Using generics (`Collection<Group>`, etc.) would improve type safety.  
5. **Resource Leaks** – No explicit resource cleanup is needed here, but any future extensions that open streams or DB connections must be closed.  
6. **Hard‑coded ServiceFactory** – Tightly couples the servlet to a static service locator; unit testing would require mocking the static factory or refactoring to DI.  
7. **Performance** – `CatalogService.loadCategoriesCache()` is invoked on startup; if the category tree is large this could delay the servlet container start‑up. Consider asynchronous initialization or lazy loading.  

### Potential Enhancements  

| Enhancement | Rationale |
|-------------|-----------|
| **Use `init(ServletConfig)` exclusively** | Guarantees that the init routine runs when the servlet starts. |
| **Propagate `ServletException`** | If initialization fails, the container will fail the startup instead of silently continuing. |
| **Adopt dependency injection (Spring, CDI)** | Decouple service lookup from a static factory, making testing and configuration easier. |
| **Add proper generics** | Prevent ClassCastException and improve code clarity. |
| **Thread‑safe singleton initialization** | Ensure `MenuFactory` and `RefCache` are safely published to all threads. |
| **Add shutdown hook** | If caches need to be cleared on application shutdown, implement `destroy()` method. |
| **Log at appropriate levels** | Replace `debug` with `info` for successful initialization, use `error` for failures. |
| **Separate concerns** | Move cache preloading logic to a dedicated `CacheInitializer` component that the servlet simply delegates to. |

### Final Verdict  

The servlet is a classic bootstrap component designed to preload reference data and caches. However, its current implementation contains a critical lifecycle flaw (overriding the wrong `init()`), minimal error handling, and tight coupling to static service lookups. Refactoring to use the proper servlet initialization contract, adding robust error handling, and adopting DI would greatly improve reliability, testability, and maintainability.

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
package com.salesmanager.central.web;

import java.io.IOException;
import java.util.Collection;

import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.log4j.Logger;

import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.service.system.SystemService;

/**
 * Servlet implementation class for Servlet: FunctionLoaderServlet
 * 
 */
public class ReferenceLoaderServlet extends javax.servlet.http.HttpServlet
		implements javax.servlet.Servlet {
	static final long serialVersionUID = 1L;

	/*
	 * (non-Java-doc)
	 * 
	 * @see javax.servlet.http.HttpServlet#HttpServlet()
	 */
	public ReferenceLoaderServlet() {
		super();
	}

	public void init(ServletConfig config) throws ServletException {
		super.init(config);
	}

	public void init() throws ServletException {

		try {

			Logger.getLogger(ReferenceLoaderServlet.class).debug(
					"********* SPRING INIT **********");
			ReferenceService ref = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			Logger.getLogger(ReferenceLoaderServlet.class).debug(
					"********* SPRING LOADED **********");


			SystemService sservice = (SystemService) ServiceFactory
					.getService(ServiceFactory.SystemService);

			Collection groups = sservice.getCentralGroups();

			Collection associations = sservice
					.getCentralRegistrationAssociations();

			Collection functions = sservice.getCentralFunctions();

			MenuFactory factory = MenuFactory.getInstance();
			factory.setGroups(groups, associations);
			factory.setFunctions(functions, associations);
			factory.setFunctionsByFunctionCode(functions);


			/**
			 * Load core cache
			 */
			com.salesmanager.core.service.cache.RefCache corecache = com.salesmanager.core.service.cache.RefCache
					.getInstance();


			CatalogService catalogService = new CatalogService();
			catalogService.loadCategoriesCache();

		} catch (Exception e) {
			Logger.getLogger(ReferenceLoaderServlet.class).error(e);
		}
	}

	/*
	 * (non-Java-doc)
	 * 
	 * @see javax.servlet.http.HttpServlet#doGet(HttpServletRequest request,
	 * HttpServletResponse response)
	 */
	protected void doGet(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
		// TODO Auto-generated method stub
	}

	/*
	 * (non-Java-doc)
	 * 
	 * @see javax.servlet.http.HttpServlet#doPost(HttpServletRequest request,
	 * HttpServletResponse response)
	 */
	protected void doPost(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
		// TODO Auto-generated method stub
	}
}


```
