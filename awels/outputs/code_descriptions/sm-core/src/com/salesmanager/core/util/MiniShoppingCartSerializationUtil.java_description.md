# MiniShoppingCartSerializationUtil.java

## Review

## 1. Summary  

The **MiniShoppingCartSerializationUtil** is a helper class that serializes a `ShoppingCart` into a compact JSON string and deserializes the same string back into a fully‑populated `ShoppingCart`.  
* **Serialization** builds a JSON representation of the cart’s products, quantities, and any selected attributes.  
* **Deserialization** parses the JSON, recreates `ShoppingCartProduct` objects, and enriches them with product and attribute data retrieved from the catalog service.  

The class relies on Jackson 1.x (`org.codehaus.jackson.map.ObjectMapper`) for JSON parsing and on several internal services (`CatalogService`, `ServiceFactory`) for database look‑ups.  It uses raw collections and manual JSON string construction instead of Jackson’s object mapping capabilities.

---

## 2. Detailed Description  

### High‑level Flow

| Step | What happens | Where it happens |
|------|--------------|------------------|
| **Input** | A JSON string (typically stored in a cookie) | `deserializeJSON(String, MerchantStore, Locale)` |
| **Parsing** | Jackson parses the string into a raw `Map<String, Object>` | `ObjectMapper.readValue(...)` |
| **Product extraction** | Iterates over the “ps” list, creating `ShoppingCartProduct` instances and their `ShoppingCartProductAttribute` lists | `for (Object o : data.keySet()) …` |
| **Catalog lookup** | Collects all product IDs and attribute IDs, fetches the corresponding `Product` and `ProductAttribute` objects from `CatalogService` | `cservice.getProducts(...)`, `cservice.getProductAttributes(...)` |
| **Re‑assembly** | Builds a new `ShoppingCart`, attaches product details (image, name, price, attributes) and sets the final list of `ShoppingCartProduct` objects | `cart = new ShoppingCart(); … cart.setProducts(shoppingCartProducts);` |
| **Return** | The fully constructed `ShoppingCart` (or `null` if input was blank) | `return cart;` |

The **serialization** side simply walks the cart’s product collection and appends JSON fragments to a `StringBuilder`.

### Design Choices

* **Manual JSON construction** – The author chose to hand‑craft the JSON string rather than using Jackson’s `ObjectMapper.writeValueAsString()`.  
* **Use of raw collections** – All lists/maps are untyped (`List`, `Map`). This simplifies code but loses compile‑time type safety.  
* **Static utility** – All operations are static; no instance state is stored, so the class is thread‑safe as long as the called services are.  
* **Service look‑ups in deserialization** – The method pulls the complete product and attribute data from the catalog, ensuring that the cart contains the most up‑to‑date pricing and product information.  

### Assumptions & Constraints

* The JSON format is rigid (`{"ps":[{...}]}`) and fields are mandatory (`pid`, `q`, optional `as`).  
* All numeric values are passed as strings in the JSON.  
* The caller provides a non‑null `MerchantStore` and `Locale`.  
* The catalog service is available and can handle bulk queries (`getProducts`, `getProductAttributes`).  
* The environment contains the following external libraries: Jackson 1.x, Apache Commons Lang‑xwork, and the SalesManager core service API.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| **`deserializeJSON(String json, MerchantStore store, Locale locale)`** | Converts a JSON string from a cookie into a fully‑populated `ShoppingCart`. | *json* – raw JSON string; *store* – merchant store context; *locale* – language/currency context. | `ShoppingCart` (or `null` if input blank). | *None* – purely functional. |
| **`serializeToJSON(ShoppingCart shoppingCart)`** | Builds a compact JSON string that represents the cart’s items, quantities, and attributes. | *shoppingCart* – the cart to serialize. | `String` (or `null` if cart empty). | *None* – purely functional. |

### Internal Helpers (inlined)

