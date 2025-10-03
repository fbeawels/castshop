# SiteMapAction.java

## Review

## 1. Summary
**Purpose**  
`SiteMapAction` is a Struts‑style action that generates XML sitemaps for a multi‑language e‑commerce store.  
It gathers three types of content – pages, products and categories – for every supported language, writes one sitemap file per content type per language and finally creates a sitemap index file.  The resulting sitemap URL is persisted in the merchant configuration.

**Key components**  
| Component | Responsibility |
|-----------|----------------|
| `MerchantService`, `CatalogService`, `ReferenceService` | Retrieve store, catalog and reference data. |
| `WebSitemapGenerator` / `SitemapIndexGenerator` (redfin‑sitemap) | Generate individual sitemap files and the index file. |
| `FileUtil`, `UrlUtil`, `ReferenceUtil` | Build file paths, URLs and catalog URIs. |
| `MerchantConfiguration` | Persist the sitemap URL for later use. |

**Design patterns / libraries**  
* **Service locator / factory** – `ServiceFactory.getService()` is used to obtain services.  
* **Builder pattern** – `WebSitemapGenerator.builder()` creates a sitemap instance.  
* **External library** – *redfin‑sitemap* handles the actual XML generation.

---

## 2. Detailed Description
### Execution Flow
1. **Service acquisition** – Obtain `MerchantService`, `CatalogService` and `ReferenceService`.  
2. **Collections initialisation** – Three maps (`categoriesMap`, `productsMap`, `pagesMap`) store language‑specific collections.  
3. **Merchant store lookup** – Fetch the current merchant store using the ID from the request context.  
4. **Per‑language processing**  
   * Retrieve supported languages (`langs`).  
   * For each language:  
     * Get categories, products and dynamic pages.  
     * Store the results in the corresponding map using the language ID as key.  
5. **Sitemap directory setup**  
   * Build the base URL, sitemap URL and directory paths.  
   * Create (or recreate) the directory that will hold individual sitemap files.  
6. **Generate sitemaps** – For each content type and for each language:  
   * Instantiate a `WebSitemapGenerator` with a file‑name prefix that encodes the content type and language.  
   * Add URLs (capped at 50 000 per file).  
   * Write the file and add its URL to the `SitemapIndexGenerator`.  
7. **Write sitemap index** – Persist the index file and store the final sitemap URL in the merchant configuration.  
8. **Return success** – Indicate that the action finished without errors.

### Assumptions & Constraints
* The application runs in a servlet container with access to the file system (sitemap files are written to disk).  
* `ServiceFactory` returns fully initialised services.  
* The merchant store and its languages are already configured.  
* The sitemap index generator writes a single `sitemap.xml` file containing up to 50 000 URLs per sub‑file.  
* No concurrency is considered – the action is assumed to be invoked serially.

### Architectural Observations
* The action mixes **business logic** (retrieving data) with **I/O logic** (file creation) and **configuration persistence**.  
* Raw collections and type‑unsafe casts are pervasive; generics are only used in a few places.  
* Error handling is minimal – any `Exception` bubbles out of `execute()`, but the action never reports a meaningful error to the user.  

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `execute()` | Main entry point for the action. Generates sitemaps and updates configuration. | None | `String` (`SUCCESS`) | Writes files to disk, updates `MerchantConfiguration`, sets success message. |
| `main()` | Not present. |

### Sub‑operations inside `execute()`
* **Service acquisition** – no separate method.  
* **Data collection** – performed in a loop; no dedicated method.  
* **Directory handling** – performed inline (creation / deletion).  
* **Sitemap generation** – repeated three times (pages, products, categories).  
* **Index writing** – performed once at the end.

> **Note**: Because all logic resides in a single method, unit testing is difficult and the code is hard to maintain.

---

## 4. Dependencies
| Dependency | Type | Purpose |
|------------|------|---------|
| `com.redfin.sitemapgenerator.WebSitemapGenerator` | Third‑party | Generates individual sitemap XML files. |
| `com.redfin.sitemapgenerator.SitemapIndexGenerator` | Third‑party | Builds the sitemap index file. |
| `com.salesmanager.core.*` | Third‑party | Core business services and entities (merchant, catalog, reference). |
| `java.io.File`, `java.util.*`, `java.util.Date` | Standard | File handling, collections, dates. |
| `com.salesmanager.core.util.*` | Third‑party | File, URL, date, reference utilities. |
| `com.salesmanager.central.BaseAction` | Internal | Base Struts‑like action providing context, messages, etc. |

There are no platform‑specific dependencies beyond the servlet container (e.g., `HttpServletRequest` via `getServletRequest()`).

---

