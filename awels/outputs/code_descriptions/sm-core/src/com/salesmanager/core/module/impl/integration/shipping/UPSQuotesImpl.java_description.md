# UPSQuotesImpl.java

## Review

## 1. Summary

**Purpose**  
The `UPSQuotesImpl` class implements the `ShippingQuotesModule` interface and is responsible for retrieving shipping quotes from UPS via its XML‑based rating API. It constructs the request XML, sends it over HTTP, parses the XML response, and returns a collection of `ShippingOption` objects that can be presented to the customer.

**Key Components**

| Class | Responsibility |
|-------|----------------|
| `UPSQuotesImpl` | Generates request XML, communicates with UPS, parses response, and builds shipping options. |
| `UPSParsedElements` | Helper container used by Apache Digester to capture parsed data from the UPS response. |
| `ShippingOption` | Domain model representing an individual shipping service option. |
| `IntegrationKeys` / `IntegrationProperties` | Hold credentials and environment settings for UPS. |
| `ServiceFactory` | Provides various services such as `CommonService` and `MerchantService`. |
| `CountryUtil`, `Zone`, `ShippingUtil`, etc. | Utility classes for mapping locales, countries, zones, and unit conversions. |

**Design Patterns & Libraries**

* **Builder / Factory** – `ServiceFactory` supplies the required services.  
* **Adapter** – `UPSQuotesImpl` adapts the UPS XML API into the application's `ShippingOption` model.  
* **Apache Digester** – XML parsing of the UPS response into POJOs.  
* **Apache HttpClient** – HTTP communication with UPS.  
* **Singleton** – `LabelUtil` and `ShippingUtil` are accessed via `getInstance()`.  

The class relies heavily on custom domain entities (`Customer`, `MerchantStore`, `PackageDetail`, etc.) and configuration objects (`ConfigurationResponse`, `MerchantConfiguration`).

---

## 2. Detailed Description

### Flow of Execution

1. **Initialization**  
   * `getShippingQuote(...)` is invoked with the store, customer, and package details.  
   * The method pulls configuration values (UPS credentials, packaging type, service filters) from the `ConfigurationResponse` via `MerchantService`.

2. **Request Construction**  
   * **Header** – `getHeader()` builds the `<AccessRequest>` XML fragment containing UPS credentials.  
   * **Body** – A static `<RatingServiceSelectionRequest>` header is concatenated with a dynamic `<Shipment>` block that includes:  
     * Shipper and ShipTo addresses (with city, state, country, postal code).  
     * Package details: weight, dimensions, packaging type.  
     * Pickup type (fixed to “03” – daily pickup).  
   * All values are formatted and sanitized (e.g., postal code trimming, rounding weight/dimensions).  

3. **HTTP Call**  
   * The full XML request is posted to the UPS endpoint (dev or prod depending on `IntegrationProperties`).  
   * `HttpClient`/`PostMethod` is used; response is retrieved as a string.

4. **Parsing Response**  
   * A custom `UPSParsedElements` object is created, and an Apache **Digester** configuration maps XML paths to setter methods.  
   * The parsed data includes error information and a list of `ShippingOption` objects (each mapped to `RatingServiceSelectionResponse/RatedShipment`).  
   * If any error or non‑successful status code is detected, the method logs and returns `null`.

5. **Post‑Processing**  
   * Currency conversion from the response currency to the store’s currency if needed.  
   * Service codes are filtered against the store’s allowed services (`service-global-upsxml`).  
   * The description of each option is built using the locale‑specific service map (`ShippingUtil.buildServiceMap`).  
   * Estimated delivery days are optionally appended to the description.  
   * Finally, a collection of filtered `ShippingOption`s is returned.

6. **Cleanup**  
   * The `finally` block ensures the HTTP connection is released and any `BufferedReader` is closed.

### Assumptions & Constraints

* **Fixed Pickup Type** – The code hardcodes pickup code “03” (Daily Pickup). No dynamic selection based on merchant settings.  
* **Error Handling** – Any error code causes the method to return `null`. No retry logic or partial result handling.  
* **Unit Code Mapping** – The mapping between store weight units (“KG”/“LB”) and UPS codes (“KGS”/“LBS”) is simplistic.  
* **No Input Validation** – The code assumes all input objects are non‑null and correctly populated (e.g., `store.getStorepostalcode()` is valid).  
* **Synchronous Request** – The method blocks until the UPS response is received; no asynchronous or queued processing.

