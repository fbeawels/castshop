# ProfileAction.java

## Review

**1.  Summary**

`ProfileAction` is a Struts 2 action that manages merchant user profiles.  
It covers:

* Viewing / editing the current user’s profile (name, e‑mail, address, language, country, credit‑card data, security questions).  
* Password reset – generating an e‑mail link, validating links, answering security questions, generating a unique temporary password.  
* Security‑question CRUD (create, edit, delete) for merchant users.  
* Role assignment for admin users, distinguishing master roles (“admin”, “superuser”, “user”) from other role definitions.  

The class pulls business logic from a `MerchantService` (obtained via a custom `ServiceFactory`) and Spring beans (`SecurityQuestionsModule`, `LabelUtil`).  JSON annotations are used for asynchronous calls.  Logging is performed with Log4j.

---

**2.  Detailed description**

| Method | Responsibility | Comments |
|--------|----------------|----------|
| `resetPassword()` | Handles an inbound request to initiate a password reset. Generates an e‑mail containing a reset link. | Uses `MerchantService` to locate the user by e‑mail. No verification that the supplied e‑mail actually belongs to the current merchant. Logging occurs before sending the e‑mail. |
| `displayResetPassword()` | Renders the “reset‑password” page based on a token in the URL. | Relies on `FileUtil.getUrlTokens()` to parse the token; minimal validation of the returned ID & DATE. Stores `resetPasswordResponse` flag for the view. |
| `saveSecurityQuestions()` | Persists the selected security question codes & answers. | Calls `mservice.deleteUserRoles()` before inserting new roles (unrelated to questions – likely a copy‑paste bug). |
| `answerQuestions()` | Processes the security‑question answer form. Validates answers, generates a new temporary password, and marks the user as having successfully reset the password. | Uses a loop to keep generating encrypted passwords until one does not exist. This is potentially expensive for a large user base. |
| `isPasswordResetSuccess()` (JSON accessor) | Exposes `passwordResetSuccess` to the front‑end. | Annotation correctly serialises the flag. |
| `display()` | Pre‑populates the profile edit page with the current user data. | Automatically updates the country in session if missing, and prepares province selections. Handles pre‑filled security questions if they exist. |
| `saveProfile()` | Persists the changes made to the current user’s profile. | Handles `ConstraintViolationException` to detect duplicate e‑mails. No validation of e‑mail format or required fields. |
| `viewUser()` | Prepares the “manage user” screen: loads master and custom roles, loads the user’s roles and checks authorisation. | Uses raw collections and unchecked casts.  The logic for default role assignment is unclear and may incorrectly default to “admin”. |
| `saveUser()` | Persists a new or updated user, along with their role assignments. | Deleting all roles and re‑adding them each time is simple but not transactional.  It does not guard against unknown role codes. |
| `editUserList()` | Retrieves the list of all merchant users for display. | No pagination – may blow memory if the list is large. |
| `deleteUser()` | Deletes a user by ID. | No confirmation step, no transaction rollback on failure. |
| `answerQuestions()` | Generates a temporary password, ensuring uniqueness. | Uses a simple loop; may hang if many users already use the same random string. |
| `displayPassword()` | Shows the password change page. | Does not set `securityQuestion1/2/3` when they are missing, leaving the form blank. |
| `savePassword()` | (Missing in the excerpt) – no method present; the password change is performed in `saveProfile()` via the `newPassword` fields. |
| `displayResetPassword()` & `resetPassword()` | The two‑step flow for external password reset. | The link generation uses `centralService.getMerchantUserInformation` to get the user; the reset link itself is built in `FileUtil.getUrlTokens()` – details hidden. |

---

**3.  Functions & Responsibilities**

| Function | Key Actions | Observations |
|----------|-------------|--------------|
| `resetPassword()` | Validates merchant ID & admin name, builds reset e‑mail, sends it. | Uses `MerchantService` twice; could be cached. Does not check whether the admin name actually belongs to the current merchant. |
| `displayResetPassword()` | Parses token from URL, sets up form fields. | Very little validation of the token. |
| `saveSecurityQuestions()` | Stores selected question codes & answers. | Does not actually persist the answers – appears to be a stub. |
| `answerQuestions()` | Validates user answers, generates a new temporary password, marks success flag. | Potential race‑condition if multiple password resets happen concurrently. |
| `isPasswordResetSuccess()` (JSON) | Getter for `passwordResetSuccess`. | Correct use of JSON annotation. |
| `display()` | Loads current user profile, sets up country/province selections. | Repeats logic seen in `saveProfile()`. |
| `saveProfile()` | Persists the profile fields, updates locale. | Handles `ConstraintViolationException` specially; other errors are generic. |
| `viewUser()` | Loads master and custom roles, checks user existence, pre‑populates role list. | Uses raw collections; no pagination. |
| `saveUser()` | Adds/updates roles, creates user if new. | Deletes all roles before re‑adding – may temporarily leave the user without any role if an error occurs. |
| `editUserList()` | Fetches all merchant users. | No pagination or filtering. |
| `deleteUser()` | Deletes a user by ID. | No confirmation prompt or soft‑delete. |

