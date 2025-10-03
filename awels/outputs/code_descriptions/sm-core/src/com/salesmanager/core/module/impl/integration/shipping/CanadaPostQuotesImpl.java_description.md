# CanadaPostQuotesImpl.java

## Review

## 1. Summary

**Purpose**  
The `CanadaPostQuotesImpl` class implements the `ShippingQuotesModule` interface to provide real‑time shipping quotes from Canada Post for orders shipped from Canada or the United States. It constructs an XML request, sends it to a Canada Post web service, parses the XML response, performs currency conversion if needed, and returns a list of `ShippingOption` objects.

**Key Components**  
- **XML Request Builder** – manually constructs the request XML using string concatenation.  
- **HTTP Client** – uses Apache HttpClient (`PostMethod`) to send the request.  
- **XML Parsing** – Apache Commons Digester is used to map XML nodes to a `CanadaPostParsedElements` object.  
- **Currency & Measurement Conversion** – util classes (`CurrencyUtil`, `ShippingUtil`, `CountryUtil`) are leveraged for unit conversion.  
- **Configuration Handling** – `getConfiguration` translates merchant configuration into integration keys/properties.  

**Notable Patterns / Libraries**  
- *Command‑like* approach to building request XML.  
- *Facade* for external service (Canada Post) wrapped in a module.  
- Uses Apache Commons libraries (HttpClient, Digester, StringUtils) and Log4j for logging.  

---

## 2. Detailed Description

### 2.1 Initialization & Configuration
1. `getConfiguration()` is invoked by the framework to populate a `ConfigurationResponse` object with `canadapost-keys` and `canadapost-properties`.  
2. These configuration objects are later retrieved in `getShippingQuote()` via `config.getConfiguration(...)`.

### 2.2 Runtime Flow (`getShippingQuote`)
1. **Validation** – Checks that the order contains packages, that the customer is from Canada (id 38) or US (id 223), and that the locale is supported (en/fr).  
2. **Currency Normalisation** – If the store’s currency is not CAD, the order total is converted to CAD.  
3. **Service Discovery** – Uses `ShippingService` and `CoreModuleService` to obtain the Canada Post endpoint (dev or prod based on properties).  
4. **Request Construction** – Builds a well‑formed XML request by iterating over `PackageDetail` objects, converting units to KG/CM, and appending customer and store shipping information.  
5. **HTTP Interaction** – Sends the request via `PostMethod`. If the response status is not 200, it throws an exception.  
6. **Response Parsing** – Uses a `Digester` configured to map elements under `<ratesAndServicesResponse>` to a `ShippingOption` instance, collecting them in a `CanadaPostParsedElements` object.  
7. **Error Handling** – Checks status code; logs and returns `null` on error.  
8. **Post‑processing** – Applies currency conversion to each option if required, attaches delivery date to the description if configured, and sets the final option price and description.  

### 2.3 Cleanup
`httppost.releaseConnection()` is executed in the `finally` block to free the HTTP connection.

### 2.4 Dependencies & Constraints
- **Assumes** Canada Post expects a specific XML format; any changes in their contract would break the module.  
- **Assumes** the external service is available over HTTPS/TCP; no retry logic is present.  
- **Assumes** the `Country` objects for the merchant store and customer are present in `RefCache`.  
- **Assumes** the `ConfigurationResponse` contains the necessary keys and properties; otherwise the method silently returns `null`.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `getShippingMethodDescription(Locale)` | Return localized description of the shipping method. | `Locale` | `String` | None |
| `getShippingQuote(ConfigurationResponse, BigDecimal, Collection<PackageDetail>, Customer, MerchantStore, Locale)` | Compute real‑time quotes for Canada Post. | See signature | `Collection<ShippingOption>` | Logs, HTTP request, currency conversion |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Translate merchant configuration into integration keys/properties. | `MerchantConfiguration`, `ConfigurationResponse` | Updated `ConfigurationResponse` | Adds entries to `vo` |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Persist configuration changes (unimplemented). | `merchantid`, `ConfigurationResponse`, `HttpServletRequest` | None | TODO |
| `class CanadaPostParsedElements` | Simple DTO for parsed response. | None | Holds status code/message and list of options | None |

### Helper Methods (inlined)
- `CurrencyUtil.convertToCurrency(...)` – Converts monetary values between currencies.  
- `CurrencyUtil.getWeight(...)` / `getMeasure(...)` – Convert weight/size units.  
- `ShippingUtil.trimPostalCode(...)` – Normalises postal codes.  
- `CountryUtil.getCountryIsoCodeById(...)` – ISO code lookup.  
- `CountryUtil.getCountryName(...)` – For the destination country.  