### Architecture & Design Choices

* The class tightly couples to external services (`HttpClient`, `Digester`, `ServiceFactory`).  
* Configuration is injected via `ConfigurationResponse` but many values are still fetched with service lookups (e.g., `MerchantService`).  
* XML generation uses string concatenation, which is error‑prone and hard to maintain.  
* Error handling uses logging and early return; this is acceptable for a small utility but may hide useful context in a production environment.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getShippingMethodDescription(Locale)` | Localized description for the module. | `locale` | `String` | None |
| `getPackageCode(String)` | Intended to map a package ID to UPS code. | `codeId` | `String` | None (currently stub). |
| `getHeader(int, ConfigurationResponse)` | Builds the `<AccessRequest>` header with credentials. | `merchantid`, `vo` | `String` XML | None |
| `getShippingQuote(ConfigurationResponse, BigDecimal, Collection<PackageDetail>, Customer, MerchantStore, Locale)` | Main entry; builds request, calls UPS, parses response, returns options. | See signature | `Collection<ShippingOption>` | Logs, HTTP request, XML parsing |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | No-op; placeholder for future config persistence. | `merchantid`, `vo`, `request` | None | None |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Reads merchant configuration values and injects them into the response. | `configurations`, `vo` | `ConfigurationResponse` | Adds configurations (`upsxml-keys`, `upsxml-properties`, `package-upsxml`, `service-global-upsxml`) |
| `getResponsecode()`, `setResponsecode(String)` | Getter/Setter for response code. | None / `String` | `String` | None |
| `getResponsetext()`, `setResponsetext(String)` | Getter/Setter for response text. | None / `String` | `String` | None |

`UPSParsedElements` has:

| Method | Purpose |
|--------|---------|
| `addOption(ShippingOption)` | Adds a parsed option to internal list. |
| `getOptions()` | Returns list of options. |
| `setStatusCode`, `setStatusMessage`, `setError`, `setErrorCode` | Standard setters for parsed data. |
| `getStatusCode`, `getStatusMessage`, `getError`, `getErrorCode` | Corresponding getters. |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.digester.Digester` | Third‑party | XML parsing into POJOs. |
| `org.apache.commons.httpclient.HttpClient`, `PostMethod`, `StringRequestEntity` | Third‑party | HTTP communication (old Apache HttpClient 3.x). |
| `org.apache.commons.lang.StringUtils` | Third‑party | String utility. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| Custom domain classes (`Customer`, `MerchantStore`, `PackageDetail`, `ShippingOption`, etc.) | Internal | Part of the SalesManager core. |
| `ServiceFactory`, `CommonService`, `MerchantService` | Internal | Service locator/factory pattern. |
| `CountryUtil`, `Zone`, `ShippingUtil`, `CurrencyUtil`, `LabelUtil`, `LanguageUtil`, `LogMerchantUtil` | Internal | Utility classes. |
| `RefCache` | Internal | Caches lookup tables for countries and zones. |

*The code uses the legacy Apache HttpClient 3.x API, which is no longer maintained. Modern code would prefer `HttpClient` 4.x or Java 11+ `HttpClient`.*

---

## 5. Additional Notes

### Strengths

* **Clear separation of concerns** – request building, communication, parsing, and post‑processing are largely isolated.  
* **Extensible configuration** – allows merchants to override packaging types, service filters, and environment settings.  
* **Locale‑aware** – uses language maps to build service labels and delivery day text.  

### Weaknesses & Edge Cases

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hardcoded Pickup Type** | Inflexible for merchants that require other pickup methods. | Expose pickup type as a configuration value. |
| **String Concatenation for XML** | Susceptible to malformed XML, difficult to maintain. | Use a proper XML builder (e.g., `DocumentBuilder`, `StringBuilder` with proper escaping) or a templating library. |
| **No Validation** | Null pointer or malformed data may crash. | Add defensive checks (e.g., `Objects.requireNonNull`). |
| **Legacy HttpClient** | Security, performance, and compatibility issues. | Migrate to Apache HttpClient 4.x or Java 11+ `HttpClient`. |
| **Error Handling** | Early return of `null` hides details; downstream code must handle `null`. | Return a custom `QuoteResult` object containing success flag, options, and error details. |
| **Synchronous Call** | Blocking thread can affect scalability. | Offload to async/async queue or integrate with a message bus. |
| **No Pagination / Chunking** | For large package collections the request may exceed UPS limits. | Batch packages or split shipments. |
| **Currency Conversion Hard‑coded** | Cost conversion is commented out, leaving potential currency mismatch. | Perform explicit conversion if response currency ≠ store currency. |
| **`getPackageCode()` Stub** | Intended mapping never used; confusing. | Remove or implement correctly. |
| **Configuration Storage** | `storeConfiguration` is a no‑op. | Implement persistence in `ShippingupsxmlAction` or another dedicated service. |