## 5. Additional Notes & Recommendations
### 5.1 Code Quality Issues
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types & unchecked casts** – e.g., `Map<Integer,Collection> categoriesMap = new HashMap();` | Compile‑time warnings, potential `ClassCastException`. | Use generics everywhere (`Map<Integer, List<Category>>`). |
| **Inefficient directory deletion** – `sm.delete()` on a directory only deletes the folder if it’s empty. | Sitemap files may remain, causing stale data. | Recursively delete files or use `FileUtils.deleteDirectory(sm)` from Apache Commons IO. |
| **Missing `mkdirs()`** – Only creates one level; intermediate directories might be missing. | File creation failures. | Call `sm.mkdirs()`. |
| **Hard‑coded 50 000 limit** – `if(i>50000) break;` | No handling for more than 50 000 items – potential lost URLs. | Generate multiple files per language/type or use redfin’s automatic splitting. |
| **No error handling** – All exceptions propagate to the caller. | Users receive generic errors; no cleanup performed. | Wrap critical blocks in try/catch, log errors, and provide meaningful messages. |
| **Repetitive code** – Three almost identical loops for pages, products, categories. | Hard to maintain. | Extract a private helper (`generateSitemapForCollection`). |
| **Directory name typo** – `fileNamePrefix("paages_")`. | Human readability suffers. | Correct to `"pages_"`. |
| **Locale usage** – `super.getLocale()` used only for page lookup. | Potential mismatch if store locale differs. | Ensure consistent locale handling. |
| **Unnecessary `new StringBuilder()` for simple concatenation** – e.g., `new StringBuilder().append(baseUrl).append(...).toString()`. | Minor performance overhead. | Use string concatenation (`baseUrl + "/content/" + ...`). |

### 5.2 Edge Cases
* **No supported languages** – The method silently returns without generating any files.  
* **Empty collections** – Index file may be created but contain no URLs; still persisted.  
* **Large product catalog** – Only first 50 000 URLs per language are written; the rest are ignored.  
* **Directory already exists with files** – `sm.delete()` will fail; leftover files could be overwritten incorrectly.  

### 5.3 Future Enhancements
1. **Refactor into smaller, testable methods** – e.g., `collectContent()`, `createSitemapForCollection()`, `writeSitemapIndex()`.  
2. **Use a templated approach** – Let the `redfin` library handle 50 000 URL splitting automatically.  
3. **Persist the entire sitemap path list** – Instead of a single `sitemap.xml` value, store an array of URLs or use a dedicated configuration entity.  
4. **Add progress logging** – Especially for large catalogs.  
5. **Support compressed sitemaps (`.gz`/`.bz2`)** – Reduce bandwidth for search engines.  
6. **Unit tests** – With a mock service layer and a temporary file system (e.g., `java.nio.file.Files.createTempDirectory`).  

### 5.4 Security & Performance
* **File path handling** – Ensure `siteMapDir` cannot be influenced by user input to avoid path traversal.  
* **Concurrency** – If multiple threads trigger sitemap generation, race conditions on the same directory may occur.  
* **Caching** – For large catalogs, regenerate sitemaps only on content changes.  

---

**Conclusion**  
`SiteMapAction` performs its core function of generating sitemaps but suffers from several design, safety, and maintainability problems. Addressing the issues above will make the code more robust, easier to test, and scalable for larger stores.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2011 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.cart;



import java.io.File;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import com.redfin.sitemapgenerator.SitemapIndexGenerator;
import com.redfin.sitemapgenerator.WebSitemapGenerator;
import com.salesmanager.central.BaseAction;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.UrlUtil;


/**
 * This class manages SiteMap file
 * @author Carl Samson
 *
 */
