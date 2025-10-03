# USPSQuotesImpl.java

## Review

## 1. Summary  
The `USPSQuotesImpl` class implements the `ShippingQuotesModule` interface to provide real‑time USPS rate quotes for an e‑commerce platform.  
Key responsibilities:

1. **Build and send a USPS RateV3 / IntlRate XML request**  
   * Builds the request body from customer, package, and merchant data.  
   * Handles domestic vs. international logic (different XML tags, URL params).  
2. **Parse the XML response**  
   * Uses Apache Commons Digester to map XML elements to a custom `USPSParsedElements` object that contains a list of `ShippingOption` objects.  
3. **Convert currency**  
   * All USPS responses are in USD; the module converts the final price to the merchant’s currency.  
4. **Provide configuration utilities**  
   * `getConfiguration` extracts credentials, properties, package codes, and service lists from the merchant’s configuration store.  
5. **Metadata & localisation**  
   * Uses `LabelUtil` for internationalised labels and `ShippingConstants` for static keys.  

The class heavily relies on third‑party libraries (Commons HttpClient, Digester, Log4j) and a large internal API surface (services, cache, util classes).  

---

## 2. Detailed Description  
### 2.1 Execution Flow  
1. **Initialisation** – `getShippingQuote` is called with order details, package list, customer, store, and locale.  
2. **Configuration Retrieval**  
   * `CommonService` obtains the *usps* module definition (`cis`).  
   * `MerchantService` loads the shipping credentials and package settings.  
3. **Request Construction**  
   * The code distinguishes between domestic (US) and international orders.  
   * It calculates aggregate weight, dimensions, girth, and total value.  
   * Builds a single `<Package>` XML node (or multiple if needed in the future) with the appropriate tags.  
4. **HTTP Call** – The XML is URL‑encoded and sent to the USPS API via `HttpClient`/`GetMethod`.  
5. **Response Parsing** – The returned XML is parsed by a `Digester` into `USPSParsedElements`, which holds a list of `ShippingOption`s.  
6. **Post‑processing**  
   * Convert prices to the store currency.  
   * Optionally append estimated delivery days.  
7. **Return** – A collection of shipping options is returned; on any error the method logs and returns `null`.  

### 2.2 Assumptions & Constraints  
* **Single package handling** – The code aggregates all package details into one request. USPS may support multiple packages, but this is not implemented.  
* **Hard‑coded API URLs & ports** – These come from the module definition; no dynamic discovery.  
* **Error handling** – Exceptions are swallowed in the `finally` block; the caller gets `null` on failure, making it hard to differentiate error types.  
* **No timeout configuration** – The HTTP client uses defaults; network latency could be problematic in a high‑traffic shop.  
* **Locale‑specific labels** – The module expects a resource bundle for US postal code trimming, but uses hard‑coded string literals for “ALL” services.  

### 2.3 Architecture & Design Choices  
* **Service Factory Pattern** – `ServiceFactory.getService` gives loose coupling to underlying implementations.  
* **Digester for XML** – Simplifies mapping, but is brittle (needs to match exact XML structure).  
* **Configuration through `ConfigurationResponse`** – Allows dynamic loading of keys and properties but couples the API to the merchant configuration model.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `Collection<ShippingOption> getShippingQuote(...)` | Main entry point: generates USPS rate request, calls API, parses results. | `ConfigurationResponse config, BigDecimal orderTotal, Collection<PackageDetail> packages, Customer customer, MerchantStore store, Locale locale` | `Collection<ShippingOption>` (or `null` on error) | Logs, HTTP call, currency conversion |
| `String getShippingMethodDescription(Locale locale)` | Internationalised shipping method name. | `Locale locale` | `String` | None |
| `ConfigurationResponse getConfiguration(MerchantConfiguration configurations, ConfigurationResponse vo)` | Extracts credentials, properties, package and service settings from a merchant configuration. | `MerchantConfiguration configurations, ConfigurationResponse vo` | `ConfigurationResponse` (augmented) | None |
| `void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)` | Persist config changes – *not implemented*; stub for action. | `int merchantid, ConfigurationResponse vo, HttpServletRequest request` | None | None |
| `void addOption(ShippingOption option)` (in `USPSParsedElements`) | Adds parsed shipping option to internal list. | `ShippingOption option` | None | Adds to `options` list |
| `List getOptions()` | Returns parsed shipping options. | None | `List<ShippingOption>` | None |
| `String getError()` / `setError(String)` | Holds any parsing error string. | `String` | `String` | None |
| `String getStatusCode()` / `setStatusCode(String)` | Holds status code from response. | `String` | `String` | None |

