# ShippingRatesAction.java

## Review

## 1. Summary  

`ShippingRatesAction` is a Struts‑style controller that handles the display and persistence of shipping configuration for a merchant in the **SalesManager** application.  
Key responsibilities:

| Component | Role |
|-----------|------|
| `MerchantService` | CRUD operations on merchant configurations |
| `ReferenceService` | CRUD operations on dynamic labels (i18n) |
| `ServiceFactory` | Factory for the above services |
| `displayQuote / displayQuoteOrder` | UI‑lookup tables for “show shipping quote” options |
| `languages / reflanguages` | Language support for dynamic labels |
| `shippingText` | Holds the localized “no‑real‑time‑quote” shipping notice |

The class uses **log4j**, **Apache Commons Lang** and a custom `LabelUtil` for i18n.

---

## 2. Detailed Description  

### 2.1 Initialization (`prepareLists`)  
* Sets the page title.  
* Builds the two lookup maps (`displayQuote` and `displayQuoteOrder`) using `LabelUtil`.  
* Loads the merchant’s supported languages into `languages` and a numeric index map (`reflanguages`).  

### 2.2 Loading existing data (`displayShippingModules`)  
* Reads all merchant configuration values that start with `"SHP_"`.  
* Parses specific keys (e.g. `MODULE_SHIPPING_INDIC_COUNTRIES_COSTS`, `MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES`, etc.).  
* Stores parsed values as request attributes so the JSP can render the configuration UI.  

### 2.3 Persisting data (`saveMethodDetails`)  
* Re‑builds the language lookup.  
* Retrieves or creates a `MerchantConfiguration` for `MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES`.  
* Persists the two numeric values (`displayQuoteDeliveryTime`, `quoteDisplayType`).  
* Handles the dynamic label for the “no‑quote” text:
  * If real‑time quotes are shown → delete the label.  
  * If real‑time quotes are hidden → create or update a `DynamicLabel` with a description per language.  
  * Performs i18n validation (non‑empty text).  
  * Removes descriptions that are no longer present.  

### 2.4 Cleanup (`cleanupkey`)  
Deletes a specific merchant configuration key.

### 2.5 Dependencies & Runtime  
* The action relies on a HTTP session containing a `ProfileConstants.context`.  
* All services are obtained via a static `ServiceFactory`.  
* Uses a lot of raw collections (`Map`, `List`) – no generics.  
* Exception handling is broad (catches `Exception`) and mostly logs and returns `INPUT`.

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs / Side‑effects |
|--------|---------|--------|------------------------|
| `cleanupkey(String key)` | Removes a merchant config entry. | `key` – configuration key to delete | Deletes via `MerchantService.cleanConfigurationKey` |
| `prepareDescriptions()` | Loads the shipping‑fee section labels into `shippingText`. | None | Fills `shippingText` list; sets up i18n descriptions |
| `prepareLists()` | Builds UI lookup maps and language collections. | None | Initializes `displayQuote`, `displayQuoteOrder`, `languages`, `reflanguages` |
| `saveMethodDetails()` | Handles form submission for shipping method configuration. | None (uses getters/setters) | Persists config and dynamic label; validates `shippingText`; returns `SUCCESS` or `INPUT` |
| `displayShippingModules()` | Prepares data for the shipping‑module configuration page. | None | Sets request attributes for JSP; returns `SUCCESS` |
| Getters / Setters | Standard JavaBeans properties (`displayQuoteDeliveryTime`, `quoteDisplayType`, `displayQuote`, `displayQuoteOrder`, `languages`, `shippingText`). | N/A | Return/modify internal state |

**Reusable utilities**  
* `LabelUtil.getInstance().getText(locale, key)` – i18n lookup.  
* `StringUtils.isBlank` – null‑safe string check.

---

## 4. Dependencies  

| External Library | Purpose | Standard / 3rd‑Party |
|------------------|---------|----------------------|
| `org.apache.commons.lang.StringUtils` | Null‑safe string operations | 3rd‑Party (Apache Commons Lang) |
| `org.apache.log4j.Logger` | Logging | 3rd‑Party (Log4j) |
| `com.salesmanager.central.*` | BaseAction, Profile constants, context | In‑house |
| `com.salesmanager.core.*` | Service interfaces, constants, entities | In‑house |
| `com.salesmanager.core.util.LabelUtil` | Internationalization lookup | In‑house |
| `com.salesmanager.core.util.LanguageUtil` | Language code helper | In‑house |

