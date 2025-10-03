# ShippingZoneAction.java

## Review

## 1. Summary  

`ShippingZoneAction` is a Struts‑style action that manages the shipping‑zone configuration of a merchant.  
* **Purpose** – Load, display, and persist the merchant’s shipping mode (domestic vs. international) and any zone exclusions.  
* **Key components**  
  * `displayZones()` – retrieves the current shipping configuration and sets it in the request for the JSP.  
  * `saveZones()` – validates the submitted zone type, updates the configuration, and persists any excluded zones.  
  * `overwriteCountriesExclusions()` – helper that writes the comma/semicolon‑delimited list of excluded zones back to the database.  
  * `updateShippingZonesAndCostsForDomestic()` – triggers a service call that rebuilds the zone/cost tables for a domestic shipper.  
* **Frameworks / libraries** –  Struts (via `BaseAction`), Apache Log4j, and a custom service layer (`MerchantService`, `ShippingService`).  The code relies heavily on custom constants from `ShippingConstants` and `ProfileConstants`.  

## 2. Detailed Description  

### Flow of execution  

| Step | Method | What happens |
|------|--------|--------------|
| **Initialization** | `displayZones()` |  * Sets page title. <br> * Pulls `Context` from session. <br> * Requests the merchant’s `SHP_ZOONES` configuration. |
| **Loading existing config** | `displayZones()` |  * If a configuration exists, iterates over all keys to find: <br>   - `MODULE_SHIPPING_ZONES_SHIPPING` → current zone type (domestic/international). <br>   - `MODULE_SHIPPING_ZONES_SKIPPED` → list of excluded zones. <br> * Stores the results as request attributes. |
| **No config found** | `displayZones()` |  * Calls `updateShippingZonesAndCostsForDomestic()` to create default domestic zone tables. <br> * Creates a `MerchantConfiguration` for “real‑time quotes” and persists it. |
| **Submitting changes** | `saveZones()` |  * Retrieves the current configuration (if any). <br> * Compares the submitted `shippingzone` with the stored value. <br> * If different: <br>   - updates the value, sets `lastModified`. <br>   - If switching to domestic → cleans out excluded zones and rebuilds domestic tables. <br>   - If switching to international → updates exclusions only. <br> * If the key does not exist → creates a new `MerchantConfiguration`. <br> * Persists excluded zones via `overwriteCountriesExclusions()`. <br> * Sets a success message. |
| **Helper methods** | `overwriteCountriesExclusions()` |  * Builds a single string from the `excludezones` list (semicolon separated). <br> * Persists or updates the `MODULE_SHIPPING_ZONES_SKIPPED` configuration. |
| | `updateShippingZonesAndCostsForDomestic()` |  * Delegates to `ShippingService.updateShippingZonesAndCostsForDomestic()` and stores the current zone type. |

### Assumptions & constraints  

* The session always contains a `ProfileConstants.context` attribute.  
* `Context` exposes `getMerchantid()` and `getCountryid()` – required for service calls.  
* All configuration keys are unique and stored as `MerchantConfiguration` entities.  
* The application uses `StringUtil.buildMultipleValueLine()` to serialize a list into a delimited string.  

### Architectural choices  

* **Service‑Layer** – business logic resides in `MerchantService`/`ShippingService`.  
* **DAO‑like pattern** – `MerchantConfiguration` is persisted via a generic `saveOrUpdateMerchantConfiguration()` method.  
* **Action‑based** – The code follows a Struts‑1 style “execute‑by‑method” approach.  

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs / Side‑effects |
|--------|---------|--------|------------------------|
| `displayZones()` | Load current shipping zone settings; populate request for the JSP. | None (reads from session). | Sets `shippingzone` and `zoonesskipped` request attributes; returns `SUCCESS`. |
| `saveZones()` | Persist submitted shipping zone and excluded zones. | Uses `shippingzone` (from form) and `excludezones` (from form). | Updates database, sets request attributes, adds a success message, returns `SUCCESS`. |
| `overwriteCountriesExclusions()` | Write the `excludezones` list into the `MODULE_SHIPPING_ZONES_SKIPPED` configuration. | None (uses instance field `excludezones`). | Persists configuration; updates `lastModified`. |
| `updateShippingZonesAndCostsForDomestic()` | Re‑build domestic zone/cost tables via the `ShippingService`. | None (uses session context). | Calls service; sets `shippingzone` to domestic. |
| `getShippingzone() / setShippingzone()` | Accessor for `shippingzone`. | `String`. | Simple getter/setter. |
| `getExcludezones() / setExcludezones()` | Accessor for `excludezones` list. | `List`. | Simple getter/setter. |