* Extraction of `ShoppingCartProduct` and `ShoppingCartProductAttribute` objects from the parsed JSON map.  
* Bulk fetching of `Product` and `ProductAttribute` instances from the catalog.  
* Price calculation using `ProductUtil` and `CurrencyUtil`.  

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| `org.codehaus.jackson.map.ObjectMapper` | Third‑party | Jackson 1.x (old). |
| `org.apache.commons.lang.xwork.StringUtils` | Third‑party | Legacy version of Apache Commons Lang. |
| `com.salesmanager.core.service.*` | Internal | Catalog service, service factory, etc. |
| `com.salesmanager.core.entity.*` | Internal | Domain entities (Product, ProductAttribute, ShoppingCart, …). |
| `com.salesmanager.core.util.*` | Internal | `LocaleUtil`, `ProductUtil`, `CurrencyUtil` (not shown in this file). |

**Platform / API** – Java SE (no platform‑specific APIs). The code assumes the presence of a running SalesManager service container that provides `ServiceFactory` and related beans.

---

## 5. Additional Notes & Recommendations  

### 5.1. Code Quality Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** – All `List`, `Map`, and `Collection` usages are untyped. | Compile‑time type safety is lost; may cause `ClassCastException` at runtime. | Add generic type parameters (`List<ShoppingCartProduct>`, `Map<Long, Product>`, etc.). |
| **Unchecked casts** – `Map m = (Map)ooo;`, `Map<String,String> data = …`. | Potential `ClassCastException` if Jackson returns a different structure. | Use `Map<String,Object>` or create POJOs and let Jackson map directly. |
| **Manual JSON building** – `serializeToJSON` concatenates strings instead of using Jackson. | Prone to syntax errors, missing escaping, and hard to maintain. | Replace with `ObjectMapper.writeValueAsString()` or a JSON builder library. |
| **Hard‑coded JSON format** – No schema validation. | If the cookie data is tampered or the format changes, parsing will fail silently or produce `null`. | Validate against a JSON schema or at least check required keys before processing. |
| **Exception handling** – Method signatures throw generic `Exception`. | Callers have to catch broad exception types; debugging is harder. | Narrow to specific checked exceptions (e.g., `IOException`, `ParseException`) or wrap in a custom runtime exception. |
| **Missing null checks** – `store`, `locale`, `productCollection` are used without guard clauses. | `NullPointerException` if any argument is `null`. | Add defensive checks at the start of the method. |
| **Potential duplicate attribute IDs** – `attributesIds` is a simple `ArrayList`; duplicates may lead to redundant DB queries. | Minor performance overhead. | Use a `Set<Long>` to deduplicate. |
| **Logging & error reporting** – No logging on failures. | Hard to diagnose problems in production. | Add `Logger` and log key events and errors. |
| **Unused imports** – `Field`, `FieldOption`. | Clutter. | Remove unused imports. |
| **Inconsistent attribute handling** – Serialization writes `scpa.getAttributeValue()` while deserialization expects an ID. | Potential mismatch between client and server representation. | Standardize on attribute ID or value consistently. |

### 5.2. Edge Cases & Limitations  

1. **Empty product list** – The method will still create a `ShoppingCart` with an empty product list; callers may prefer a `null` or throw an exception.  
2. **Malformed JSON** – If the JSON string is partially corrupted, `ObjectMapper` will throw an `IOException`; this propagates out as `Exception`. No graceful fallback.  
3. **Large carts** – All product/attribute data is fetched in memory. For very large carts, memory consumption may spike. Consider streaming or pagination.  
4. **Price recalculation** – The utility re‑computes price on deserialization; if the product catalog changes (price, currency), the cart will reflect new values. This is intentional but may surprise users who expect cart values to be frozen.  
5. **Locale & currency mismatch** – The method assumes that the provided `locale` and `store.getCurrency()` match the user’s session; mismatches could lead to incorrect price formatting.

### 5.3. Suggested Enhancements  

