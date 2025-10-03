# SalesManagerCustomerWSImpl.java

## Review

## 1. Summary

**Purpose**  
`SalesManagerCustomerWSImpl` is a SOAP web‑service implementation that exposes two operations to manage customers:

1. **createCustomer** – Creates a new customer or updates an existing one.  
2. **getCustomer** – Retrieves a customer by ID.

The service performs credential validation, input validation, and interacts with the underlying `CustomerService` layer to persist/retrieve customer entities. Responses are wrapped in custom response objects (`CreateCustomerWebServiceResponse`, `GetCustomerWebServiceResponse`).

**Key Components**

| Component | Role |
|-----------|------|
| `@WebService` / `@WebMethod` annotations | Expose SOAP endpoints. |
| `WebServiceCredentials` | Encapsulates merchant ID and API key for authentication. |
| `Customer` | DTO used in the SOAP contract (different from JPA entity). |
| `CustomerService` | Business‑layer service that interacts with the data store. |
| `RefCache` | In‑memory cache of reference data (countries, zones). |
| `EncryptionUtil` | Generates and verifies API keys. |
| `LocaleUtil`, `LanguageUtil` | Resolve `Locale` objects and language numbers. |
| `MessageSource` | Resolve i18n error messages. |
| `BeanUtils` | Copy properties between DTO and entity. |

**Design Patterns / Libraries**

* **Facade** – The WS implementation delegates to a service layer (`CustomerService`).  
* **DAO/Service** – Interaction with persistence is abstracted via `ServiceFactory`.  
* **Spring** – Dependency injection (via `SpringUtil.getBean`) and message source.  
* **Apache Commons BeanUtils / Lang** – Property copying and string utilities.  
* **Log4j** – Logging.  
* **JAX‑WS** – SOAP web service annotations.  

---

## 2. Detailed Description

### Flow Overview

| Step | Description |
|------|-------------|
| **Invocation** | Client calls either `createCustomer` or `getCustomer`. |
| **Locale Determination** | Default locale (`LocaleUtil.getDefaultLocale()`) is overridden if the incoming DTO supplies a language code. |
| **Credential Validation** | `validateCredentials()` checks the merchant ID / API key pair. |
| **Input Validation** | `validate()` performs a series of field‑level checks; any errors short‑circuit with status 2 (create) or status 0 (get). |
| **Entity Mapping** | The DTO (`Customer`) is copied into a JPA entity (`com.salesmanager.core.entity.customer.Customer`) via `BeanUtils`. Additional properties (billing address, zone names, etc.) are set manually. |
| **Reference Data Lookup** | If zone/country IDs are present, the service resolves the corresponding names from `RefCache`. |
| **Persistence** | `CustomerService.saveOrUpdateCustomer()` is called with a system entry type and locale. |
| **Response Construction** | Success results in status 1, error results in status 0 or 2, and messages are populated from the `MessageSource`. |
| **Exception Handling** | `ServiceException` is propagated with the error message; generic exceptions are logged and result in a generic “technical error” message. |

### Assumptions & Constraints

* **Thread Safety** – The service uses shared static state only for logging; all other objects are method‑local, so it is safe for concurrent use.  
* **Cache Availability** – `RefCache` is expected to be pre‑populated; missing data leads to validation failures.  
* **Locale Support** – Only the default locale or a language code supplied by the client is used; no country‑specific locale fallback.  
* **API Key Generation** – Relies on `EncryptionUtil.encrypt()`; any change to key algorithm will break existing clients.  

### Architecture Choices

* **Explicit DTOs** – Separate web‑service DTOs from persistence entities reduce coupling but introduce manual mapping logic.  
* **ServiceFactory** – A simple factory pattern is used instead of Spring dependency injection for services.  
* **Manual Response Building** – Each method builds its own response; no generic error handling helper is used beyond `setStatusMsg`.  

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `validateCredentials(Locale, WebServiceCredentials)` | Checks that the supplied API key matches the generated key for the merchant ID. | `locale`, `credentials` | void (throws `ServiceException`) | Logs errors |
| `createCustomer(WebServiceCredentials, Customer)` | Handles the customer creation/update flow. | `credentials`, `customer` | `CreateCustomerWebServiceResponse` | Persists customer, logs, updates response |
| `getCustomer(WebServiceCredentials, Customer)` | Retrieves a customer by ID. | `credentials`, `customer` | `GetCustomerWebServiceResponse` | Reads customer, logs, updates response |
| `setStatusMsg(MessageSource, Locale, WebServiceResponse, String, int)` | Helper to set status & message on a response. | `messageSource`, `locale`, `response`, `messageKey`, `status` | void | Sets fields |
| `validate(Customer, Locale, MessageSource)` | Validates required fields; returns array of error messages. | `customer`, `locale`, `messageSource` | `String[]` | None |
| `validate(String, String, List<String>, Locale, MessageSource)` | Field‑level validation; adds message to list. | `valueToValidate`, `validationErrorKey`, `validationErrorList`, `locale`, `messageSource` | void | Adds to list |

