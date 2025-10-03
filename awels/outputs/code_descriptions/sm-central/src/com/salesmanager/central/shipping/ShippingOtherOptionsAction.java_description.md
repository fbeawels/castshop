# ShippingOtherOptionsAction.java

## Review

## 1. Summary  

**Purpose**  
`ShippingOtherOptionsAction` is a Struts‑2 action that manages the “other” shipping options for a merchant: tax handling, free‑shipping logic, and handling fees. It persists these settings in the merchant’s configuration store and exposes the current values to the view layer.

**Key Components**  
| Component | Responsibility |
|-----------|----------------|
| `save()` | Validates input, updates the merchant configuration for tax, free‑shipping, and handling fees, and displays a confirmation message. |
| `prepare()` | Loads the current configuration and tax classes, populating request attributes that the JSP/velocity template consumes. |
| `display()` | Simple success‑return that forwards to the view. |
| Getter/Setter methods | Bind form parameters and expose internal state to the view. |

**Design Patterns & Frameworks**  
* **Action/Controller** – Implements Struts‑2 `Action` semantics (`Preparable`, `getServletRequest`).  
* **DAO Service** – Uses `ServiceFactory` to obtain `MerchantService` and `TaxService`.  
* **Configuration Store** – Persists settings in `MerchantConfiguration` objects.  
* **Utility helpers** – `CurrencyUtil`, `LabelUtil`, `MessageUtil`, and `LabelUtil` provide internationalisation and error handling.  

---

## 2. Detailed Description  

### Execution Flow

1. **Request Handling**  
   * When a user submits the “other shipping options” form, Struts calls `prepare()` first, followed by `save()`.  
   * `display()` is used when the page is simply rendered (GET request).

2. **Preparation (`prepare()`)**  
   * Sets the page title.  
   * Retrieves the current merchant context (`Context`).  
   * Loads tax classes via `TaxService` and builds a `TreeMap` (tax ID → title) for UI selection.  
   * Sends a `ConfigurationRequest` (prefixed with `"SHP_"`) to `MerchantService` to fetch all shipping‑related configurations.  
   * Iterates over the configurations, populating request attributes:
     * Tax class (`shiptaxclass`)
     * Free‑shipping indicator, region, and amount
     * Handling fees
     * Shipping zones (unused in this snippet but presumably used elsewhere)
   * All values are stored as strings or `BigDecimal` for use in the view.

3. **Saving (`save()`)**  
   * Validates free‑shipping amount and handling fee using `CurrencyUtil.validateCurrency`.  
   * Updates the tax class:
     * If the configuration exists and the form unchecks the box → deletes the key via `cleanupkey()`.  
     * If a new tax class is selected → updates the value.  
   * Updates free‑shipping indicator:
     * Handles three cases: enable + region + amount, enable + only region, or disable.  
   * Updates handling fees similarly to tax.  
   * Persists each change with `MerchantService.saveOrUpdateMerchantConfiguration`.  
   * Adds a success message; if an exception occurs, an error message is shown and the stack trace logged.

4. **Cleanup** – No explicit resource cleanup; relies on framework to release request objects.

### Dependencies & Constraints  

* **Frameworks**: Struts‑2 (`Preparable`, request handling), log4j for logging.  
* **Services**: `MerchantService`, `TaxService` obtained via `ServiceFactory`.  
* **Domain objects**: `MerchantConfiguration`, `TaxClass`.  
* **Utility classes**: `CurrencyUtil`, `LabelUtil`, `MessageUtil`, `ShippingConstants`.  
* **Assumptions**:  
  * The merchant’s context is always present in the session.  
  * All configuration keys follow the `"SHP_"` prefix convention.  
  * The front‑end form submits expected parameter names (`applytax`, `taxclass`, etc.).  
  * The application uses a single currency per merchant (hence `ctx.getCurrency()`).

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `public String save() throws Exception` | Persists form data to merchant configuration; validates numeric values. | None (uses fields). | `"success"` string for Struts navigation. | Writes to DB via `MerchantService`; sets request messages; logs errors. |
| `public void prepare() throws Exception` | Loads existing configuration and tax classes, populating request attributes for the view. | None. | `void` | Sets request attributes; logs errors. |
| `public String display() throws Exception` | Simple forward to the JSP/velocity page. | None. | `"success"` | None. |
| Getters/Setters for `shiptaxclass`, `handlingfees`, `applytax`, `taxclass`, `freeshipdest`, `handling`, `freeshipamnt`, `applyfreeshipping` | Bind form fields and expose internal state. | `String`/`MerchantConfiguration`. | Corresponding type | None. |