---

**4.  Dependencies & External Services**

| External | Purpose | Notes |
|----------|---------|-------|
| `com.salesmanager.core.business.common.constants.*` | Constants like default country ID, error messages | Hard‑coded defaults. |
| `com.salesmanager.core.business.customer.model.CreditCardUtil` | (Commented out) | Not used. |
| `com.salesmanager.core.business.customer.model.Customer` | (Commented out) | Not used. |
| `com.salesmanager.core.business.merchant.model.*` | Domain objects | `MerchantUserInformation`, `MerchantUserRole`, `MerchantUserRoleDef`. |
| `com.salesmanager.core.business.merchant.service.*` | `MerchantService` operations | Fetched via `ServiceFactory`. |
| `com.salesmanager.core.business.merchant.util.*` | `LabelUtil`, `MessageUtil`, `PropertiesUtil` | For i18n and messaging. |
| `com.salesmanager.core.business.security.model.*` | Role/role definition objects | `MerchantUserRole`, `MerchantUserRoleDef`. |
| `com.salesmanager.core.business.service.ServiceFactory` | Singleton factory for services | Custom service factory – could be replaced by Spring injection. |
| `com.salesmanager.core.business.util.*` | `LabelUtil`, `MessageUtil`, `PropertiesUtil` | Utilities. |
| `com.salesmanager.core.constants.*` | `ProfileConstants`, `MasterRoles`, etc. | Holds context key. |
| `org.apache.logging.log4j.*` | Logging | Log4j used. |
| `org.apache.struts2.json.annotations.JSON` | Expose fields for AJAX | Only used on `isPasswordResetSuccess()`. |
| `org.apache.struts2.util.ServletContextUtils` | (Unused) | Possibly leftover. |
| `com.opensymphony.xwork2.Action` | Struts2 action contract | Base class. |

---

**5.  Issues & Recommendations**

| Area | Issue | Recommendation |
|------|-------|----------------|
| **Generics / type safety** | Raw `Map` and `Collection` types in many places. | Use generic types (`Map<String, String>`, `Collection<MerchantUserRole>`) to catch compile‑time errors. |
| **Code duplication** | Repeated `MerchantService` retrieval and identical try‑catches. | Create a private helper `getMerchantService()` or inject the service via Spring. |
| **Validation** | Minimal input validation (e.g., e‑mail format, numeric ID). | Add Struts2 validation or custom validators. |
| **Security** | Encryption key derived from a constant. | Review the encryption scheme; consider a per‑user salt and a secure hash (e.g., PBKDF2, bcrypt). |
| **Concurrency / state** | Instance fields (`answers`, `answersText`, `securityQuestions`) persist across requests. | Clear or initialise these in each action method; avoid shared mutable state. |
| **Error handling** | Broad catch blocks; swallow `Exception` in many places. | Prefer specific exceptions; propagate or wrap in custom action‑level exceptions. |
| **Null checks** | Some methods (e.g., `displayPassword()`) assume non‑null fields. | Guard against `null` or use optional chaining. |
| **Performance** | Password uniqueness check loops over entire user base. | Store a hash set of existing passwords or let DB enforce uniqueness. |
| **Readability** | Long methods with inline comments and commented‑out code blocks. | Break out logic into smaller private helper methods; remove dead code. |
| **Internationalisation** | Calls to `LabelUtil` inside loops; may cause performance hits. | Cache the `LabelUtil` instance or use a dedicated i18n service. |
| **Authorization** | `super.authorize((IMerchant)merchantProfile);` called in `viewUser()` but never checked for success. | Ensure proper error handling if user lacks permissions. |
| **Resource leaks** | Not applicable – no streams or DB connections are explicitly opened. |
| **Testing** | No unit tests shown. | Add tests for password reset, security questions, role assignment. |
| **Framework usage** | Struts2 actions are not thread‑safe; current design relies on request‑scoped instance. | No issue, but avoid static mutable state. |

---

**6.  Suggested Refactorings**

1. **Inject Services**  
   Replace `ServiceFactory.getService()` and `SpringUtil.getBean()` with proper Spring dependency injection. E.g.,  
   ```java
   @Autowired private MerchantService merchantService;
   @Autowired private SecurityQuestionsModule securityQuestionsModule;
   ```

2. **Use Generics Everywhere**  
   Replace raw types (`Map`, `Collection`) with parameterised types.  
   ```java
   private Map<String, String> securityQuestions = new HashMap<>();
   ```

3. **Separate Concerns**  
   Move business logic (e.g., password generation, encryption) to dedicated service classes.  
   Keep the action thin: only bind request parameters, invoke services, and populate the model.

4. **Centralise Error Handling**  
   Create an `ActionException` hierarchy (e.g., `EmailAlreadyExistsException`) and handle it once in a global exception interceptor.

5. **Clear Request‑Specific State**  
   Ensure that lists (`answers`, `answersText`, `submitRoles`, etc.) are re‑initialised or cleared in each request or in `reset()`.

6. **Improve Token Validation**  
   In `displayResetPassword()` validate the token format, check expiration, and use a secure random ID rather than plain numeric.