| Feature | Implementation Idea |
|---------|----------------------|
| **Generic POJO mapping** – Create `MiniCartDTO` classes that mirror the JSON structure. Use `ObjectMapper` for both serialization and deserialization. | Simplifies parsing, improves type safety, and removes manual casting. |
| **Builder pattern** – Use a `ShoppingCartBuilder` to assemble the cart, making the code more readable and testable. | Encapsulates cart construction logic. |
| **Unit tests** – Add tests covering normal, empty, and malformed JSON scenarios. | Guarantees future changes do not break serialization. |
| **Exception hierarchy** – Define `MiniCartSerializationException` for all serialization errors. | Cleaner API for callers. |
| **Caching** – Cache product and attribute look‑ups if the same IDs appear frequently (e.g., in a user’s shopping session). | Improves performance for repeated deserializations. |
| **Internationalization** – Use `StringEscapeUtils` for attribute values to avoid JSON injection. | Enhances security. |
| **Configuration** – Externalize the JSON field names (`ps`, `p`, `pid`, `q`, `as`, `a`) to constants or properties. | Easier to change format if needed. |

---

### 6. Conclusion  

The `MiniShoppingCartSerializationUtil` provides a straightforward mechanism to persist a lightweight shopping cart in a cookie and restore it later. Its core logic is clear, but the implementation suffers from several common Java pitfalls:

* Unchecked raw types and casts,
* Manual JSON string construction,
* Lack of defensive programming,
* Limited error handling.

Addressing these issues will make the utility more robust, maintainable, and future‑proof. The recommended refactor to use Jackson’s mapping facilities and generics will dramatically improve type safety and readability, while additional logging and exception handling will aid troubleshooting in production.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Mar 7, 2011 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;



import org.apache.commons.lang.xwork.StringUtils;
import org.codehaus.jackson.map.ObjectMapper;

import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.entity.orders.ShoppingCartProduct;
import com.salesmanager.core.entity.orders.ShoppingCartProductAttribute;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.entity.system.FieldOption;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;

/**
 * Responsible for serializing the mini shopping cart in JSON format so it can
 * be saved in the cookie' Also contains a de serialization to rebuild a
 * ShoppingCart from a JSON String
 * 
 * @author Carl Samson
 * 
 */
public class MiniShoppingCartSerializationUtil {

