# CacheFactory.java

## Review

## 1. Summary  
**Purpose & Core Functionality**  
`CacheFactory` is a lightweight, singleton factory that manages a set of named caches. Each cache is either a `Map` or a `List`, both wrapped in a thread‑safe wrapper (`Collections.synchronizedMap` / `synchronizedList`). The factory keeps a central registry (`cachesm`) that maps a cache name to the corresponding collection instance.  

**Key Components**  
- **Singleton pattern** – ensures a single instance of `CacheFactory` exists in the JVM.  
- **Factory methods** – `createCacheMap`, `createCacheList` instantiate a cache if one with the same name does not already exist.  
- **Lookup & removal** – `getCacheMap`, `getCacheList`, `containsCache`, `removeCache`.  

**Design Patterns / Libraries**  
- Singleton (explicit, non‑thread‑safe implementation).  
- Factory (provides a factory interface for cache creation).  
- Uses only standard Java collections (`java.util.*`).

---

## 2. Detailed Description  
1. **Singleton Initialization**  
   ```java
   public static CacheFactory getInstance() {
       if (cachefactory == null) {
           cachefactory = new CacheFactory();
       }
       return cachefactory;
   }
   ```  
   The first call creates the instance; subsequent calls return the same object.  
   **Assumption**: Single‑threaded initialization. In a multi‑threaded environment this can lead to multiple instances.

2. **Central Registry (`cachesm`)**  
   ```java
   private static Map cachesm = Collections.synchronizedMap(new HashMap());
   ```  
   A synchronized `HashMap` holds all named caches. All cache collections themselves are also synchronized wrappers, so every cache is thread‑safe.

3. **Creation Methods**  
   - `createCacheMap(String name)`  
     *If the name already exists, the existing cache is returned.*  
     Otherwise a new synchronized `HashMap` is created, stored in `cachesm`, and returned.  
   - `createCacheList(String name)` – analogous but uses `ArrayList` wrapped with `synchronizedList`.

4. **Lookup Methods**  
   `getCacheMap` / `getCacheList` simply return the object stored under the given name (or `null` if absent). No type safety guarantees.

5. **Utility Methods**  
   - `containsCache(String name)` checks for existence.  
   - `removeCache(String name)` removes the entry from the registry.

**Execution Flow**  
- The first client calls `CacheFactory.getInstance()` → singleton created.  
- Client requests a cache: `createCacheMap("orders")` → new map created & stored.  
- Subsequent requests for `"orders"` receive the same map instance.  
- All cache operations are synchronized at the collection level.  
- No automatic cleanup or eviction logic – caches remain until `removeCache` is invoked.

**Constraints & Dependencies**  
- Requires a single JVM process; does not support distributed caching.  
- Uses only Java SE collections; no external libraries.  
- Relies on Java's `synchronized` wrappers for thread safety.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects / Notes |
|--------|---------|------------|---------|----------------------|
| `private CacheFactory()` | Private constructor to enforce singleton. | – | – | – |
| `public static CacheFactory getInstance()` | Retrieve singleton instance. | – | `CacheFactory` | Non‑thread‑safe; may create multiple instances in concurrent init. |
| `public Map createCacheMap(String name)` | Create (or fetch existing) synchronized `Map`. | `String name` | `Map` | Returns existing map if name already used; silently ignores duplicate creation. |
| `public List createCacheList(String name)` | Create (or fetch existing) synchronized `List`. | `String name` | `List` | Same semantics as `createCacheMap`. |
| `public Map getCacheMap(String name)` | Retrieve map by name. | `String name` | `Map` | Returns `null` if not found. |
| `public List getCacheList(String name)` | Retrieve list by name. | `String name` | `List` | Returns `null` if not found. |
| `public boolean containsCache(String name)` | Check existence. | `String name` | `boolean` | – |
| `public void removeCache(String name)` | Remove cache entry. | `String name` | `void` | Removes the reference; underlying collection is eligible for GC. |

*Reusable/Utility Methods*: None explicitly, but `createCacheMap` and `createCacheList` are essentially the same logic, just with different collection types.

---

## 4. Dependencies  
- **Standard Java API**  
  - `java.util.Map`, `java.util.HashMap`, `java.util.Collections`, `java.util.List`, `java.util.ArrayList`  
  - No third‑party libraries or frameworks.  
- **Platform**: Works on any Java SE environment (Java 5+ due to generics).

---

## 5. Additional Notes & Recommendations  

