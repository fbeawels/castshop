# CacheUtil.java

## Review

## 1. Summary  
`CacheUtil` is a lightweight, thread‑susceptible singleton that offers a central repository for per‑named caches.  
- **Core purpose**: Provide a single point where callers can request a named cache that is either a `Map` or a `List`.  
- **Key components**:  
  - `cachefactory`: the singleton instance.  
  - `cachesm`: a synchronized `HashMap` that holds the actual named caches.  
  - Factory methods (`createCacheMap`, `createCacheList`) that lazily create and store synchronized collections.  
  - Retrieval, inspection and removal helpers (`getCacheMap`, `getCacheList`, `containsCache`, `removeCache`).  
- **Design patterns / libraries**: Implements a *singleton* pattern, uses Java’s `Collections.synchronized*` wrappers for thread safety, and relies exclusively on the JDK (`java.util`).  

## 2. Detailed Description  
1. **Initialization**  
   - The static `cachesm` map is initialized once at class load time, wrapped with `Collections.synchronizedMap`.  
   - The singleton `cachefactory` starts as `null`.  

2. **Runtime behaviour**  
   - `getInstance()` lazily creates the singleton instance.  
   - `createCacheMap(String name)` / `createCacheList(String name)`  
     - Check if the named cache already exists in `cachesm`.  
     - If not, instantiate a synchronized `HashMap` or `ArrayList`, store it, and return it.  
   - Retrieval methods simply pull the value from `cachesm`.  
   - `containsCache` and `removeCache` are straightforward delegates to `cachesm`.  

3. **Cleanup**  
   - No explicit cleanup beyond `removeCache`; the singleton never releases the underlying `cachesm`.  

4. **Assumptions & constraints**  
   - Cache names are unique per type; the code does not enforce a distinction between map and list caches – a name can map to a `Map` **or** a `List`.  
   - All methods accept a `String` name; no null‑check or validation.  
   - Raw types are used (`Map`, `List`) without generics.  

5. **Architecture & design choices**  
   - Simplicity: the implementation relies on synchronized wrappers rather than more sophisticated concurrent collections.  
   - The singleton pattern ensures a single cache store but is not thread‑safe in its current form.  

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑effects | Notes |
|--------|-----------|---------|--------|---------|--------------|-------|
| `getInstance()` | `static CacheUtil` | Return the singleton instance. | none | `CacheUtil` | Lazy instantiation | Not thread‑safe; could return multiple instances under concurrent access. |
| `createCacheMap(String name)` | `Map` | Lazily create/retrieve a named `Map`. | `name` | `Map` | Stores new map in `cachesm` if absent. | Returns raw `Map`; type safety lost. |
| `createCacheList(String name)` | `List` | Lazily create/retrieve a named `List`. | `name` | `List` | Stores new list in `cachesm` if absent. | Returns raw `List`. |
| `getCacheMap(String name)` | `Map` | Retrieve existing named map. | `name` | `Map` or `null` | None | If `name` points to a `List`, casting will fail at runtime. |
| `getCacheList(String name)` | `List` | Retrieve existing named list. | `name` | `List` or `null` | None | Same type‑casting risk. |
| `containsCache(String name)` | `boolean` | Check existence of a named cache. | `name` | `true/false` | None | |
| `removeCache(String name)` | `void` | Delete named cache. | `name` | none | Removes entry from `cachesm`. | |

**Reusable/utility methods**: None beyond the core factory and retrieval logic.  

## 4. Dependencies  
- **Standard library only**  
  - `java.util.ArrayList`, `HashMap`, `List`, `Map`, `Collections`  
- No external frameworks, APIs or platform‑specific code.  

## 5. Additional Notes  

### Edge cases / Potential Issues  
1. **Thread‑safety of the singleton**  
   - `getInstance()` is not synchronized; two threads can concurrently create two instances.  
   - The underlying `cachesm` is a synchronized map, but the creation of a new cache for the same name is not atomic. Two threads could each create a different `Map` or `List` for the same key, with one overwriting the other.  

2. **Type safety**  
   - Raw types (`Map`, `List`) mean callers must cast and risk `ClassCastException`.  
   - Using the same name for a map and a list leads to runtime failures when retrieving with the wrong method.  

3. **Null handling**  
   - Passing `null` for `name` will lead to a `NullPointerException` when interacting with the map.  

4. **Memory leaks**  
   - Caches are never automatically evicted or time‑out; they persist until `removeCache` is called.  

### Suggested Enhancements  
- **Thread‑safe singleton**: use an `enum` singleton or `private static final CacheUtil INSTANCE = new CacheUtil();` (eager initialization) or double‑checked locking with `volatile`.  
- **Concurrent collections**: replace `Collections.synchronizedMap`/`List` with `ConcurrentHashMap` and `CopyOnWriteArrayList` or `ConcurrentLinkedDeque` for better performance.  
- **Generics**: expose type‑parameterized methods (`<K,V> Map<K,V> createCacheMap(String name)`), or create a `Cache<K,V>` abstraction.  
- **Separate namespace**: maintain distinct maps for map‑caches and list‑caches to avoid name clashes.  
- **Null‑check & validation**: validate `name` to prevent accidental `null` keys.  
- **Eviction policy**: integrate a time‑based or size‑based eviction mechanism (e.g., Guava’s `CacheBuilder` or `java.util.concurrent.ConcurrentMap` with custom logic).  
- **Documentation & naming**: better JavaDoc comments and clearer method names (e.g., `getOrCreateMapCache`).  

Overall, `CacheUtil` provides a minimal, easy‑to‑understand caching façade, but its current implementation is fragile in multi‑threaded environments and lacks type safety. Addressing the above concerns would make it robust, maintainable, and ready for production use.

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

public class CacheUtil {

	private static CacheUtil cachefactory = null;
	private static Map cachesm = Collections.synchronizedMap(new HashMap());

	private CacheUtil() {
	}

	public static CacheUtil getInstance() {
		if (cachefactory == null) {
			cachefactory = new CacheUtil();
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