7. **Remove Dead Code**  
   Delete commented blocks and unused imports to improve readability.

8. **Unit Tests**  
   Write tests for each public method, mocking `MerchantService` and ensuring role logic works as expected.

9. **Logging**  
   Use parameterised logging to avoid string concatenation, e.g.,  
   ```java
   log.info("Reset password for merchant {}", merchantProfile.getAdminEmail());
   ```

10. **Security Audits**  
    Review the encryption implementation; ensure salts, key management, and hash functions meet current standards.

---

**7.  Conclusion**

`ProfileAction` demonstrates a classic, service‑oriented Struts 2 action that ties together business services, i18n, and user session handling.  While it fulfills its functional requirements, the code would benefit from modernisation:

* Adopt Spring dependency injection fully (remove the custom `ServiceFactory`).  
* Replace raw collections with generics.  
* Consolidate repeated logic, improve validation, and remove dead code.  
* Strengthen error handling and logging.  
* Re‑evaluate the password encryption strategy for compliance with current security best‑practices.

With these changes the action would become more maintainable, easier to test, and safer to run in a production environment.

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
package com.salesmanager.central.profile;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.TreeMap;

import javax.servlet.http.HttpSession;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.json.annotations.JSON;
import org.hibernate.exception.ConstraintViolationException;

import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.CountrySelectBaseAction;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.IMerchant;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.merchant.MerchantUserRole;
import com.salesmanager.core.entity.merchant.MerchantUserRoleDef;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.module.model.application.PasswordGeneratorModule;
import com.salesmanager.core.module.model.application.SecurityQuestionsModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.LogMerchantUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.SpringUtil;

/**
 * Manage User Profile
 * 
 * @author Carl Samson
 * 
 */
public class ProfileAction extends CountrySelectBaseAction {

	private Logger log = Logger.getLogger(ProfileAction.class);

	private MerchantUserInformation merchantProfile;
	private Integer countryCode;
	private String ccMonth;
	private String ccYear;
	private String securityCode;

	private String newPassword;
	private String repeatNewPassword;
	
	//when submiting reset password
	private String merchantId;
	private String adminName;
	
	private boolean resetPasswordResponse;
	
	private Map securityQuestions = new HashMap();
	private List<Integer> answers = new ArrayList<Integer>();
	private List<String> answersText = new ArrayList<String>();
	private List<String> questionsText = new ArrayList<String>();
	private boolean passwordResetSuccess = false;
	private String merchantPassword = "";
	
	Collection<MerchantUserInformation> merchantUserInformations;
	
	//security
	private Map<String,String> roles = new TreeMap();
	private Map<String,String> masterRoles = new TreeMap();
	private Collection<String> userRoles = new ArrayList();
	
	private String submitMasterRole = null;
	private List<String> submitRoles = new ArrayList();
	
	
	
	public String resetPassword() {
		
		
		try {
			
			int merchantId = Integer.parseInt(this.getMerchantId());
			

			//log attempt
			LogMerchantUtil.log(merchantId, "Attempt to change password for user name " + this.getAdminName());
			
			//validate merchant id admin
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			MerchantUserInformation merchantUserInformation = mservice.getMerchantUserInformation(this.getAdminName());
			MerchantStore store = mservice.getMerchantStore(merchantId);
			
			if(merchantUserInformation!=null) {
				if(merchantUserInformation.getMerchantId()!=merchantId) {
					LogMerchantUtil.log(merchantId, "Attempt to change password for user name " + this.getAdminName() + " and inputed wrong merchantId " + merchantId);
				}
			}
			
			//create email link
			Configuration config = PropertiesUtil.getConfiguration();
			String l = config.getString("core.system.defaultlanguage", "en");
			if (!StringUtils.isBlank(merchantUserInformation.getUserlang())) {
				l = merchantUserInformation.getUserlang();
			}

			LabelUtil lhelper = LabelUtil.getInstance();
			lhelper.setLocale(super.getLocale());
			String subject = lhelper.getText(l, "label.admin.resetpassword.subject");
			String message = lhelper.getText(l, "label.admin.resetpassword");

			String centralUri = ReferenceUtil.buildCentralUri(store);

			String link = FileUtil.getAdminPasswordResetUrl(merchantUserInformation,store);
			
			String url = "<a href=\""
					+ link
					+ "\">"
					+ link
					+ "</a>";

			Map emailctx = new HashMap();
			emailctx.put("EMAIL_PASSWORD_TEXT", message);
			emailctx.put("EMAIL_PASSWORD_LINK", url);

			CommonService cservice = new CommonService();
			cservice.sendHtmlEmail(merchantUserInformation.getAdminEmail(), subject,
					store, emailctx,
					"email_template_user_password_link.ftl",merchantUserInformation.getAdminEmail());
			
			//send email
			
		} catch (Exception e) {
			log.error(e);
			resetPasswordResponse = false;
			return SUCCESS;
		}

		resetPasswordResponse = true;
		return SUCCESS;

	}
	
