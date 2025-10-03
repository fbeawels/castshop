# ProductImageTag.java

## Review

## 1. Summary  

**Purpose**  
`ProductImageTag` is a JSP custom tag that renders an `<img>` element for a product.  
It accepts a `Product` object and a handful of optional parameters (image source type, resizing ratio, id, CSS class, etc.) and produces an image tag whose width/height is derived from the store configuration or from a percentage-based resize ratio.

**Key components**  
| Component | Role |
|-----------|------|
| `doStartTag()` | Core rendering logic – obtains image path, loads store‑specific configuration, calculates dimensions, builds the `<img>` tag and writes it to the page. |
| `doEndTag()` | Simply indicates that the rest of the page should be evaluated. |
| Setters/Getters | Expose tag attributes to the JSP page. |

**Notable patterns / libraries**  
* Uses **TagSupport** from `javax.servlet.jsp.tagext`, a classic JSP tag‑library pattern.  
* Relies on **Apache Commons Configuration** (for reading properties) and **Commons Lang** (`StringUtils`).  
* Calls out to the application’s own services (`ReferenceService`, `ServiceFactory`, `UrlUtil`, `FileUtil`, `PropertiesUtil`).  
* Logging is done with **Log4j**.

---

## 2. Detailed Description  

### Execution Flow  
1. **Request/Session Retrieval** – obtains `HttpServletRequest`, `HttpSession`, and the `Locale`.  
2. **Image Path Construction** – based on the `source` attribute (`smallImage` vs. others), calls `FileUtil.getSmallProductImagePath` or `getLargeProductImagePath`.  
3. **Optional URL Prefix** – if `addSchemeHostAndPort` is `true`, prepends the domain (`UrlUtil.getUnsecuredDomain`).  
4. **Store Configuration**  
   * Tries to fetch a pre‑cached `STORECONFIGURATION` map from the session.  
   * If missing, calls `ReferenceService.getModuleConfigurationsKeyValue(...)` to load configuration from the database.  
   * As a fallback, loads defaults from `PropertiesUtil`.  
5. **Dimension Selection** – picks small or large image dimensions from the configuration map.  
6. **Resize Ratio** – if `resizeratio > 0`, multiplies the selected width/height by that percentage.  
7. **Tag Construction** – builds an `<img>` tag string, adding attributes such as `src`, `width`, `height`, optional `id` and `class`.  
8. **Output** – writes the tag to the JSP page (`pageContext.getOut().print`).  

### Assumptions & Constraints  
* A `Product` instance **must** be provided; the code does not guard against `null`.  
* The configuration map always contains the expected keys (`smallimagewidth`, etc.).  
* The store configuration is stored in the session and is considered thread‑safe because each request gets a fresh tag instance.  
* Resizeratio is interpreted as a percentage; negative values are silently ignored.  

### Architecture & Design Choices  
* The tag is a **stateless helper** that delegates image path logic to `FileUtil`.  
* Store‑specific configuration is cached in the session to avoid repeated database hits.  
* The class mixes concerns: configuration loading, URL construction, and HTML generation all live inside `doStartTag()`.  
* Error handling is limited to a single catch block that logs the exception and swallows it, allowing the page to continue rendering (though the tag would silently fail to output any image).

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `doStartTag()` | Core rendering logic; writes `<img>` to the response. | N/A (reads tag attributes & request/session data) | `SKIP_BODY` (returns int) | Writes to `JspWriter`; logs errors |
| `doEndTag()` | Indicates page evaluation continues. | N/A | `EVAL_PAGE` | None |
| `getProduct()/setProduct(Product)` | Getter/Setter for product. | `Product` | `Product` | None |
| `getSource()/setSource(String)` | Image source type (`smallImage` or others). | `String` | `String` | None |
| `getResizeratio()/setResizeratio(int)` | Resize percentage. | `int` | `int` | None |
| `getId()/setId(String)` | Optional HTML id attribute. | `String` | `String` | None |
| `getCssClass()/setCssClass(String)` | Optional CSS class. | `String` | `String` | None |
| `isAddSchemeHostAndPort()/setAddSchemeHostAndPort(boolean)` | Flag to prepend domain. | `boolean` | `boolean` | None |