### Reusable / Utility Methods

* `validate()` and its helper `validate(String...)` are reusable for other DTOs.  
* `setStatusMsg()` can be used by future endpoints to standardize error responses.  

---

## 4. Dependencies

| Library | Type | Purpose |
|---------|------|---------|
| **JAX‑WS** (`javax.jws.*`) | Standard | Expose SOAP endpoints. |
| **Apache Commons BeanUtils** | Third‑party | Copy bean properties. |
| **Apache Commons Lang** | Third‑party | `StringUtils` for blank checks. |
| **Log4j** | Third‑party | Logging. |
| **Spring Framework** (`org.springframework.context.MessageSource`) | Third‑party | i18n messages, bean lookup (`SpringUtil.getBean`). |
| **SalesManager Core** | Internal | Entity classes, services (`CustomerService`, `ServiceFactory`, `RefCache`). |
| **EncryptionUtil** | Internal | API key generation/verification. |
| **LocaleUtil / LanguageUtil** | Internal | Resolve `Locale` objects and language numbers. |

*All dependencies are standard or internal to the SalesManager project; no external runtime requirements beyond the JVM.*

---

## 5. Additional Notes

### Edge Cases & Potential Issues

1. **API Key Regeneration** – If `EncryptionUtil.encrypt()` algorithm changes, existing credentials will fail. Consider persisting a hash or versioning keys.  
2. **Partial Updates** – The service copies all properties blindly; null fields in the DTO will overwrite existing values. A merge strategy or patch mechanism would be safer.  
3. **Zone/Country Name Resolution** – The logic first attempts to resolve zone names via `RefCache`. If the cache is stale or missing, the customer’s billing state may remain null. Cache refresh logic could be added.  
4. **Exception Granularity** – All non‑`ServiceException` errors are treated as a generic “technical” error. Exposing more specific error codes would improve client diagnostics.  
5. **MessageSource Retrieval** – `SpringUtil.getBean("messageSource")` is called in every method; this could be moved to a singleton field to reduce lookup overhead.  
6. **Locale Handling** – If a client supplies an unsupported language code, `LocaleUtil.getLocale()` may return null, leading to a `NullPointerException`. Input validation for locale should be added.  
7. **Thread‑Safe Cache Access** – `RefCache` is used without synchronization; if it can be modified at runtime, concurrent access may produce inconsistent reads.  
8. **Logging Sensitive Info** – Credentials are never logged, which is good, but logging the generated `apiKeyGen` in error scenarios could expose secrets. Ensure sensitive data is redacted.  

### Future Enhancements

| Area | Suggested Improvement |
|------|------------------------|
| **DTO Mapping** | Use MapStruct or a dedicated mapping layer to avoid manual property handling and reduce bugs. |
| **Validation Framework** | Replace custom `validate()` logic with Bean Validation (JSR‑380) annotations for cleaner code and reusable constraints. |
| **Error Codes** | Define an enum of error codes alongside messages to make the response machine‑readable. |
| **Caching** | Introduce a TTL or event‑based cache refresh for reference data to keep it up‑to‑date. |
| **API Security** | Move credential validation to a dedicated filter or interceptor to keep service methods focused on business logic. |
| **Unit Tests** | Add comprehensive tests covering happy paths, validation failures, and security checks. |
| **API Key Revocation** | Implement a revocation list or expiration for API keys to improve security. |
| **Localization** | Support dynamic locale selection based on request headers or client preference. |

### Code‑Level Suggestions