	public String displayResetPassword() {
		

		try {
			
			super.setPageTitle("security.question.title");
			
			//parse tokens
			String fileId = getServletRequest().getParameter("urlId");
			String lang = getServletRequest().getParameter("lang");
			
			Locale l = LocaleUtil.getDefaultLocale();
			try {
				l = new Locale(Constants.CA_ISOCODE,lang);
			} catch (Exception e) {
				log.warn("Wrong locale language " + lang);
			}
			
			
			super.setLocale(l);

			if (StringUtils.isBlank(fileId)) {
				List msg = new ArrayList();
				msg.add(getText("error.invalid.url"));
				super.setActionErrors(msg);
				return "ANONYMOUSGENERICERROR";
			}
			
			Map fileInfo = FileUtil.getUrlTokens(fileId);

			String id = (String) fileInfo.get("ID");
			String date = (String) fileInfo.get("DATE");
			
			if(StringUtils.isBlank(id) || StringUtils.isBlank(date)) {
				List msg = new ArrayList();
				msg.add(getText("error.invalid.url"));
				super.setActionErrors(msg);
				return "ANONYMOUSGENERICERROR";
			}
			
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			MerchantUserInformation merchantUserInformation = mservice.getMerchantUserInformation(new Long(id));
			this.setMerchantProfile(merchantUserInformation);
			
			if(merchantUserInformation==null) {
				log.error("merchantUserInformation " + id + " is null");
				List msg = new ArrayList();
				msg.add(getText("error.invalid.url"));
				super.setActionErrors(msg);
				return "ANONYMOUSGENERICERROR";
			}
			
			//display security questions
			SecurityQuestionsModule module = (SecurityQuestionsModule)SpringUtil.getBean("securityQuestions");
			Map questions = module.getSecurityQuestions(super.getLocale());
			this.setSecurityQuestions(questions);
			
			questionsText.add(0, module.getQuestionText(1,super.getLocale()));
			questionsText.add(1, module.getQuestionText(2,super.getLocale()));
			questionsText.add(2, module.getQuestionText(3,super.getLocale()));
			

			//set merchantUserId in http session
			HttpSession session = super.getServletRequest().getSession(true);
			session.setAttribute("merchantUserId", merchantUserInformation.getMerchantUserId());

			
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "ANONYMOUSGENERICERROR";
		}
		
		return SUCCESS;
		
	}
	
	public String saveSecurityQuestions() {
		
		if(this.getMerchantProfile()==null) {
			log.error("MerchantUserId not submited");
			super.setTechnicalMessage();
			return INPUT;
		}
		
		try {
			//this will submit the questions and the answers
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			MerchantUserInformation merchantUserInformation = mservice.getMerchantUserInformation(this.getMerchantProfile().getMerchantUserId());
			
			if(merchantUserInformation==null) {
				log.error("MerchantUserInformation is null");
				super.setTechnicalMessage();
				return INPUT;
			}
			
			int i = 0;
			for(int j = 0; j < answers.size(); j++) {
				
				Integer questionId = answers.get(j);
				String answer = answersText.get(j);
				
				//only 3 are stored in merchant user
				if(i==0) {
					merchantUserInformation.setSecurityQuestion1(String.valueOf(questionId));
					merchantUserInformation.setSecurityAnswer1(answer.trim().toLowerCase());
				} else if(i==1) {
					merchantUserInformation.setSecurityQuestion2(String.valueOf(questionId));
					merchantUserInformation.setSecurityAnswer2(answer.trim().toLowerCase());
				} else if(i==2) {
					merchantUserInformation.setSecurityQuestion3(String.valueOf(questionId));
					merchantUserInformation.setSecurityAnswer3(answer.trim().toLowerCase());
				} else {
					break;
				}
				
				i++;
			}
			
			if(
			
				!StringUtils.isBlank(merchantUserInformation.getSecurityQuestion1())
					
				&& !StringUtils.isBlank(merchantUserInformation.getSecurityQuestion2())
				
						&& !StringUtils.isBlank(merchantUserInformation.getSecurityQuestion3())
								
							&&	(merchantUserInformation.getSecurityQuestion1().equals(merchantUserInformation.getSecurityQuestion2()) || merchantUserInformation.getSecurityQuestion2().equals(merchantUserInformation.getSecurityQuestion3()) || merchantUserInformation.getSecurityQuestion1().equals(merchantUserInformation.getSecurityQuestion3())))

			{
				
				super.setErrorMessage("security.questions.differentmessages");
				return INPUT;
			}


			mservice.saveOrUpdateMerchantUserInformation(merchantUserInformation);
			this.setMerchantProfile(merchantUserInformation);
			
			super.setSuccessMessage();
			
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}
		
		return SUCCESS;
	}
	
