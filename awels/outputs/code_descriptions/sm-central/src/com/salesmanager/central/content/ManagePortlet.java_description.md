# ManagePortlet.java

## Review

## 1. Summary
**Purpose** – The `ManagePortlet` class is a DWR‑enabled bean that exposes three operations for manipulating portal portlets on a merchant’s dashboard:

| Method | What it does |
|--------|--------------|
| `setVisible` | Toggle the visibility flag of an existing portlet. |
| `movePortlet` | Add, move, or delete a portlet in a page column and re‑order the surrounding portlets. |
| `configurePortlet` | Persist configuration values (fields) for a module‑specific portlet on a page. |

**Key components**

* **ReferenceService** – CRUD helper for `Portlet` and other reference tables.  
* **MerchantService** – Handles merchant‑specific configuration persistence.  
* **LabelUtil / LocaleUtil** – For localized messages.  
* **DWR WebContextFactory** – Provides access to the current `HttpServletRequest` (and therefore session).  

The class relies on a few business‑layer services rather than on any persistence framework directly, and uses plain POJOs (`Portlet`, `MerchantConfiguration`, `Field`, etc.) for data transfer. No explicit design pattern beyond DAO/service abstraction is evident, but the code shows a fairly standard “service‑layer” approach.

---

## 2. Detailed Description
### Execution Flow
1. **Context acquisition** – Every method obtains the current `HttpServletRequest` via `WebContextFactory.get()` and pulls the logged‑in merchant context (`Context ctx = ...`).  
2. **Locale/Label** – A `LabelUtil` instance is set up for i18n.  
3. **Business logic** – The core of each method interacts with the appropriate service (ReferenceService for portlets, MerchantService for configuration).  
4. **Error handling** – Broad `try / catch (Exception e)` blocks wrap all logic; errors are logged and a generic error message is returned.  

### Assumptions & Constraints
* The caller guarantees a non‑null `Context` in the session; if missing, the code creates a dummy `Portlet` with an error message.  
* `Portlet` IDs are assumed to be unique per merchant and are managed by the persistence layer.  
* Column identifiers are strings; `"deck"` is a special value meaning “remove from page and put back in the deck”.  
* Sorting of portlets relies on the `sortOrder` integer. The code manually re‑orders all affected portlets, which may be expensive on large columns.

### Architecture
* **Thin controller** – The class serves as a façade for the UI (via DWR).  
* **Service‑based persistence** – All database access is abstracted behind `ReferenceService` and `MerchantService`.  
* **No explicit transaction handling** – All changes are performed in a single `try` block; if an exception occurs, no rollback is explicitly requested. Presumably the underlying services handle transactions.

---

## 3. Functions/Methods
| Method | Inputs | Outputs | Side‑Effects | Notes |
|--------|--------|---------|--------------|-------|
| `public void setVisible(long portletId, boolean visible)` | `portletId` (PK), `visible` flag | void | Persists the new visibility state of the portlet if it belongs to the current merchant. | Does **not** return any indication of success/failure. |
| `public Portlet movePortlet(Portlet portlet)` | A `Portlet` object containing at least `portletId`, `columnId`, `sortOrder`, `page`, etc. | Returns the updated `Portlet` (or a new instance with an error message). | *Add* (new ID = 0), *move*, or *delete* (if column=`deck`). Reorders surrounding portlets. | Handles three scenarios: delete, move, create. Uses manual list manipulation for re‑ordering. |
| `public DisplayMessage configurePortlet(String module, String page, Field[] fields)` | `module`, `page`, array of `Field` | `DisplayMessage` with success or error text | Persists the module configuration in `MerchantConfiguration`. | Parses and rebuilds configuration string; overwrites existing values for the given key. |

### Reusable/Utility Methods
* **LabelUtil / LocaleUtil** – Not part of this class but heavily used for message resolution.  
* **ConfigurationFieldUtil** – For parsing/building configuration strings (not defined here).  
* **StringUtils** – From Apache Commons for string checks.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Java EE | Standard. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Commons Lang. |
| `org.apache.log4j.Logger` | Third‑party | Log4j. |
| `uk.ltd.getahead.dwr.WebContextFactory` | Third‑party | DWR framework. |
| `com.salesmanager.*` | Third‑party (internal) | Business services, entities, utilities. |
| `java.util.*` | Standard | Collections, Locale. |