*Utility methods*  
* `super.cleanupkey(String key)` – assumed to delete a configuration key from the store.  
* `super.getServletRequest()` – retrieves the current `HttpServletRequest`.

---

## 4. Dependencies  

| Library / Class | Role | Type |
|-----------------|------|------|
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.opensymphony.xwork2.Preparable` | Struts action interface | Third‑party |
| `com.salesmanager.central.profile.Context` | Holds merchant session data | Domain |
| `com.salesmanager.central.profile.ProfileConstants` | Session key constants | Domain |
| `com.salesmanager.core.constants.ShippingConstants` | Shipping key constants | Domain |
| `com.salesmanager.core.entity.merchant.MerchantConfiguration` | Configuration entity | Domain |
| `com.salesmanager.core.entity.tax.TaxClass` | Tax class entity | Domain |
| `com.salesmanager.core.service.ServiceFactory` | Service locator | Domain |
| `com.salesmanager.core.service.merchant.ConfigurationRequest` | Request DTO for config read | Domain |
| `com.salesmanager.core.service.merchant.ConfigurationResponse` | Response DTO for config read | Domain |
| `com.salesmanager.core.service.merchant.MerchantService` | CRUD for merchant configuration | Domain |
| `com.salesmanager.core.service.tax.TaxService` | Retrieve tax classes | Domain |
| `com.salesmanager.core.util.CurrencyUtil` | Currency formatting / validation | Utility |
| `com.salesmanager.core.util.LabelUtil` | Internationalisation lookup | Utility |
| `com.salesmanager.core.util.MessageUtil` | Adds messages to request | Utility |

All dependencies are either core libraries of the Sales Manager platform or standard third‑party libraries (log4j, Struts‑2).

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`List`, `Map`, `Iterator`) | Loss of compile‑time type safety; potential `ClassCastException`. | Use generics: `List<TaxClass>`, `Map<String, String>`, etc. |
| **Unnecessary `dt` and `new Date(dt.getTime())`** | Redundant object creation. | Use a single `Date now = new Date();` and reuse. |
| **Duplicated logic for updating configurations** | Hard to maintain, error‑prone. | Extract helper methods: `updateOrCreateConfig(key, value, value1, value2)` and `deleteConfig(key)` |
| **Magic strings** (`"1"`, `"true"`, `"false"`, `"SHP_"`) | Hard to change; prone to typos. | Use constants in `ShippingConstants` or an enum. |
| **Exception handling** – `catch (Exception e)` in both methods | Swallows specific errors; always returns `"success"`. | Catch specific exceptions; return `"error"` or redirect to an error page; rethrow or set error status. |
| **Logging** – only stack trace logged, no context | Hard to debug. | Log method entry, key values, and detailed exception stack traces. |
| **Unused local variables** (`config` in `save()`, `shipping`, `szones` in `prepare()`) | Minor clutter. | Remove them. |
| **NPE risk** – accessing request or session objects without null checks. | Potential `NullPointerException`. | Guard against null or throw a descriptive exception. |
| **Hard‑coded region default** (`ShippingConstants.DOMESTIC_SHIPPING`) | Unclear origin. | Document or make it a configuration. |
| **Missing validation for empty strings** – e.g., `this.getFreeshipamnt()` might be an empty string. | `CurrencyUtil.validateCurrency` may throw, but the error message is generic. | Add explicit check for empty string before parsing. |
| **No unit tests** – code heavily relies on external services. | Hard to guarantee correctness. | Write unit tests with mocked `MerchantService` and `TaxService`. |

### 5.2 Functional Enhancements  

1. **Internationalise UI labels** – already using `LabelUtil`, but ensure all hard‑coded strings are replaced.  
2. **Persist `shippingZones`** – the code fetches `MODULE_SHIPPING_ZONES_SHIPPING` but never uses it; either expose it to the view or remove the logic.  
3. **Concurrency handling** – if two admin users update the same configuration concurrently, last write wins. Consider optimistic locking if needed.  
4. **Form validation** – perform server‑side validation in a separate validator class (Struts‑2 `Validator`) to keep `save()` focused.  
5. **Clean‑up helper** – use a `ConfigurationHelper` bean that encapsulates CRUD operations and key management.

### 5.3 Security & Permissions  

* The action assumes the session holds a `Context` with a valid `merchantId`.  
* There is no explicit permission check – any authenticated user with a session can modify shipping settings.  
* Recommend adding a security interceptor or a `@RequiresPermission("MANAGE_SHIPPING")` annotation.

### 5.4 Performance  

* The current implementation retrieves **all** shipping configurations on every request (`SHP_` prefix).  
* If the configuration set grows large, consider filtering by key or caching results per session.  
* Use lazy loading or paging if the number of tax classes becomes large.

---

**Conclusion**  
`ShippingOtherOptionsAction` performs its core responsibilities—reading and writing shipping configuration values—adequately, but it suffers from a number of maintainability and robustness issues. Refactoring to use generics, encapsulating repeated logic, improving exception handling, and adding unit tests would significantly raise the quality and reliability of this component.

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
package com.salesmanager.central.shipping;

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

import org.apache.log4j.Logger;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.tax.TaxClass;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.tax.TaxService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

/**
 * Manages 'Other Options'
 * 
 * @author Administrator
 * 
 */
public class ShippingOtherOptionsAction extends ShippingRatesAction implements
		Preparable {

	private Logger log = Logger.getLogger(ShippingOtherOptionsAction.class);

	private MerchantConfiguration shiptaxclass;
	private MerchantConfiguration freeshipinddestamnt;
	private MerchantConfiguration handlingfees;

	private String applytax;
	private String taxclass;
	private String handling;
	private String freeshipdest;
	private String freeshipamnt;
	private String applyfreeshipping;

	public String save() throws Exception {

		List config = null;

		try {

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			TaxService tservice = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			java.util.Date dt = new java.util.Date();

			// validate BigDecimals -- only if checkbox
			BigDecimal fsamount;
			BigDecimal hamount;
			if (this.getApplyfreeshipping() != null
					&& this.getFreeshipamnt() != null) {
				try {
					// strip , from amount
					fsamount = CurrencyUtil.validateCurrency(this
							.getFreeshipamnt(), ctx.getCurrency());

				} catch (Exception e) {
					
					MessageUtil.addErrorMessage(super.getServletRequest(),
							LabelUtil.getInstance().getText(
									"message.error.invalidfreeshippingamount"));
					return SUCCESS;
				}
			}

			if (this.getHandling() != null) {
				if (!this.getHandling().equals("")) {
					try {
						// strip , from amount
						hamount = CurrencyUtil.validateCurrency(this
								.getHandling(), ctx.getCurrency());
					} catch (Exception e) {
						
						MessageUtil
								.addErrorMessage(
										super.getServletRequest(),
										LabelUtil
												.getInstance()
												.getText(
														"message.error.invalidhandlingfeeamount"));
						return SUCCESS;
					}
				}
			}

			// update tax class
			if (shiptaxclass != null) {// exist in database
				if (this.getApplytax() == null) {
					// remove shipping tax lines
					super
							.cleanupkey(ShippingConstants.MODULE_SHIPPING_TAX_CLASS);
				} else {
					if (this.getTaxclass() != null
							&& !this.getTaxclass().equals(
									shiptaxclass.getConfigurationValue())) {
						shiptaxclass.setConfigurationValue(this.getTaxclass());
						shiptaxclass.setLastModified(new java.util.Date(dt
								.getTime()));
						mservice
								.saveOrUpdateMerchantConfiguration(shiptaxclass);

					}
				}
			} else {// does not exist in database
				if (this.getApplytax() != null) {// submitted

					// Get tax class id
					MerchantConfiguration conf = new MerchantConfiguration();
					conf
							.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_TAX_CLASS);
					conf.setMerchantId(ctx.getMerchantid());
					conf.setConfigurationValue(this.getTaxclass());
					mservice.saveOrUpdateMerchantConfiguration(conf);

				}
			}

			// update freeshipping indicator
			if (freeshipinddestamnt != null) {// exist in database

				freeshipinddestamnt.setLastModified(new java.util.Date(dt
						.getTime()));
				if (this.getApplyfreeshipping() != null) {

					freeshipinddestamnt.setConfigurationValue("true");

					if (this.getFreeshipdest() != null) {
						freeshipinddestamnt.setConfigurationValue1(this
								.getFreeshipdest());
					}
					if (this.getFreeshipamnt() != null) {
						freeshipinddestamnt.setConfigurationValue2(this
								.getFreeshipamnt());
					}

				} else {

					freeshipinddestamnt.setConfigurationValue("false");
				}

				mservice.saveOrUpdateMerchantConfiguration(freeshipinddestamnt);

			} else {// does not exist in database
				// cleanup first

				if (this.getApplyfreeshipping() != null) {

					freeshipinddestamnt = new MerchantConfiguration();
					freeshipinddestamnt
							.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_FREE_IND_DEST_AMNT);

					freeshipinddestamnt.setConfigurationValue("true");
					freeshipinddestamnt.setMerchantId(ctx.getMerchantid());

					if (this.getFreeshipdest() != null) {
						freeshipinddestamnt.setConfigurationValue1(this
								.getFreeshipdest());
					}
					if (this.getFreeshipamnt() != null) {
						freeshipinddestamnt.setConfigurationValue2(this
								.getFreeshipamnt());
					}
					mservice
							.saveOrUpdateMerchantConfiguration(freeshipinddestamnt);

				}

			}

			// update handling fees
			if (handlingfees != null) {// exist in database
				if (this.getHandling() == null) {
					// remove shipping tax lines
					super
							.cleanupkey(ShippingConstants.MODULE_SHIPPING_HANDLING_FEES);
				} else {
					if (this.getHandling() != null
							&& !this.getHandling().equals(
									handlingfees.getConfigurationValue())) {
						handlingfees.setConfigurationValue(this.getHandling());
						handlingfees.setLastModified(new java.util.Date(dt
								.getTime()));
						mservice
								.saveOrUpdateMerchantConfiguration(handlingfees);

					}
				}
			} else {// does not exist in database
				if (this.getHandling() != null) {// submitted
					MerchantConfiguration conf = new MerchantConfiguration();
					conf
							.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_HANDLING_FEES);
					conf.setMerchantId(ctx.getMerchantid());
					conf.setConfigurationValue(this.getHandling());
					mservice.saveOrUpdateMerchantConfiguration(conf);

				}
			}

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {

			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);

		}

		return SUCCESS;

	}

	public void prepare() throws Exception {

		try {
			
			super.setPageTitle("leftmenu.shipping.shippinghandling");

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			// Set tax classes
			TaxService tservice = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);
			List txscl = tservice.getTaxClasses(ctx.getMerchantid());

			Map classtaxesid = new TreeMap();

			if (txscl != null) {
				Iterator it = txscl.iterator();
				while (it.hasNext()) {
					TaxClass x = (TaxClass) it.next();
					classtaxesid.put(String.valueOf(x.getTaxClassId()), x
							.getTaxClassTitle());
				}
			}

			classtaxesid.put("1", "Tax");
			super.getServletRequest().setAttribute("taxclassmap", classtaxesid);

			ConfigurationRequest requestvo = new ConfigurationRequest(
					merchantid.intValue(), true, "SHP_");
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationResponse responsevo = mservice
					.getConfiguration(requestvo);
			List config = responsevo.getMerchantConfigurationList();

			String shipping = null;
			Map szones = new HashMap();

			if (config != null) {
				Iterator it = config.iterator();
				while (it.hasNext()) {
					MerchantConfiguration m = (MerchantConfiguration) it.next();
					String key = m.getConfigurationKey();

					if (key.equals(ShippingConstants.MODULE_SHIPPING_TAX_CLASS)) {
						shiptaxclass = m;
						super.getServletRequest().setAttribute("shiptaxclass",
								m.getConfigurationValue());

					}
					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_FREE_IND_DEST_AMNT)) {

						freeshipinddestamnt = m;

						if (m.getConfigurationValue() != null
								&& !m.getConfigurationValue().equals("")) {
							super.getServletRequest().setAttribute(
									"freeshippingindicator",
									m.getConfigurationValue());
						}
						if (m.getConfigurationValue1() != null
								&& !m.getConfigurationValue1().equals("")) {
							super.getServletRequest().setAttribute(
									"freeshippingregion",
									m.getConfigurationValue1());
						}
						if (m.getConfigurationValue2() != null
								&& !m.getConfigurationValue2().equals("")) {
							BigDecimal value = new BigDecimal(0);
							try {
								value = new BigDecimal(m
										.getConfigurationValue2());
							} catch (Exception e) {
								log.error("Invalid big decimal value "
										+ m.getConfigurationValue2());
							}
							super.getServletRequest().setAttribute(
									"freeshippingamount", value);
						}

						super.getServletRequest().setAttribute(
								"freeshippingindicator",
								m.getConfigurationValue());
					}

					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_HANDLING_FEES)) {
						handlingfees = m;
						BigDecimal value = new BigDecimal(0);
						try {
							value = new BigDecimal(m.getConfigurationValue());
						} catch (Exception e) {
							log.error("Invalid big decimal value "
									+ m.getConfigurationValue());
						}
						super.getServletRequest().setAttribute("handlingfees",
								value);
					}
					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING)) {
						super.getServletRequest().setAttribute("zoonesshipping",
								m.getConfigurationValue());
					}

				}

			}

		} catch (Exception e) {

			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);

		}

	}

	public String display() throws Exception {
		return SUCCESS;
	}

	public MerchantConfiguration getShiptaxclass() {
		return shiptaxclass;
	}

	public MerchantConfiguration getHandlingfees() {
		return handlingfees;
	}

	public String getApplytax() {
		return applytax;
	}

	public void setApplytax(String applytax) {
		this.applytax = applytax;
	}

	public String getTaxclass() {
		return taxclass;
	}

	public void setTaxclass(String taxclass) {
		this.taxclass = taxclass;
	}

	public String getFreeshipdest() {
		if (freeshipdest == null) {
			freeshipdest = ShippingConstants.DOMESTIC_SHIPPING;
		}
		return freeshipdest;
	}

	public void setFreeshipdest(String freeshipdest) {
		this.freeshipdest = freeshipdest;
	}

	public String getHandling() {
		return handling;
	}

	public void setHandling(String handling) {
		this.handling = handling;
	}

	public String getFreeshipamnt() {
		return freeshipamnt;
	}

	public void setFreeshipamnt(String freeshipamnt) {
		this.freeshipamnt = freeshipamnt;
	}

	public String getApplyfreeshipping() {
		return applyfreeshipping;
	}

	public void setApplyfreeshipping(String applyfreeshipping) {
		this.applyfreeshipping = applyfreeshipping;
	}

}



```