### Reusable / Utility Methods  
* `CurrencyUtil.getMeasure` – converts dimensions to inches.  
* `CurrencyUtil.getWeight` – converts weight to pounds.  
* `CurrencyUtil.convertToCurrency` – currency conversion.  
* `DateUtil.addDaysToCurrentDate` – shipping date calculation.  
* `ShippingUtil.getKeys` / `getProperties` – parse configuration strings into objects.  

---

## 4. Dependencies  

| Library | Purpose | Is Standard? |
|---------|---------|--------------|
| `org.apache.commons.digester.Digester` | XML parsing into Java objects | 3rd‑party |
| `org.apache.commons.httpclient.HttpClient` / `GetMethod` | HTTP GET to USPS API | 3rd‑party |
| `org.apache.commons.lang.StringUtils` | String utilities | 3rd‑party |
| `org.apache.log4j.Logger` | Logging | 3rd‑party |
| `javax.servlet.http.HttpServletRequest` | Web request context for config store | Standard (Servlet API) |
| `com.salesmanager.*` | Internal services, entities, utilities, constants | Internal |
| `java.util.*` | Collections, Map, Iterator | Standard |

No platform‑specific assumptions beyond the servlet container.  

---

## 5. Additional Notes  

### 5.1 Edge Cases & Potential Issues  
1. **Package Count > 1** – Aggregating all packages into one request may exceed USPS weight or dimension limits; USPS supports separate package requests.  
2. **International MailType & Girth** – The code does not include `<GXG>` block for international packages; USPS may require more detailed information for certain countries.  
3. **Currency Rounding** – `CurrencyUtil.convertToCurrency` may produce rounding errors; no explicit rounding mode is specified.  
4. **Error Handling** – Returning `null` on any exception hides the cause. Callers cannot distinguish between a connection failure, XML parsing error, or missing configuration.  
5. **Hard‑coded Service Code “ALL”** – Domestic request uses `"ALL"` to request all services; if USPS deprecates this or changes the API, the module will fail silently.  
6. **URL Encoding** – Uses `java.net.URLEncoder.encode(xmlbuffer.toString())`; this encodes spaces as `+`. USPS may expect `%20`.  
7. **Thread Safety** – `Logger` and `HttpClient` are instance‑level; the class is not thread‑safe if used concurrently.  
8. **Performance** – For every quote request a new `HttpClient` and `Digester` are created; pooling or reuse could reduce overhead.  

### 5.2 Suggested Enhancements  
* **Refactor Request Building** – Separate a `USPSRequestBuilder` that constructs XML using a DOM or JAXB model.  
* **Support Multiple Packages** – Iterate over `packages`, creating separate `<Package>` nodes.  
* **Better Error Propagation** – Throw custom exceptions (`ShippingQuoteException`) instead of returning `null`.  
* **Configure HttpClient** – Use connection pooling, timeouts, and request retries.  
* **Unit‑Testable** – Inject `HttpClient` and a parser; mock responses to test various XML scenarios.  
* **Use Java 8 Streams** – Cleaner dimension/weight aggregation and conversion.  
* **Locale‑Aware Service Mapping** – Load service codes and descriptions from a resource bundle to avoid hard‑coded strings.  
* **Logging Enhancements** – Include response payload snippets and request IDs for debugging.  

### 5.3 Code‑Quality Observations  
* **Long Method** – `getShippingQuote` is ~400 lines long; consider splitting into smaller private methods (`buildRequestXml`, `callApi`, `parseResponse`, `processOptions`).  
* **Variable Naming** – Many variables (`st`, `intl`, `rtdetails`) are ambiguous; more descriptive names improve readability.  
* **Magic Numbers** – `ShippingConstants.DISPLAY_RT_QUOTE_TIME` etc. are used; comments explaining the meaning of each constant would help.  
* **Redundant Imports / Unused Variables** – The class declares `reader` but never uses it; `reader` and its closing logic can be removed.  

---