*All external services are resolved via a `ServiceFactory` singleton, so the code has a tight coupling to the SalesManager service layer.*

---

## 5. Additional Notes
### Strengths
* **Clear separation** – Business logic is delegated to services; the bean only orchestrates calls.  
* **Localized messaging** – Error/success strings come from `LabelUtil`.  
* **Error handling** – Exceptions are logged; a user‑friendly message is returned.

### Weaknesses & Edge Cases
1. **No transactional boundary** – If `saveOrUpdateAllPortlets` fails after a partial reorder, data may be left in an inconsistent state.  
2. **Inefficient re‑ordering** – Every move/create/delete operation loads the entire column into memory, manipulates it, and writes it back. For columns with many portlets, this can be costly.  
3. **Missing validation** – The method trusts the caller for values such as `sortOrder`, `columnId`, etc. Out‑of‑range or invalid values could produce unexpected ordering.  
4. **Thread‑safety** – The bean is effectively stateless, but the services it calls may maintain internal state. No synchronization is visible.  
5. **Error messaging** – In `setVisible`, failures silently result in no feedback to the caller.  
6. **Use of raw types** – Collections (`List portletList = new ArrayList();`) and `Map fieldValues = new HashMap();` lack generics, leading to unchecked warnings.  
7. **Hard‑coded `"deck"` constant** – If the value changes in the UI layer, the code will break.  
8. **Deprecated `StringUtils.isBlank`** – In newer commons‑lang3 it’s still valid but consider upgrading for better readability.

### Suggested Enhancements
| Area | Recommendation |
|------|----------------|
| **Transaction** | Wrap each operation in a service‑layer transaction (e.g., `@Transactional`). |
| **Ordering logic** | Replace manual re‑ordering with a single SQL `UPDATE` using `CASE` statements or a dedicated ordering service. |
| **Validation** | Add pre‑condition checks (e.g., `sortOrder >= 0`, valid column ID) and return informative error messages. |
| **Generics** | Replace raw collections with generic types (`List<Portlet>`, `Map<String, List<Field>>`). |
| **Exception handling** | Use custom exception types; return a structured error object instead of just a message. |
| **Unit tests** | Write unit tests for each scenario (create, move, delete) and edge cases (invalid IDs, out‑of‑range sort order). |
| **Code cleanup** | Remove commented out code (`movePortleet`) and unused imports. |
| **Logging** | Log at debug level the incoming parameters for troubleshooting; consider masking sensitive data. |

### Potential Future Extensions
* **Bulk reorder API** – Accept a list of portlet IDs with target positions.  
* **Undo/redo** – Keep a history of portlet arrangements per page.  
* **Role‑based access** – Validate that the user has permission to modify a given portlet.  
* **WebSocket notifications** – Push real‑time updates to all sessions viewing the page.  

Overall, the class accomplishes its core responsibilities but would benefit from tighter validation, transactional safety, and more modern Java practices (generics, streams, proper exception handling).

## Code Critique



## Code Preview