### Edge Cases & Potential Issues  
1. **Thread‑safety of Singleton**  
   - `getInstance()` is not double‑checked locked; concurrent calls can spawn multiple instances, breaking the singleton contract.  
2. **Raw Types & Type Safety**  
   - All maps/lists are raw (`Map`, `List`). Clients risk `ClassCastException`.  
3. **Duplicate Cache Names**  
   - The current implementation silently returns the existing cache when a duplicate name is requested. This might hide bugs; consider throwing an exception or returning a flag.  
4. **Memory Leaks**  
   - Caches are never evicted; if many caches are created, memory consumption can grow unbounded.  
5. **No Eviction / Expiration**  
   - No support for time‑based or size‑based cache eviction; useful in real‑world caching scenarios.  
6. **Central Registry Synchronization**  
   - `cachesm` is a synchronized map, but the create methods still perform a `containsKey` followed by `put` which is not atomic. In a concurrent scenario two threads could create two different instances for the same name.

### Suggested Enhancements  
1. **Thread‑safe Singleton**  
   ```java
   private static volatile CacheFactory instance;

   public static CacheFactory getInstance() {
       if (instance == null) {
           synchronized (CacheFactory.class) {
               if (instance == null) {
                   instance = new CacheFactory();
               }
           }
       }
       return instance;
   }
   ```  
2. **Use Generics**  
   ```java
   public <K,V> Map<K,V> createCacheMap(String name) { … }
   public <E> List<E> createCacheList(String name) { … }
   ```  
   Store the collections in a generic wrapper or a custom `Cache<T>` class to maintain type safety.  

3. **Atomic Cache Creation**  
   Replace the two‑step `containsKey` / `put` with `computeIfAbsent` (Java 8+):
   ```java
   cachesm.computeIfAbsent(name, k -> Collections.synchronizedMap(new HashMap()));
   ```

4. **Eviction Policy**  
   Consider integrating with a lightweight caching library (e.g., Caffeine or Guava’s `CacheBuilder`) if expiration or size limits are required.

5. **Configuration & Namespacing**  
   Support cache configuration (e.g., time‑to‑live, maximum size) and optional namespaces to avoid accidental name collisions.

6. **Unit Tests**  
   Add tests that cover concurrent cache creation, retrieval, and removal to verify thread safety.

7. **Documentation**  
   Update Javadoc to clarify return types, thread‑safety guarantees, and potential pitfalls.

---

### Conclusion  
`CacheFactory` provides a minimal, thread‑safe (via synchronized wrappers) caching façade suitable for simple use cases. However, the lack of proper singleton implementation, raw types, and missing eviction logic limit its applicability in production‑grade systems. The code is clean and straightforward, but refactoring with generics, proper concurrency handling, and optional eviction policies would greatly enhance its robustness and usability.

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
package com.salesmanager.central.util;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Manage the cache
 * 
 * @author Carl Samson
 * 
 */

public class CacheFactory {

	private static CacheFactory cachefactory = null;
	private static Map cachesm = Collections.synchronizedMap(new HashMap());

	private CacheFactory() {
	}

	public static CacheFactory getInstance() {
		if (cachefactory == null) {
			cachefactory = new CacheFactory();
		}
		return cachefactory;
	}

	/**
	 * Create a synchronized HashMap for caching
	 * 
	 * @param name
	 * @return
	 * @throws Exception
	 */
	public Map createCacheMap(String name) {
		if (cachesm.containsKey(name)) {
			// throw new Exception("Cache name " + name + " already exist");
			return (Map) cachesm.get(name);
		}

		Map newcache = Collections.synchronizedMap(new HashMap());
		cachesm.put(name, newcache);
		return newcache;

	}

	/**
	 * Create a synchronized List for caching
	 * 
	 * @param name
	 * @return
	 * @throws Exception
	 */
	public List createCacheList(String name) {
		if (cachesm.containsKey(name)) {
			// throw new Exception("Cache name " + name + " already exist");
			return (List) cachesm.get(name);
		}

		List newcache = Collections.synchronizedList(new ArrayList());
		cachesm.put(name, newcache);
		return newcache;

	}

	public Map getCacheMap(String name) {
		Map returnmap = (Map) cachesm.get(name);
		return returnmap;
	}

	public List getCacheList(String name) {
		List returnlist = (List) cachesm.get(name);
		return returnlist;
	}

	public boolean containsCache(String name) {
		return cachesm.containsKey(name);
	}

	public void removeCache(String name) {
		cachesm.remove(name);
	}

}



```
