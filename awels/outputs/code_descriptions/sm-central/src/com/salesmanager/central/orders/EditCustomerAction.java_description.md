# EditCustomerAction.java

## Review

## 1. Summary  
`EditCustomerAction` is a Struts 2 action that allows merchants to edit the customer‑related data (email, billing address, shipping address) of an existing order.  
- **Core responsibilities**  
  1. Load an order, verify that the current merchant owns it, and expose it to the view.  
  2. Render the edit screens for email, billing, and shipping.  
  3. Persist the edited fields back to the database.  
- **Key collaborators**  
  - `OrderService` – CRUD on `Order` objects.  
  - `ReferenceService` / `RefCache` – lookup of countries and zones.  
  - `MerchantService` – read configuration values (e.g. shipping method).  
  - `Context` – session‑level data such as merchant id and language.  
- **Patterns & frameworks**  
  - *MVC* – the action is the controller, Struts 2 templates the view.  
  - *Service layer* – business logic is delegated to injected services.  
  - *Cache* – `RefCache` keeps lookup tables in memory.  
  - *Preparable* – Struts 2 lifecycle hook used to set the page title.  

## 2. Detailed Description  
The action follows a typical request‑processing pattern:

1. **Initialization** – `prepare()` sets the page title.  
2. **Authorization & data loading** –  
   * `prepareOrderDetails()` pulls the order using its ID, checks ownership, and prepares the UI selections (country/zone).  
3. **View rendering** –  
   * `displayOrderEmailAddress()`, `viewShippingCustomer()`, `viewBillingCustomer()` populate the `order` object for the view.  
4. **Edit submission** –  
   * `editOrderEmailAddress()`, `editShipping()`, `editBilling()` accept form data, validate it, update the order, and persist it.  
5. **Error handling** – Each method catches `AuthorizationException` and generic `Exception`, logs the error, and sets an appropriate message on the `ActionSupport` superclass.  

The action relies on `Context` stored in the HTTP session for merchant ID, language, and country data. The `Order` entity is used to hold both customer data and the order itself, which is a design decision specific to the underlying domain model.

### Assumptions & Constraints
- The current session always contains a `Context` object.  
- `OrderService` and other services are thread‑safe and can be reused per request.  
- `RefCache` holds immutable country/zone maps.  
- The view layer correctly uses the action’s fields (`order`, `customerinformation`, `countryid`, etc.).  

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `prepare()` | Struts `Preparable` hook – sets page title. | none | `SUCCESS` (inherited) | Sets page title via superclass |
| `prepareOrderDetails()` | Loads the order, authorises, and preps country/zone selections. | none | none | Sets `order`, `countryid`, UI selections |
| `editOrderEmailAddress()` | Validates and updates the customer’s email address. | `order` (from form) | `SUCCESS` or `INPUT` | Persists email, sets success/technical message |
| `displayOrderEmailAddress()` | Loads full order for email view. | `order` (from form) | `SUCCESS` | Sets `order` field |
| `viewShippingCustomer()` | Pre‑pares the edit‑shipping view (country/zone, shipping method). | `order` (from form) | `SUCCESS` | Sets UI attributes (`showaddressvalidation`, country) |
| `viewBillingCustomer()` | Pre‑pares the edit‑billing view. | `order` (from form) | `SUCCESS` | None beyond `order` |
| `editShipping()` | Applies changes from the shipping form to the order. | `customerinformation` (shipping fields) | `SUCCESS` | Persists order, sets success message |
| `editBilling()` | Applies changes from the billing form to the order. | `customerinformation` (billing fields) | `SUCCESS` | Persists order, sets success message |
| `getCustomerinformation()` / `setCustomerinformation()` | Getter/Setter for a temporary `Order` holding form data. | N/A | `Order` | None |
| `getCountryid()` / `setCountryid()` | Getter/Setter for the selected country ID. | N/A | `int` | None |
| `getOrder()` / `setOrder()` | Getter/Setter for the current order. | N/A | `Order` | None |

### Reusable / Utility
- `prepareOrderDetails()` is the only reusable piece of logic for loading and authorising an order.  
- The action relies heavily on `ServiceFactory.getService()` to obtain domain services; this indirection makes unit‑testing harder because the factories are static.

## 4. Dependencies  

| Library/Framework | Purpose | Note |
|-------------------|---------|------|
| **Struts 2** | MVC framework (Action, `Preparable`) | Core framework |
| **Apache Commons Lang** (`StringUtils`) | String helpers | Standard |
| **log4j** | Logging | Deprecated; consider SLF4J/Logback |
| **salesmanager.core** | Domain services, entities, util classes (`CountryUtil`, `CustomerUtil`, `LanguageUtil`) | Third‑party, internal |
| **salesmanager.central** | Base action classes (`BaseAction`, `CountrySelectBaseAction`) | Internal |
| **Servlet API** (via `HttpServletRequest`) | Session & request handling | Standard |

All dependencies are either standard Java/Servlet APIs or internal to the SalesManager platform. No external third‑party libraries beyond Commons and log4j.

## 5. Additional Notes  

