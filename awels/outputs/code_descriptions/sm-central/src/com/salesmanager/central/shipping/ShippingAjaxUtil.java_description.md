# ShippingAjaxUtil.java

## Review

## 1. Summary

`ShippingAjaxUtil` is a helper class that exposes two Ajax‑style methods for enabling or disabling a per‑country shipping estimate configuration for a merchant.  
The configuration is stored as a single `MerchantConfiguration` record whose value (`configurationValue1`) is a string that encodes a list of `index:minDays;maxDays` entries separated by “|”. The methods read the current configuration, modify the relevant entry (or create it if it does not exist), and persist the updated configuration back to the database.  

Key components  
| Component | Role |
|-----------|------|
| `MerchantService` (via `ServiceFactory`) | Fetches and updates the `MerchantConfiguration` record. |
| `ConfigurationRequest/Response` | Request/response objects used by `MerchantService` to fetch the configuration. |
| `ShippingConstants` | Holds the module key (`MODULE_SHIPPING_ESTIMATE_BYCOUNTRY`) and a constant for the maximum number of ranges (`MAX_PRICE_RANGE_COUNT`). |
| `Context`, `ProfileConstants` | Provide the current merchant id and session locale. |
| `LabelUtil` | Generates internationalised status messages. |

The class uses **DWR** (`WebContextFactory`) to obtain the current `HttpServletRequest`, indicating it is designed to be called from a web front‑end.

---

## 2. Detailed Description

### Flow of execution

1. **Entry point** – The caller invokes either `enableShippingEstimate(int index, int mindays, int maxdays)` or `disableShippingEstimate(int index)` from the front‑end.
2. **Request context** – The method obtains the current HTTP request/session via `WebContextFactory`.  
   - It pulls the merchant id from the `Context` stored in the session.  
   - It also fetches the `Locale` to localise status messages.
3. **Validation** – In `enableShippingEstimate` the code ensures `maxdays >= mindays`; otherwise it forces `mindays = 1`. No validation is performed on `index` or `maxdays` values beyond this.
4. **Read current configuration** – A `ConfigurationRequest` is built and sent to `MerchantService`. The returned `ConfigurationResponse` holds the existing `MerchantConfiguration` for the key `MODULE_SHIPPING_ESTIMATE_BYCOUNTRY`.
5. **Modify / create entry** –  
   *If a configuration exists and contains a non‑blank value:*  
   - The value string is tokenised on “|”.  
   - Each token is examined; the one whose position matches `index` is replaced with the new `mindays/maxdays` pair, the rest are left unchanged.  
   *If the value is blank or the configuration does not exist:*  
   - A helper (`buildInitialLine`) constructs a string of `MAX_PRICE_RANGE_COUNT` tokens where the token at `index` contains the new pair and all others contain “*”.
6. **Persist** – The updated `MerchantConfiguration` (or a new one if none existed) is written back using `MerchantService.saveOrUpdateMerchantConfiguration()`.
7. **Return status** – A simple HTML string indicating success or error is returned to the caller.  

`disableShippingEstimate` follows the same pattern but, when a matching token is found, it replaces it with a single asterisk (“*”) rather than a numeric range.

### Assumptions / constraints

- The `index` parameter is 1‑based and expected to be within `1 … MAX_PRICE_RANGE_COUNT`. No runtime check is performed.  
- The configuration string is always formatted as `index:mindays;maxdays` per token, separated by `|`.  
- A disabled entry is represented by a token that is simply “*”.  
- The system uses DWR to expose these methods; they are therefore expected to be called asynchronously from a web page.  
- The code assumes that `MerchantService` will correctly insert or update the `MerchantConfiguration` record when called.  

### Architectural observations

- The class is a thin service layer wrapper around the persistence API; it performs no business logic beyond string manipulation.  
- String handling is manual (using `StringTokenizer` and `StringBuffer`), which is error‑prone and not locale‑safe.  
- The status messages are built in‑line; a better approach would be to return a structured JSON object and let the front‑end format the UI.  

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `public String enableShippingEstimate(int index, int mindays, int maxdays)` | Adds or updates the shipping estimate range for a specific country index. | `index` – 1‑based position in the list. <br> `mindays` – minimum days. <br> `maxdays` – maximum days. | HTML string with status icon and message. | Writes/updates a `MerchantConfiguration` record. |
| `public String disableShippingEstimate(int index)` | Marks the shipping estimate for the given index as disabled by replacing the entry with `*`. | `index` – 1‑based position. | HTML string with status icon and message. | Updates the `MerchantConfiguration` record. |
| `private String buildInitialLine(int index, String line)` | Builds a full configuration string with `MAX_PRICE_RANGE_COUNT` tokens, placing `line` at the given `index` and `*` elsewhere. | `index` – target position. <br> `line` – string to insert (e.g. `"index:mindays;maxdays"`). | Configuration string. | None. |