### Suggested Refactoring Roadmap

1. **Abstract HTTP Layer** – Create an `HttpClientProvider` that hides the underlying client implementation.  
2. **XML Layer** – Replace manual concatenation with a typed XML model (e.g., JAXB or a simple DOM builder).  
3. **Use `javax.xml.parsers` / `StAX`** for lightweight parsing, optionally still using Digester for convenience.  
4. **Introduce a `QuoteContext` object** that carries store, customer, and package data, simplifying method signatures.  
5. **Unit Tests** – Extract request construction and parsing into unit‑testable helpers.  
6. **Retry & Back‑off** – Add simple retry logic for transient network failures.  

### Minor Improvements

* Rename local variables to follow Java conventions (`reader`, `httppost`) – they are declared but never used meaningfully.  
* Replace `Collection returnColl` with `List<ShippingOption> returnColl`.  
* Use generics everywhere (`Collection<ShippingOption>`) to avoid raw types.  
* Consider returning an empty collection instead of `null` when no options are available; the caller can then display a friendly message.

---

### Verdict

`UPSQuotesImpl` is functional and meets its core requirement of fetching UPS quotes. However, the code would benefit from modernization, better error handling, and a more robust XML handling strategy. Addressing the legacy dependencies and improving configurability will make the module more maintainable and merchant‑friendly in the long term.

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

import java.io.BufferedReader;
import java.io.Reader;
import java.io.StringReader;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.digester.Digester;
import org.apache.commons.httpclient.HttpClient;
import org.apache.commons.httpclient.methods.PostMethod;
import org.apache.commons.httpclient.methods.RequestEntity;
import org.apache.commons.httpclient.methods.StringRequestEntity;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.entity.shipping.PackageDetail;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.module.model.integration.ShippingQuotesModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LogMerchantUtil;

public class UPSQuotesImpl implements ShippingQuotesModule {

	private Logger log = Logger.getLogger(UPSQuotesImpl.class);

	public String getShippingMethodDescription(Locale locale) {
		return LabelUtil.getInstance().getText(locale, "module.upsxml");
	}

	private String getPackageCode(String codeId) {

		if (StringUtils.isBlank(codeId)) {
			log.warn("codeId is blank or null, will return standard overnight");

		}

		/**
		 *'00' (unknown), '01' (UPS letter), '02' (customer supplied package),
		 * '03'(tube), '04' (PAK), '21' (UPS express box), '2a' (UPS small
		 * express box), '2b' (UPS medium express box), '2c' (UPS large express
		 * box), '24' (UPS 25KG box), '25' (UPS 10KG box), '2a' (small express
		 * box), '2b' (medium express box), '2c' (large express box), or '30'
		 * (pallet)
		 */

		return null;

	}

	protected String getHeader(int merchantid, ConfigurationResponse vo)
			throws Exception {

		IntegrationKeys sk = (IntegrationKeys) vo
				.getConfiguration("upsxml-keys");

		if (sk == null) {
			throw new Exception(
					"Integration keys are null from configuration request upsxml");
		}

		StringBuffer xmlreqbuffer = new StringBuffer();
		xmlreqbuffer.append("<?xml version=\"1.0\"?>");
		xmlreqbuffer.append("<AccessRequest>");
		xmlreqbuffer.append("<AccessLicenseNumber>");
		xmlreqbuffer.append(sk.getKey1());
		xmlreqbuffer.append("</AccessLicenseNumber>");
		xmlreqbuffer.append("<UserId>");
		xmlreqbuffer.append(sk.getUserid());
		xmlreqbuffer.append("</UserId>");
		xmlreqbuffer.append("<Password>");
		xmlreqbuffer.append(sk.getPassword());
		xmlreqbuffer.append("</Password>");
		xmlreqbuffer.append("</AccessRequest>");

		return xmlreqbuffer.toString();

	}