### Strengths  
- **Clear separation**: Action delegates business logic to services.  
- **Reusable helpers**: `prepareOrderDetails()` centralises common authorization logic.  
- **Internationalisation support**: Uses language codes for country/zone lookups.  

### Issues & Suggestions  

| Issue | Why it matters | Suggested fix |
|-------|----------------|---------------|
| **Null checks** | `Context` or `order` may be null; NPE can propagate to the view. | Add guard clauses or use `Objects.requireNonNull`. |
| **Hard‑coded strings** (`ShippingConstants.MODULE_SHIPPING_RT_QUOTES_FREE`) | Makes future changes fragile. | Store as constants or config. |
| **Parsing user input without validation** (`Integer.parseInt(dstate)`) | If the value is not numeric, an exception is swallowed silently. | Validate with regex or try‑catch and return error. |
| **Unused `showaddressvalidation`** | Always false; likely a bug. | Set to `true` when address fields are present and valid. |
| **Mix of data models** | Using `Order` to hold temporary form data (`customerinformation`) conflates concerns. | Create dedicated DTOs (`CustomerEmailForm`, `ShippingForm`, `BillingForm`). |
| **Duplicate code** in `editShipping()` and `editBilling()` | Violates DRY; harder to maintain. | Extract common validation and persistence logic into a private helper. |
| **Logging** | `log.error(e)` prints only the exception object. | Use `log.error("message", e)` to include stack trace. |
| **ServiceFactory static usage** | Hard to mock for unit tests. | Inject services via dependency injection (e.g., Spring). |
| **log4j vs SLF4J** | log4j is legacy. | Replace with SLF4J + Logback or Log4j2. |
| **Thread‑safety of fields** | Struts 2 actions are per‑request, but if configured otherwise, shared fields could leak data. | Ensure no static state and use local variables when possible. |
| **Error handling consistency** | Some methods return `INPUT`, others `AUTHORIZATIONEXCEPTION`. | Define a unified error‑handling strategy (e.g., exception mappers). |
| **Hard‑coded `0` order ID check** | Using `0` to denote “no order” is unclear. | Use `null` or throw a custom `OrderNotFoundException`. |

### Future Enhancements  
1. **DTO separation** – Decouple form data from the persistence entity.  
2. **Unit tests** – Use mocks for services and `Context` to cover success and failure paths.  
3. **Internationalisation** – Extract all message keys and ensure that `OrderService` supports locale‑aware queries.  
4. **Validation framework** – Replace manual string checks with JSR‑303 bean validation.  
5. **Error page routing** – Centralise exception handling via Struts 2 `ExceptionMappings`.  

---

**Overall**, the action implements the required functionality in a straightforward manner, but it would benefit from modernizing its architecture, tightening validation, and improving testability.

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
package com.salesmanager.central.orders;

import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.CountrySelectBaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CustomerUtil;
import com.salesmanager.core.util.LanguageUtil;

public class EditCustomerAction extends CountrySelectBaseAction implements Preparable {

	private Logger log = Logger.getLogger(EditCustomerAction.class);

	private int countryid = -1;

	private Order customerinformation;

	private Order order = null;

	//private String deliveryState;

	protected void prepareOrderDetails() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		// Get the order
		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		Order o = oservice.getOrder(this.getOrder().getOrderId());
		


		// check if that entity realy belongs to merchantid
		if (o == null) {
			throw new AuthorizationException("Order is null for orderId "
					+ this.getOrder().getOrderId());
		}

		// Check if user is authorized (entity belongs to merchant)
		super.authorize(o);
		
		Country c = CountryUtil.getCountryByName(o.getDeliveryCountry(),
				LanguageUtil.getLanguageNumberCode(ctx.getLang()));
		
		if(c!=null) {
			super.prepareSelections(c.getCountryId());
			this.setCountryid(c.getCountryId());
		} else {
			super.prepareSelections();
		}
		
		this.setZoneText(o.getDeliveryState());