The **utility methods** (`FileUtil`, `UrlUtil`, `PropertiesUtil`) are not part of this class but are crucial for its operation.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.jsp.tagext.TagSupport` | Standard Java EE | Provides lifecycle hooks for custom tags. |
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads `.properties` files; older `commons-configuration` 1.x. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Basic string utilities. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.salesmanager.core.*` | Internal | Application‑specific utilities and services (`FileUtil`, `UrlUtil`, `ReferenceService`, etc.). |
| `javax.servlet.http.HttpServletRequest/HttpSession` | Standard | Servlet API. |

No platform‑specific code beyond standard servlet/JSP.

---

## 5. Additional Notes & Recommendations  

### Strengths  
* Keeps the tag logic simple and focused on rendering.  
* Caches configuration to reduce database load.  
* Provides flexible sizing via a percentage ratio.  

### Weaknesses / Edge Cases  
1. **Null Safety**  
   * No check for `product` being `null` → `NullPointerException`.  
   * `product.getMerchantId()` or `product.getProductImage()` could be `null`.  
2. **Error Handling**  
   * Swallows all exceptions after logging; the user sees a broken or missing image but no diagnostic.  
3. **Configuration Map**  
   * Uses raw `Map` with unchecked casts; can lead to `ClassCastException`.  
   * If a configuration key is missing, `get("smallimagewidth")` returns `null`, causing `NumberFormatException`.  
4. **Resizeratio Logic**  
   * Negative values are ignored silently; may be better to validate and log.  
5. **Hard‑coded Strings**  
   * `"smallImage"` magic value; consider an enum or constants.  
6. **Deprecated HTML**  
   * Uses `border="0"`; modern HTML prefers CSS.  
7. **Potential Security Issue**  
   * The `src` attribute is constructed from application data; ensure that the resulting path is safe and not subject to XSS if `product.getProductImage()` can be controlled by a user.  
8. **Testing & Reusability**  
   * The tag is tightly coupled to session attributes (`STORE`, `STORECONFIGURATION`). Unit testing would be more difficult.  

### Suggested Enhancements  
| Area | Suggested Change |
|------|------------------|
| **Type Safety** | Use `Map<String, String>` for configurations; cast once after loading. |
| **Null Checks** | Guard against `product == null`; fallback to a placeholder image. |
| **Validation** | Validate `resizeratio` (e.g., 1–100) and log a warning if out of range. |
| **Configuration Loading** | Move the configuration loading into a separate helper/service method; cache using a session‑scoped bean. |
| **String Constants** | Replace `"smallImage"` with an enum (`ImageSource.SMALL`, `ImageSource.LARGE`). |
| **Logging** | Use `log.error("Error formatting image size", e)` to include stack trace. |
| **Deprecated Attributes** | Remove `border="0"`; use CSS (`class="product-image"`) for styling. |
| **Alt Text** | Add an `alt` attribute derived from the product name or description. |
| **Security** | Validate or sanitize the image path to prevent directory traversal or injection. |
| **Unit Testing** | Expose the core logic (path building, size calculation) in a separate non‑tag class so it can be unit‑tested without a servlet container. |
| **Taglib Declarations** | Ensure the tag library descriptor (TLD) declares the attributes with correct types and default values. |

### Final Thoughts  
`ProductImageTag` fulfills a clear UI need but would benefit from modern Java practices (generics, optional, builder patterns) and better separation of concerns. The primary goal should be to make the tag resilient to mis‑configured data, easier to test, and compliant with current web standards. Implementing the above recommendations would significantly improve maintainability and robustness.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.util.www.tags;

import java.util.HashMap;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.tagext.TagSupport;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;


import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.UrlUtil;


public class ProductImageTag extends TagSupport {
	
	private Logger log = Logger.getLogger(ProductImageTag.class);
	
	private Product product;
	private String source;
	private int resizeratio;
	private String id;
	private String cssClass;
	private boolean addSchemeHostAndPort = false;