No obvious platform‑specific dependencies – the code is pure Java.  

---

## 5. Additional Notes & Recommendations  

### 5.1 Coding Practices  
* **Generics** – All raw types (`Map`, `List`, `Set`) should be parameterised to avoid unchecked casts.  
* **Collections** – Use `Map<Integer, String>` for the lookup maps and `List<String>` for `shippingText`.  
* **Iterator Loops** – Prefer enhanced for‑loops or streams; current `Iterator` style is verbose.  
* **StringTokenizer** – Replace with `String.split(";")`; Tokenizer is legacy.  
* **Exception Handling** – Catch specific exceptions (`NumberFormatException`, `ServiceException`, etc.) and propagate meaningful messages.  
* **Validation** – Use a dedicated validation framework (e.g. Struts2 `Validator` or Spring MVC `@Valid`) rather than manual field‑error logic.  
* **Logging** – Include the merchant ID and the key in log messages for better traceability.  

### 5.2 Functional Issues  
* **`reflanguages`** – It is a map from an index to a language ID. The code assumes the indices are contiguous from 0. This is fragile if languages are added/removed. Using a `List<Integer>` would be clearer.  
* **`prepareDescriptions()`** – The method iterates over `reflanguages.size()` but accesses `reflanguages.get(icount)`. If the map is not contiguous, a `null` value will be used leading to a `NullPointerException`.  
* **`shippingText` Validation** – The code validates that each element is non‑blank but does not verify that the list size matches `reflanguages.size()`. Out‑of‑bounds errors may occur.  
* **Duplicate Parsing** – `displayShippingModules()` performs many repetitive `if (c.getConfigurationValue1() != null)` checks. A helper method could centralise parsing of integer values.  
* **Dynamic Label Deletion** – When real‑time quotes are enabled, the code deletes the entire label (`rservice.deleteDynamicLabel(currentLabel);`). If the label contains other descriptions, they will be lost.  
* **Thread‑Safety** – The action class is a Struts action and is instantiated per request, so thread‑safety is not an issue, but the use of mutable static collections would be problematic if introduced.  

### 5.3 Security & Robustness  
* **Session Handling** – `Context ctx = (Context) super.getServletRequest().getSession().getAttribute(ProfileConstants.context);` assumes the attribute is always present. A `NullPointerException` will be thrown if the session has expired.  
* **Input Sanitisation** – The shipping text is stored directly; consider escaping to avoid XSS in the UI.  
* **Error Reporting** – The generic `setTechnicalMessage()` masks the root cause for the user; a more granular error page could improve debugging.  

### 5.4 Potential Enhancements  
1. **Refactor to Service Layer** – Move business logic (configuration parsing, dynamic label handling) into dedicated service classes.  
2. **Use DTOs** – Create a `ShippingConfigurationDTO` to encapsulate all config values, making the action leaner.  
3. **Unit Tests** – Add tests for the parsing logic and dynamic label creation; mock the services.  
4. **Internationalisation** – Use a `Map<Language, String>` for `shippingText` to avoid index‑based lookups.  
5. **Config Key Constants** – Replace hard‑coded strings (`"SHP_"`, `"MODULE_SHIPPING_..."`) with a dedicated constants class.  

---

**Overall**  
The class accomplishes its purpose but would benefit significantly from modern Java practices, better separation of concerns, and tighter validation. Addressing the raw types, improving exception handling, and decoupling business logic from the action layer would make the code more maintainable, testable, and robust.

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

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.StringTokenizer;
import java.util.TreeMap;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;

public class ShippingRatesAction extends BaseAction {

	private Logger log = Logger.getLogger(ShippingRatesAction.class);

	private int displayQuoteDeliveryTime = ShippingConstants.NO_DISPLAY_RT_QUOTE_TIME;
	private int quoteDisplayType = 0;

	Map displayQuote;// for displaying shipping stuff
	Map displayQuoteOrder;// for displaying shipping stuff

	private Collection<Language> languages;// used in the page as an index
	private Map<Integer, Integer> reflanguages = new HashMap();// reference
																// count -
																// languageId
	private List<String> shippingText = new ArrayList<String>();// text submited