### 5.4 Final Assessment  
The implementation is functional for a simple use case (single package domestic or international quotes) and integrates cleanly with the existing SalesManager framework.  
However, it is fragile to API changes, lacks robust error handling, and does not scale well with multiple packages or high request volume.  
Addressing the suggestions above would yield a more maintainable, testable, and production‑ready USPS shipping module.

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
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.digester.Digester;
import org.apache.commons.httpclient.HttpClient;
import org.apache.commons.httpclient.methods.GetMethod;
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
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LogMerchantUtil;

public class USPSQuotesImpl implements ShippingQuotesModule {

	private Logger log = Logger.getLogger(USPSQuotesImpl.class);

	public Collection<ShippingOption> getShippingQuote(
			ConfigurationResponse config, BigDecimal orderTotal,
			Collection<PackageDetail> packages, Customer customer,
			MerchantStore store, Locale locale) {

		CoreModuleService cis = null;

		StringBuffer xmlbuffer = new StringBuffer();
		BufferedReader reader = null;

		GetMethod httpget = null;

		try {

			CommonService cservice = (CommonService) ServiceFactory
					.getService(ServiceFactory.CommonService);

			String countrycode = CountryUtil.getCountryIsoCodeById(store
					.getCountry());
			cis = cservice.getModule(countrycode, "usps");

			if (cis == null) {
				log.error("Can't retreive an integration service [countryid "
						+ store.getCountry() + " usps subtype 1]");
			}

			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationRequest request_prefs = new ConfigurationRequest(store
					.getMerchantId(),
					ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
			ConfigurationResponse vo_prefs = service
					.getConfiguration(request_prefs);

			String pack = (String) vo_prefs.getConfiguration("package-usps");
			if (pack == null) {
				log
						.debug("Will assign packaging type 02 to USPS shipping for merchantid "
								+ store.getMerchantId());
				pack = "PARCEL";
			}

			ConfigurationRequest request = new ConfigurationRequest(store
					.getMerchantId(), ShippingConstants.MODULE_SHIPPING_RT_CRED);
			ConfigurationResponse vo = service.getConfiguration(request);

			if (vo == null) {
				throw new Exception("ConfigurationVO is null upsxml");
			}

			// if not shipping to USA
			boolean domestic = true;
			int shippingCountryId = customer.getCustomerCountryId();
			if (shippingCountryId != Constants.US_COUNTRY_ID) {
				domestic = false;
			}

			IntegrationKeys keys = (IntegrationKeys) vo
					.getConfiguration("usps-keys");

			if (keys == null) {
				throw new Exception(
						"Integration keys are null from configuration request usps");
			}

			String xmlheader = "<RateV3Request USERID=\"" + keys.getUserid()
					+ "\">";
			if (!domestic) {
				xmlheader = "<IntlRateRequest USERID=\"" + keys.getUserid()
						+ "\">";
			}

			StringBuffer xmldatabuffer = new StringBuffer();

			Map countriesMap = (Map) RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			Country customerCountry = (Country) countriesMap.get(customer
					.getCustomerCountryId());

			Iterator packagesIterator = packages.iterator();

			double totalW = 0;
			double totalH = 0;
			double totalL = 0;
			double totalG = 0;
			double totalP = 0;

			while (packagesIterator.hasNext()) {

				PackageDetail detail = (PackageDetail) packagesIterator.next();

				// need size in inch
				double w = CurrencyUtil.getMeasure(detail.getShippingWidth(),
						store, Constants.INCH_SIZE_UNIT);
				double h = CurrencyUtil.getMeasure(detail.getShippingHeight(),
						store, Constants.INCH_SIZE_UNIT);
				double l = CurrencyUtil.getMeasure(detail.getShippingLength(),
						store, Constants.INCH_SIZE_UNIT);

				totalW = totalW + w;
				totalH = totalH + h;
				totalL = totalL + l;

				// Girth = Length + (Width x 2) + (Height x 2)
				double girth = l + (w * 2) + (h * 2);

				totalG = totalG + girth;

				// need weight in pounds
				double p = CurrencyUtil.getWeight(detail.getShippingWeight(),
						store, Constants.LB_WEIGHT_UNIT);

				totalP = totalP + p;

			}

			BigDecimal convertedOrderTotal = CurrencyUtil.convertToCurrency(
					orderTotal, store.getCurrency(),
					Constants.CURRENCY_CODE_USD);

			// calculate total shipping volume

			// ship date is 3 days from here
			Date newDate = DateUtil.addDaysToCurrentDate(3);
			String shipDate = DateUtil.formatDateMonthString(newDate);

			int i = 1;

			// need pounds and ounces
			int pounds = (int) totalP;
			String ouncesString = String.valueOf(totalP - pounds);
			int ouncesIndex = ouncesString.indexOf(".");
			String ounces = "00";
			if (ouncesIndex > -1) {
				ounces = ouncesString.substring(ouncesIndex + 1);
			}

			String size = "REGULAR";

			if (totalL + totalG <= 64) {
				size = "REGULAR";
			} else if (totalL + totalG <= 108) {
				size = "LARGE";
			} else {
				size = "OVERSIZE";
			}

			/**
			 * Domestic <Package ID="1ST"> <Service>ALL</Service>
			 * <ZipOrigination>90210</ZipOrigination>
			 * <ZipDestination>96698</ZipDestination> <Pounds>8</Pounds>
			 * <Ounces>32</Ounces> <Container/> <Size>REGULAR</Size>
			 * <Machinable>true</Machinable> </Package>
			 * 
			 * //MAXWEIGHT=70 lbs
			 * 
			 * 
			 * //domestic container default=VARIABLE whiteSpace=collapse
			 * enumeration=VARIABLE enumeration=FLAT RATE BOX enumeration=FLAT
			 * RATE ENVELOPE enumeration=LG FLAT RATE BOX
			 * enumeration=RECTANGULAR enumeration=NONRECTANGULAR
			 * 
			 * //INTL enumeration=Package enumeration=Postcards or aerogrammes
			 * enumeration=Matter for the blind enumeration=Envelope
			 * 
			 * Size May be left blank in situations that do not Size. Defined as
			 * follows: REGULAR: package plus girth is 84 inches or less; LARGE:
			 * package length plus girth measure more than 84 inches not more
			 * than 108 inches; OVERSIZE: package length plus girth is more than
			 * 108 but not 130 inches. For example: <Size>REGULAR</Size>
			 * 
			 * International <Package ID="1ST"> <Machinable>true</Machinable>
			 * <MailType>Envelope</MailType> <Country>Canada</Country>
			 * <Length>0</Length> <Width>0</Width> <Height>0</Height>
			 * <ValueOfContents>250</ValueOfContents> </Package>
			 * 
			 * <Package ID="2ND"> <Pounds>4</Pounds> <Ounces>3</Ounces>
			 * <MailType>Package</MailType> <GXG> <Length>46</Length>
			 * <Width>14</Width> <Height>15</Height> <POBoxFlag>N</POBoxFlag>
			 * <GiftFlag>N</GiftFlag> </GXG>
			 * <ValueOfContents>250</ValueOfContents> <Country>Japan</Country>
			 * </Package>
			 */

			xmldatabuffer.append("<Package ID=\"").append(i).append("\">");
			// if domestic

			// user selected services
			// Map selectedintlservices =
			// (Map)config.getConfiguration("service-global-usps");

			// now get corresponding code
			// Map allservices =
			// com.salesmanager.core.util.ShippingUtil.buildServiceMapLabelByCode("usps",locale);
			// TreeBidiMap bidiMap = new TreeBidiMap(allservices);
			// Map invertedservicesmap = (Map)bidiMap.inverseBidiMap();

			// Iterator selServices = selectedintlservices.keySet().iterator();
			// while(selServices.hasNext()) {

			// String serviceId = (String)selServices.next();

			// String svc = (String)invertedservicesmap.get(serviceId);

			if (domestic) {

				xmldatabuffer.append("<Service>");
				xmldatabuffer.append("ALL");
				xmldatabuffer.append("</Service>");
				xmldatabuffer.append("<ZipOrigination>");
				xmldatabuffer.append(com.salesmanager.core.util.ShippingUtil
						.trimPostalCode(store.getStorepostalcode()));
				xmldatabuffer.append("</ZipOrigination>");
				xmldatabuffer.append("<ZipDestination>");
				xmldatabuffer.append(com.salesmanager.core.util.ShippingUtil
						.trimPostalCode(customer.getCustomerPostalCode()));
				xmldatabuffer.append("</ZipDestination>");
				xmldatabuffer.append("<Pounds>");
				xmldatabuffer.append(pounds);
				xmldatabuffer.append("</Pounds>");
				xmldatabuffer.append("<Ounces>");
				xmldatabuffer.append(ounces);
				xmldatabuffer.append("</Ounces>");
				xmldatabuffer.append("<Container>");
				xmldatabuffer.append(pack);
				xmldatabuffer.append("</Container>");
				xmldatabuffer.append("<Size>");
				xmldatabuffer.append(size);
				xmldatabuffer.append("</Size>");
				xmldatabuffer.append("<ShipDate>");
				xmldatabuffer.append(shipDate);
				xmldatabuffer.append("</ShipDate>");
			} else {
				// if international
				xmldatabuffer.append("<Pounds>");
				xmldatabuffer.append(pounds);
				xmldatabuffer.append("</Pounds>");
				xmldatabuffer.append("<Ounces>");
				xmldatabuffer.append(ounces);
				xmldatabuffer.append("</Ounces>");
				xmldatabuffer.append("<MailType>");
				xmldatabuffer.append("Package");
				xmldatabuffer.append("</MailType>");
				xmldatabuffer.append("<ValueOfContents>");
				xmldatabuffer.append(convertedOrderTotal);
				xmldatabuffer.append("</ValueOfContents>");
				xmldatabuffer.append("<Country>");
				xmldatabuffer.append(customerCountry.getCountryName());
				xmldatabuffer.append("</Country>");
			}

			// }

			// if international & CXG
			/*
			 * xmldatabuffer.append("<CXG>"); xmldatabuffer.append("<Length>");
			 * xmldatabuffer.append(""); xmldatabuffer.append("</Length>");
			 * xmldatabuffer.append("<Width>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</Width>");
			 * xmldatabuffer.append("<Height>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</Height>");
			 * xmldatabuffer.append("<POBoxFlag>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</POBoxFlag>");
			 * xmldatabuffer.append("<GiftFlag>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</GiftFlag>");
			 * xmldatabuffer.append("</CXG>");
			 */

			/*
			 * xmldatabuffer.append("<Width>"); xmldatabuffer.append(totalW);
			 * xmldatabuffer.append("</Width>");
			 * xmldatabuffer.append("<Length>"); xmldatabuffer.append(totalL);
			 * xmldatabuffer.append("</Length>");
			 * xmldatabuffer.append("<Height>"); xmldatabuffer.append(totalH);
			 * xmldatabuffer.append("</Height>");
			 * xmldatabuffer.append("<Girth>"); xmldatabuffer.append(totalG);
			 * xmldatabuffer.append("</Girth>");
			 */

			xmldatabuffer.append("</Package>");

			String xmlfooter = "</RateRequest>";
			if (!domestic) {
				xmlfooter = "</IntlRateRequest>";
			}

			xmlbuffer.append(xmlheader.toString()).append(
					xmldatabuffer.toString()).append(xmlfooter.toString());

			log.debug("USPS QUOTE REQUEST " + xmlbuffer.toString());

			String data = "";

			IntegrationProperties props = (IntegrationProperties) config
					.getConfiguration("usps-properties");

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

			String encoded = java.net.URLEncoder.encode(xmlbuffer.toString());

			String completeUri = uri + "?API=RateV3&XML=" + encoded;
			if (!domestic) {
				completeUri = uri + "?API=IntlRate&XML=" + encoded;
			}

			// ?API=RateV3

			httpget = new GetMethod(protocol + "://" + host + ":" + port
					+ completeUri);
			// RequestEntity entity = new
			// StringRequestEntity(xmlbuffer.toString(),"text/plain","UTF-8");
			// httpget.setRequestEntity(entity);

			int result = client.executeMethod(httpget);
			if (result != 200) {
				log.error("Communication Error with usps quote " + result + " "
						+ protocol + "://" + host + ":" + port + uri);
				throw new Exception("USPS quote communication error " + result);
			}
			data = httpget.getResponseBodyAsString();
			log.debug("usps quote response " + data);

			UPSParsedElements parsed = new UPSParsedElements();

			/**
			 * <RateV3Response> <Package ID="1ST">
			 * <ZipOrigination>44106</ZipOrigination>
			 * <ZipDestination>20770</ZipDestination>
			 */

			Digester digester = new Digester();
			digester.push(parsed);

			if (domestic) {

				digester.addCallMethod("RateV3Response/Package/Error",
						"setError", 0);
				digester
						.addObjectCreate(
								"RateV3Response/Package/Postage",
								com.salesmanager.core.entity.shipping.ShippingOption.class);
				digester.addSetProperties("RateV3Response/Package/Postage",
						"CLASSID", "optionId");
				digester.addCallMethod(
						"RateV3Response/Package/Postage/MailService",
						"optionName", 0);
				digester.addCallMethod(
						"RateV3Response/Package/Postage/MailService",
						"optionCode", 0);
				digester.addCallMethod("RateV3Response/Package/Postage/Rate",
						"optionPrice", 0);
				digester
						.addCallMethod(
								"RateV3Response/Package/Postage/Commitment/CommitmentDate",
								"estimatedNumberOfDays", 0);
				digester.addSetNext("RateV3Response/Package/Postage",
						"addOption");

			} else {

				digester.addCallMethod("IntlRateResponse/Package/Error",
						"setError", 0);
				digester
						.addObjectCreate(
								"IntlRateResponse/Package/Service",
								com.salesmanager.core.entity.shipping.ShippingOption.class);
				digester.addSetProperties("IntlRateResponse/Package/Service",
						"ID", "optionId");
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/SvcDescription",
						"setOptionName", 0);
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/SvcDescription",
						"setOptionCode", 0);
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/Postage",
						"setOptionPriceText", 0);
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/SvcCommitments",
						"setEstimatedNumberOfDays", 0);
				digester.addSetNext("IntlRateResponse/Package/Service",
						"addOption");

			}

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
			// cost is in USD, need to do conversion

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

			LabelUtil labelUtil = LabelUtil.getInstance();
			// Map serviceMap =
			// com.salesmanager.core.util.ShippingUtil.buildServiceMap("usps",locale);

			List options = parsed.getOptions();

			Collection returnColl = null;

			if (options != null && options.size() > 0) {

				returnColl = new ArrayList();
				// Map selectedintlservices =
				// (Map)config.getConfiguration("service-global-usps");
				// need to create a Map of LABEL - LABLEL
				// Iterator servicesIterator =
				// selectedintlservices.keySet().iterator();
				// Map services = new HashMap();

				// ResourceBundle bundle = ResourceBundle.getBundle("usps",
				// locale);

				// while(servicesIterator.hasNext()) {
				// String key = (String)servicesIterator.next();
				// String value =
				// bundle.getString("shipping.quote.services.label." + key);
				// services.put(value, key);
				// }

				Iterator it = options.iterator();
				while (it.hasNext()) {
					ShippingOption option = (ShippingOption) it.next();
					option.setCurrency(Constants.CURRENCY_CODE_USD);

					StringBuffer description = new StringBuffer();
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

					// if(!services.containsKey(option.getOptionCode())) {
					// if(returnColl==null) {
					// returnColl = new ArrayList();
					// }
					// returnColl.add(option);
					// }
					returnColl.add(option);
				}

				// if(options.size()==0) {
				// CommonService.logServiceMessage(store.getMerchantId(),
				// " none of the service code returned by UPS [" +
				// selectedintlservices.keySet().toArray(new
				// String[selectedintlservices.size()]) +
				// "] for this shipping is in your selection list");
				// }

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
			if (httpget != null) {
				httpget.releaseConnection();
			}
		}

	}

	public String getShippingMethodDescription(Locale locale) {
		return LabelUtil.getInstance().getText(locale, "module.usps");
	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo)
			throws Exception {
		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_CRED)) {// handle
																// credentials

			if (!StringUtils.isBlank(configurations.getConfigurationValue1())) {

				IntegrationKeys keys = ShippingUtil.getKeys(configurations
						.getConfigurationValue1());
				vo.addConfiguration("usps-keys", keys);

			}

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationProperties props = ShippingUtil
						.getProperties(configurations.getConfigurationValue2());
				vo.addConfiguration("usps-properties", props);

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
				vo.addConfiguration("package-usps", configurations
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
				vo.addConfiguration("service-global-usps", globalmap);
			}
		}

		vo.addMerchantConfiguration(configurations);
		return vo;
	}

	public void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {
		// implemented in ShippinguspsAction

	}

}

class USPSParsedElements {

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