	/**
	 * Deserialize a Shopping cart string from the cookie
	 * {"ps":[{"p":{"pid","<productid>","qty":"<quantity>","as":[{"a":"<attributeid>"},{"a":"<attributeid>"}]}},{"p":...}]
	 * 
	 * @param json
	 * @return com.salesmanager.core.entity.orders.ShoppingCart
	 * @throws Exception
	 */
	public static ShoppingCart deserializeJSON(String json, MerchantStore store, Locale locale) throws Exception {

		
		if(StringUtils.isBlank(json)) {
			return null;
		}
		ShoppingCart cart = null;
		Map<String, String> data = new ObjectMapper().readValue(json, HashMap.class);
		
		Map<Long,ShoppingCartProduct> productsMap = null;
		List productsList = null;
		
		if(data!=null) {
			
			//Collection products = new ArrayList();
			productsMap = new HashMap();
			productsList = new ArrayList(); 
			
			for(Object o: data.keySet()) {

				if(o instanceof String && ((String)o).equals("ps")) {
					// can parse
					
					Object oo = data.get(o);
					if(oo instanceof List) {//List 
						
						for(Object ooo:(List)oo) {

							if(ooo instanceof LinkedHashMap) {
								
								Map m = (Map)ooo;
								
								//get each products
								Map field = (Map)m.get("p");

								String productId = (String)field.get("pid");
								String qty = (String)field.get("q");
								
								long pId = Long.parseLong(productId);
								ShoppingCartProduct scp = new ShoppingCartProduct();
								scp.setProductId(pId);
								scp.setQuantity(Integer.parseInt(qty));
								productsMap.put(pId, scp);
								productsList.add(scp);
								
								List attrList = (List)field.get("as");
								
								if(attrList!=null) {
									List attributesList  = new ArrayList();
									for(Object oooo:attrList) {

										Map values = (Map)oooo;
										String attrId = (String)values.get("a");
										ShoppingCartProductAttribute attribute = new ShoppingCartProductAttribute();
										attribute.setAttributeId(Long.parseLong(attrId));
										attributesList.add(attribute);
										
									}
									scp.setAttributes(attributesList);
								}
							}
						}
					}
				}
			}
		}
		
		//if shoppingcart != null
		//get all products, query the catalog and re-create a new ShoppingCart
		//using values from the database
		
		if(productsList!=null) {
			List productsIds = new ArrayList();
			List attributesIds = new ArrayList();
			//Map productAttributes = new HashMap();
				//int i = 0;
				for(Object o: productsList) {

					ShoppingCartProduct p = (ShoppingCartProduct)o;
					//p.setInternalId(i);

					productsIds.add(p.getProductId());
					List attrs = p.getAttributes();
					if(attrs!=null) {
						List attrsPerProduct = new ArrayList();
						for(Object oo: attrs) {
							ShoppingCartProductAttribute attr = (ShoppingCartProductAttribute)oo;
							attributesIds.add(attr.getAttributeId());//for doing the query
							attrsPerProduct.add(attr.getAttributeId());
						}
						//productAttributes.put(p.getInternalId(), attrsPerProduct);
					}
					//i++;
					
				}
				
				CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
				Collection productCollection = cservice.getProducts(productsIds);
				Collection attributes = null;
				
				Map<Long,Product> pMap = null;
				if(productCollection!=null && productCollection.size()>0) {
					pMap = new HashMap();
					for(Object o : productCollection) {
						Product p = (Product)o;
						pMap.put(p.getProductId(), p);
					}
				}
				
				
				Map<Long,ProductAttribute> productAttributesMap = null;
				if(attributesIds!=null && attributesIds.size()>0) {
					attributes = cservice.getProductAttributes(attributesIds, locale.getLanguage());
					productAttributesMap = new HashMap();
					for(Object o : attributes) {
						ProductAttribute pa = (ProductAttribute)o;
						productAttributesMap.put(pa.getProductAttributeId(), pa);
					}
				}
				

				
				
				
				//recreate ShoppingCart
				cart = new ShoppingCart();
				List shoppingCartProducts = new ArrayList();
				
				if(pMap!=null && pMap.size()>0) {
					LocaleUtil.setLocaleToEntityCollection(productCollection, locale, store.getCurrency());
					for(Object o: productsList) {
						
						ShoppingCartProduct scp = (ShoppingCartProduct)o;
						
						if(pMap.containsKey(scp.getProductId())) {
							
							Product p = pMap.get(scp.getProductId());
							
							scp.setImage(p.getSmallImagePath());
							scp.setProductName(p.getProductDescription().getProductName());
							
							List productAttributesList = null;
							List shoppingCartAttributesList = null;
							shoppingCartProducts.add(scp);
							
							List attrs = scp.getAttributes();
							if(attrs!=null && attrs.size()>0) {
								
								for(Object oo: attrs) {
									
									ShoppingCartProductAttribute scpa = (ShoppingCartProductAttribute)oo;
									
									if(productAttributesMap.containsKey(scpa.getAttributeId())) {
										
										
										ProductAttribute pa = (ProductAttribute)productAttributesMap.get(scpa.getAttributeId());
										if(productAttributesList==null) {
											productAttributesList = new ArrayList();
										}
										if(shoppingCartAttributesList==null) {
											shoppingCartAttributesList = new ArrayList();
										}
										productAttributesList.add(pa);
										shoppingCartAttributesList.add(scpa);
										
									}
									
								}
							}
							
							if(productAttributesList!=null) {
								BigDecimal priceWithAttributes = ProductUtil
								.determinePriceWithAttributes(p, productAttributesList, locale,
										store.getCurrency());
										scp.setPrice(priceWithAttributes);
										scp.setPriceText(CurrencyUtil
												.displayFormatedAmountWithCurrency(
														priceWithAttributes, store
														.getCurrency()));
										
								scp.setAttributes(shoppingCartAttributesList);
							} else {
								
								scp.setPrice(ProductUtil.determinePrice(p, locale,
										store.getCurrency()));
								BigDecimal price = ProductUtil.determinePrice(p,
										locale, store.getCurrency());
								scp.setPriceText(CurrencyUtil
										.displayFormatedAmountWithCurrency(price, store
												.getCurrency()));
								
							}

						}

					}
				}
				
	/*			
				for(Object o : productCollection){
					Product p = (Product)o;
					//p.setLocale(locale, currency);
					if(p.getMerchantId()==store.getMerchantId()) {
						
						ShoppingCartProduct scp = new ShoppingCartProduct();
						scp.setProductId(p.getProductId());
						scp.setImage(p.getSmallImagePath());
						scp.setQuantity(1);
						
						ShoppingCartProduct temp = productsMap.get(p.getProductId());
						
						if(temp!=null) {
							scp.setQuantity(temp.getQuantity());
						}

						scp.setProductName(p.getName());
						shoppingCartProducts.add(scp);
						
						if(attributes!=null) {
							
							List productAttributesList = new ArrayList();
							
							List productAttrs = (List)productAttributes.get(p.getProductId());
							
							for(Object x : productAttrs) {
								Long productAttribute = (Long)x;
								//get the object from loaded collection
								for(Object z : attributes) {
									
									ProductAttribute pa = (ProductAttribute)z;
									if(pa.getProductAttributeId()==productAttribute) {
										
										ShoppingCartProductAttribute productAttr = new ShoppingCartProductAttribute();
										productAttr.setAttributeId(productAttribute);
										productAttr.setAttributeValue(productAttr.getAttributeValue());
										productAttr.setTextValue(productAttr.getTextValue());
										productAttributesList.add(productAttr);	
										
										
										
									}
								}
							}
							
							if(productAttributesList.size()>0 && productAttributesMap.containsKey(p.getProductId())) {
								
								List attrs = productAttributesMap.get(p.getProductId());
								
								BigDecimal priceWithAttributes = ProductUtil
								.determinePriceWithAttributes(p, attrs, locale,
										store.getCurrency());
										scp.setPrice(priceWithAttributes);
										scp.setPriceText(CurrencyUtil
												.displayFormatedAmountWithCurrency(
														priceWithAttributes, store
														.getCurrency()));
								
								
								scp.setAttributes(productAttributesList);
							}	else {
								
								scp.setPrice(ProductUtil.determinePrice(p, locale,
										store.getCurrency()));
								BigDecimal price = ProductUtil.determinePrice(p,
										locale, store.getCurrency());
								scp.setPriceText(CurrencyUtil
										.displayFormatedAmountWithCurrency(price, store
												.getCurrency()));
							}
						}	
					}
				}*/
				
				
				cart.setProducts(shoppingCartProducts);
		}
		
		return cart;

	}

	
	