All public methods return a plain string that the caller is expected to inject into the DOM. They perform all error handling internally, logging the exception and replacing the message with a generic “technical error” string.

---

## 4. Dependencies

| Library / Framework | Purpose | Standard / 3rd‑party |
|---------------------|---------|----------------------|
| `javax.servlet.http.*` | Access to `HttpServletRequest` / `HttpSession`. | Java EE / Servlet API |
| `org.apache.commons.lang.StringUtils` | String utilities (`isBlank`). | Apache Commons Lang |
| `org.apache.log4j.Logger` | Logging. | Log4j (3rd‑party) |
| `uk.ltd.getahead.dwr.WebContextFactory` | DWR integration (access to current HTTP request). | DWR (3rd‑party) |
| `com.salesmanager.central.*` | Project‑specific context and constants. | In‑house |
| `com.salesmanager.core.*` | Core services and entities (`MerchantService`, `MerchantConfiguration`, etc.). | In‑house |

There are no database or ORM frameworks directly referenced; persistence is mediated by `MerchantService`.

---

## 5. Additional Notes & Recommendations

### Edge cases & potential bugs
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Index out of bounds** – No check that `index` lies between `1` and `MAX_PRICE_RANGE_COUNT`. | Crashes or incorrect configuration if an out‑of‑range value is passed. | Validate and clamp `index`, or return an error message. |
| **maxdays < mindays** – The code simply sets `mindays = 1`. | Potentially hides the real problem and produces misleading data. | Either swap the values or reject the request with a clear error. |
| **Disabled entry format** – The `disableShippingEstimate` method writes `*` for a disabled token, but the `enableShippingEstimate` logic treats any non‑blank token as a valid range. | After disabling, re‑enabling may fail to find the original token and insert a duplicate. | Keep the original token format and use a flag field, or store disabled entries with a distinct pattern that can be parsed correctly. |
| **Confusing reuse of `conf`** – When the value is blank, the code creates a *new* `MerchantConfiguration` and discards the old one. | Duplicate or orphaned configuration records may be created. | Reuse the existing `conf` instance or explicitly delete the old record if necessary. |
| **String parsing is fragile** – Uses `StringTokenizer` and manual loops. | Easy to break if the format changes; hard to read. | Replace with `String.split("\\|")` and use `StringBuilder`. |
| **No transactional guarantees** – Two concurrent requests could race and overwrite each other. | Lost updates. | Use a database transaction or a lock around the read‑modify‑write cycle. |
| **Returning raw HTML** – The caller must inject the string into the page. | Tight coupling between server and UI; difficult to test. | Return a JSON object (`{ success:true, message:"..." }`) and let the front‑end handle presentation. |
| **Hard‑coded status icons** – `<div class="icon-ok">` and `<div class="icon-error">`. | If UI changes, the HTML here becomes stale. | Use a status code or enum instead. |

### Suggested improvements
1. **Refactor string handling**  
   * Use `String.split` and `StringBuilder`.  
   * Store the configuration as a JSON array or a dedicated database table for ranges instead of a serialized string.
2. **Input validation** – Verify all numeric arguments and that `index` is within bounds.
3. **Error reporting** – Throw a checked exception or return a structured error instead of hiding the exception in a generic message.
4. **Unit tests** – Add tests for each method covering enabled, disabled, boundary, and error scenarios.
5. **Decouple UI** – Return a DTO or JSON instead of raw HTML; the front‑end should decide how to display success or error.
6. **Logging** – Log the exact exception stack trace and include contextual data (merchant id, index, days).  
7. **Concurrency** – Synchronise the update or leverage database constraints (unique key on merchant id + module key) to prevent duplicates.

Overall, the class achieves its intended purpose but would benefit from a cleaner design, stronger validation, and a clearer separation between business logic and presentation concerns.

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

import java.util.Locale;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.LabelUtil;

public class ShippingAjaxUtil {

	private Logger log = Logger.getLogger(ShippingAjaxUtil.class);

