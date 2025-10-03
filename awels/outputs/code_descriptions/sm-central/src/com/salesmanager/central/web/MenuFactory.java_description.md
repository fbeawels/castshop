# MenuFactory.java

## Review

## 1. Summary  

**Purpose**  
`MenuFactory` is a singleton helper that builds and caches the menu structure (groups + functions) for merchant‑registration workflows. It pulls data from the core entities (`CentralFunction`, `CentralGroup`, `CentralRegistrationAssociation`, etc.) and organizes it per registration definition. The class exposes methods to set the groups/functions and retrieve them, optionally filtered by a specific registration code and group code.

**Key components**

| Component | Responsibility |
|-----------|----------------|
| `groups` | `Map<registrationCode, List<CentralGroupRegistration>>` – holds the group hierarchy for each registration |
| `functions` | `Map<registrationCode, List<CentralFunctionRegistration>>` – holds the function list for each registration |
| `functionsByFunctionCode` | `Map<functionCode, CentralFunction>` – quick lookup of a function by its code |
| `CacheFactory` | Lightweight cache provider used to persist the computed lists between calls |
| `MenuFactory` | Singleton that orchestrates building and retrieving the above maps |

**Design patterns & libraries**

* Singleton (lazy initialization, non‑thread‑safe)  
* Factory (creating lists/maps via `CacheFactory`)  
* Utility library: `org.apache.commons.lang.StringUtils`  
* Logging: `org.apache.log4j.Logger` (used only for errors)

---

## 2. Detailed Description  

### Flow of execution  

1. **Instantiation** – `MenuFactory.getInstance()` lazily creates a single instance; constructors populate the `groups` and `functions` caches via `CacheFactory`.  
2. **Setting data**  
   * `setGroups(grouplist, registrationList)` processes a collection of `CentralGroup` objects and a list of `CentralRegistrationAssociation` objects, producing a `CentralGroupRegistration` for every match and storing the result in the `groups` map keyed by the registration code.  
   * `setFunctions(functionsColl, registrationList)` does a similar job for functions: it creates `CentralFunctionRegistration` objects, builds a helper `functionsurl` cache mapping URLs to `AuthorizationCodes`, and stores the registrations in `functions`.  
3. **Retrieving data**  
   * `getGroups(registrationCode)` fetches the pre‑computed group list from the `groups` map.  
   * `getFunctions(registrationCode, group)` first checks a per‑registration/group cache. If missing, it delegates to `getFunctionsList` which rebuilds the list by filtering the master `functions` map and stores the result in a cache list.  
4. **Utility** – `getFunctionsByFunctionCode()` and `setFunctionsByFunctionCode(Collection)` maintain a map for quick lookup by function code.

### Assumptions & constraints  

| Assumption | Implication |
|------------|-------------|
| `CacheFactory` is thread‑safe and will keep data across calls | Concurrency is only partially addressed (only `getFunctionsList` is synchronized) |
| Every `CentralRegistrationAssociation` is associated with a valid `CentralGroup` or `CentralFunction` | No defensive checks for missing associations |
| `registrationCode` is never negative | No guard against bad input |
| `StringUtils.isBlank` is used to validate codes | Avoids NPE but still relies on the caller to pass non‑null objects |

### Architecture & design choices  