	/**
	 * {"ps":[{"p":{"pid","<productid>","qty":"<quantity>","as":[{"a":"<attributeid>"},{"a":"<attributeid>"}]}},{"p":...}]}
	 * @param shoppingCart
	 * @return String
	 * @throws Exception
	 */
	public static String serializeToJSON(ShoppingCart shoppingCart) throws Exception {


		if(shoppingCart==null) {
			return null;
		}
		
		Collection products = shoppingCart.getProducts();
		if(products == null || products.size()==0) {
			return null;
		}
		
		StringBuilder json = new StringBuilder();
		json.append("{\"ps\":[");
		
		int i = 1;
		

		
		for(Object o: products) {
			
				ShoppingCartProduct product = (ShoppingCartProduct)o;
				json.append("{\"p\":");
				json.append("{\"pid\":\"");
				json.append(product.getProductId());
				json.append("\"");
				json.append(",\"q\":");
				json.append("\"");
				json.append(product.getQuantity());
				json.append("\"");
				List attributes = product.getAttributes();
				if(attributes!=null) {
					json.append(",\"as\":[");
					int j = 1;
					for(Object oo : attributes) {
						
						ShoppingCartProductAttribute scpa = (ShoppingCartProductAttribute)oo;
						json.append("{\"a\":");
						json.append("\"");
						json.append(scpa.getAttributeValue());
						json.append("\"");
						json.append("}");
						if(j<attributes.size()) {
							json.append(",");
						}
						j++;
					}
					json.append("]");
				}
				json.append("}}");
				if(i<products.size()) {
					json.append(",");
				}
				i++;
		}
		json.append("]}");
		return json.toString();
	}

}



```
