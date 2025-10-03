# CSVFileUploadImpl.java

## Review

## 1. Summary
`CSVFileUploadImpl` is a concrete implementation of the `IFileUploadService` interface that processes CSV files for three domain entities – **Category**, **Manufacturer** and **Product** – and persists them through a `CatalogService`.  
The class reads a CSV file using a `CSVFileReader`, validates the data row‑by‑row, constructs domain objects, and delegates persistence to the service layer. Validation errors are collected into a `Map<Integer, List<String>>` keyed by the CSV line number and returned to the caller.  

Key components  
| Component | Role |
|-----------|------|
| `CSVFileReader` | Parses the raw CSV file into a list of string lists (rows). |
| `CatalogService` | CRUD service for categories, manufacturers and products. |
| `RefCache` | In‑memory cache used to validate language IDs. |
| `CurrencyUtil` | Validates currency amount and symbol combinations. |
| `Log` | Provides debug/error logging. |

The code uses the **Data‑Transfer‑Object** pattern to build the domain objects and relies on **factory lookup** (`ServiceFactory.getService`) to obtain the `CatalogService`. No advanced design patterns are employed beyond basic encapsulation.

---

## 2. Detailed Description
### Flow of execution
1. **Service acquisition** – In every upload method the `CatalogService` is retrieved via `ServiceFactory`.  
2. **CSV parsing** – `fileReader.processCSV()` returns a `List<List<String>>` where each inner list represents a row.  
3. **Row validation & object construction** – For each row, the code:
   * Checks the number of columns.
   * Parses/validates key fields (`languageId`, `categoryId`, `manufacturerId`, etc.).
   * Builds the corresponding domain object (`Category`, `Manufacturers`, `Product`) and its description entities.
4. **Persistence** – The constructed objects are passed to the service layer (`uploadCategories`, `saveOrUpdateManufacturers`, `saveOrUpdateProduct`).  
5. **Error handling** – If any validation or persistence step fails, an error message is added to `errorMap`. The loop then continues with the next row.  
6. **Result** – After all rows have been processed, `errorMap` is returned.

### Assumptions & constraints
* The CSV files have a strict column ordering and count (5 for categories, 3 for manufacturers, 14 for products).  
* All numeric fields are expected to be integers, except weight/size fields that should be decimal but are parsed as integers (see issues below).  
* The `CatalogService` and `CSVFileReader` are assumed to be thread‑safe or used in a single‑threaded context; no synchronization is performed.  
* The `RefCache` must already be populated with language entries before the service runs.

### Architecture & design choices
* **Single responsibility** – The class focuses on CSV parsing and data validation, leaving persistence to the service layer.  
* **Static utility helpers** – `parseInt`, `addErrorMsg`, `isValidLanguage` are static, making them reusable but also less testable (no mocking).  
* **Error collection strategy** – Instead of throwing exceptions, the implementation aggregates all validation errors, enabling batch processing with feedback.  
* **Hard‑coded constants** – Validation messages and special values (`ROOT_CATEGORY_ID`) are defined as static finals.

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `uploadCategory(File, Integer)` | Reads a category CSV, validates rows, builds `Category` & `CategoryDescription` objects, persists via `catalogService.uploadCategories`. | `csvCategoryFile` – source file, `merchantId` – merchant identifier. | `Map<Integer, List<String>>` – line‑number keyed error messages. | Uses `catalogService` & `fileReader`; logs errors. |
| `uploadManufacturers(File)` | Similar to categories, but creates `Manufacturers` and `ManufacturersInfo`. | `csvManufacturersFile`. | Error map. | Persists with `catalogService.saveOrUpdateManufacturers`. |
| `uploadProducts(File, Integer)` | Processes product CSV; builds `Product` and `ProductDescription`; persists with `catalogService.saveOrUpdateProduct`. | `csvProductsFile`, `merchantId`. | Error map. | Uses `CurrencyUtil.validateCurrency`. |
| `parseInt(String)` | Safe integer parsing. Returns `0` on failure. | `val`. | `Integer`. | None. |
| `addErrorMsg(Map<Integer, List<String>>, Integer, String)` | Adds a message to the error map for a specific line. | `errorMap`, `lineNo`, `errorMsg`. | None. | Modifies the map. |
| `isValidLanguage(Integer)` | Checks if a language ID exists in the cache. | `langId`. | `boolean`. | None. |
| `getFileReader()` / `setFileReader(CSVFileReader)` | Bean accessors for `fileReader`. | – / `fileReader`. | `CSVFileReader` / void. | Setter mutates internal state. |