* Replace manual array conversion: `validationErrorList.toArray(new String[0])`.  
* Avoid casting from raw `Map` – use generics: `Map<Integer, Country>`.  
* Extract repeated `MessageSource` retrieval into a `@PostConstruct` bean.  
* Add `@Transactional` annotation on service layer methods instead of handling transactions in WS code.  

---

**Overall Assessment**

The implementation correctly exposes the required SOAP operations and follows a clear, layered design. The code is readable, with adequate error handling and logging. However, there are opportunities to reduce boilerplate, improve validation, and enhance security. Addressing the edge cases and refactoring the mapping/validation logic would make the service more robust and maintainable.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-3 Sep, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.service.ws.impl;

import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

import javax.jws.WebMethod;
import javax.jws.WebParam;
import javax.jws.WebResult;
import javax.jws.WebService;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.springframework.context.MessageSource;

import com.salesmanager.core.entity.customer.ws.CreateCustomerWebServiceResponse;
import com.salesmanager.core.entity.customer.ws.Customer;
import com.salesmanager.core.entity.customer.ws.GetCustomerWebServiceResponse;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.SystemUrlEntryType;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.ws.SalesManagerCustomerWS;
import com.salesmanager.core.service.ws.WebServiceCredentials;
import com.salesmanager.core.service.ws.WebServiceResponse;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.SpringUtil;


@WebService
public class SalesManagerCustomerWSImpl implements SalesManagerCustomerWS{

	private Logger log = Logger.getLogger(SalesManagerCustomerWSImpl.class);
	private static final String MESSAGE_SEPERATOR=",";
	
	/**
	 * Validates web service credentials
	 * @param locale
	 * @param credentials
	 * @throws ServiceException
	 */
	private void validateCredentials(Locale locale, WebServiceCredentials credentials) throws ServiceException {
		MessageSource messageSource = (MessageSource)SpringUtil.getBean("messageSource");
		
		try {
			
			int merchantId = credentials.getMerchantId();
			
			String k = EncryptionUtil.generatekey(String.valueOf(merchantId));
			String apiKeyGen = EncryptionUtil.encrypt(k, String.valueOf(merchantId));
			
			if(StringUtils.isBlank(apiKeyGen) || apiKeyGen.length()<16) {
				log.error("Problem with API KEY GENERATION " + apiKeyGen);
				throw new ServiceException(messageSource.getMessage("errors.technical", 
						null, locale));
			}
			
			String apiKey = credentials.getApiKey();
			
			if(StringUtils.isBlank(apiKey)) {
				throw new ServiceException(messageSource.getMessage("messages.error.ws.invalidcredentials", 
						null, locale));
			}
			
			if(!apiKeyGen.equals(apiKey)) {
				throw new ServiceException(messageSource.getMessage("messages.error.ws.invalidcredentials", 
						null, locale));
			}
			
			
		} catch (Exception e) {
			
			if(e instanceof ServiceException) {
				throw (ServiceException)e;
			}
			
			log.error(e);
			throw new ServiceException(messageSource.getMessage("errors.technical", 
					null, locale));
		}
		
	}
	