* **Maps per registration code** – keeps the menu data bounded to the registration type, making retrieval fast.  
* **Separate caches for URLs** – `functionsurl` maps a function URL to its associated registration/promotions, presumably for quick auth checks elsewhere.  
* **Raw collections** – the code predates generics; it mixes `Map`, `List`, `Iterator` without type parameters.  
* **Lazy singleton** – a classic but non‑thread‑safe lazy init pattern.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `public static MenuFactory getInstance()` | Returns the singleton instance (lazy init). | None | `MenuFactory` | Creates instance if absent |
| `private MenuFactory()` | Constructor that initialises the `groups` and `functions` maps via `CacheFactory`. | None | None | Fills caches |
| `public void setGroups(Collection groupsColl, Collection registrationList)` | Builds `CentralGroupRegistration` objects for each matching group/registration pair and stores them in `groups`. | `Collection` of `CentralGroup` (mis‑named `groupsColl`), `Collection` of `CentralRegistrationAssociation` | None | Overwrites `groups` map |
| `public List getGroups(int registrationcode)` | Retrieves the list of groups for a given registration code. | `int` | `List` of `CentralGroupRegistration` | None |
| `public void setFunctions(Collection functionsColl, Collection registrationList)` | Builds `CentralFunctionRegistration` objects, updates the `functions` map, and populates the `functionsurl` cache. | `Collection` of `CentralFunction` (mis‑named `functionsColl`), `Collection` of `CentralRegistrationAssociation` | None | Overwrites `functions` map, updates `functionsurl` |
| `public List getFunctions(int registrationcode, String group)` | Returns a cached or freshly computed list of functions for a registration+group. | `int`, `String` | `List` of `CentralFunctionRegistration` | Caches result if not present |
| `private synchronized List getFunctionsList(int registrationcode, String group)` | Filters the master `functions` list for a specific registration/group and populates a cache list. | `int`, `String` | `List` of `CentralFunctionRegistration` | Creates cache list |
| `public Map getFunctions()` | Lazily returns the `functions` map. | None | `Map` | Instantiates map if null |
| `public Map<String, CentralFunction> getFunctionsByFunctionCode()` | Returns the map of functions keyed by function code. | None | `Map` | None |
| `public void setFunctionsByFunctionCode(Collection functions)` | Builds `functionsByFunctionCode` from a collection of `CentralFunction`. | `Collection` of `CentralFunction` | None | Populates map |
| `public void setFunctionsByFunctionCode(Collection functions)` (overloaded) – actually only one; see above. | N/A | N/A | N/A | N/A |

**Reusable/utility methods** – none; all methods are tightly coupled to the cache logic.

---

## 4. Dependencies  

| Library / Framework | Type | Role |
|---------------------|------|------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String blank checks |
| `org.apache.log4j.Logger` | Third‑party | Error logging |
| `com.salesmanager.core.entity.system.*` | Core entities | Domain objects (`CentralFunction`, `CentralGroup`, etc.) |
| `com.salesmanager.central.util.CacheFactory` | Core util | Provides cache maps/lists |
| `com.salesmanager.central.entity.functions.*` | Core entities | Registration/association wrappers |
| `java.util.*` | Standard | Collections, iterators |

*No external web frameworks (Servlets, Spring, etc.) are used.*  
*The code is platform‑agnostic but assumes a Java SE environment with the above libraries.*

---

## 5. Additional Notes & Recommendations  

### 5.1 Code‑quality issues  
1. **Raw types** – All collections use raw `Map`, `List`, `Iterator`. Modern Java (post‑Java‑5) encourages generics (`Map<Integer, List<CentralGroupRegistration>>`). This would prevent class‑cast errors and improve readability.  
2. **Deprecated constructors** – `new Integer(code)` should be replaced with `Integer.valueOf(code)` (autoboxing handles this now).  
3. **Thread safety** –  
   * `getInstance()` is not thread‑safe: two threads may create two instances.  
   * Only `getFunctionsList` is synchronized; other mutating methods (`setGroups`, `setFunctions`) are not. Concurrency on `groups`/`functions` maps could lead to data races.  
   * Suggested fix: use an `enum`‑based singleton or static initializer, and guard all mutable operations with a `ReentrantReadWriteLock`.  
4. **Logging vs `printStackTrace`** – The code mixes `e.printStackTrace()` with `Logger.error(e)`. Prefer `logger.error("message", e)` consistently.  
5. **Hard‑coded cache names** – Strings like `"functions" + registrationcode + group` make the code fragile. Extract these as constants or use a key‑generation method.  
6. **Redundant loops** – The nested loops in `setGroups` and `setFunctions` can be simplified (e.g., using Java 8 streams or indexed maps).  
7. **Null checks** – The methods assume non‑null inputs; defensive programming could improve robustness.

### 5.2 Performance & caching  
* `getFunctions` first checks the cache and, if missing, calls `getFunctionsList`, which builds the list and **does not** store the result back into the cache. The cache is created inside `getFunctionsList`, but the list is returned directly; the caller may expect a cached list. Clarify this behavior or add a `putCacheList` step.  
* `setFunctions` rebuilds the entire `functions` map each time it is called. If this method is invoked frequently (e.g., per request), it could become a bottleneck. Consider incremental updates or a read‑only snapshot.