---

## 4. Dependencies

| Library | Purpose | Standard / Third‑party |
|---------|---------|------------------------|
| `org.apache.commons.httpclient` | HTTP client to POST XML | Third‑party |
| `org.apache.commons.digester` | XML parsing via rules | Third‑party |
| `org.apache.commons.lang.StringUtils` | String utilities | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.salesmanager.core.*` | Core domain, services, utilities | Internal to the project |

No external services beyond the Canada Post web service are required. The module is tightly coupled to the internal service factories and cache (`ServiceFactory`, `RefCache`).

---

## 5. Additional Notes & Recommendations

### 5.1 Code Quality & Maintainability
- **String concatenation for XML** is error‑prone and hard to maintain. Consider using an XML builder (e.g., `javax.xml.stream.XMLOutputFactory`) or a templated approach (XSLT or Velocity).  
- **Hard‑coded country IDs (38, 223)** reduce flexibility. Prefer constants or look‑ups.  
- **Magic numbers** (`-6`, `-7`, `"24"`, `"1"`, etc.) should be defined as constants with descriptive names.  
- **Exception handling** is too broad. Catching `Exception` hides the specific failure reasons. Use finer‑grained catch blocks (e.g., `IOException`, `SAXException`).  
- **Logging** uses `log.error(e)` but does not include stack trace or message context. Prefer `log.error("message", e)`.  
- **Resource leaks** – The `HttpClient` is not closed; the underlying connection manager may leak resources over time. Use try‑with‑resources or explicitly close.  

### 5.2 Security & Validation
- The request body is built by concatenating user data; this could lead to XML injection if not properly escaped. Use an XML library to escape values.  
- No input validation for postal codes or package dimensions.  
- Credentials are stored in memory (`IntegrationKeys`) but no encryption or secure transport details are shown.  

### 5.3 Error Handling & User Feedback
- The method returns `null` on any error, which may lead to NPEs downstream. Consider returning an empty list or a custom error object.  
- The status codes `-6` and `-7` are logged via `LogMerchantUtil`, but other codes are just logged as errors. Provide user‑friendly messages or retry logic.  

### 5.4 Performance
- For each quote request, the method rebuilds the entire XML request and parses the response. Caching repeated requests (e.g., identical package configurations) could improve performance.  
- The `CurrencyUtil.convertToCurrency` call is performed per option; if many options exist, this could be optimized.

### 5.5 Testability
- The current implementation tightly couples HTTP client, digester, and external caches. Refactor to inject these dependencies, enabling unit testing with mocks.  

### 5.6 Future Enhancements
1. **Retry logic** for transient network failures.  
2. **Graceful degradation**: fallback to a default rate if Canada Post is unavailable.  
3. **Internationalization**: support more locales, not just en/fr.  
4. **Extensibility**: allow different shipping carriers to be plugged in with minimal code changes.  
5. **Metrics**: log request latency and response size for monitoring.  

---

### Final Assessment

The module accomplishes its primary goal—obtaining Canada Post quotes—but is fragile, difficult to test, and prone to bugs from string‑based XML manipulation. Refactoring for clarity, error handling, and security would significantly improve robustness and maintainability.

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
package com.salesmanager.core.module.impl.integration.shipping;

import java.io.Reader;
import java.io.StringReader;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.digester.Digester;
import org.apache.commons.httpclient.HttpClient;
import org.apache.commons.httpclient.methods.PostMethod;
import org.apache.commons.httpclient.methods.RequestEntity;
import org.apache.commons.httpclient.methods.StringRequestEntity;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.shipping.PackageDetail;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.module.model.integration.ShippingQuotesModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LogMerchantUtil;

public class CanadaPostQuotesImpl implements ShippingQuotesModule {

	private Logger log = Logger.getLogger(CanadaPostQuotesImpl.class);

	public String getShippingMethodDescription(Locale locale) {
		return LabelUtil.getInstance().getText(locale, "module.canadapost");
	}

	public Collection<ShippingOption> getShippingQuote(
			ConfigurationResponse config, BigDecimal orderTotal,
			Collection<PackageDetail> packages, Customer customer,
			MerchantStore store, Locale locale) {

		BigDecimal total = orderTotal;

		if (packages == null) {
			return null;
		}

		// only applies to Canada and US
		if (customer.getCustomerCountryId() != 38
				&& customer.getCustomerCountryId() != 223) {
			return null;
		}

		// supports en and fr
		String language = locale.getLanguage();
		if (!language.equals(Constants.FRENCH_CODE)
				&& !language.equals(Constants.ENGLISH_CODE)) {
			language = Constants.ENGLISH_CODE;
		}

		// get canadapost credentials
		if (config == null) {
			log
					.error("CanadaPostQuotesImp.getShippingQuote requires ConfigurationVO for key SHP_RT_CRED");
			return null;
		}

		// if store is not CAD
		if (!store.getCurrency().equals(Constants.CURRENCY_CODE_CAD)) {
			total = CurrencyUtil.convertToCurrency(total, store.getCurrency(),
					Constants.CURRENCY_CODE_CAD);
		}

		PostMethod httppost = null;

		CanadaPostParsedElements canadaPost = null;

		try {

			int icountry = store.getCountry();
			String country = CountryUtil.getCountryIsoCodeById(icountry);

			ShippingService sservice = (ShippingService) ServiceFactory
					.getService(ServiceFactory.ShippingService);
			CoreModuleService cms = sservice.getRealTimeQuoteShippingService(
					country, "canadapost");

			IntegrationKeys keys = (IntegrationKeys) config
					.getConfiguration("canadapost-keys");
			IntegrationProperties props = (IntegrationProperties) config
					.getConfiguration("canadapost-properties");

			if (cms == null) {
				// throw new
				// Exception("Central integration services not configured for "
				// + PaymentConstants.PAYMENT_PSIGATENAME + " and country id " +
				// origincountryid);
				log
						.error("CoreModuleService not configured for  canadapost and country id "
								+ icountry);
				return null;
			}

			String host = cms.getCoreModuleServiceProdDomain();
			String protocol = cms.getCoreModuleServiceProdProtocol();
			String port = cms.getCoreModuleServiceProdPort();
			String url = cms.getCoreModuleServiceProdEnv();
			if (props.getProperties1().equals(
					String.valueOf(ShippingConstants.TEST_ENVIRONMENT))) {
				host = cms.getCoreModuleServiceDevDomain();
				protocol = cms.getCoreModuleServiceDevProtocol();
				port = cms.getCoreModuleServiceDevPort();
				url = cms.getCoreModuleServiceDevEnv();
			}

			// accept KG and CM

			StringBuffer request = new StringBuffer();

			request.append("<?xml version=\"1.0\" ?>");
			request.append("<eparcel>");
			request.append("<language>").append(language).append("</language>");

			request.append("<ratesAndServicesRequest>");
			request.append("<merchantCPCID>").append(keys.getUserid()).append(
					"</merchantCPCID>");
			request.append("<fromPostalCode>").append(
					com.salesmanager.core.util.ShippingUtil
							.trimPostalCode(store.getStorepostalcode()))
					.append("</fromPostalCode>");
			request.append("<turnAroundTime>").append("24").append(
					"</turnAroundTime>");
			request.append("<itemsPrice>").append(
					CurrencyUtil.displayFormatedAmountNoCurrency(total, "CAD"))
					.append("</itemsPrice>");
			request.append("<lineItems>");

			Iterator packageIterator = packages.iterator();
			while (packageIterator.hasNext()) {
				PackageDetail pack = (PackageDetail) packageIterator.next();
				request.append("<item>");
				request.append("<quantity>").append(pack.getShippingQuantity())
						.append("</quantity>");
				request.append("<weight>").append(
						String.valueOf(CurrencyUtil.getWeight(pack
								.getShippingWeight(), store,
								Constants.KG_WEIGHT_UNIT))).append("</weight>");
				request.append("<length>").append(
						String.valueOf(CurrencyUtil.getMeasure(pack
								.getShippingLength(), store,
								Constants.CM_SIZE_UNIT))).append("</length>");
				request.append("<width>").append(
						String.valueOf(CurrencyUtil.getMeasure(pack
								.getShippingWidth(), store,
								Constants.CM_SIZE_UNIT))).append("</width>");
				request.append("<height>").append(
						String.valueOf(CurrencyUtil.getMeasure(pack
								.getShippingHeight(), store,
								Constants.CM_SIZE_UNIT))).append("</height>");
				request.append("<description>").append(pack.getProductName())
						.append("</description>");
				request.append("<readyToShip/>");
				request.append("</item>");
			}

			Country c = null;
			Map countries = (Map) RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			c = (Country) countries.get(store.getCountry());

			request.append("</lineItems>");
			request.append("<city>").append(customer.getCustomerCity()).append(
					"</city>");

			request.append("<provOrState>").append(customer.getShippingSate())
					.append("</provOrState>");
			Map cs = (Map) RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			Country customerCountry = (Country) cs.get(customer
					.getCustomerCountryId());
			request.append("<country>")
					.append(customerCountry.getCountryName()).append(
							"</country>");
			request.append("<postalCode>").append(
					com.salesmanager.core.util.ShippingUtil
							.trimPostalCode(customer.getCustomerPostalCode()))
					.append("</postalCode>");
			request.append("</ratesAndServicesRequest>");
			request.append("</eparcel>");

			/**
			 * <?xml version="1.0" ?> <eparcel>
			 * <!--********************************--> <!-- Prefered language
			 * for the --> <!-- response (FR/EN) (optional) -->
			 * <!--********************************--> <language>en</language>
			 * 
			 * <ratesAndServicesRequest>
			 * <!--**********************************--> <!-- Merchant
			 * Identification assigned --> <!-- by Canada Post --> <!-- --> <!--
			 * Note: Use 'CPC_DEMO_HTML' or ask --> <!-- our Help Desk to change
			 * your --> <!-- profile if you want HTML to be --> <!-- returned to
			 * you --> <!--**********************************--> <merchantCPCID>
			 * CPC_DEMO_XML </merchantCPCID>
			 * 
			 * <!--*********************************--> <!--Origin Postal Code
			 * --> <!--This parameter is optional -->
			 * <!--*********************************-->
			 * <fromPostalCode>m1p1c0</fromPostalCode>
			 * 
			 * <!--**********************************--> <!-- Turn Around Time
			 * (hours) --> <!-- This parameter is optional -->
			 * <!--**********************************--> <turnAroundTime> 24
			 * </turnAroundTime>
			 * 
			 * <!--**********************************--> <!-- Total amount in $
			 * of the items --> <!-- for insurance calculation --> <!-- This
			 * parameter is optional -->
			 * <!--**********************************-->
			 * <itemsPrice>0.00</itemsPrice>
			 * 
			 * <!--**********************************--> <!-- List of items in
			 * the shopping --> <!-- cart --> <!-- Each item is defined by : -->
			 * <!-- - quantity (mandatory) --> <!-- - size (mandatory) --> <!--
			 * - weight (mandatory) --> <!-- - description (mandatory) --> <!--
			 * - ready to ship (optional) -->
			 * <!--**********************************--> <lineItems> <item>
			 * <quantity> 1 </quantity> <weight> 1.491 </weight> <length> 1
			 * </length> <width> 1 </width> <height> 1 </height> <description>
			 * KAO Diskettes </description> </item>
			 * 
			 * <item> <quantity> 1 </quantity> <weight> 1.5 </weight> <length>
			 * 20 </length> <width> 30 </width> <height> 20 </height>
			 * <description> My Ready To Ship Item</description>
			 * <!--**********************************************--> <!-- By
			 * adding the 'readyToShip' tag, Sell Online --> <!-- will not pack
			 * this item in the boxes --> <!-- defined in the merchant profile.
			 * --> <!-- Instead, this item will be shipped in its --> <!--
			 * original box: 1.5 kg and 20x30x20 cm -->
			 * <!--**********************************************-->
			 * <readyToShip/> </item> </lineItems>
			 * 
			 * <!--********************************--> <!-- City where the
			 * parcel will be --> <!-- shipped to -->
			 * <!--********************************--> <city> </city>
			 * 
			 * <!--********************************--> <!-- Province (Canada) or
			 * State (US)--> <!-- where the parcel will be --> <!-- shipped to
			 * --> <!--********************************--> <provOrState>
			 * Wisconsin </provOrState>
			 * 
			 * <!--********************************--> <!-- Country or ISO
			 * Country code --> <!-- where the parcel will be --> <!-- shipped
			 * to --> <!--********************************--> <country> CANADA
			 * </country>
			 * 
			 * <!--********************************--> <!-- Postal Code (or ZIP)
			 * where the --> <!-- parcel will be shipped to -->
			 * <!--********************************--> <postalCode>
			 * H3K1E5</postalCode> </ratesAndServicesRequest> </eparcel>
			 **/

			log.debug("canadapost request " + request.toString());

			HttpClient client = new HttpClient();
			
			StringBuilder u = new StringBuilder().append(protocol).append("://").append(host).append(":").append(port);
			if(!StringUtils.isBlank(url)) {
				u.append(url);
			}
			
			log.debug("Canadapost URL " + u.toString());

			httppost = new PostMethod(u.toString());
			RequestEntity entity = new StringRequestEntity(request.toString(),
					"text/plain", "UTF-8");
			httppost.setRequestEntity(entity);

			int result = client.executeMethod(httppost);

			if (result != 200) {
				log.error("Communication Error with canadapost " + protocol
						+ "://" + host + ":" + port + url);
				throw new Exception("Communication Error with canadapost "
						+ protocol + "://" + host + ":" + port + url);
			}
			String stringresult = httppost.getResponseBodyAsString();
			log.debug("canadapost response " + stringresult);

			canadaPost = new CanadaPostParsedElements();
			Digester digester = new Digester();
			digester.push(canadaPost);

			digester.addCallMethod(
					"eparcel/ratesAndServicesResponse/statusCode",
					"setStatusCode", 0);
			digester.addCallMethod(
					"eparcel/ratesAndServicesResponse/statusMessage",
					"setStatusMessage", 0);
			digester.addObjectCreate(
					"eparcel/ratesAndServicesResponse/product",
					com.salesmanager.core.entity.shipping.ShippingOption.class);
			digester.addSetProperties(
					"eparcel/ratesAndServicesResponse/product", "sequence",
					"optionId");
			digester.addCallMethod(
					"eparcel/ratesAndServicesResponse/product/shippingDate",
					"setShippingDate", 0);
			digester.addCallMethod(
					"eparcel/ratesAndServicesResponse/product/deliveryDate",
					"setDeliveryDate", 0);
			digester.addCallMethod(
					"eparcel/ratesAndServicesResponse/product/name",
					"setOptionName", 0);
			digester.addCallMethod(
					"eparcel/ratesAndServicesResponse/product/rate",
					"setOptionPriceText", 0);
			digester.addSetNext("eparcel/ratesAndServicesResponse/product",
					"addOption");

			/**
			 * response
			 * 
			 * <?xml version="1.0" ?> <!DOCTYPE eparcel (View Source for full
			 * doctype...)> - <eparcel> - <ratesAndServicesResponse>
			 * <statusCode>1</statusCode> <statusMessage>OK</statusMessage>
			 * <requestID>1769506</requestID> <handling>0.0</handling>
			 * <language>0</language> - <product id="1040" sequence="1">
			 * <name>Priority Courier</name> <rate>38.44</rate>
			 * <shippingDate>2008-12-22</shippingDate>
			 * <deliveryDate>2008-12-23</deliveryDate>
			 * <deliveryDayOfWeek>3</deliveryDayOfWeek>
			 * <nextDayAM>true</nextDayAM> <packingID>P_0</packingID> </product>
			 * - <product id="1020" sequence="2"> <name>Expedited</name>
			 * <rate>16.08</rate> <shippingDate>2008-12-22</shippingDate>
			 * <deliveryDate>2008-12-23</deliveryDate>
			 * <deliveryDayOfWeek>3</deliveryDayOfWeek>
			 * <nextDayAM>false</nextDayAM> <packingID>P_0</packingID>
			 * </product> - <product id="1010" sequence="3">
			 * <name>Regular</name> <rate>16.08</rate>
			 * <shippingDate>2008-12-22</shippingDate>
			 * <deliveryDate>2008-12-29</deliveryDate>
			 * <deliveryDayOfWeek>2</deliveryDayOfWeek>
			 * <nextDayAM>false</nextDayAM> <packingID>P_0</packingID>
			 * </product> - <packing> <packingID>P_0</packingID> - <box>
			 * <name>Small Box</name> <weight>1.691</weight>
			 * <expediterWeight>1.691</expediterWeight> <length>25.0</length>
			 * <width>17.0</width> <height>16.0</height> - <packedItem>
			 * <quantity>1</quantity> <description>KAO Diskettes</description>
			 * </packedItem> </box> - <box> <name>My Ready To Ship Item</name>
			 * <weight>2.0</weight> <expediterWeight>1.5</expediterWeight>
			 * <length>30.0</length> <width>20.0</width> <height>20.0</height> -
			 * <packedItem> <quantity>1</quantity> <description>My Ready To Ship
			 * Item</description> </packedItem> </box> </packing> -
			 * <shippingOptions> <insurance>No</insurance>
			 * <deliveryConfirmation>Yes</deliveryConfirmation>
			 * <signature>No</signature> </shippingOptions> <comment />
			 * </ratesAndServicesResponse> </eparcel> - <!-- END_OF_EPARCEL -->
			 */

			Reader reader = new StringReader(stringresult);

			digester.parse(reader);

		} catch (Exception e) {
			log.error(e);
		} finally {
			if (httppost != null) {
				httppost.releaseConnection();
			}
		}

		if (canadaPost == null || canadaPost.getStatusCode() == null) {
			return null;
		}

		if (canadaPost.getStatusCode().equals("-6")
				|| canadaPost.getStatusCode().equals("-7")) {
			LogMerchantUtil.log(store.getMerchantId(),
					"Can't process CanadaPost statusCode="
							+ canadaPost.getStatusCode() + " message= "
							+ canadaPost.getStatusMessage());
		}

		if (!canadaPost.getStatusCode().equals("1")) {
			log.error("An error occured with canadapost request (code-> "
					+ canadaPost.getStatusCode() + " message-> "
					+ canadaPost.getStatusMessage() + ")");
			return null;
		}

		String carrier = getShippingMethodDescription(locale);
		// cost is in CAD, need to do conversion

		boolean requiresCurrencyConversion = false;
		String storeCurrency = store.getCurrency();
		if (!storeCurrency.equals(Constants.CURRENCY_CODE_CAD)) {
			requiresCurrencyConversion = true;
		}

		/** Details on whit RT quote information to display **/
		MerchantConfiguration rtdetails = config
				.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES);
		int displayQuoteDeliveryTime = ShippingConstants.NO_DISPLAY_RT_QUOTE_TIME;
		if (rtdetails != null) {

			if (!StringUtils.isBlank(rtdetails.getConfigurationValue1())) {// display
																			// or
																			// not
																			// quotes
				try {
					displayQuoteDeliveryTime = Integer.parseInt(rtdetails
							.getConfigurationValue1());

				} catch (Exception e) {
					log.error("Display quote is not an integer value ["
							+ rtdetails.getConfigurationValue1() + "]");
				}
			}
		}
		/**/

		List options = canadaPost.getOptions();
		if (options != null) {
			Iterator i = options.iterator();
			while (i.hasNext()) {
				ShippingOption option = (ShippingOption) i.next();
				option.setCurrency(store.getCurrency());
				StringBuffer description = new StringBuffer();
				description.append(option.getOptionName());
				if (displayQuoteDeliveryTime == ShippingConstants.DISPLAY_RT_QUOTE_TIME) {
					description.append(" (").append(option.getDeliveryDate())
							.append(")");
				}
				option.setDescription(description.toString());
				if (requiresCurrencyConversion) {
					option.setOptionPrice(CurrencyUtil.convertToCurrency(option
							.getOptionPrice(), Constants.CURRENCY_CODE_CAD,
							store.getCurrency()));
				}
				// System.out.println(option.getOptionPrice().toString());

			}
		}

		return options;
	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo)
			throws Exception {
		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_CRED)) {// handle
																// credentials

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationKeys keys = ShippingUtil.getKeys(configurations
						.getConfigurationValue1());
				vo.addConfiguration("canadapost-keys", keys);

			}

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {
				IntegrationProperties props = ShippingUtil
						.getProperties(configurations.getConfigurationValue2());
				vo.addConfiguration("canadapost-properties", props);
			}

		}

		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT)) {// handle
																	// packages
																	// &
																	// services
			Map domesticmap = null;
			Map globalmap = null;
			// PKGOPTIONS
			if (!StringUtils.isBlank(configurations.getConfigurationValue())) {
				vo.addConfiguration("package-canadapost", configurations
						.getConfigurationValue());
			}
		}

		vo.addMerchantConfiguration(configurations);
		return vo;
	}

	public void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {
		// TODO Auto-generated method stub

	}

}

class CanadaPostParsedElements {

	private String statusCode;
	private String statusMessage;
	private List options = new ArrayList();

	public void addOption(ShippingOption option) {
		options.add(option);
	}

	public List getOptions() {
		return options;
	}

	public String getStatusCode() {
		return statusCode;
	}

	public void setStatusCode(String statusCode) {
		this.statusCode = statusCode;
	}

	public String getStatusMessage() {
		return statusMessage;
	}

	public void setStatusMessage(String statusMessage) {
		this.statusMessage = statusMessage;
	}

}



```