	/**
	 * Creates a new Customer
	 */
	@WebMethod
	public @WebResult CreateCustomerWebServiceResponse createCustomer(@WebParam(name="credentials")WebServiceCredentials credentials,@WebParam(name="customer")Customer customer) {
		MessageSource messageSource = (MessageSource)SpringUtil.getBean("messageSource");
		
		Locale locale = LocaleUtil.getDefaultLocale();
		if(StringUtils.isNotBlank(customer.getCustomerLang())) {
			locale = LocaleUtil.getLocale(customer.getCustomerLang());
		}

		CreateCustomerWebServiceResponse response = new CreateCustomerWebServiceResponse();
		try {
			
			//check credentials
			validateCredentials(locale,credentials);
			
			String[] validationErrorList = validate(customer, locale,messageSource);
			if(validationErrorList != null && validationErrorList.length>0){
				response.setMessages(validationErrorList);
				response.setStatus(2);
				return response;
			}
			
			CustomerService cservice = (CustomerService)ServiceFactory.getService(ServiceFactory.CustomerService);
			
			//if customer has customer id >0 check that it belongs to this merchant id
			com.salesmanager.core.entity.customer.Customer tmpCustomer = null;
			if(customer.getCustomerId()>0) {
				tmpCustomer = cservice.getCustomer(customer.getCustomerId());
				if(tmpCustomer!=null) {
					if(tmpCustomer.getMerchantId()!=credentials.getMerchantId()) {
						response.setMessages(new String[]{messageSource.getMessage("messages.authorization", 
								null, locale)});
						response.setStatus(0);
					}
				}
			}
			
			
			com.salesmanager.core.entity.customer.Customer newCustomer = new com.salesmanager.core.entity.customer.Customer();
			
			if(tmpCustomer!=null) {//modify existing customer
				newCustomer = tmpCustomer;
			}
			BeanUtils.copyProperties(newCustomer, customer);
			
			//copy properties to billing
			newCustomer.setCustomerBillingCity(customer.getCustomerCity());
			newCustomer.setCustomerBillingCountryId(customer.getCustomerCountryId());
			newCustomer.setCustomerBillingCountryName(newCustomer.getBillingCountry());
			newCustomer.setCustomerBillingFirstName(customer.getCustomerFirstname());
			newCustomer.setCustomerBillingLastName(customer.getCustomerLastname());
			newCustomer.setCustomerBillingPostalCode(customer.getCustomerPostalCode());
			newCustomer.setCustomerBillingState(newCustomer.getStateProvinceName());
			newCustomer.setCustomerBillingStreetAddress(customer.getCustomerStreetAddress());
			newCustomer.setCustomerBillingZoneId(customer.getCustomerZoneId());

			
			
			newCustomer.setLocale(locale);
			newCustomer.setMerchantId(credentials.getMerchantId());
			
			
			if(StringUtils.isBlank(customer.getZoneName()) && customer.getCustomerZoneId()>0) {
				java.util.Map zones = (java.util.Map)RefCache.getAllZonesmap(LanguageUtil.getLanguageNumberCode(locale.getLanguage()));
				if(zones!=null) {
					Zone z = (Zone)zones.get(customer.getCustomerZoneId());
					if(z!=null) {
						newCustomer.setCustomerState(z.getZoneName());
					}
				}
			}
			
			if(StringUtils.isBlank(newCustomer.getBillingState()) && newCustomer.getCustomerZoneId()>0) {
				java.util.Map zones = (java.util.Map)RefCache.getAllZonesmap(LanguageUtil.getLanguageNumberCode(locale.getLanguage()));
				if(zones!=null) {
					Zone z = (Zone)zones.get(newCustomer.getCustomerZoneId());
					if(z!=null) {
						newCustomer.setCustomerBillingState(z.getZoneName());
					}
				}
			}

			
			
			
			cservice.saveOrUpdateCustomer(newCustomer, SystemUrlEntryType.WEB,locale);
			
			response.setMessages(new String[]{messageSource.getMessage("messages.customer.customerregistered", 
					null, locale)});
			response.setStatus(1);
			response.setCustomerId(newCustomer.getCustomerId());

		} catch(Exception e){
			
			if(e instanceof ServiceException) {
				String msg[] = {((ServiceException)e).getMessage()};
				response.setMessages(msg);
				response.setStatus(0);
			} else {
			
				log.error("Exception occurred while creating Customer",e);
				response.setMessages(new String[]{messageSource.getMessage("errors.technical", 
						null, locale)});
				response.setStatus(0);
				
			}
		}
		return response;
	}
	
	/**
	 * Get customer for a customerId and merchantId
	 */
	@WebMethod
	public @WebResult GetCustomerWebServiceResponse getCustomer(@WebParam(name="credentials")
			WebServiceCredentials credentials, @WebParam(name="customer")Customer customer) {
		MessageSource messageSource = (MessageSource)SpringUtil.getBean("messageSource");
		
		Locale locale = LocaleUtil.getDefaultLocale();
		if(StringUtils.isNotBlank(customer.getCustomerLang())) {
			locale = LocaleUtil.getLocale(customer.getCustomerLang());
		}

		GetCustomerWebServiceResponse response = new GetCustomerWebServiceResponse();
		try {
			
			if(customer.getCustomerId() == 0){
				setStatusMsg(messageSource, locale, response,"messages.authorization",0);
				return response;
			}
			//check credentials
			validateCredentials(locale,credentials);
			
			CustomerService cservice = (CustomerService)ServiceFactory.getService(ServiceFactory.CustomerService);
			com.salesmanager.core.entity.customer.Customer entityCustomer  = cservice.getCustomer(customer.getCustomerId());
			if(entityCustomer == null){
				setStatusMsg(messageSource, locale, response,"messages.customer.doesnotexist",0);
				return response;
			}
			
			if(entityCustomer.getMerchantId()!=credentials.getMerchantId()) {
				setStatusMsg(messageSource, locale, response,"messages.authorization",0);
				return response;
			}
			
			Customer webCustomer = new Customer();
			BeanUtils.copyProperties(webCustomer, entityCustomer);
			response.setCustomer(webCustomer);
			response.setStatus(1);

		} catch(Exception e){
			
			if(e instanceof ServiceException) {
				String[] msg = {((ServiceException)e).getMessage()};
				response.setMessages(msg);
				response.setStatus(0);
			} else {
			
				log.error("Exception occurred while creating Customer",e);
				response.setMessages(new String[]{messageSource.getMessage("errors.technical", 
						null, locale)});
				response.setStatus(0);
				
			}
		}
		return response;		
		
	}