	public String answerQuestions() {
		
		super.setPageTitle("security.question.title");
		
		SecurityQuestionsModule module = (SecurityQuestionsModule)SpringUtil.getBean("securityQuestions");
		Map questions = module.getSecurityQuestions(super.getLocale());
		this.setSecurityQuestions(questions);
		
		questionsText.add(0, module.getQuestionText(1,super.getLocale()));
		questionsText.add(1, module.getQuestionText(2,super.getLocale()));
		questionsText.add(2, module.getQuestionText(3,super.getLocale()));
		
		

		
		try {
			
			HttpSession session = super.getServletRequest().getSession(true);
			Long merchantUserId = (Long)session.getAttribute("merchantUserId");

			
			if(merchantUserId==null) {
				log.error("MerchantUserInformationId is null");
				super.setTechnicalMessage();
				return INPUT;
			}
			
			//this will submit the questions and the answers
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			MerchantUserInformation merchantUserInformation = mservice.getMerchantUserInformation(merchantUserId);
			
			if(merchantUserInformation==null) {
				log.error("MerchantUserInformation is null");
				super.setTechnicalMessage();
				return INPUT;
			}
			
			int i = 0;
			for(int j = 0; j < answersText.size(); j++) {
				

				String answer = answersText.get(j);
				
				//only 3 are stored in merchant user
				if(i==0) {
					if(!answer.trim().toLowerCase().equals(merchantUserInformation.getSecurityAnswer1())) {
						super.setErrorMessage("message.invalid.security.answers");
						return INPUT;
					}
				} else if(i==1) {
					if(!answer.trim().toLowerCase().equals(merchantUserInformation.getSecurityAnswer2())) {
						super.setErrorMessage("message.invalid.security.answers");
						return INPUT;
					}
				} else if(i==2) {
					if(!answer.trim().toLowerCase().equals(merchantUserInformation.getSecurityAnswer3())) {
						super.setErrorMessage("message.invalid.security.answers");
						return INPUT;
					}
				} else {
					break;
				}
				
				i++;
			}
			

			
			//generate new password
			
			PasswordGeneratorModule passwordGenerator = (PasswordGeneratorModule) SpringUtil
			.getBean("passwordgenerator");

			// encrypt
			String key = EncryptionUtil.generatekey(String
					.valueOf(SecurityConstants.idConstant));
			boolean found = true;

			String password = null;
			String encrypted = null;
			// validate if already exist
			while (found) {

				password = passwordGenerator.generatePassword();
				encrypted = EncryptionUtil.encrypt(key, password);
				MerchantUserInformation info =mservice.getMerchantInformationByUserNameAndPassword(
						merchantUserInformation.getAdminName(), encrypted);
				if (info == null) {
					found = false;
				}
			}
			
			merchantPassword = password;
			
			
			merchantUserInformation.setAdminPass(encrypted);
			mservice.saveOrUpdateMerchantUserInformation(merchantUserInformation);
			this.setMerchantProfile(merchantUserInformation);
			
			passwordResetSuccess = true;
			
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}
		
		return SUCCESS;
		
	}
	

	public String changeLanguage() {

		Context context = super.getContext();

		String language = super.getServletRequest().getParameter("lang");
		if (!StringUtils.isBlank(language)) {

			RefCache cache = RefCache.getInstance();

			Map countries = cache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(context.getLang()));
			Country c = (Country) countries.get(context.getCountryid());

			Locale locale = new Locale(language, c.getCountryIsoCode2());
			super.setLocale(locale);
			
			Context ctx = super.getContext();
			ctx.setLang(locale.getLanguage());
			
		}