### Reusable / Utility methods  

* `StringUtil.buildMultipleValueLine()` – Serialises a list into a delimited string.  
* `MessageUtil.addErrorMessage()/addMessage()` – Standard Struts message handling.  

## 4. Dependencies  

| Library / API | Category | Notes |
|---------------|----------|-------|
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.central.*` | Custom | BaseAction, Context, constants. |
| `com.salesmanager.core.*` | Custom | Service factory, entity, util classes. |
| `java.util` classes | Standard | Collections, Date, StringTokenizer. |

*No external web frameworks beyond Struts 1 are required; the code is pure Java‑EE.*  

## 5. Additional Notes  

### Strengths  

* Clear separation between action logic and service/business layer.  
* Consistent use of configuration constants to avoid “magic strings”.  

### Weaknesses & Edge Cases  

1. **Raw types** – `List excludezones = new ArrayList();` and the generic-less `Map` usage make the code prone to `ClassCastException` and hinder compile‑time type safety.  
2. **Deprecated APIs** – `StringTokenizer` and `java.util.Date` should be replaced with `String.split()` and the `java.time` API.  
3. **Potential NPEs** –  
   * `ctx` may be `null` if the session has no `ProfileConstants.context`.  
   * `shipping` may remain `null` when the configuration map is empty, which is then stored as a request attribute.  
4. **Error handling** – All exceptions are caught and a generic “technical error” is shown, but the stack trace is logged. This hides the root cause from the user and may make debugging harder.  
5. **Duplicate `setConfigurationKey` call** – In `saveZones()` the `configurationKey` is set twice, the second call is redundant.  
6. **No validation** – The submitted `shippingzone` and `excludezones` are not validated (e.g., for allowed values or duplicates).  
7. **Concurrency** – The action is stateless, but shared session attributes (`Context`) could lead to race conditions if accessed concurrently.  
8. **Hard‑coded constants** – Strings such as `"SHP_ZOONES"` and `"MODULE_SHIPPING_ZONES_SKIPPED"` are hard‑coded in the action; if these change the code must be updated.  

### Suggested Improvements  

* **Generics** – Replace raw collections with parameterised types (`List<String>`, `Map<String, MerchantConfiguration>`).  
* **Refactor string handling** – Use `String.join()` / `String.split()` and `StringTokenizer` only for legacy code.  
* **Modern date/time** – Switch to `java.time.Instant` / `LocalDateTime` and let the persistence layer handle conversion.  
* **Validation layer** – Introduce a validator (Struts `Validator` or custom) to enforce allowed zone values and unique exclusions.  
* **Better error handling** – Differentiate between user errors and system errors; propagate exceptions to a global error handler.  
* **Unit tests** – Write JUnit tests for the action methods, mocking the service layer.  
* **Remove duplicated `setConfigurationKey`** – Keep only one call.  
* **Encapsulate configuration keys** – Expose them via an enum or a dedicated constants class to avoid typos.  

Overall, the class fulfills its intended role but would benefit from modernising the codebase, tightening type safety, and improving robustness.

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
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.StringUtil;

/**
 * First Page invoked in Shipping Handles Shipping Configuration
 * 
 * @author Administrator
 * 
 */
public class ShippingZoneAction extends BaseAction {

	private Logger log = Logger.getLogger(ShippingZoneAction.class);
	public String shippingzone;
	private List excludezones = new ArrayList();

	/**
	 * Firts Method to invoke from the shipping menu, will set default shipping
	 * values
	 * 
	 * @return
	 * @throws Exception
	 */
	public String displayZones() throws Exception {

		
		super.setPageTitle("label.shipping.title");
		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			// ** New method
			ConfigurationRequest requestvo = new ConfigurationRequest(
					merchantid.intValue(), true, "SHP_ZOONES");
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationResponse responsevo = mservice
					.getConfiguration(requestvo);
			Map config = responsevo.getMerchantConfigurations();

			String shipping = null;// type of shipping
			Map szones = new HashMap();// shipping zones

			if (config == null || config.size() == 0) {// Nothing configured
														// yet, set default
														// values to national

				/**
				 * INITIAL SHIPPING DEFAULT VALUES
				 */
				this.updateShippingZonesAndCostsForDomestic(); // set to
																// domestic
				// set display real time shipping estimate
				MerchantConfiguration quoteDisplay = new MerchantConfiguration();
				quoteDisplay
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES);
				quoteDisplay.setConfigurationValue1(String
						.valueOf(ShippingConstants.DISPLAY_RT_QUOTE_TIME));
				quoteDisplay.setConfigurationValue2(String
						.valueOf(ShippingConstants.ALL_QUOTES_DISPLAYED));
				quoteDisplay.setDateAdded(new Date());
				quoteDisplay.setLastModified(new Date());
				quoteDisplay.setMerchantId(super.getContext().getMerchantid());

				mservice.saveOrUpdateMerchantConfiguration(quoteDisplay);

			} else {
				// Iterator it = config.iterator();
				Iterator it = config.keySet().iterator();
				while (it.hasNext()) {
					// MerchantConfiguration m =
					// (MerchantConfiguration)it.next();
					String key = (String) it.next();
					// String key = m.getConfigurationKey();
					MerchantConfiguration m = (MerchantConfiguration) config
							.get(key);
					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING)) {// national
																						// or
																						// international
						shipping = m.getConfigurationValue();
					}
					if (key
							.equals(ShippingConstants.MODULE_SHIPPING_ZONES_SKIPPED)) {// zones
																						// where
																						// shipping
																						// occurs
						String skipped = m.getConfigurationValue();
						StringTokenizer st = new StringTokenizer(skipped, ";");
						while (st.hasMoreTokens()) {
							String token = st.nextToken();
							szones.put(token, token);
						}
					}
				}
			}

			super.getServletRequest().setAttribute("shippingzone", shipping);
			super.getServletRequest().setAttribute("zoonesskipped", szones);

		} catch (Exception e) {

			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);
		}

		return SUCCESS;
	}

	/**
	 * Saves the shipping zone (domestic or international)
	 */
	public String saveZones() throws Exception {

		MerchantConfiguration config = null;
		super.setPageTitle("label.shipping.title");
		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			ConfigurationRequest requestvo = new ConfigurationRequest(
					merchantid.intValue(), false,
					ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING);
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationResponse responsevo = mservice
					.getConfiguration(requestvo);
			Map configs = responsevo.getMerchantConfigurations();

			if (configs != null) {
				config = responsevo
						.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING);
			}

			// international or domestic

			java.util.Date dt = new java.util.Date();

			if (config != null) {
				// zone is different than the one configured
				if (!config.getConfigurationValue().equals(
						this.getShippingzone())) {// submit a different value
					config.setConfigurationValue(this.getShippingzone());
					config.setLastModified(new java.util.Date(dt.getTime()));

					// only if the user switch from international to domestic
					if (this.getShippingzone().equals(
							ShippingConstants.DOMESTIC_SHIPPING)) {

						mservice
								.cleanConfigurationKey(
										ShippingConstants.MODULE_SHIPPING_ZONES_SKIPPED,
										merchantid);
						this.updateShippingZonesAndCostsForDomestic();
					} else {// else if international, overwrite country
							// exclusions
						this.overwriteCountriesExclusions();
					}
					mservice.saveOrUpdateMerchantConfiguration(config);
				} else if (config.getConfigurationValue().equals(
						ShippingConstants.INTERNATIONAL_SHIPPING)) {
					this.overwriteCountriesExclusions();
				}
			} else {

				// if shipping domestic, create an entry in zone_countries and
				// zone_costs
				if (this.getShippingzone().equals(
						ShippingConstants.DOMESTIC_SHIPPING)) {
					this.updateShippingZonesAndCostsForDomestic();
				} else {// else if international, check for any country
						// exclusion
					this.overwriteCountriesExclusions();
				}

				// create an entry for zone_shipping
				config = new MerchantConfiguration();
				config
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING);
				config.setDateAdded(new java.util.Date(dt.getTime()));
				config.setLastModified(new java.util.Date(dt.getTime()));
				config
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING);
				config.setConfigurationValue(this.getShippingzone());
				config.setMerchantId(ctx.getMerchantid());
				config.setConfigurationModule("");
				mservice.saveOrUpdateMerchantConfiguration(config);

			}
			super.getServletRequest().setAttribute("shippingzone",
					this.getShippingzone());

			Map szones = new HashMap();
			if (this.getExcludezones() != null
					&& this.getExcludezones().size() > 0) {
				Iterator ezit = this.getExcludezones().iterator();
				while (ezit.hasNext()) {
					String ezvalue = (String) ezit.next();
					szones.put(ezvalue, ezvalue);
				}
			}
			super.getServletRequest().setAttribute("zoonesskipped", szones);
			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {

			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);

		}

		return SUCCESS;
	}

	private void overwriteCountriesExclusions() throws Exception {

		java.util.Date dt = new java.util.Date();

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		int countryid = ctx.getCountryid();

		MerchantConfiguration excl = null;

		ConfigurationRequest requestvo = new ConfigurationRequest(merchantid
				.intValue(), false,
				ShippingConstants.MODULE_SHIPPING_ZONES_SKIPPED);
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		ConfigurationResponse responsevo = mservice.getConfiguration(requestvo);
		Map configs = responsevo.getMerchantConfigurations();

		if (configs != null) {
			excl = responsevo
					.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_ZONES_SKIPPED);
		}

		List exclusions = this.getExcludezones();

		if (exclusions != null & exclusions.size() > 0) {

			String exclusionlinebuffer;
			exclusionlinebuffer = StringUtil.buildMultipleValueLine(exclusions);

			if (excl != null) {
				excl.setLastModified(new java.util.Date(dt.getTime()));
				excl.setConfigurationValue(exclusionlinebuffer);
				excl.setConfigurationModule("");
				mservice.saveOrUpdateMerchantConfiguration(excl);
			} else {
				excl = new MerchantConfiguration();
				excl
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_ZONES_SKIPPED);
				excl.setMerchantId(merchantid);
				excl.setConfigurationValue(exclusionlinebuffer.toString());
				excl.setConfigurationModule("");
				mservice.saveOrUpdateMerchantConfiguration(excl);
			}
		}
	}

	private void updateShippingZonesAndCostsForDomestic() throws Exception {

		ShippingService sservice = (ShippingService) ServiceFactory
				.getService(ServiceFactory.ShippingService);

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);

		sservice.updateShippingZonesAndCostsForDomestic(ctx.getMerchantid(),
				ctx.getCountryid());

		this.setShippingzone(ShippingConstants.DOMESTIC_SHIPPING);
	}

	public String getShippingzone() {
		return shippingzone;
	}

	public void setShippingzone(String shippingzone) {
		this.shippingzone = shippingzone;
	}

	public List getExcludezones() {
		return excludezones;
	}

	public void setExcludezones(List excludezones) {
		this.excludezones = excludezones;
	}
}



```