```java
package com.salesmanager.central.content;


import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;


import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.Portlet;
import com.salesmanager.core.entity.system.DisplayMessage;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.ConfigurationFieldUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;

/*
 * DWR Bean for portlet management
 */
public class ManagePortlet {
	
	private final static String COLUMN_DECK = "deck";
	private Logger log = Logger.getLogger(ManagePortlet.class);
	
	public void setVisible(long portletId, boolean visible) {
		
		
		try {
			
			ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
			Portlet p = rservice.getPortlet(portletId);
			
			HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();
			Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);
			
			if(p!=null && p.getMerchantId()==ctx.getMerchantid()) {
				p.setVisible(visible);
				rservice.saveOrUpdatePortlet(p);
			}
			
		} catch (Exception e) {
			log.error(e);
		}
		
		
	}
	
	/**
	 * 
	 * @param pageId - Portal page id
	 * @param portleetId - from Portleet table, can be 0 if it was not assigned
	 * @param title - module or content title
	 * @param labelId - DynamicLabel id
	 * @param columnId - id where the column is drawn
	 * @param type - portlet type (content (1) or module (2)
	 * @param order - order of the portleet in the portlet area
	 * @return
	 */
	//public Portleet movePortleet(long pageId, long portleetId, String title, long labelId, String columnId, int type, int order) {
	public Portlet movePortlet(Portlet portlet) {
		
		
		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();
		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);
		
		Locale locale = LocaleUtil.getLocale(req);
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(locale);
		
		if(ctx==null) {
			Portlet p = new Portlet();
			p.setMessage(label.getText("error.sessionexpired"));
			return p;
		}
		
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		
		
		
		
		Portlet p = null;
		
		try {
			
			//log message error when no columnid
		
			if(!StringUtils.isBlank(portlet.getColumnId())) {
				
				p = rservice.getPortlet(portlet.getPortletId());
				
								
				
				//was in the portal area, returns to deck, so there is a portlet id
				if(portlet.getColumnId().equals(COLUMN_DECK))  {
					//remove from Portlet
					
					if(p==null) {
						p = new Portlet();
						//p.setMessage(label.getText("integration.messages.error.generic"));
						return p;
					}
					
					//check if it belongs to merchant
					if(p.getMerchantId()!=ctx.getMerchantid()) {
						log.warn("Poortlet poortletId [" + portlet.getPortletId() + "] does not belong to merchant id [" + ctx.getMerchantid() + "]");
						return p;
					}
					

					rservice.deletePortlet(p);
					p.setPortletId(0);//re-initialize portlet id
					
					//reorder order portlets in the same column
					List portletList = new ArrayList();
					Collection coll = rservice.getPortlets(portlet.getPage(), p.getColumnId(), ctx.getMerchantid());
					int count = 0;
					for (Object o: coll) {
						Portlet portletOrder = (Portlet)o;
						portletOrder.setSortOrder(count);
						portletList.add(portletOrder);

						count++;
					}
					
					rservice.saveOrUpdateAllPortlets(portletList);
					
					return p;
				
				} else {
					
					List portletList = new ArrayList();
					
					if(portlet.getPortletId()>0) { //moving portlet
						
						if(p==null) {
							log.error("Cannot remove this poortlet poortletId [" + portlet.getPortletId() + "] title [" + portlet.getTitle() + "]");
							p = new Portlet();
							p.setMessage(label.getText("integration.messages.error.generic"));
							return p;
						}
						
						if(p.getMerchantId()!=ctx.getMerchantid()) {
							log.warn("Poortlet poortletId [" + portlet.getPortletId() + "] does not belong to merchant id [" + ctx.getMerchantid() + "]");
							return p;
						}
						
						String originalColumn = p.getColumnId();
						
						
						//change column
						p.setColumnId(portlet.getColumnId());
						p.setSortOrder(portlet.getSortOrder());
						
						//re-arrange order of current column
						Collection coll = rservice.getPortlets(portlet.getPage(), portlet.getColumnId(), ctx.getMerchantid());
						int count = 0;
						//int newCount = 0;
						for (Object o: coll) {
							
							
							Portlet portletOrder = (Portlet)o;
							if(portletOrder.getPortletId()!=p.getPortletId()) {
								if(portletOrder.getSortOrder().intValue()==portlet.getSortOrder().intValue()) {
									
									int newCount = portlet.getSortOrder().intValue()+1;
									if(count<portlet.getSortOrder()) {
										newCount = count;
									} 
									portletOrder.setSortOrder(newCount);
									count ++;
								} else {
									portletOrder.setSortOrder(count);
								}
								portletList.add(portletOrder);
								count++;
							}
						}
						portletList.add(p);
						
						//rearange order of original column
						coll = rservice.getPortlets(portlet.getPage(), originalColumn, ctx.getMerchantid());
						count = 0;
						//int newCount = 0;
						for (Object o: coll) {
							Portlet portletOrder = (Portlet)o;
							if(portletOrder.getPortletId()!=p.getPortletId()) {
								portletOrder.setSortOrder(count);
								portletList.add(portletOrder);
								count++;
							}

						}
						
					} else {//create a portlet
						
						p = new Portlet();
						p.setColumnId(portlet.getColumnId());
						p.setLabelId(portlet.getLabelId());
						p.setPage(portlet.getPage());
						p.setMerchantId(ctx.getMerchantid());
						p.setPortletType(portlet.getPortletType());
						p.setTitle(portlet.getTitle());
						p.setSortOrder(portlet.getSortOrder());
						
						//get portlet according to type
						if(portlet.getPortletType()==1) {
							CoreModuleService cms = rservice.getCoreModuleService("XX", portlet.getTitle());
							p.setName(cms.getCoreModuleServiceDescription());
						} else {
							p.setName(portlet.getTitle());
						}
						
						p.setPortletType(portlet.getPortletType());
						
						//re-arrange order
						Collection coll = rservice.getPortlets(portlet.getPage(), portlet.getColumnId(), ctx.getMerchantid());
						int count = 0;
						//int newCount = 0;
						for (Object o: coll) {
							
							
							Portlet portletOrder = (Portlet)o;
							if(portletOrder.getPortletId()!=p.getPortletId()) {
								if(portletOrder.getSortOrder().intValue()==portlet.getSortOrder().intValue()) {
									int newCount = portlet.getSortOrder().intValue()+1;
									if(count<portlet.getSortOrder()) {
										newCount = count;
									} 
									portletOrder.setSortOrder(newCount);
									count ++;
								} else {
									portletOrder.setSortOrder(count);
								}
								portletList.add(portletOrder);
								count++;
							}
						}
					}
					rservice.saveOrUpdateAllPortlets(portletList);
					rservice.saveOrUpdatePortlet(p);
				}
			}
			
			
		
		} catch (Exception e) {
			log.error(e);
			if(p==null) {
				p = new Portlet();
				p.setMessage(label.getText("integration.messages.error.generic"));
			}
		}
		
		return p;
		

	}
	
	
	/**
	 * Configure portlet with submited fields values
	 * @param fields
	 * @return
	 */
	public DisplayMessage configurePortlet(String module, String page, Field[] fields) {
		
		
		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();
		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);
		
		Locale locale = LocaleUtil.getLocale(req);
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(locale);
		
		DisplayMessage message = new DisplayMessage();
		
		if(StringUtils.isBlank(module) || StringUtils.isBlank(page)) {
			message.setErrorMessage(label.getText("messages.error.integration.invalidparameter"));
			return message;
		}
		
		try {
			
		
			//get configured portlets for this page
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			ConfigurationRequest configRequest = new ConfigurationRequest(ctx.getMerchantid(),ConfigurationFieldUtil.getMerchantConfigurationKey(page, module));
			ConfigurationResponse configResponse = mservice.getConfiguration(configRequest);
			
			Map fieldValues = new HashMap();
			MerchantConfiguration conf = configResponse.getMerchantConfiguration(ConfigurationFieldUtil.getMerchantConfigurationKey(page, module));
			if(conf!=null) {
				String f = conf.getConfigurationValue();
				if(!StringUtils.isBlank(f)) {
					fieldValues = ConfigurationFieldUtil.parseFieldsValues(f);
				}
			}
			
			List newFieldList = new ArrayList();
			for(int i=0;i<fields.length;i++) {
				newFieldList.add(fields[i]);
			}
			
			fieldValues.put(module, newFieldList);
			
			
			String fieldString = ConfigurationFieldUtil.buildFieldValuesString(fieldValues);
			
			if(conf==null) {
				conf = new MerchantConfiguration();
				conf.setConfigurationKey(ConfigurationFieldUtil.getMerchantConfigurationKey(page, module));
				conf.setMerchantId(ctx.getMerchantid());
			}
			
			conf.setConfigurationValue(fieldString);
			mservice.saveOrUpdateMerchantConfiguration(conf);
			
			message.setSuccessMessage(label.getText("message.confirmation.success"));
			
			return message;
		
		} catch (Exception e) {
			log.error(e);
			message.setErrorMessage(label.getText("errors.technical") + " " + e.getMessage());
			return message;
		}
		
		
	}

}



```