		return SUCCESS;

	}

	public String displayPassword() {

		try {
			
			super.setPageTitle("label.changepassword");

			int merchantId = super.getContext().getMerchantid();
			String user = super.getPrincipal().getRemoteUser();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			merchantProfile = mservice.getMerchantUserInformation(user);

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public String changePassword() {

		try {
			
			super.setPageTitle("label.changepassword");

			if (merchantProfile == null) {
				log.error("merchantProfileIsNull");
				super.setTechnicalMessage();
				return INPUT;
			}

			int merchantId = super.getContext().getMerchantid();
			String user = super.getPrincipal().getRemoteUser();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			MerchantUserInformation profile = mservice
					.getMerchantUserInformation(user);

			// new paswords match
			if (StringUtils.isBlank(merchantProfile.getAdminPass())) {
				super.setErrorMessage("messages.required.currentpassword");
				return INPUT;
			}

			if (StringUtils.isBlank(this.getNewPassword())) {
				super.setErrorMessage("messages.required.newpassword");
				return INPUT;
			}

			if (StringUtils.isBlank(this.getRepeatNewPassword())) {
				super.setErrorMessage("messages.required.repeatnewpassword");
				return INPUT;
			}

			if (!this.getNewPassword().equals(this.getRepeatNewPassword())) {
				super.setErrorMessage("messages.password.match");
				return INPUT;
			}

			// 6 to 8 characters
			if (this.getNewPassword().length() < 6
					|| this.getNewPassword().length() > 8) {
				super.setErrorMessage("messages.password.length");
				return INPUT;
			}

			String key = EncryptionUtil.generatekey(String
					.valueOf(SecurityConstants.idConstant));
			String enc = EncryptionUtil.encrypt(key, this.getNewPassword());

			profile.setAdminPass(enc);

			mservice.saveOrUpdateMerchantUserInformation(profile);

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

		return SUCCESS;

	}

	public String display() {


		try {
			
			//@TODO set security questions and answers
			
			super.setPageTitle("label.menu.group.profile");
			
			SecurityQuestionsModule module = (SecurityQuestionsModule)SpringUtil.getBean("securityQuestions");
			Map questions = module.getSecurityQuestions(super.getLocale());
			this.setSecurityQuestions(questions);

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			merchantProfile = mservice.getMerchantUserInformation(super
					.getPrincipal().getUserPrincipal().getName());
			if (merchantProfile == null) {// should be created from the original
											// subscribtion process
				log.error("Profile does not exist for merchantid "
						+ super.getPrincipal().getRemoteUser());
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText("errors.technical"));
				return ERROR;
			}

			// NEED TO SET COUNTRY ID IN THE SESSION IN ORDER TO RETREIVE
			// ASSOCIATE PROVINCES
			if (merchantProfile.getUsercountrycode() == 0) {// original default
															// country code
				Configuration conf = PropertiesUtil.getConfiguration();
				int defaultCountry = conf
						.getInt("core.system.defaultcountryid",
								Constants.US_COUNTRY_ID);
				MessageUtil.addNoticeMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(
								"messages.updateinformation"));
				merchantProfile.setUsercountrycode(defaultCountry);
				super.prepareSelections(defaultCountry);

			} else {

				super.prepareSelections(merchantProfile.getUsercountrycode());
			}
			
			if(!StringUtils.isBlank(merchantProfile.getSecurityQuestion1())
					
				|| !StringUtils.isBlank(merchantProfile.getSecurityQuestion2())
				
				|| !StringUtils.isBlank(merchantProfile.getSecurityQuestion3())
				) {
				
				
			
			answers.add(0,Integer.parseInt(merchantProfile.getSecurityQuestion1()));
			answers.add(1,Integer.parseInt(merchantProfile.getSecurityQuestion2()));
			answers.add(2,Integer.parseInt(merchantProfile.getSecurityQuestion3()));
			
			answersText.add(0, merchantProfile.getSecurityAnswer1());
			answersText.add(1, merchantProfile.getSecurityAnswer2());
			answersText.add(2, merchantProfile.getSecurityAnswer3());
			
		}

			/**
			 * //@todo get credit card //parse expiration date to mm yy format
			 * String ccDate = profile.getCcExpires(); if(ccDate!=null &&
			 * !ccDate.equals("")) { int length = ccDate.length(); String ccm =
			 * null; String ccy = null; if(length==4) { ccm =
			 * ccDate.substring(0,2); ccy = ccDate.substring(3); } else { ccm =
			 * "0" + ccDate.substring(0,1); ccy = ccDate.substring(2); }
			 * this.setCcYear(ccy); this.setCcMonth(ccm);
			 * 
			 * this.setSecurityCode(new String(profile.getCcCvv())); } else {
			 * java.util.Calendar cal = new java.util.GregorianCalendar();
			 * this.setCcYear
			 * (String.valueOf((cal.get(java.util.Calendar.YEAR))));
			 * this.setCcMonth
			 * (String.valueOf((cal.get(java.util.Calendar.MONTH))));
			 * this.setSecurityCode(""); }
			 * 
			 * 
			 * if(profile.getCcNumber()!=null &&
			 * !profile.getCcNumber().trim().equals("")) { //decrypt credit card
			 * String decryptedvalue =
			 * EncryptionUtil.decrypt(EncryptionUtil.generatekey
			 * (String.valueOf(merchantid.intValue())), profile.getCcNumber());
			 * //mask value CreditCardUtil util = new CreditCardUtil(); String
			 * card = util.maskCardNumber(decryptedvalue);
			 * profile.setCcNumber(card); }
			 **/

			// }

		} catch (Exception e) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);
		}

		return SUCCESS;

	}

	/**
	 * saveProfile
	 * 
	 * @return "SUCCESS" or "ERROR"
	 * @throws Exception
	 */
	public String saveProfile() throws Exception {


		try {
			
			super.setPageTitle("label.menu.group.profile");
			
			SecurityQuestionsModule module = (SecurityQuestionsModule)SpringUtil.getBean("securityQuestions");
			Map questions = module.getSecurityQuestions(super.getLocale());
			this.setSecurityQuestions(questions);


			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantUserInformation mUserInfo = mservice
					.getMerchantUserInformation(super.getPrincipal()
							.getRemoteUser());

			if (mUserInfo == null) {
				mUserInfo = new MerchantUserInformation();
			}



			java.util.Date dt = new java.util.Date();

			mUserInfo.setAdminEmail(this.merchantProfile.getAdminEmail());
			mUserInfo.setUserfname(this.merchantProfile.getUserfname());
			mUserInfo.setUserlname(this.merchantProfile.getUserlname());
			mUserInfo.setUseraddress(this.merchantProfile.getUseraddress());
			mUserInfo.setUsercity(this.merchantProfile.getUsercity());
			mUserInfo.setUserphone(this.merchantProfile.getUserphone());
			mUserInfo.setUserpostalcode(this.merchantProfile
					.getUserpostalcode());
			mUserInfo.setUserstate(this.merchantProfile.getUserstate());
			mUserInfo.setUsercountrycode(this.merchantProfile
					.getUsercountrycode());
			mUserInfo.setUserlang(this.merchantProfile.getUserlang());

			super.prepareSelections(mUserInfo.getUsercountrycode());

			mUserInfo.setLastModified(new java.util.Date(dt.getTime()));

			mservice.saveOrUpdateMerchantUserInformation(mUserInfo);


			ctx.setLang(mUserInfo.getUserlang());


			super.setSuccessMessage();

		} catch (Exception e) {

			if (e instanceof ConstraintViolationException) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(
								"messages.emailalreadyexist"));
			} else {
				log.error(e);
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText("errors.technical"));
			}
		}

		return SUCCESS;

	}
	
	public String viewUser() throws Exception {
		
		super.setPageTitle("label.admin.security.manageuser");
		
		//get roles
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(super.getLocale());
		
		
		
		//label.admin.security.admin
		//label.admin.security.store
		//label.admin.security.catalog
		//label.admin.security.checkout
		//label.admin.security.order
		
		masterRoles.put("admin", label.getText("label.admin.security.admin"));
		masterRoles.put("user", label.getText("label.admin.security.user"));
		
		//get roles from def
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);

		
		Collection<MerchantUserRoleDef> defs = mservice.getMerchantUserRoleDef();
		for(Object o : defs) {
			
			MerchantUserRoleDef def = (MerchantUserRoleDef)o;
			
			if(!def.getRoleCode().equals("admin") && !def.getRoleCode().equals("superuser")) {
				roles.put(def.getRoleCode(),label.getText("label.admin.security." + def.getRoleCode()));
			}
		}
		
		//see if user exist
		
		MerchantUserInformation userInformation = this.getMerchantProfile();
		
		if(userInformation!=null && userInformation.getMerchantUserId()!=null) {
			

			merchantProfile = mservice.getMerchantUserInformation(userInformation.getMerchantUserId());
			
			super.authorize((IMerchant)merchantProfile);
			

			
			Collection rollColl = mservice.getUserRoles(merchantProfile.getAdminName());
			for(Object o : rollColl) {
				MerchantUserRole r = (MerchantUserRole)o;
				userRoles.add(r.getRoleCode());
			}
		} else {
			
			if(!userRoles.contains("superuser") || !userRoles.contains("admin") || !userRoles.contains("user")) {
				userRoles.add("admin");
			}
			
		}
		
		
		
		//if user == superuser -> admin if user = admin -> admin else -> user and populate checkboxes
		

		
		return SUCCESS;
		
	}
	
	public String saveUser() throws Exception {

		
		if(this.getMerchantProfile()==null) {
			log.error("MerchantUserInformation should not be null");
			super.setTechnicalMessage();
			return INPUT;
		}
		
		this.viewUser();
		
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		
		if(StringUtils.isBlank(this.getSubmitMasterRole())) {
			//set to admin by default
			this.setSubmitMasterRole("admin");
		}
		
		//see if user exist
		
		if(this.getMerchantProfile()!=null && this.getMerchantProfile().getMerchantUserId()!=null && this.getMerchantProfile().getMerchantUserId().longValue() >0) {
			
			//Collection rollColl = mservice.getUserRoles(merchantProfile.getAdminName());

			mservice.deleteUserRoles(merchantProfile.getAdminName());
			
		} 
		
		if(this.getMerchantProfile()!=null && this.getMerchantProfile().getMerchantUserId()==null) {
		
			//check if email already exist
			MerchantUserInformation user = mservice.getMerchantUserInformationByAdminEmail(this.getMerchantProfile().getAdminEmail());
			if(user!=null) {
				super.setErrorMessage("messages.emailalreadyexist");
				return INPUT;
			}
		
		}
			
		List<MerchantUserRole> newRoles = new ArrayList();
		
		if(this.getMerchantProfile()!=null && this.getMerchantProfile().getMerchantUserId()==null) {
			this.getMerchantProfile().setAdminName(this.getAdminName());
		}
		
		
		if(!StringUtils.isBlank(this.getSubmitMasterRole()) && !this.getSubmitMasterRole().equals("admin")) {
			for(Object o: submitRoles) {
				
				String role = (String)o;
				
				MerchantUserRole mur = new MerchantUserRole();
				mur.setAdminName(this.getMerchantProfile().getAdminName());
				mur.setRoleCode(role);
				
				newRoles.add(mur);
				
			}
		}
		

		
		//add master role
		MerchantUserRole mur = new MerchantUserRole();
		mur.setAdminName(this.getMerchantProfile().getAdminName());
		mur.setRoleCode(this.getSubmitMasterRole());
		newRoles.add(mur);
		
		if(this.getMerchantProfile()!=null && this.getMerchantProfile().getMerchantUserId()==null) {
			mservice.createMerchantUserInformation(super.getContext().getMerchantid(),this.getMerchantProfile(),newRoles,super.getLocale());
			
		} else {
			
			mservice.saveOrUpdateRoles(newRoles);
		}
		
		super.setSuccessMessage();
		
		return SUCCESS;
		
	}
	
	/**
	 * Get a user list for a given store
	 * @return
	 * @throws Exception
	 */
	public String editUserList() throws Exception {
		
		super.setPageTitle("label.user.userlist");
		
		//view user list
		
		//get users from merchantId
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		
		merchantUserInformations = mservice.getMerchantUserInfo(super.getContext().getMerchantid());
		
		return SUCCESS;
		
	}
	
	public String deleteUser() throws Exception {
		
		if(this.getMerchantProfile()==null || this.getMerchantProfile().getMerchantUserId()==null) {
			log.error("Should receive MerchantUserInformation id");
			super.setTechnicalMessage();
			return INPUT;
		}
		
		//get users from merchantId
		MerchantService mservice = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		
		
		MerchantUserInformation user = mservice.getMerchantUserInformation(this.getMerchantProfile().getMerchantUserId());
		
		mservice.deleteMerchantUserInformation(user);
		
		super.setSuccessMessage();
		

		return SUCCESS;
		
	}

	public MerchantUserInformation getMerchantProfile() {
		return merchantProfile;
	}

	public void setMerchantProfile(MerchantUserInformation merchantProfile) {
		this.merchantProfile = merchantProfile;
	}

	public Integer getCountryCode() {
		return countryCode;
	}

	public void setCountryCode(Integer countryCode) {
		this.countryCode = countryCode;
	}

	public String getCcMonth() {
		return ccMonth;
	}

	public void setCcMonth(String ccMonth) {
		this.ccMonth = ccMonth;
	}

	public String getCcYear() {
		return ccYear;
	}

	public void setCcYear(String ccYear) {
		this.ccYear = ccYear;
	}

	public String getSecurityCode() {
		return securityCode;
	}

	public void setSecurityCode(String securityCode) {
		this.securityCode = securityCode;
	}

	public String getNewPassword() {
		return newPassword;
	}

	public void setNewPassword(String newPassword) {
		this.newPassword = newPassword;
	}

	public String getRepeatNewPassword() {
		return repeatNewPassword;
	}

	public void setRepeatNewPassword(String repeatNewPassword) {
		this.repeatNewPassword = repeatNewPassword;
	}

	public String getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(String merchantId) {
		this.merchantId = merchantId;
	}



	public boolean isResetPasswordResponse() {
		return resetPasswordResponse;
	}

	public void setResetPasswordResponse(boolean resetPasswordResponse) {
		this.resetPasswordResponse = resetPasswordResponse;
	}

	public String getAdminName() {
		return adminName;
	}

	public void setAdminName(String adminName) {
		this.adminName = adminName;
	}

	public Map getSecurityQuestions() {
		return securityQuestions;
	}

	public void setSecurityQuestions(Map securityQuestions) {
		this.securityQuestions = securityQuestions;
	}

	public List<Integer> getAnswers() {
		return answers;
	}

	public void setAnswers(List<Integer> answers) {
		this.answers = answers;
	}

	public List<String> getAnswersText() {
		return answersText;
	}

	public void setAnswersText(List<String> answersText) {
		this.answersText = answersText;
	}

	public List<String> getQuestionsText() {
		return questionsText;
	}

	public void setQuestionsText(List<String> questionsText) {
		this.questionsText = questionsText;
	}
	
	@JSON(name="passwordResetSuccess")
	public boolean isPasswordResetSuccess() {
		return passwordResetSuccess;
	}

	public void setPasswordResetSuccess(boolean passwordResetSuccess) {
		this.passwordResetSuccess = passwordResetSuccess;
	}

	public String getMerchantPassword() {
		return merchantPassword;
	}

	public void setMerchantPassword(String merchantPassword) {
		this.merchantPassword = merchantPassword;
	}

	public Map<String, String> getRoles() {
		return roles;
	}

	public void setRoles(Map<String, String> roles) {
		this.roles = roles;
	}

	public List<String> getSubmitRoles() {
		return submitRoles;
	}

	public void setSubmitRoles(List<String> submitRoles) {
		this.submitRoles = submitRoles;
	}

	public Map<String, String> getMasterRoles() {
		return masterRoles;
	}

	public void setMasterRoles(Map<String, String> masterRoles) {
		this.masterRoles = masterRoles;
	}

	public Collection<String> getUserRoles() {
		return userRoles;
	}

	public void setUserRoles(Collection<String> userRoles) {
		this.userRoles = userRoles;
	}

	public String getSubmitMasterRole() {
		return submitMasterRole;
	}

	public void setSubmitMasterRole(String submitMasterRole) {
		this.submitMasterRole = submitMasterRole;
	}

	public Collection<MerchantUserInformation> getMerchantUserInformations() {
		return merchantUserInformations;
	}

	public void setMerchantUserInformations(
			Collection<MerchantUserInformation> merchantUserInformations) {
		this.merchantUserInformations = merchantUserInformations;
	}

	//public String getMasterRole() {
	//	return masterRole;
	//}

	//public void setMasterRole(String masterRole) {
	//	this.masterRole = masterRole;
	//}



}



```