public class SiteMapAction extends BaseAction {
	
	
	/**
	 * Creates a sitemap
	 */
	public String execute() throws Exception {
		
		MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
		CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		Map<Integer,Collection> categoriesMap = new HashMap();
		Map<Integer,Collection<Product>> productsMap = new HashMap();
		Map<Integer,Collection<DynamicLabel>> pagesMap = new HashMap();
		
		 
		
		
		//for each supported language
		MerchantStore store = mservice.getMerchantStore(super.getContext().getMerchantid().intValue());
		
		
		Map langs = store.getGetSupportedLanguages();
		
		if(langs==null) {
			//@todo return a message
		}
		
		for(Object o : langs.keySet()) {
			
			Integer lang = (Integer)o;
			Language l = (Language)langs.get(lang);
			
			//get all categories
			Map categories = cservice.getCategoriesByLang(store.getMerchantId(),l.getCode());
			if(categories!=null && categories.size()>0) {
				
				
				List<Object> list = new ArrayList<Object>(categories.entrySet());

				
				categoriesMap.put(l.getLanguageId(), list);
			}
			
			//get all products
			Collection products = cservice.getProductsByMerchantIdAndLanguageId(store.getMerchantId(),l.getLanguageId());
			if(products!=null && products.size()>0) {
				productsMap.put(l.getLanguageId(), products);
			}
			
			//get pages
			Collection pages = rservice.getDynamicLabels(store.getMerchantId(),LabelConstants.STORE_FRONT_CUSTOM_PAGES,super.getLocale());
			if(pages!=null && pages.size()>0) {
				pagesMap.put(l.getLanguageId(), pages);
			}
			
		}
		
		
		WebSitemapGenerator wsg; 
		// generate pages
		
		//urls
		String baseUrl = FileUtil.getDefaultCataloguePageUrl(store, super.getServletRequest());
		String siteMapUrl = new StringBuilder().append(UrlUtil.getUnsecuredDomain(super.getServletRequest())).append(FileUtil.getSiteMapUrl()).append(store.getMerchantId()).toString();

		//dirs
		String siteMapDir = new StringBuilder().append(FileUtil.getSiteMapFilePath()).append("/").append(store.getMerchantId()).toString();
		String index_file = new StringBuilder().append(FileUtil.getSiteMapFilePath()).append("/").append(store.getMerchantId()).append("/").append("sitemap.xml").toString();
		
		
		SitemapIndexGenerator sig = new SitemapIndexGenerator(siteMapUrl, new File(index_file));
		
		if(pagesMap.size()>0 || productsMap.size()>0 || categoriesMap.size()>0) {
			
			//check if folder exists
			File sm = new File(siteMapDir);
			boolean exists = sm.exists();
			if(exists) {//delete
				sm.delete();
			}
			boolean mkdir = sm.mkdir();
			
		}
		
		if(pagesMap.size()>0) {
			 
			for(Object o : langs.keySet()) {//languages
					Integer languageId = (Integer)o;
				
					Collection pages = (Collection)pagesMap.get(languageId);
				
					if(pages!=null && pages.size()>0) {
					
					wsg = WebSitemapGenerator.builder(baseUrl,new File(siteMapDir)).fileNamePrefix("paages_" + languageId).build();
					
					for(Object oo : pages) {
						
						DynamicLabel l = (DynamicLabel)oo;
						wsg.addUrl(new StringBuilder().append(baseUrl).append("/content/").append(l.getDynamicLabelDescription().getSeUrl()).toString());
					}
					wsg.write(); // generate pages sitemap 
					sig.addUrl(siteMapUrl + "/paages_" + languageId +  ".xml");
				
				}
			}
		}
		
		
		if(productsMap.size()>0) {
			 
			for(Object o : langs.keySet()) {//languages
				Integer languageId = (Integer)o;
				
				Collection products = (Collection)productsMap.get(languageId);
				
					if(products!=null && products.size()>0) {
					
					wsg = WebSitemapGenerator.builder(baseUrl,new File(siteMapDir)).fileNamePrefix("products_" + languageId).build();
					
					int i = 1;
					for(Object oo : products) {
						if(i>50000) {break;}
						Product p = (Product)oo;
						wsg.addUrl(new StringBuilder().append(ReferenceUtil.buildCatalogUri(store)).append("/product/").append(p.getProductDescription().getSeUrl()).toString());
						i++;
					}
					wsg.write(); // generate pages sitemap 
					sig.addUrl(siteMapUrl + "/products_" + languageId +  ".xml");
				
				}
			}
			
		}
		
		if(categoriesMap.size()>0) {
			
			
			for(Object o : langs.keySet()) {//languages 
				Integer languageId = (Integer)o;
				
				Collection categories = (Collection)categoriesMap.get(languageId);
				if(categories!=null && categories.size()>0) {
					wsg = WebSitemapGenerator.builder(baseUrl,new File(siteMapDir)).fileNamePrefix("categories_"+languageId).build();
					
					int i = 1;
					for(Object oo : categories) {
						if(i>50000) {break;}
						Map.Entry entry = (Map.Entry)oo;
						Category c = (Category)entry.getValue();
						if(c.getCategoryId()>0) {
							wsg.addUrl(new StringBuilder().append(ReferenceUtil.buildCatalogUri(store)).append("/category/").append(c.getCategoryDescription().getSeUrl()).toString());
						}
						i++;
					}
					wsg.write(); // generate pages sitemap
					sig.addUrl(siteMapUrl + "/categories_" + languageId +  ".xml");
				}
			}
			
		}
		
		 
		
		//for (int i = 0; i &lt; 5; i++) 
		//	wsg.addUrl("http://www.example.com/foo"+i+".html"); 
		
		
		
		
		//wsg = WebSitemapGenerator.builder("http://www.example.com", myDir).fileNamePrefix("bar").build(); 
		//for (int i = 0; i &lt; 5; i++) 
		//	wsg.addUrl("http://www.example.com/bar"+i+".html"); 
		//wsg.write(); // generate sitemap index for foo + bar  
		

		

		
		if(pagesMap.size()>0 || productsMap.size()>0 || categoriesMap.size()>0) {
		
			sig.write();
			
			//write in configuration
			ConfigurationRequest req = new ConfigurationRequest(store.getMerchantId(),ConfigurationConstants.SITEMAP);
			ConfigurationResponse resp = mservice.getConfiguration(req);
			MerchantConfiguration conf = resp.getMerchantConfiguration(ConfigurationConstants.SITEMAP);
			
			if(conf==null) {
				conf = new MerchantConfiguration();
			}
			
			conf.setConfigurationKey(ConfigurationConstants.SITEMAP);
			conf.setMerchantId(store.getMerchantId());
			conf.setConfigurationValue(siteMapUrl + "/sitemap.xml");
			conf.setConfigurationValue1(DateUtil.formatDate(new Date()));
			
			mservice.saveOrUpdateMerchantConfiguration(conf);
	
			
			super.setSuccessMessage();
		
		}
		
		return SUCCESS;
		
	}

}



```