	public Collection<ShippingOption> getShippingQuote(
			ConfigurationResponse config, BigDecimal orderTotal,
			Collection<PackageDetail> packages, Customer customer,
			MerchantStore store, Locale locale) {

		CoreModuleService cis = null;

		StringBuffer xmlbuffer = new StringBuffer();
		BufferedReader reader = null;
		PostMethod httppost = null;

		try {

			CommonService cservice = (CommonService) ServiceFactory
					.getService(ServiceFactory.CommonService);

			String countrycode = CountryUtil.getCountryIsoCodeById(store
					.getCountry());
			cis = cservice.getModule(countrycode, "upsxml");

			if (cis == null) {
				log.error("Can't retreive an integration service [countryid "
						+ store.getCountry() + " ups subtype 1]");
				// throw new
				// Exception("UPS getQuote Can't retreive an integration service");
			}

			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationRequest request_prefs = new ConfigurationRequest(store
					.getMerchantId(),
					ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
			ConfigurationResponse vo_prefs = service
					.getConfiguration(request_prefs);

			String pack = (String) vo_prefs.getConfiguration("package-upsxml");
			if (pack == null) {
				log
						.debug("Will assign packaging type 02 to UPS shipping for merchantid "
								+ store.getMerchantId());
				pack = "02";
			}

			ConfigurationRequest request = new ConfigurationRequest(store
					.getMerchantId(), ShippingConstants.MODULE_SHIPPING_RT_CRED);
			ConfigurationResponse vo = service.getConfiguration(request);

			if (vo == null) {
				throw new Exception("ConfigurationVO is null upsxml");
			}

			String xmlhead = getHeader(store.getMerchantId(), vo);

			String weightCode = store.getWeightunitcode();
			String measureCode = store.getSeizeunitcode();

			if (weightCode.equals("KG")) {
				weightCode = "KGS";
			} else {
				weightCode = "LBS";
			}

			String xml = "<?xml version=\"1.0\"?><RatingServiceSelectionRequest><Request><TransactionReference><CustomerContext>SalesManager Data</CustomerContext><XpciVersion>1.0001</XpciVersion></TransactionReference><RequestAction>Rate</RequestAction><RequestOption>Shop</RequestOption></Request>";
			StringBuffer xmldatabuffer = new StringBuffer();

			/**
			 * <Shipment>
			 * 
			 * <Shipper> <Address> <City></City>
			 * <StateProvinceCode>QC</StateProvinceCode>
			 * <CountryCode>CA</CountryCode> <PostalCode></PostalCode>
			 * </Address> </Shipper>
			 * 
			 * <ShipTo> <Address> <City>Redwood Shores</City>
			 * <StateProvinceCode>CA</StateProvinceCode>
			 * <CountryCode>US</CountryCode> <PostalCode></PostalCode>
			 * <ResidentialAddressIndicator/> </Address> </ShipTo>
			 * 
			 * <Package> <PackagingType> <Code>21</Code> </PackagingType>
			 * <PackageWeight> <UnitOfMeasurement> <Code>LBS</Code>
			 * </UnitOfMeasurement> <Weight>1.1</Weight> </PackageWeight>
			 * <PackageServiceOptions> <InsuredValue>
			 * <CurrencyCode>CAD</CurrencyCode>
			 * <MonetaryValue>100</MonetaryValue> </InsuredValue>
			 * </PackageServiceOptions> </Package>
			 * 
			 * 
			 * </Shipment>
			 * 
			 * <CustomerClassification> <Code>03</Code>
			 * </CustomerClassification> </RatingServiceSelectionRequest>
			 * **/

			Map countriesMap = (Map) RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			Map zonesMap = (Map) RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));

			Country storeCountry = (Country) countriesMap.get(store
					.getCountry());

			Country customerCountry = (Country) countriesMap.get(customer
					.getCustomerCountryId());

			int sZone = -1;
			try {
				sZone = Integer.parseInt(store.getZone());
			} catch (Exception e) {
				// TODO: handle exception
			}

			Zone storeZone = (Zone) zonesMap.get(sZone);
			Zone customerZone = (Zone) zonesMap.get(customer
					.getCustomerZoneId());

			xmldatabuffer.append("<PickupType><Code>03</Code></PickupType>");
			// xmldatabuffer.append("<Description>Daily Pickup</Description>");
			xmldatabuffer.append("<Shipment><Shipper>");
			xmldatabuffer.append("<Address>");
			xmldatabuffer.append("<City>");
			xmldatabuffer.append(store.getStorecity());
			xmldatabuffer.append("</City>");
			// if(!StringUtils.isBlank(store.getStorestateprovince())) {
			if (storeZone != null) {
				xmldatabuffer.append("<StateProvinceCode>");
				xmldatabuffer.append(storeZone.getZoneCode());// zone code
				xmldatabuffer.append("</StateProvinceCode>");
			}
			xmldatabuffer.append("<CountryCode>");
			xmldatabuffer.append(storeCountry.getCountryIsoCode2());
			xmldatabuffer.append("</CountryCode>");
			xmldatabuffer.append("<PostalCode>");
			xmldatabuffer.append(com.salesmanager.core.util.ShippingUtil
					.trimPostalCode(store.getStorepostalcode()));
			xmldatabuffer.append("</PostalCode></Address></Shipper>");

			// ship to
			xmldatabuffer.append("<ShipTo>");
			xmldatabuffer.append("<Address>");
			xmldatabuffer.append("<City>");
			xmldatabuffer.append(customer.getCustomerCity());
			xmldatabuffer.append("</City>");
			// if(!StringUtils.isBlank(customer.getCustomerState())) {
			if (customerZone != null) {
				xmldatabuffer.append("<StateProvinceCode>");
				xmldatabuffer.append(customerZone.getZoneCode());// zone code
				xmldatabuffer.append("</StateProvinceCode>");
			}
			xmldatabuffer.append("<CountryCode>");
			xmldatabuffer.append(customerCountry.getCountryIsoCode2());
			xmldatabuffer.append("</CountryCode>");
			xmldatabuffer.append("<PostalCode>");
			xmldatabuffer.append(com.salesmanager.core.util.ShippingUtil
					.trimPostalCode(customer.getCustomerPostalCode()));
			xmldatabuffer.append("</PostalCode></Address></ShipTo>");
			// xmldatabuffer.append("<Service><Code>11</Code></Service>");

			Iterator packagesIterator = packages.iterator();
			while (packagesIterator.hasNext()) {

				PackageDetail detail = (PackageDetail) packagesIterator.next();
				xmldatabuffer.append("<Package>");
				xmldatabuffer.append("<PackagingType>");
				xmldatabuffer.append("<Code>");
				xmldatabuffer.append(pack);
				xmldatabuffer.append("</Code>");
				xmldatabuffer.append("</PackagingType>");

				// weight
				xmldatabuffer.append("<PackageWeight>");
				xmldatabuffer.append("<UnitOfMeasurement>");
				xmldatabuffer.append("<Code>");
				xmldatabuffer.append(weightCode);
				xmldatabuffer.append("</Code>");
				xmldatabuffer.append("</UnitOfMeasurement>");
				xmldatabuffer.append("<Weight>");
				xmldatabuffer.append(new BigDecimal(detail.getShippingWeight())
						.setScale(1, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Weight>");
				xmldatabuffer.append("</PackageWeight>");

				// dimension
				xmldatabuffer.append("<Dimensions>");
				xmldatabuffer.append("<UnitOfMeasurement>");
				xmldatabuffer.append("<Code>");
				xmldatabuffer.append(measureCode);
				xmldatabuffer.append("</Code>");
				xmldatabuffer.append("</UnitOfMeasurement>");
				xmldatabuffer.append("<Length>");
				xmldatabuffer.append(new BigDecimal(detail.getShippingLength())
						.setScale(2, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Length>");
				xmldatabuffer.append("<Width>");
				xmldatabuffer.append(new BigDecimal(detail.getShippingWidth())
						.setScale(2, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Width>");
				xmldatabuffer.append("<Height>");
				xmldatabuffer.append(new BigDecimal(detail.getShippingHeight())
						.setScale(2, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Height>");
				xmldatabuffer.append("</Dimensions>");
				xmldatabuffer.append("</Package>");

			}

			xmldatabuffer.append("</Shipment>");
			xmldatabuffer.append("</RatingServiceSelectionRequest>");

			xmlbuffer.append(xmlhead).append(xml).append(
					xmldatabuffer.toString());

			log.debug("UPS QUOTE REQUEST " + xmlbuffer.toString());

			String data = "";

			IntegrationKeys keys = (IntegrationKeys) config
					.getConfiguration("upsxml-keys");

			IntegrationProperties props = (IntegrationProperties) config
					.getConfiguration("upsxml-properties");

			String host = cis.getCoreModuleServiceProdDomain();
			String protocol = cis.getCoreModuleServiceProdProtocol();
			String port = cis.getCoreModuleServiceProdPort();
			String uri = cis.getCoreModuleServiceProdEnv();

			if (props.getProperties1().equals(
					String.valueOf(ShippingConstants.TEST_ENVIRONMENT))) {
				host = cis.getCoreModuleServiceDevDomain();
				protocol = cis.getCoreModuleServiceDevProtocol();
				port = cis.getCoreModuleServiceDevPort();
				uri = cis.getCoreModuleServiceDevEnv();
			}

			HttpClient client = new HttpClient();
			httppost = new PostMethod(protocol + "://" + host + ":" + port
					+ uri);
			RequestEntity entity = new StringRequestEntity(
					xmlbuffer.toString(), "text/plain", "UTF-8");
			httppost.setRequestEntity(entity);

			int result = client.executeMethod(httppost);
			if (result != 200) {
				log.error("Communication Error with ups quote " + result + " "
						+ protocol + "://" + host + ":" + port + uri);
				throw new Exception("UPS quote communication error " + result);
			}
			data = httppost.getResponseBodyAsString();
			log.debug("ups quote response " + data);

			UPSParsedElements parsed = new UPSParsedElements();

			Digester digester = new Digester();
			digester.push(parsed);
			digester.addCallMethod(
					"RatingServiceSelectionResponse/Response/Error",
					"setErrorCode", 0);
			digester.addCallMethod(
					"RatingServiceSelectionResponse/Response/ErrorDescriprion",
					"setError", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/Response/ResponseStatusCode",
							"setStatusCode", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/Response/ResponseStatusDescription",
							"setStatusMessage", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/Response/Error/ErrorDescription",
							"setError", 0);

			digester.addObjectCreate(
					"RatingServiceSelectionResponse/RatedShipment",
					com.salesmanager.core.entity.shipping.ShippingOption.class);
			// digester.addSetProperties(
			// "RatingServiceSelectionResponse/RatedShipment", "sequence",
			// "optionId" );
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/Service/Code",
							"setOptionId", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/TotalCharges/MonetaryValue",
							"setOptionPriceText", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/TotalCharges/CurrencyCode",
							"setCurrency", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/Service/Code",
							"setOptionCode", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/GuaranteedDaysToDelivery",
							"setEstimatedNumberOfDays", 0);
			digester.addSetNext("RatingServiceSelectionResponse/RatedShipment",
					"addOption");

			// <?xml
			// version="1.0"?><AddressValidationResponse><Response><TransactionReference><CustomerContext>SalesManager
			// Data</CustomerContext><XpciVersion>1.0</XpciVersion></TransactionReference><ResponseStatusCode>0</ResponseStatusCode><ResponseStatusDescription>Failure</ResponseStatusDescription><Error><ErrorSeverity>Hard</ErrorSeverity><ErrorCode>10002</ErrorCode><ErrorDescription>The
			// XML document is well formed but the document is not
			// valid</ErrorDescription><ErrorLocation><ErrorLocationElementName>AddressValidationRequest</ErrorLocationElementName></ErrorLocation></Error></Response></AddressValidationResponse>

			Reader xmlreader = new StringReader(data);

			digester.parse(xmlreader);

			if (!StringUtils.isBlank(parsed.getErrorCode())) {
				log.error("Can't process UPS statusCode="
						+ parsed.getErrorCode() + " message= "
						+ parsed.getError());
				return null;
			}
			if (!StringUtils.isBlank(parsed.getStatusCode())
					&& !parsed.getStatusCode().equals("1")) {
				LogMerchantUtil.log(store.getMerchantId(),
						"Can't process UPS statusCode="
								+ parsed.getStatusCode() + " message= "
								+ parsed.getError());
				log.error("Can't process UPS statusCode="
						+ parsed.getStatusCode() + " message= "
						+ parsed.getError());
				return null;
			}

			if (parsed.getOptions() == null || parsed.getOptions().size() == 0) {
				log.warn("No options returned from UPS");
				return null;
			}

			String carrier = getShippingMethodDescription(locale);
			// cost is in CAD, need to do conversion

			/*
			 * boolean requiresCurrencyConversion = false; String storeCurrency
			 * = store.getCurrency();
			 * if(!storeCurrency.equals(Constants.CURRENCY_CODE_CAD)) {
			 * requiresCurrencyConversion = true; }
			 */

			LabelUtil labelUtil = LabelUtil.getInstance();
			Map serviceMap = com.salesmanager.core.util.ShippingUtil
					.buildServiceMap("upsxml", locale);

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

			Collection returnColl = null;

			List options = parsed.getOptions();
			if (options != null) {

				Map selectedintlservices = (Map) config
						.getConfiguration("service-global-upsxml");

				Iterator i = options.iterator();
				while (i.hasNext()) {
					ShippingOption option = (ShippingOption) i.next();
					// option.setCurrency(store.getCurrency());
					StringBuffer description = new StringBuffer();

					String code = option.getOptionCode();
					option.setOptionCode(code);
					// get description
					String label = (String) serviceMap.get(code);
					if (label == null) {
						log
								.warn("UPSXML cannot find description for service code "
										+ code);
					}

					option.setOptionName(label);

					description.append(option.getOptionName());
					if (displayQuoteDeliveryTime == ShippingConstants.DISPLAY_RT_QUOTE_TIME) {
						if (!StringUtils.isBlank(option
								.getEstimatedNumberOfDays())) {
							description.append(" (").append(
									option.getEstimatedNumberOfDays()).append(
									" ").append(
									labelUtil.getText(locale,
											"label.generic.days.lowercase"))
									.append(")");
						}
					}
					option.setDescription(description.toString());

					// get currency
					if (!option.getCurrency().equals(store.getCurrency())) {
						option.setOptionPrice(CurrencyUtil.convertToCurrency(
								option.getOptionPrice(), option.getCurrency(),
								store.getCurrency()));
					}

					if (!selectedintlservices.containsKey(option
							.getOptionCode())) {
						if (returnColl == null) {
							returnColl = new ArrayList();
						}
						returnColl.add(option);
						// options.remove(option);
					}

				}

				if (options.size() == 0) {
					LogMerchantUtil
							.log(
									store.getMerchantId(),
									" none of the service code returned by UPS ["
											+ selectedintlservices
													.keySet()
													.toArray(
															new String[selectedintlservices
																	.size()])
											+ "] for this shipping is in your selection list");
				}
			}



			return returnColl;

		} catch (Exception e1) {
			log.error(e1);
			return null;
		} finally {
			if (reader != null) {
				try {
					reader.close();
				} catch (Exception ignore) {
				}
			}

			if (httppost != null) {
				httppost.releaseConnection();
			}
		}

	}

	public void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {
		// TODO Auto-generated method stub
		// stored in ShippingupsxmlAction class

	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo)
			throws Exception {

		// what is the MerchantConfiguration ??

		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_CRED)) {// handle
																// credentials

			if (!StringUtils.isBlank(configurations.getConfigurationValue1())) {

				IntegrationKeys keys = ShippingUtil.getKeys(configurations
						.getConfigurationValue1());
				vo.addConfiguration("upsxml-keys", keys);

			}

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationProperties props = ShippingUtil
						.getProperties(configurations.getConfigurationValue2());
				vo.addConfiguration("upsxml-properties", props);
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
				vo.addConfiguration("package-upsxml", configurations
						.getConfigurationValue());
			}
			// Global
			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {
				globalmap = new HashMap();
				String intl = configurations.getConfigurationValue2();
				StringTokenizer st = new StringTokenizer(intl, ";");
				while (st.hasMoreTokens()) {
					String token = st.nextToken();
					globalmap.put(token, token);
				}
				vo.addConfiguration("service-global-upsxml", globalmap);
			}
		}

		vo.addMerchantConfiguration(configurations);
		return vo;

	}

	private String responsecode = null;
	private String responsetext = null;

	public String getResponsecode() {
		return responsecode;
	}

	public void setResponsecode(String responsecode) {
		this.responsecode = responsecode;
	}

	public String getResponsetext() {
		return responsetext;
	}

	public void setResponsetext(String responsetext) {
		this.responsetext = responsetext;
	}

}

class UPSParsedElements {

	private String statusCode;
	private String statusMessage;
	private String error = "";
	private String errorCode = "";
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

	public String getError() {
		return error;
	}

	public void setError(String error) {
		this.error = error;
	}

	public String getErrorCode() {
		return errorCode;
	}

	public void setErrorCode(String errorCode) {
		this.errorCode = errorCode;
	}

}



```