### 5.3 API clarity  
* Method names such as `setGroups` and `setFunctions` are misleading because they *replace* the entire internal map rather than *add* to it. Renaming to `replaceGroups` / `replaceFunctions` or documenting the behavior would help.  
* Parameters are poorly named (`groupsColl`, `registrationList`). Use more descriptive names (`groupCollection`, `registrationAssociations`).  

### 5.4 Extensibility  
* If new menu item types (e.g., sub‑menus, separators) are added, the current structure will need to be extended. A hierarchy‑aware model (e.g., a `MenuItem` base class) could simplify future changes.  

### 5.5 Future enhancements  
1. **Generics & Java 8** – Refactor to use generics and streams for cleaner, safer code.  
2. **Thread‑safe singleton** – Implement enum‑based or `volatile` double‑checked locking.  
3. **Cache abstraction** – Wrap `CacheFactory` behind a more expressive interface (`MenuCache`) to hide cache specifics.  
4. **Unit tests** – Add tests covering all public methods, especially edge cases (empty collections, nulls, concurrent access).  
5. **Configuration** – Externalise cache names/keys to a config file or constants.  

---

### Verdict  

`MenuFactory` implements a functional approach to building menu structures for different merchant registration scenarios. It leverages caching and domain entities appropriately. However, the implementation is dated, lacks type safety, and is partially thread‑unsafe. Refactoring to modern Java practices (generics, concurrency utilities, clean logging) would greatly improve maintainability, reliability, and future extensibility.

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

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.entity.functions.CentralFunctionRegistration;
import com.salesmanager.central.entity.functions.CentralGroupRegistration;
import com.salesmanager.central.util.CacheFactory;
import com.salesmanager.core.entity.system.CentralFunction;
import com.salesmanager.core.entity.system.CentralGroup;
import com.salesmanager.core.entity.system.CentralRegistrationAssociation;

public class MenuFactory {

	private static MenuFactory menuFactory = null;
	private Map groups;
	private Map functions;
	private Map<String, CentralFunction> functionsByFunctionCode;

	public Map<String, CentralFunction> getFunctionsByFunctionCode() {
		return functionsByFunctionCode;
	}

	public void setFunctionsByFunctionCode(Collection functions) {

		if (functions != null) {
			// functionsByFunctionCode = functions;
			functionsByFunctionCode = new TreeMap();
			Iterator i = functions.iterator();
			while (i.hasNext()) {
				CentralFunction cf = (CentralFunction) i.next();
				functionsByFunctionCode.put(cf.getCentralFunctionCode(), cf);
			}
		}

	}