	protected void cleanupkey(String key) throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			ConfigurationRequest requestvo = new ConfigurationRequest(
					merchantid.intValue(), true, "SHP_");
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			mservice.cleanConfigurationKey(key, merchantid);

		} catch (Exception e) {
			log.error(e);
			throw e;

		}
	}

	private void prepareDescriptions() throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		DynamicLabel dynamicLabel = null;
		List labels = (List) rservice.getDynamicLabels(super.getContext()
				.getMerchantid().intValue(),
				LabelConstants.SHIPPING_FEES_SECTION);

		if (labels != null && labels.size() > 0) {
			dynamicLabel = (DynamicLabel) labels.get(0);
		}

		if (dynamicLabel != null) {
			Set dynamicLabelSet = dynamicLabel.getDescriptions();

			Map labelMap = new HashMap();
			Iterator labelSetIterator = dynamicLabelSet.iterator();
			while (labelSetIterator.hasNext()) {
				DynamicLabelDescription description = (DynamicLabelDescription) labelSetIterator
						.next();
				labelMap.put(description.getId().getLanguageId(), description);
			}

			for (int icount = 0; icount < reflanguages.size(); icount++) {
				int langid = (Integer) reflanguages.get(icount);
				DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
						.get(langid);
				if (desc != null) {
					shippingText.add(desc.getDynamicLabelDescription());
				}
			}

		}

	}

	private void prepareLists() throws Exception {

		try {
			
			super.setPageTitle("label.shipping.rates.title");

			displayQuote = new TreeMap();
			displayQuote.put(0, LabelUtil.getInstance().getText(
					super.getLocale(), "label.shipping.displayquote.0"));
			displayQuote.put(1, LabelUtil.getInstance().getText(
					super.getLocale(), "label.shipping.displayquote.1"));

			displayQuoteOrder = new TreeMap();
			displayQuoteOrder.put(0, LabelUtil.getInstance().getText(
					super.getLocale(), "label.shipping.displayquoteorder.0"));
			displayQuoteOrder.put(1, LabelUtil.getInstance().getText(
					super.getLocale(), "label.shipping.displayquoteorder.1"));
			displayQuoteOrder.put(2, LabelUtil.getInstance().getText(
					super.getLocale(), "label.shipping.displayquoteorder.2"));

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore mstore = mservice.getMerchantStore(super.getContext()
					.getMerchantid());

			Map languagesMap = mstore.getGetSupportedLanguages();

			languages = languagesMap.values();// collection reverse the map

			int count = 0;
			Iterator langit = languagesMap.keySet().iterator();
			while (langit.hasNext()) {
				Integer langid = (Integer) langit.next();
				Language lang = (Language) languagesMap.get(langid);
				reflanguages.put(count, langid);
				count++;
			}

		} catch (Exception e) {
			log.error(e);
		}

	}

	/**
	 * Real time method details (options) from the realt time shipping method
	 * list
	 * 
	 * @return
	 */
	public String saveMethodDetails() {

		try {

			boolean hasError = false;

			this.prepareLists();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			// query for an existing value first
			ConfigurationRequest requestvo = new ConfigurationRequest(super
					.getContext().getMerchantid(), false,
					ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES);
			ConfigurationResponse responsevo = mservice
					.getConfiguration(requestvo);

			MerchantConfiguration config = responsevo
					.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES);
			if (config == null) {

				// save merchant configuration
				config = new MerchantConfiguration();
				config
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES);
				config.setMerchantId(super.getContext().getMerchantid());
				config.setDateAdded(new Date());
			}

			config.setConfigurationValue1(String
					.valueOf(displayQuoteDeliveryTime));
			config.setConfigurationValue2(String.valueOf(quoteDisplayType));
			config.setLastModified(new Date());

			// get shipping text
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			DynamicLabel currentLabel = null;
			List labels = (List) rservice.getDynamicLabels(super.getContext()
					.getMerchantid(), LabelConstants.SHIPPING_FEES_SECTION);

			if (labels != null && labels.size() > 0) {
				currentLabel = (DynamicLabel) labels.get(0);
			}

			// if display real time and label != null
			if (displayQuoteDeliveryTime == ShippingConstants.DISPLAY_RT_QUOTE_TIME
					&& currentLabel != null) {

				// delete label
				rservice.deleteDynamicLabel(currentLabel);

			}

			if (displayQuoteDeliveryTime == ShippingConstants.NO_DISPLAY_RT_QUOTE_TIME
					&& shippingText != null && shippingText.size() > 0) {

				// add shipping text description

				// try to get first

				if (currentLabel == null) {// create a new one
					currentLabel = new DynamicLabel();
					currentLabel.setMerchantId(super.getContext()
							.getMerchantid());
					currentLabel
							.setSectionId(LabelConstants.SHIPPING_FEES_SECTION);
					currentLabel.setTitle("--");
				}

				Set dynamicLabelSet = currentLabel.getDescriptions();

				Map labelMap = new HashMap();// existing labels [lang,
												// description]
				if (dynamicLabelSet != null && dynamicLabelSet.size() > 0) {
					Iterator labelSetIterator = dynamicLabelSet.iterator();
					while (labelSetIterator.hasNext()) {
						DynamicLabelDescription description = (DynamicLabelDescription) labelSetIterator
								.next();
						labelMap.put(description.getId().getLanguageId(),
								description);
					}
				}

				HashSet descriptionset = new HashSet();// new set
				Map newMap = new HashMap();

				// I18N validation
				Iterator i = reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					int submitedlangid = (Integer) reflanguages.get(langcount);
					String code = LanguageUtil
							.getLanguageStringCode(submitedlangid);

					// validate text
					String text = (String) this.getShippingText()
							.get(langcount);
					if (StringUtils.isBlank(text)) {
						super.addFieldError("shippingText[" + langcount + "]",
								getText("error.customeshipping.text.required")
										+ " (" + code + ")");
						hasError = true;
					}

					// update existing or create a new one

					DynamicLabelDescription description = null;
					if (labelMap.containsKey(submitedlangid)) {
						description = (DynamicLabelDescription) labelMap
								.get(submitedlangid);
						// dynamicLabelSet.remove(description);
					} else {
						description = new DynamicLabelDescription();
						DynamicLabelDescriptionId id = new DynamicLabelDescriptionId();
						id.setLanguageId(submitedlangid);
						description.setId(id);
						description.setDynamicLabelTitle("shippingText");
					}
					newMap.put(submitedlangid, description);
					description.setDynamicLabelDescription(text);
					descriptionset.add(description);

				}

				if (hasError) {
					return INPUT;
				}

				currentLabel.setDescriptions(descriptionset);
				rservice.saveOrUpdateDynamicLabel(currentLabel);

				// delete unused
				i = reflanguages.keySet().iterator();
				Set deletedSet = new HashSet();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					int submitedlangid = (Integer) reflanguages.get(langcount);
					if (!newMap.containsKey(submitedlangid)) {
						DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
								.get(submitedlangid);
						deletedSet.add(desc);
					}
				}

				if (deletedSet.size() > 0) {
					rservice.deleteDynamicLabelDescriptions(deletedSet);
				}
			}

			mservice.saveOrUpdateMerchantConfiguration(config);

			super.setSuccessMessage();
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

	}

	/**
	 * Displays the page for selecting the shipping company and custom rates
	 * 
	 * @return
	 * @throws Exception
	 */
	public String displayShippingModules() throws Exception {

		try {

			this.prepareLists();
			this.prepareDescriptions();

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			ConfigurationRequest requestvo = new ConfigurationRequest(
					merchantid.intValue(), true, "SHP_");
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationResponse responsevo = mservice
					.getConfiguration(requestvo);
			List config = responsevo.getMerchantConfigurationList();

			String shipping = null;
			Map szones = new HashMap();

			Map domesticservices = new HashMap();
			Map intlservices = new HashMap();

			Map modules = new HashMap();

			if (config != null) {

				// Iterator it = config.iterator();
				Iterator it = config.iterator();
				while (it.hasNext()) {

					MerchantConfiguration c = (MerchantConfiguration) it.next();
					String key = c.getConfigurationKey();

					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_INDIC_COUNTRIES_COSTS)) {// Custom
																								// costs
						if (c.getConfigurationValue1() != null
								&& !c.getConfigurationValue1().equals("")) {
							super.getServletRequest().setAttribute(
									"shippingcountries",
									c.getConfigurationValue1());// countries
						}
						if (c.getConfigurationValue2() != null
								&& !c.getConfigurationValue2().equals("")) {
							super.getServletRequest()
									.setAttribute("shippingcosts",
											c.getConfigurationValue2());// costs
						}
						if (c.getConfigurationValue() != null
								&& !c.getConfigurationValue().equals("")) {
							super.getServletRequest().setAttribute(
									"shippingzoneindicator",
									c.getConfigurationValue());// domestic or
																// international
						}
					}

					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES)) {

						if (!StringUtils.isBlank(c.getConfigurationValue1())) {// display
																				// or
																				// not
																				// quotes
							displayQuoteDeliveryTime = ShippingConstants.NO_DISPLAY_RT_QUOTE_TIME;
							try {
								displayQuoteDeliveryTime = Integer.parseInt(c
										.getConfigurationValue1());

							} catch (Exception e) {
								log
										.error("Display quote is not an integer value ["
												+ c.getConfigurationValue1()
												+ "]");
							}
						}

						if (!StringUtils.isBlank(c.getConfigurationValue2())) {// which
																				// quotes
																				// (all,
																				// less
																				// expensive,
																				// hghest)

							quoteDisplayType = 0;
							try {
								quoteDisplayType = Integer.parseInt(c
										.getConfigurationValue2());

							} catch (Exception e) {
								log
										.error("Display type quote is not an integer value ["
												+ c.getConfigurationValue2()
												+ "]");
							}

						}

					}

					// How do you display shipping quotes
					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME)) {

						if (!StringUtils.isBlank(c.getConfigurationValue1())) {
							modules.put(c.getConfigurationValue1(), c);
						}
					}

					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT)) {

						// PKG
						if (c.getConfigurationValue() != null
								&& !c.getConfigurationValue().equals("")) {

						}
						// DOM
						if (c.getConfigurationValue1() != null
								&& !c.getConfigurationValue1().equals("")) {
							String domestic = c.getConfigurationValue1();
							StringTokenizer st = new StringTokenizer(domestic,
									";");
							while (st.hasMoreTokens()) {
								String token = st.nextToken();
								domesticservices.put(token, token);
							}
						}
						// INTL
						if (c.getConfigurationValue2() != null
								&& !c.getConfigurationValue2().equals("")) {
							String intl = c.getConfigurationValue2();
							StringTokenizer st = new StringTokenizer(intl, ";");
							while (st.hasMoreTokens()) {
								String token = st.nextToken();
								intlservices.put(token, token);
							}
						}
					}

					if (key
							.equals(ShippingConstants.INTERNATIONAL_SHIPPING)) {
						super.getServletRequest().setAttribute("shippingrtdomestic",
								c.getConfigurationValue());
					}
				}
			}

			super.getServletRequest().setAttribute("shippingrtmodule", modules);
			super.getServletRequest().setAttribute("shippingrtdomestic",
					domesticservices);
			super.getServletRequest().setAttribute("shippingrtinternational",
					intlservices);

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
		}

		return SUCCESS;
	}

	public int getDisplayQuoteDeliveryTime() {
		return displayQuoteDeliveryTime;
	}

	public void setDisplayQuoteDeliveryTime(int displayQuoteDeliveryTime) {
		this.displayQuoteDeliveryTime = displayQuoteDeliveryTime;
	}

	public int getQuoteDisplayType() {
		return quoteDisplayType;
	}

	public void setQuoteDisplayType(int quoteDisplayType) {
		this.quoteDisplayType = quoteDisplayType;
	}

	public Map getDisplayQuote() {
		return displayQuote;
	}

	public void setDisplayQuote(Map displayQuote) {
		this.displayQuote = displayQuote;
	}

	public Map getDisplayQuoteOrder() {
		return displayQuoteOrder;
	}

	public void setDisplayQuoteOrder(Map displayQuoteOrder) {
		this.displayQuoteOrder = displayQuoteOrder;
	}

	public Collection<Language> getLanguages() {
		return languages;
	}

	public void setLanguages(Collection<Language> languages) {
		this.languages = languages;
	}

	public List<String> getShippingText() {
		return shippingText;
	}

	public void setShippingText(List<String> shippingText) {
		this.shippingText = shippingText;
	}

}



```