		this.setOrder(o);

	}

	public String editOrderEmailAddress() {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			Order o = this.getOrder();

			// check email address

			if (StringUtils.isBlank(o.getCustomerEmailAddress())) {
				super.addFieldError("order.customerEmailAddress",
						"messages.required.email");
				return INPUT;
			} else {
				if (!CustomerUtil.validateEmail(o.getCustomerEmailAddress())) {
					super.addFieldError("order.customerEmailAddress",
							"messages.invalid.email");
					return INPUT;
				}
			}

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			Order newOrder = oservice.getOrder(o.getOrderId());

			newOrder.setCustomerEmailAddress(o.getCustomerEmailAddress());

			oservice.saveOrUpdateOrder(newOrder);

			super.setSuccessMessage();
			return SUCCESS;

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

	}

	public String displayOrderEmailAddress() {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			Order o = this.getOrder();

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			Order completeOrder = oservice.getOrder(o.getOrderId());

			this.setOrder(completeOrder);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	/**
	 * User enters the edit shipping address page
	 * 
	 * @return
	 * @throws Exception
	 */
	public String viewShippingCustomer() throws Exception {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			prepareOrderDetails();

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			/**
			 * prepare address validation authorization only if it contains a
			 * shipping address
			 **/
			Order o = this.getOrder();

			Country c = null;

			ReferenceService ref = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);

			boolean showaddressvalidation = false;
			if (o.getDeliveryStreetAddress() != null
					&& !o.getDeliveryStreetAddress().trim().equals("")
					&& o.getDeliveryCity() != null
					&& !o.getDeliveryCity().trim().equals("")
					&& o.getDeliveryPostcode() != null
					&& !o.getDeliveryPostcode().trim().equals("")
					&& o.getDeliveryState() != null
					&& !o.getDeliveryState().trim().equals("")
					&& o.getDeliveryCountry() != null
					&& !o.getDeliveryCountry().trim().equals("")) {

				c = CountryUtil.getCountryByName(o.getDeliveryCountry(),
						LanguageUtil.getLanguageNumberCode(ctx.getLang()));
				
				

				//this.setCountryid(c.getCountryId());

				// get the zone

				Zone z = CountryUtil.getZoneCodeByName(o.getDeliveryState(),
						LanguageUtil.getLanguageNumberCode(ctx.getLang()));

				if (z != null) {
					//this.setDeliveryState(String.valueOf(z.getZoneId()));
					super.setZoneText(String.valueOf(z.getZoneId()));
				} else {
					super.setZoneText(o.getDeliveryState());
					//this.setDeliveryState(o.getDeliveryState());
				}

				super.getServletRequest().getSession().setAttribute("COUNTRY",
						c.getCountryId());

				// check shipping method configured

				String shippingmethod = o.getShippingModuleCode();

				// If we are dealing with free shipping...
				if (shippingmethod !=null && shippingmethod
						.equals(ShippingConstants.MODULE_SHIPPING_RT_QUOTES_FREE)) {
					// get the shipping method configured
					Integer merchantid = ctx.getMerchantid();
					ConfigurationRequest request = new ConfigurationRequest(
							merchantid, true, "SHP_");
					MerchantService service = new MerchantService();
					ConfigurationResponse returnvo = service
							.getConfiguration(request);
					shippingmethod = (String) returnvo
							.getConfiguration("shippingmethod");
				}

			}

			super.getServletRequest().setAttribute("showaddressvalidation",
					showaddressvalidation);

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;
	}

	/**
	 * User enters the edit billing address page
	 * 
	 * @return
	 * @throws Exception
	 */
	public String viewBillingCustomer() throws Exception {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			prepareOrderDetails();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;
	}

	public String editShipping() throws Exception {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			prepareOrderDetails();

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			Map zones = RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(ctx.getLang()));

			Order ord = this.getOrder();

			String dstate = customerinformation.getDeliveryState();
			try {
				int state = Integer.parseInt(dstate);
				Zone z = (Zone) zones.get(state);
				if (z != null) {
					customerinformation.setDeliveryState(z.getZoneName());
				}
			} catch (Exception ignore) {
				// TODO: handle exception
			}

			ord.setDeliveryName(customerinformation.getDeliveryName());
			ord.setDeliveryStreetAddress(customerinformation
					.getDeliveryStreetAddress());
			ord.setDeliveryCity(customerinformation.getDeliveryCity());
			ord.setDeliveryPostcode(customerinformation.getDeliveryPostcode());
			ord.setDeliveryState(customerinformation.getDeliveryState());

			// Change the country to text
			ReferenceService ref = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);

			// Country c =
			// ser.getCountryById(Integer.parseInt(customerinformation.getDeliveryCountry()));
			Map countries = RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(ctx.getLang()));

			int countryid = ctx.getCountryid();
			try {
				countryid = Integer.parseInt(customerinformation
						.getDeliveryCountry());
			} catch (Exception e) {
				// TODO: handle exception
			}

			Country c = (Country) countries.get(countryid);

			if (c != null) {
				ord.setDeliveryCountry(c.getCountryName());
			} else {
				ord.setDeliveryCountry("");
			}

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			oservice.saveOrUpdateOrder(ord);

			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;
	}

	public String editBilling() throws Exception {

		try {
			
			

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			prepareOrderDetails();

			Order ord = this.getOrder();

			ord.setBillingName(customerinformation.getBillingName());
			ord.setBillingStreetAddress(customerinformation
					.getBillingStreetAddress());
			ord.setBillingCity(customerinformation.getBillingCity());
			ord.setBillingPostcode(customerinformation.getBillingPostcode());
			ord.setBillingState(customerinformation.getBillingState());
			ord.setBillingCountry(customerinformation.getBillingCountry());

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			oservice.saveOrUpdateOrder(ord);

			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;
	}

	public Order getCustomerinformation() {
		return customerinformation;
	}

	public void setCustomerinformation(Order customerinformation) {
		this.customerinformation = customerinformation;
	}

	public int getCountryid() {
		return countryid;
	}

	public void setCountryid(int countryid) {
		this.countryid = countryid;
	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	//public String getDeliveryState() {
	//	return deliveryState;
	//}

	//public void setDeliveryState(String deliveryState) {
	//	this.deliveryState = deliveryState;
	//}

	public void prepare() throws Exception {
		super.setPageTitle("label.order.editcustomer.title");
		
	}

}



```