	private MenuFactory() {
		CacheFactory cfactory = CacheFactory.getInstance();
		try {
			groups = cfactory.createCacheMap("groups");
			functions = cfactory.createCacheMap("functions");
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	public static MenuFactory getInstance() {
		if (menuFactory == null) {
			menuFactory = new MenuFactory();
		}
		return menuFactory;
	}

	private synchronized List getFunctionsList(int registrationcode,
			String group) {

		CacheFactory cfactory = CacheFactory.getInstance();
		List functionslist = (List) functions
				.get(new Integer(registrationcode));
		// Logger.getLogger(MenuFactory.class).debug("*** ANALYZE F LIST " +
		// registrationcode + " SIZE " + functionslist.size());
		Iterator i = functionslist.iterator();
		try {
			List rfunctionlist = cfactory.createCacheList("functions"
					+ registrationcode + group);
			while (i.hasNext()) {
				CentralFunctionRegistration function = (CentralFunctionRegistration) i
						.next();
				int code = function.getMerchantRegistrationDefCode();
				String groupcode = function.getCentralGroupCode();

				if (!StringUtils.isBlank(groupcode) && registrationcode == code
						&& groupcode.equalsIgnoreCase(group)) {
					rfunctionlist.add(function);
				}
			}
			return rfunctionlist;
		} catch (Exception e) {
			e.printStackTrace();
			return new ArrayList();
		}
	}

	public List getFunctions(int registrationcode, String group) {
		CacheFactory cfactory = CacheFactory.getInstance();
		// Logger.getLogger(MenuFactory.class).debug("*** ANALYZE F " +
		// registrationcode + " " + group);
		if (cfactory.containsCache("functions" + registrationcode + group)) {
			// Logger.getLogger(MenuFactory.class).debug("*** ANALYZE F - GOT FROM CACHE");
			return cfactory
					.getCacheList("functions" + registrationcode + group);
		}
		List functionslist = getFunctionsList(registrationcode, group);
		return functionslist;

	}

	public List getGroups(int registrationcode) {
		List grouplist = (List) groups.get(new Integer(registrationcode));

		if (grouplist == null)
			grouplist = new ArrayList();

		return grouplist;
	}

	/**
	 * Set groups in a map per registration code
	 * 
	 * @param grouplist
	 */
	public void setGroups(Collection groupsColl, Collection registrationList) {

		if (registrationList == null || registrationList.size() == 0
				|| groupsColl == null || groupsColl.size() == 0)
			return;

		groups = new HashMap();

		// Iterator i = registrationList.iterator();
		Iterator i = groupsColl.iterator();
		while (i.hasNext()) {
			// CentralRegistrationAssociation association =
			// (CentralRegistrationAssociation)i.next();
			CentralGroup gr = (CentralGroup) i.next();

			Iterator regIterator = registrationList.iterator();
			while (regIterator.hasNext()) {
				CentralRegistrationAssociation association = (CentralRegistrationAssociation) regIterator
						.next();
				if (!StringUtils.isBlank(association.getCentralGroupCode())
						&& association.getCentralGroupCode().equals(
								gr.getCentralGroupCode())) {

					int code = association.getMerchantRegistrationDefCode();
					Integer akey = new Integer(code);

					CentralGroupRegistration registration = new CentralGroupRegistration();
					registration.setCentralGroupCode(association
							.getCentralGroupCode());
					registration.setMerchantRegistrationDefCode(association
							.getMerchantRegistrationDefCode());
					registration.setPromotionCode(association
							.getPromotionCode());

					registration.setCentralGroupDescription(gr
							.getCentralGroupDescription());
					registration.setCentralGroupNewUntil(gr
							.getCentralGroupNewUntil());
					registration.setCentralGroupNew(gr.isCentralGroupNew());
					registration.setCentralGroupPosition(gr
							.getCentralGroupPosition());
					registration.setCentralGroupVisible(gr
							.isCentralGroupVisible());

					// Logger.getLogger(MenuFactory.class).debug("*** CHECKING KEY "
					// + akey);
					if (!groups.containsKey(akey)) {
						// Logger.getLogger(MenuFactory.class).debug("*** KEY "
						// + akey + " does not exixt");
						List newlist = new ArrayList();
						// Logger.getLogger(MenuFactory.class).debug("***WILL ADD to key "
						// + akey + group.getCentralGroupCode() + " desc " +
						// group.getCentralGroupDescription());
						newlist.add(registration);
						groups.put(akey, newlist);
					} else {
						// Logger.getLogger(MenuFactory.class).debug("*** KEY "
						// + akey + " exixt");
						List thelist = (List) groups.get(akey);
						// Logger.getLogger(MenuFactory.class).debug("***WILL ADD to key "
						// + akey + group.getCentralGroupCode() + " desc " +
						// group.getCentralGroupDescription());
						thelist.add(registration);
					}

				}
			}

		}

		// order groups

		// receive group map and registrationassociation

		// iterate through registration get group
		// create CentralGroupRegistration
		// store in map

		// iterate groups
		// get CentralGroupRegistration

		/*
		 * groups = new HashMap(); if(grouplist==null) return;
		 * 
		 * //List keys = new ArrayList(); Iterator i = grouplist.iterator();
		 * 
		 * //Logger.getLogger(MenuFactory.class).debug("*** GROUP LIST SIZE " +
		 * grouplist.size());
		 * 
		 * while(i.hasNext()) { CentralGroupRegistration group =
		 * (CentralGroupRegistration)i.next(); int code =
		 * group.getMerchantRegistrationDefCode(); Integer akey = new
		 * Integer(code);
		 * //Logger.getLogger(MenuFactory.class).debug("*** CHECKING KEY " +
		 * akey); if(!groups.containsKey(akey)) {
		 * //Logger.getLogger(MenuFactory.class).debug("*** KEY " + akey +
		 * " does not exixt"); List newlist = new ArrayList();
		 * //Logger.getLogger(MenuFactory.class).debug("***WILL ADD to key " +
		 * akey + group.getCentralGroupCode() + " desc " +
		 * group.getCentralGroupDescription()); newlist.add(group);
		 * groups.put(akey, newlist); } else {
		 * //Logger.getLogger(MenuFactory.class).debug("*** KEY " + akey +
		 * " exixt"); List thelist = (List)groups.get(akey);
		 * //Logger.getLogger(MenuFactory.class).debug("***WILL ADD to key " +
		 * akey + group.getCentralGroupCode() + " desc " +
		 * group.getCentralGroupDescription()); thelist.add(group); } }
		 */
	}

	/**
	 * Set functions based on registration code
	 * 
	 * @param functions
	 */
	public void setFunctions(Collection functionsColl,
			Collection registrationList) {

		if (registrationList == null || registrationList.size() == 0)
			return;

		functions = new HashMap();

		try {
			CacheFactory cfactory = CacheFactory.getInstance();

			cfactory.removeCache("functionsurl");
			Map functionsurl = cfactory.createCacheMap("functionsurl");

			// Iterator i = registrationList.iterator();
			Iterator i = functionsColl.iterator();
			while (i.hasNext()) {
				// CentralRegistrationAssociation association =
				// (CentralRegistrationAssociation)i.next();

				CentralFunction f = (CentralFunction) i.next();

				Iterator regIterator = registrationList.iterator();
				while (regIterator.hasNext()) {
					CentralRegistrationAssociation association = (CentralRegistrationAssociation) regIterator
							.next();
					if (!StringUtils.isBlank(association
							.getCentralFunctionCode())
							&& association.getCentralFunctionCode().equals(
									f.getCentralFunctionCode())) {

						CentralFunctionRegistration registration = new CentralFunctionRegistration();

						registration.setCentralFunctionCode(association
								.getCentralFunctionCode());
						registration.setMerchantRegistrationDefCode(association
								.getMerchantRegistrationDefCode());
						registration.setPromotionCode(association
								.getPromotionCode());

						registration.setCentralFunctionDescription(f
								.getCentralFunctionDescription());
						registration.setCentralFunctionNewUntil(f
								.getCentralFunctionNewUntil());
						registration.setCentralFunctionNew(f
								.isCentralFunctionNew());
						registration.setCentralFunctionPosition(f
								.getCentralFunctionPosition());
						registration.setCentralFunctionVisible(f
								.isCentralFunctionVisible());
						registration.setCentralFunctionUrl(f
								.getCentralFunctionUrl());
						registration.setCentralGroupCode(f
								.getCentralGroupCode());
						registration.setRole(f.getRole());

						// CentralFunctionRegistration function =
						// (CentralFunctionRegistration)i.next();
						int code = registration
								.getMerchantRegistrationDefCode();
						int pcode = registration.getPromotionCode();
						String url = registration.getCentralFunctionUrl();

						AuthorizationCodes codes = (AuthorizationCodes) functionsurl
								.get(url);
						if (codes == null) {
							codes = new AuthorizationCodes(f
									.getCentralFunctionUrl());
							functionsurl.put(url, codes);
						}
						if (!codes.containsRegistrationCode(String
								.valueOf(code))) {
							codes.addRegistrationCode(String.valueOf(code));

						}
						if (pcode > 0) {
							if (!codes.containsPromotionCode(String
									.valueOf(pcode))) {
								codes.addPromotionCode(String.valueOf(pcode));

							}
						}
						Integer akey = new Integer(code);
						if (!functions.containsKey(akey)) {
							List newlist = new ArrayList();
							newlist.add(registration);
							functions.put(akey, newlist);
						} else {
							List thelist = (List) functions.get(akey);
							thelist.add(registration);
						}

					}

				}

			}

		} catch (Exception e) {
			Logger.getLogger(MenuFactory.class).error(e);
		}
	}

	public Map getFunctions() {
		if (functions == null) {
			functions = new HashMap();
		}
		return functions;
	}

}



```