### Reusable / utility methods
* `parseInt` – Should return `null` for invalid input; current implementation hides errors by returning `0`.  
* `addErrorMsg` – Centralised error collection; could be extracted to a dedicated error‑handling helper.  
* `isValidLanguage` – Depends on `RefCache`; could be decoupled via an interface for better testability.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.logging.Log` | Third‑party logging | Simple wrapper; no issue. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Only used for `isNotBlank`; could replace with `StringUtils.isNotEmpty` from Apache Commons Lang 3. |
| `com.opensymphony.xwork2.validator.ValidationException` | Third‑party | Used when currency validation fails. |
| `com.salesmanager.core.*` | In‑house | Core entity, service, constants, cache, and utility classes. |
| `CSVFileReader` | In‑house | Parses CSV; not shown but assumed thread‑safe. |
| `ServiceFactory` | In‑house | Factory for service lookup; static global access. |
| `CatalogService` | In‑house | Business logic for persistence. |
| `CurrencyUtil` | In‑house | Validates amount/currency combinations. |

All external dependencies are either standard Java SE or widely used open‑source libraries. No platform‑specific APIs are present.

---

## 5. Additional Notes

### Potential Issues & Edge Cases
1. **`parseInt` Returning 0**  
   * All numeric fields rely on `parseInt`; an invalid numeric string (e.g., `"abc"`) silently becomes `0`.  
   * For mandatory IDs, this leads to incorrect “Invalid … Id” messages rather than a “missing or malformed value” error.  
   * **Fix:** Return `null` on `NumberFormatException` and handle `null` appropriately.

2. **Weight & Dimension Parsing**  
   * `new BigDecimal(parseInt(row.get(10)))` assumes integer values; real‑world data may contain decimals.  
   * **Fix:** Use `new BigDecimal(row.get(10))` or `new BigDecimal(Double.parseDouble(row.get(10)))` with proper validation.

3. **ProductDescription ID**  
   * `new ProductDescriptionId(0, languageId)` sets the product part of the composite key to `0`.  
   * The persistence layer may generate a real ID later; however, this can cause conflicts or orphan records if the service does not handle the `0` case.  
   * **Fix:** Let the `Product` entity generate its ID first, then set the `ProductDescription` ID accordingly.

4. **Parent Category Retrieval**  
   * Uses `StringUtils.isNotBlank(row.get(4))` to decide whether to parse a parent ID.  
   * If the CSV has an empty string, it falls back to the root ID; however, a missing field still counts as an empty string (size check passes).  
   * **Improvement:** Validate the length of `row` before accessing indices.

5. **Thread Safety**  
   * `fileReader` and `catalogService` are instance fields set via setter injection.  
   * In a multi‑threaded environment (e.g., servlet container), concurrent calls could corrupt shared state.  
   * **Fix:** Make the service stateless or synchronize access.

6. **Logging & Error Handling**  
   * The `IOException` caught during CSV parsing is logged but not re‑thrown, so callers receive an empty error map.  
   * **Fix:** Either rethrow a wrapped exception or include a global error in the map.

7. **Use of `Arrays.asList(new CategoryDescription[] { desc })`**  
   * Creating a single‑element array just to wrap in a list is unnecessary; `Collections.singletonList(desc)` is clearer.

8. **Missing `@Override` Annotations**  
   * The class implements `IFileUploadService` but none of the methods are annotated with `@Override`.  
   * While not harmful, it reduces readability and safety.

9. **Hard‑coded Validation Messages**  
   * Error strings are concatenated with data objects (`row`) leading to potentially noisy logs.  
   * Consider using parameterized messages or a dedicated error code system.

10. **Null Checks on `languageId`**  
    * After calling `parseInt`, the code checks `languageId == null`. Since `parseInt` never returns `null`, this check is ineffective.  
    * **Fix:** Update `parseInt` to return `null` on failure.

### Suggested Enhancements
* **Refactor Validation** – Extract validation logic into a separate validator class, enabling unit testing and reuse across other import mechanisms.  
* **DTO Layer** – Create dedicated DTOs for each CSV row to encapsulate parsing logic and reduce boilerplate.  
* **Batch Persistence** – Instead of persisting each row individually, accumulate objects and batch persist them to improve performance and reduce transaction overhead.  
* **Configuration‑Driven Mapping** – Allow the CSV column order/size to be configurable, making the importer adaptable to format changes without code changes.  
* **Unit Tests** – Provide tests for each upload method covering normal, boundary, and error cases.  
* **Logging Levels** – Use `DEBUG` for detailed row processing logs and `ERROR` for failures; avoid flooding the logs with success messages.  

Overall, the implementation achieves its core goal but would benefit from tighter validation, clearer error handling, and better separation of concerns to improve maintainability and robustness.

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
package com.salesmanager.core.util.file.csv;

import java.io.File;
import java.io.IOException;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Date;
import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.catalog.CategoryDescription;
import com.salesmanager.core.entity.catalog.CategoryDescriptionId;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductDescriptionId;
import com.salesmanager.core.entity.reference.Manufacturers;
import com.salesmanager.core.entity.reference.ManufacturersInfo;
import com.salesmanager.core.entity.reference.ManufacturersInfoId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.catalog.CatalogException;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.file.IFileUploadService;

public class CSVFileUploadImpl implements IFileUploadService {

	private static final Log log = LogFactory.getLog(CSVFileUploadImpl.class);
	private static final String INSUFF_DATA_STR = "Insufficient Data:";
	private static final String INVALID_LANGUAGE_ID = "Invalid Language Id:";
	private static final String INVALID_CATEGORY_ID = "Invalid Category Id:";
	public static final String INVALID_MANUFACTURER_ID = "Invalid Manufacturer Id:";
	public static final String INVALID_AMT_CURRENCY = "Invalid Amount or Currency:";
	public static final String UNABLE_TO_PROCESS = "Unable to Process:";
	private CatalogService catalogService;
	private CSVFileReader fileReader;

	public Map<Integer, List<String>> uploadCategory(File csvCategoryFile,
			Integer merchantId) throws CatalogException {
		Map<Integer, List<String>> errorMap = new LinkedHashMap<Integer, List<String>>();
		try {

			catalogService = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			List<List<String>> categoryRowList = fileReader
					.processCSV(csvCategoryFile);
			int lineNo = 0;
			for (List<String> row : categoryRowList) {
				// Language code|category id|category name|category
				// description|Parent Category id
				lineNo++;
				Integer languageId = parseInt(row.get(0));
				if (row.size() != 5) {
					addErrorMsg(errorMap, lineNo, INSUFF_DATA_STR + row);
					continue;
				}
				if (languageId == null || !isValidLanguage(languageId)) {
					addErrorMsg(errorMap, lineNo, INVALID_LANGUAGE_ID
							+ languageId);
					continue;
				}
				Integer categoryId = parseInt(row.get(1));
				if (categoryId == null || categoryId == 0) {
					addErrorMsg(errorMap, lineNo, INVALID_CATEGORY_ID
							+ categoryId);
					continue;
				}
				long parentCategoryId;
				if (StringUtils.isNotBlank(row.get(4))) {
					parentCategoryId = Long.valueOf(row.get(4));
				} else {
					parentCategoryId = new Long(
							CatalogConstants.ROOT_CATEGORY_ID);
				}

				Category parentCat = null;
				if (parentCategoryId != CatalogConstants.ROOT_CATEGORY_ID) {
					parentCat = catalogService.getCategory(parentCategoryId);
				}
				Category cat = new Category();
				cat.setParent(parentCat);
				cat.setParentId(parentCategoryId);
				cat.setCategoryId(categoryId);
				cat.setMerchantId(merchantId);
				cat.setLastModified(new Date());
				cat.setDateAdded(new Date());

				CategoryDescription desc = new CategoryDescription();
				desc.setCategoryDescription(row.get(3));
				desc.setCategoryName(row.get(2));
				desc.setId(new CategoryDescriptionId(categoryId, languageId));
				try {
					catalogService.uploadCategories(cat, Arrays
							.asList(new CategoryDescription[] { desc }));
				} catch (Exception e) {
					log.error("Error occurred while uploading Categor.", e);
					addErrorMsg(errorMap, lineNo, UNABLE_TO_PROCESS + row);
					continue;
				}
			}
		} catch (IOException e) {
			log.error("Error occurred while uploading Categ.", e);
		}
		return errorMap;
	}

	public Map<Integer, List<String>> uploadManufacturers(
			File csvManufacturersFile) {
		Map<Integer, List<String>> errorMap = new LinkedHashMap<Integer, List<String>>();
		try {
			catalogService = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			List<List<String>> manuRowList = fileReader
					.processCSV(csvManufacturersFile);
			int lineNo = 0;
			for (List<String> row : manuRowList) {
				lineNo++;
				if (row.size() != 3) {
					addErrorMsg(errorMap, lineNo, INSUFF_DATA_STR + row);
					continue;
				}
				Integer languageId = parseInt(row.get(0));
				if (languageId == null || !isValidLanguage(languageId)) {
					addErrorMsg(errorMap, lineNo, INVALID_LANGUAGE_ID
							+ languageId);
					continue;
				}
				Manufacturers manufacturers = new Manufacturers();
				Integer manufacturerId = parseInt(row.get(1));
				if (manufacturerId == null || manufacturerId == 0) {
					addErrorMsg(errorMap, lineNo, INVALID_MANUFACTURER_ID
							+ manufacturerId);
					continue;
				}
				manufacturers.setManufacturersId(manufacturerId);
				manufacturers.setManufacturersName(row.get(2));
				ManufacturersInfo manuInfo = new ManufacturersInfo();
				manuInfo.setId(new ManufacturersInfoId(manufacturerId,
						languageId));
				catalogService.saveOrUpdateManufacturers(manufacturers,
						manuInfo);
			}

		} catch (IOException e) {
			log.error("Error occurred while uploading Categ.", e);
		}
		return errorMap;
	}

	public Map<Integer, List<String>> uploadProducts(File csvProductsFile,
			Integer merchantId) {
		Map<Integer, List<String>> errorMap = new LinkedHashMap<Integer, List<String>>();
		try {
			catalogService = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			List<List<String>> prodowList = fileReader
					.processCSV(csvProductsFile);
			int lineNo = 0;
			for (List<String> row : prodowList) {
				lineNo++;
				if (row.size() != 14) {
					addErrorMsg(errorMap, lineNo, INSUFF_DATA_STR + row);
					continue;
				}
				Integer languageId = parseInt(row.get(0));
				if (languageId == null || !isValidLanguage(languageId)) {
					addErrorMsg(errorMap, lineNo, INVALID_LANGUAGE_ID
							+ languageId);
					continue;
				}
				Product product = new Product();
				product.setMerchantId(merchantId);
				product.setSku(row.get(1));
				product.setProductModel(row.get(4));
				product.setProductQuantity(parseInt(row.get(5)));
				try {
					product.setProductPrice(CurrencyUtil.validateCurrency(row
							.get(6), row.get(7)));
				} catch (ValidationException e) {
					addErrorMsg(errorMap, lineNo, INVALID_AMT_CURRENCY
							+ languageId);
					continue;
				}
				product.setProductManufacturersId(parseInt(row.get(8)));
				product.setProductVirtual(Boolean.parseBoolean(row.get(9)));
				product.setProductWeight(new BigDecimal(parseInt(row.get(10))));
				product.setProductHeight(new BigDecimal(parseInt(row.get(11))));
				product.setProductLength(new BigDecimal(parseInt(row.get(12))));
				product.setProductWidth(new BigDecimal(parseInt(row.get(13))));

				ProductDescription prodDesc = new ProductDescription();
				prodDesc.setProductName(row.get(2));
				prodDesc.setProductDescription(row.get(3));
				prodDesc.setId(new ProductDescriptionId(0, languageId));

				Set<ProductDescription> descSet = new HashSet<ProductDescription>();
				descSet.add(prodDesc);
				product.setDescriptions(descSet);
				try {
					catalogService.saveOrUpdateProduct(product);
				} catch (CatalogException e) {
					addErrorMsg(errorMap, lineNo, UNABLE_TO_PROCESS + row);
					continue;
				}
			}
		} catch (IOException e) {
			log.error("Error occurred while uploading Categori", e);
		}
		return errorMap;
	}

	public static Integer parseInt(String val) {
		try {
			return Integer.parseInt(val);
		} catch (NumberFormatException e) {
			return 0;
		}
	}

	public static void addErrorMsg(Map<Integer, List<String>> errorMap,
			Integer lineNo, String errorMsg) {
		if (errorMap.get(lineNo) == null) {
			errorMap.put(lineNo, new ArrayList<String>());

		}
		errorMap.get(lineNo).add(errorMsg);
	}

	public static boolean isValidLanguage(Integer langId) {
		return (RefCache.getLanguageswithindex().get(langId) != null);
	}

	public CSVFileReader getFileReader() {
		return fileReader;
	}

	public void setFileReader(CSVFileReader fileReader) {
		this.fileReader = fileReader;
	}

}



```