	private void setStatusMsg(MessageSource messageSource, Locale locale,
			WebServiceResponse response,String messageKey,int status) {
		response.setMessages(new String[]{messageSource.getMessage(messageKey, 
				null, locale)});
		response.setStatus(status);
	}	
	
	private static String[] validate(Customer customer,Locale locale,
			MessageSource messageSource){
		List<String> validationErrorList = new ArrayList<String>();
		validate(customer.getCustomerFirstname(), "messages.required.firstname", 
				validationErrorList, locale, messageSource);
		validate(customer.getCustomerLastname(), "messages.required.lastname", 
				validationErrorList, locale, messageSource);
		validate(customer.getCustomerEmailAddress(), "messages.required.email", 
				validationErrorList, locale, messageSource);
		validate(customer.getCustomerTelephone(), "messages.required.phone", 
				validationErrorList, locale, messageSource);
		validate(customer.getCustomerCity(), "messages.required.city", 
				validationErrorList, locale, messageSource);
		validate(customer.getCustomerPostalCode(), "messages.required.postalcode", 
				validationErrorList, locale, messageSource);		
		validate(customer.getCustomerStreetAddress(), "messages.required.streetaddress", 
				validationErrorList, locale, messageSource);
		validate(customer.getCustomerLang(), "messages.required.language", 
				validationErrorList, locale, messageSource);
		
		//validate country
		if(customer.getCustomerCountryId()==0) {
			validationErrorList.add(messageSource.getMessage("messages.required.customercountrycode", 
					null, locale));
		} else {
		
			java.util.Map countries = RefCache.getAllcountriesmap(1);
			Country c = (Country)countries.get(customer.getCustomerCountryId());
			if(c==null) {
				validationErrorList.add(messageSource.getMessage("messages.required.customercountrycode", 
						null, locale));
			}
		
	   }
		
		//validate zone
		if(customer.getCustomerZoneId()==0) {
			if(StringUtils.isBlank(customer.getZoneName())) {
				validationErrorList.add(messageSource.getMessage("messages.required.customerzonecode", 
						null, locale));
			}
		} else {
			java.util.Map zones = RefCache.getAllZonesmap(1);
			Zone z = (Zone)zones.get(customer.getCustomerZoneId());
			if(z==null) {
				validationErrorList.add(messageSource.getMessage("messages.required.customerzonecode", 
						null, locale));
			}
		}
		
		if(validationErrorList.size()>0) {
			 String[] messages = (String[])validationErrorList.toArray();
			 return messages;
		} else {
			return null;
		}

	}
	
	/**
	 * Utility method to validate for not-null/empty check.
	 * @param valueToValidate String to check for not-null/empty
	 * @param validationErrorKey If validation fails then error message key to use.
	 * @param validationErrorList List of error messages
	 * @param locale Locale
	 * @param messageSource MessageSource
	 */
	private static void validate(String valueToValidate,String validationErrorKey,
			List<String> validationErrorList,Locale locale,MessageSource messageSource){
		if (StringUtils.isBlank(valueToValidate)) {
			validationErrorList.add(messageSource.getMessage(validationErrorKey, 
					null, locale));
		}
	}
	
/*	private static String getMessages(List<String> errorMessages,String seperator){
		String message = null;
		for(String msg:errorMessages){
			if(message != null){
				message += seperator+msg;
			}else{
				message = msg;
			}
		}
		return message;
	}*/

}



```