	public int doStartTag() throws JspException {
		try {



			HttpServletRequest request = (HttpServletRequest) pageContext
					.getRequest();
			
			HttpSession session = request.getSession();
			
			Locale locale = (Locale) request.getAttribute("LOCALE");
			
			String imagePath = null;
			
			if("smallImage".equals(this.getSource())) {
				imagePath = FileUtil.getSmallProductImagePath(this.getProduct().getMerchantId(), this
						.getProduct().getProductImage());
			} else {
				imagePath = FileUtil.getLargeProductImagePath(this.getProduct().getMerchantId(), this
						.getProduct().getProductImage());
			}
			
			if(addSchemeHostAndPort) {
				imagePath = UrlUtil.getUnsecuredDomain(request) + imagePath;
			}
			
			//get configuration
			MerchantStore store = (MerchantStore)session.getAttribute("STORE");
			
			Map configurations = (Map)session.getAttribute("STORECONFIGURATION");
			
			if(configurations==null) {
				
				ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
				Map storeConfiguration = rservice.getModuleConfigurationsKeyValue(
				store.getTemplateModule(), store.getCountry());
				if (storeConfiguration != null) {
					session.setAttribute("STORECONFIGURATION",
							storeConfiguration);
				} else {
					configurations = new HashMap();
					Configuration conf = PropertiesUtil.getConfiguration();
					configurations.put("largeimagewidth", conf.getString("core.product.config.large.image.width"));
					configurations.put("largeimageheight", conf.getString("core.product.config.large.image.height"));
					configurations.put("smallimagewidth", conf.getString("core.product.config.small.image.width"));
					configurations.put("smallimageheight", conf.getString("core.product.config.small.image.height"));
				}
				
			}
			
			
			String simageWidth = null;
			String simageHeight = null;
			
			if("smallImage".equals(this.getSource())) {
				simageWidth = (String)configurations.get("smallimagewidth");
				simageHeight = (String)configurations.get("smallimageheight");
			} else {
				simageWidth = (String)configurations.get("largeimagewidth");
				simageHeight = (String)configurations.get("largeimageheight");
			}

			
			StringBuilder imageTag = new StringBuilder();
			imageTag.append("<img src=\"").append(imagePath).append("\"");
			
			if(this.getResizeratio()>0) {
				
				int imageWidth = 0;
				int imageHeight = 0;
				
				try {
					if("smallImage".equals(this.getSource())) {//or largeImage
						
						imageWidth = Integer.parseInt((String)configurations.get("smallimagewidth"));
						imageHeight = Integer.parseInt((String)configurations.get("smallimageheight"));
					} else {
						imageWidth = Integer.parseInt((String)configurations.get("largeimagewidth"));
						imageHeight = Integer.parseInt((String)configurations.get("largeimageheight"));	
					}
					
					imageWidth = imageWidth * this.getResizeratio()/100;
					imageHeight = imageHeight * this.getResizeratio()/100;
					
					simageWidth = String.valueOf(imageWidth);
					simageHeight = String.valueOf(imageHeight);
		
				} catch (Exception e) {
					log.error("Error formating image size " + e);
				}
				
			}
			
			imageTag.append(" border=\"0\" width=\"").append(simageWidth).append("\"");
			imageTag.append(" border=\"0\" height=\"").append(simageHeight).append("\"");
			
			if(!StringUtils.isBlank(this.getId())) {
				imageTag.append(" id=\"").append(this.getId()).append("\"");
			}
			
			if(!StringUtils.isBlank(this.getCssClass())) {
				imageTag.append(" class=\"").append(this.getCssClass()).append("\"");
			}
			
			imageTag.append(">");

			pageContext.getOut().print(imageTag.toString());


			
		} catch (Exception ex) {
			log.error(ex);
		}
		return SKIP_BODY;
	}

	public int doEndTag() {
		return EVAL_PAGE;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public String getSource() {
		return source;
	}

	public void setSource(String source) {
		this.source = source;
	}

	public int getResizeratio() {
		return resizeratio;
	}

	public void setResizeratio(int resizeratio) {
		this.resizeratio = resizeratio;
	}

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public String getCssClass() {
		return cssClass;
	}

	public void setCssClass(String cssClass) {
		this.cssClass = cssClass;
	}

	public boolean isAddSchemeHostAndPort() {
		return addSchemeHostAndPort;
	}

	public void setAddSchemeHostAndPort(boolean addSchemeHostAndPort) {
		this.addSchemeHostAndPort = addSchemeHostAndPort;
	}



}



```