	public String enableShippingEstimate(int index, int mindays, int maxdays) {

		// COUNTRYCODE;DAYS|COUNTRYCODE;DAYS

		// get actual configuration

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();

		HttpSession session = req.getSession();

		// validation
		if (maxdays < mindays) {
			mindays = 1;
		}

		Context ctx = (Context) req.getSession().getAttribute(
				ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		Locale locale = (Locale) session.getAttribute("WW_TRANS_I18N_LOCALE");

		String message = new StringBuffer().append("<div class=\"icon-ok\">")
				.append(
						LabelUtil.getInstance().getText(locale,
								"message.confirmation.success")).append(
						"</div>").toString();

		try {

			ConfigurationRequest request = new ConfigurationRequest(merchantid,
					false, ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY);

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationResponse res = mservice.getConfiguration(request);

			StringBuffer newLine = new StringBuffer().append(index).append(":")
					.append(mindays).append(";").append(maxdays);

			MerchantConfiguration conf = res
					.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY);
			if (conf != null) {

				String value = conf.getConfigurationValue1();

				if (!StringUtils.isBlank(value)) {

					String line = conf.getConfigurationValue1();

					StringTokenizer cvtk = new StringTokenizer(value, "|");// index:<MINCOST>;<MAXCOST>|
					int count = 1;
					StringBuffer newLineBuffer = new StringBuffer();
					while (cvtk != null && cvtk.hasMoreTokens()) {
						String lnToken = cvtk.nextToken();
						if (count == index) {
							newLineBuffer.append(newLine);
						} else {
							newLineBuffer.append(lnToken);
						}
						if (count < ShippingConstants.MAX_PRICE_RANGE_COUNT) {
							newLineBuffer.append("|");
						}
						count++;
					}

					conf.setConfigurationValue1(newLineBuffer.toString());
					conf.setConfigurationValue("true");

				} else {
					String line = buildInitialLine(index, newLine.toString());
					conf = new MerchantConfiguration();
					conf.setMerchantId(ctx.getMerchantid());
					conf
							.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY);
					conf.setConfigurationValue("true");
					conf.setConfigurationValue1(line);

				}

			} else {
				String line = buildInitialLine(index, newLine.toString());
				conf = new MerchantConfiguration();
				conf.setMerchantId(ctx.getMerchantid());
				conf
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY);
				conf.setConfigurationValue("true");
				conf.setConfigurationValue1(line);

			}

			mservice.saveOrUpdateMerchantConfiguration(conf);

		} catch (Exception e) {
			log.error(e);
			message = new StringBuffer().append("<div class=\"icon-error\">")
					.append(
							LabelUtil.getInstance().getText(locale,
									"errors.technical")).append("</div>")
					.toString();
		}

		return message;

	}

	public String disableShippingEstimate(int index) {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();

		HttpSession session = req.getSession();

		Context ctx = (Context) req.getSession().getAttribute(
				ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		Locale locale = (Locale) session.getAttribute("WW_TRANS_I18N_LOCALE");

		String message = new StringBuffer().append("<div class=\"icon-ok\">")
				.append(
						LabelUtil.getInstance().getText(locale,
								"message.confirmation.success")).append(
						"</div>").toString();

		try {

			ConfigurationRequest request = new ConfigurationRequest(merchantid,
					false, ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY);

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationResponse res = mservice.getConfiguration(request);

			MerchantConfiguration conf = res
					.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY);
			if (conf != null) {

				String value = conf.getConfigurationValue1();

				if (!StringUtils.isBlank(value)) {

					String line = conf.getConfigurationValue1();

					StringTokenizer cvtk = new StringTokenizer(value, "|");// index:<MINCOST>;<MAXCOST>|
					int count = 1;
					StringBuffer newLineBuffer = new StringBuffer();
					while (cvtk != null && cvtk.hasMoreTokens()) {
						String lnToken = cvtk.nextToken();
						if (count == index) {
							newLineBuffer.append("*");
						} else {
							newLineBuffer.append(lnToken);
						}
						if (count < ShippingConstants.MAX_PRICE_RANGE_COUNT) {
							newLineBuffer.append("|");
						}
						count++;
					}

					conf.setConfigurationValue1(newLineBuffer.toString());
					conf.setConfigurationValue("true");

					mservice.saveOrUpdateMerchantConfiguration(conf);

				}

			}

		} catch (Exception e) {
			log.error(e);
			message = new StringBuffer().append("<div class=\"icon-error\">")
					.append(
							LabelUtil.getInstance().getText(locale,
									"errors.technical")).append("</div>")
					.toString();
		}

		return message;

	}

	private String buildInitialLine(int index, String line) {

		StringBuffer lineBuffer = new StringBuffer();
		for (int i = 1; i <= ShippingConstants.MAX_PRICE_RANGE_COUNT; i++) {
			if (i == index) {
				lineBuffer.append(line);
			} else {
				lineBuffer.append("*");
			}
			if (i < ShippingConstants.MAX_PRICE_RANGE_COUNT) {
				lineBuffer.append("|");
			}
		}
		return lineBuffer.toString();
	}

}



```
