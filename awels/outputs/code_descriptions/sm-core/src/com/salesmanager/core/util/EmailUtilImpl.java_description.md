# EmailUtilImpl.java

## Review

## 1. Summary

**Purpose**  
`EmailUtilImpl` is an abstract helper that prepares the context for HTML e‑mail templates and wires the mail‑sending infrastructure (Spring’s `JavaMailSender` and FreeMarker). Concrete subclasses provide the actual sending logic by implementing the `send()` method.

**Key components**

| Component | Responsibility |
|-----------|----------------|
| `prepareEmailContext()` | Builds a `Map<String,Object>` that holds the dynamic data needed by a FreeMarker template (store name, logo, disclaimer, footer, etc.). |
| `send()` (abstract) | Subclass‑specific logic that assembles the `MimeMessage` and dispatches it via the configured `JavaMailSender`. |
| Configuration fields (`mailSender`, `configuration`, `freemarkerTemplate`, etc.) | Store the dependencies required by the template engine and mail API. |
| `set*()` / `get*()` methods | Simple setters/getters for dependency injection (mostly used by Spring). |

**Design patterns & frameworks**

* **Template Method** – The abstract `send()` method leaves the “sending” part to subclasses while the base class takes care of context creation.
* **Dependency Injection** – Properties are injected via setters (typical Spring style).
* **Template Engine** – FreeMarker is used for HTML e‑mail rendering.
* **Mail API** – Spring’s `JavaMailSender` (backed by JavaMail) handles the actual SMTP communication.

---

## 2. Detailed Description

### Execution Flow

1. **Initialization** – A Spring bean of a concrete subclass will have its dependencies wired:
   * `JavaMailSender` (configured with host, port, credentials, etc.)
   * FreeMarker `Configuration` (template loader, encoding, etc.)
   * `freemarkerTemplate` (the name of the FreeMarker file to use)

2. **Context Creation** – When a client calls `prepareEmailContext(profile, lang)` the method:
   * Pulls internationalised strings via `LabelUtil`.
   * Builds HTML fragments for the logo, disclaimer, and footer.
   * Stores all values in a `HashMap` (raw type) that will be merged into the FreeMarker template.

3. **Sending** – The subclass implements `send(email, subject, entries)`:
   * Typically, it would create a `MimeMessage`, process the template with the `entries` map, and send it via `mailSender`.

4. **Cleanup** – No explicit cleanup is required; all resources (streams, mail sessions) are managed by the underlying libraries.

### Assumptions & Constraints

| Assumption | Why it matters |
|------------|----------------|
| `profile` is never `null` | The method throws a generic `Exception` if it is; callers must enforce this pre‑condition. |
| `profile.getStorelogo()` contains a valid relative path | The logo URL is constructed by concatenating many strings; any typo may break the image link. |
| `LabelUtil.getInstance()` returns a singleton | The code assumes a thread‑safe singleton. |
| FreeMarker template exists under the configured loader path | Missing templates will throw `TemplateException` at runtime. |

### Architecture & Design Choices

* **Abstract Base** – Keeps the plumbing (context building, dependency wiring) in one place, promoting reuse across different e‑mail types.
* **Raw Types** – The use of raw `Map` instead of generics is legacy style; it can cause unchecked casts downstream.
* **Hard‑coded String Concatenation** – URL construction is brittle; using `UriComponentsBuilder` or similar would be safer.
* **Deprecated Import** – `StringBufferInputStream` is imported but never used; it was removed in JDK 9 and is considered unsafe.

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `prepareEmailContext(MerchantStore, String)` | `Map prepareEmailContext(...)` | Builds the context map for a FreeMarker template. | `MerchantStore profile`, `String lang` | `Map` containing keys such as `EMAIL_STORE_NAME`, `LOGOPATH`, etc. | Throws `Exception` if profile is null; logs nothing. |
| `send(String, String, Map)` | `abstract void send(...)` | Subclass‑specific e‑mail sending logic. | `String email`, `String subject`, `Map entries` | None (throws `Exception`) | May send an e‑mail via `mailSender`. |
| `setEmailTemplate(String)` | `void setEmailTemplate(String)` | Convenience setter that delegates to `setFreemarkerTemplate`. | `String template` | None | Updates `freemarkerTemplate`. |
| `setFreemarkerMailConfiguration(Configuration)` | `void setFreemarkerMailConfiguration(Configuration)` | Stores the FreeMarker configuration instance. | `Configuration configuration` | None | Sets `this.configuration`. |
| `setFreemarkerTemplate(String)` | `void setFreemarkerTemplate(String)` | Sets the template file name. | `String freemarkerTemplate` | None | Updates `freemarkerTemplate`. |
| `setMailSender(JavaMailSender)` | `void setMailSender(JavaMailSender)` | Injects the Spring mail sender. | `JavaMailSender mailSender` | None | Updates `this.mailSender`. |
| `getFromEmail()` | `String getFromEmail()` | Returns the “From” display name. | None | `String` | None |
| `setFromEmail(String)` | `void setFromEmail(String)` | Sets the “From” display name. | `String` | None | Updates `fromEmail`. |
| `getFromAddress()` | `String getFromAddress()` | Returns the e‑mail address used in the “From” header. | None | `String` | None |
| `setFromAddress(String)` | `void setFromAddress(String)` | Sets the e‑mail address used in the “From” header. | `String` | None | Updates `fromAddress`. |
| `getConfiguration()` | `Configuration getConfiguration()` | Returns the FreeMarker configuration. | None | `Configuration` | None |
| `getFreemarkerTemplate()` | `String getFreemarkerTemplate()` | Returns the template file name. | None | `String` | None |
| `getMailSender()` | `JavaMailSender getMailSender()` | Returns the configured mail sender. | None | `JavaMailSender` | None |

*Utility methods* – none beyond simple getters/setters.

---

## 4. Dependencies

| Dependency | Type | Role |
|------------|------|------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads application properties (`core.store.mediaurl`). |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `org.springframework.mail.javamail.JavaMailSender` | Spring | SMTP mail abstraction. |
| `javax.mail.*` | JavaMail | Low‑level mail APIs (`MimeMessage`, `Multipart`, etc.). |
| `freemarker.template.*` | FreeMarker | Templating engine for HTML e‑mails. |
| `com.salesmanager.core.*` | Internal | Entities (`MerchantStore`, `MerchantUserInformation`), utilities (`LabelUtil`, `ReferenceUtil`, `DateUtil`, `PropertiesUtil`). |
| `java.io.*` | Java SE | I/O classes (note: `StringBufferInputStream` is deprecated/removed). |
| `java.util.*` | Java SE | Collection utilities. |

*Platform / Java version considerations*  
`StringBufferInputStream` is present only up to Java 8. Since it is unused, it can be safely removed to avoid confusion.  

---

## 5. Additional Notes & Recommendations

### Code‑quality improvements

1. **Generics** – Replace raw `Map` with `Map<String, Object>` to avoid unchecked casts.
2. **Remove unused import** – Delete `StringBufferInputStream`.
3. **Exception handling** – `prepareEmailContext` throws a generic `Exception`; consider a custom checked exception or `IllegalArgumentException`.
4. **Null checks** – The method assumes `profile` is non‑null but still throws a generic exception. A defensive check in constructors or setters could be added.
5. **String concatenation** – Use `StringBuilder` or `UriComponentsBuilder` for URLs to avoid hard‑coded separators.
6. **Logging** – Add at least an error log in `prepareEmailContext` when mandatory data is missing.
7. **Immutability** – The context map could be wrapped with `Collections.unmodifiableMap()` before returning, preventing accidental modifications by callers.
8. **Documentation** – JavaDoc for `prepareEmailContext` and the abstract `send` method should explain the contract and the expected keys in the map.
9. **Unit tests** – A test harness that mocks `LabelUtil` and `ReferenceUtil` would validate that all keys are populated correctly.

### Functional concerns

* **Template existence** – If the specified FreeMarker template is missing, `TemplateException` will be thrown during rendering. The subclass should handle this gracefully.
* **Encoding** – The code does not explicitly set character encoding when writing the email body; ensure UTF‑8 is used consistently.
* **Attachment handling** – The base class does not expose any mechanism for adding attachments; if needed, the subclass should provide a helper or extend the context map to include attachment data.

### Future enhancements

1. **Internationalisation support** – Currently, only the static strings are fetched via `LabelUtil`. Consider parameterising the template with a locale and letting FreeMarker handle localisation.
2. **Email analytics** – Store send status, timestamps, and bounce information in a database table.
3. **Template caching** – FreeMarker already caches templates, but ensure the configuration is set appropriately for production.
4. **Security** – Validate that the `email` address passed to `send` is a legitimate address (e.g., regex validation) to avoid injection into headers.
5. **Template pre‑validation** – A startup hook that loads all expected templates could catch missing files early.

---

### Bottom line

`EmailUtilImpl` is a reasonable, if somewhat dated, foundation for HTML e‑mail generation in a Spring/FreeMarker environment. It encapsulates context preparation and dependency wiring, but it would benefit from modern Java practices (generics, try‑with‑resources, immutable data structures) and a few clean‑up changes. Once the abstract `send()` method is implemented in concrete subclasses, the overall mailing subsystem can be robust, testable, and easier to maintain.

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
package com.salesmanager.core.util;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.StringBufferInputStream;
import java.io.StringWriter;
import java.util.HashMap;
import java.util.Map;

import javax.activation.DataSource;
import javax.mail.BodyPart;
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Multipart;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeBodyPart;
import javax.mail.internet.MimeMessage;
import javax.mail.internet.MimeMultipart;

import org.apache.log4j.Logger;
import org.springframework.mail.MailPreparationException;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessagePreparator;

import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;

import freemarker.template.Configuration;
import freemarker.template.Template;
import freemarker.template.TemplateException;

/**
 * Prepares HTML emails to be sent by the system
 * 
 * @author Carl Samson
 * 
 */

public abstract class EmailUtilImpl implements EmailUtil {

	private static org.apache.commons.configuration.Configuration config = PropertiesUtil
			.getConfiguration();

	private String chkpath = config.getString("core.store.mediaurl");

	private Configuration configuration;

	private String freemarkerTemplate;
	private JavaMailSender mailSender;

	private String fromEmail = null;

	private String fromAddress = null;

	private Logger log = Logger.getLogger(EmailUtilImpl.class);

	public Map prepareEmailContext(
			MerchantStore profile, String lang) throws Exception {

		Map emailcontext = new HashMap();

		String domain = ReferenceUtil.getUnSecureDomain(profile);

		if (profile == null) {
			throw new Exception("Profile is null");
		}

		String disclaim = LabelUtil.getInstance().getText(lang,
				"email.disclaimer");
		String spam = LabelUtil.getInstance().getText(lang,
				"email.spam.disclaimer");
		String footer = LabelUtil.getInstance().getText(lang,
				"footer.copywright");
		String msgfrom = LabelUtil.getInstance().getText(lang,
				"email.message.from");

		emailcontext.put("EMAIL_STORE_NAME", msgfrom + " "
				+ profile.getStorename());

		if (profile.getStorelogo() != null
				&& !profile.getStorelogo().equals("")) {
			StringBuffer logopath = new StringBuffer();
			logopath.append("<div class=\"header\">").append("<img src=\"");
			logopath.append(ReferenceUtil.getUnSecureDomain(profile)).append(
					"/").append(chkpath).append("/images/brandings/").append(
					profile.getMerchantId()).append("/header/").append(
					profile.getStorelogo()).append("\"");
			logopath.append(" alt=\"logo\" /></div>");
			emailcontext.put("LOGOPATH", logopath.toString());
		} else {
			emailcontext.put("LOGOPATH", "");
		}

		fromEmail = profile.getStorename();
		fromAddress = profile.getStoreemailaddress();

		StringBuffer disclaimbuffer = new StringBuffer();
		disclaimbuffer.append(disclaim).append(" ").append("<a href=\"mailto:")
				.append(profile.getStoreemailaddress()).append("\">").append(
						profile.getStoreemailaddress()).append("</a>");
		emailcontext.put("EMAIL_DISCLAIMER", disclaimbuffer.toString());

		emailcontext.put("EMAIL_SPAM_DISCLAIMER", spam);

		StringBuffer footerbuffer = new StringBuffer();
		footerbuffer.append(footer).append(" ").append(
				DateUtil.getPresentYear());
		footerbuffer.append(" ").append("<a href=\"").append(domain).append(
				"\">").append(profile.getStorename()).append("</a>");
		emailcontext.put("EMAIL_FOOTER_COPYRIGHT", footerbuffer.toString());

		return emailcontext;
	}



	public abstract void send(final String email, final String subject,
			final Map entries) throws Exception;

	public void setEmailTemplate(String template) {
		this.setFreemarkerTemplate(template);
	}

	public void setFreemarkerMailConfiguration(Configuration configuration) {
		this.configuration = configuration;
	}

	public void setFreemarkerTemplate(String freemarkerTemplate) {
		this.freemarkerTemplate = freemarkerTemplate;
	}

	public void setMailSender(JavaMailSender mailSender) {
		this.mailSender = mailSender;
	}

	public String getFromEmail() {
		return fromEmail;
	}

	public void setFromEmail(String fromEmail) {
		this.fromEmail = fromEmail;
	}

	public String getFromAddress() {
		return fromAddress;
	}

	public void setFromAddress(String fromAddress) {
		this.fromAddress = fromAddress;
	}

	public Configuration getConfiguration() {
		return configuration;
	}

	public String getFreemarkerTemplate() {
		return freemarkerTemplate;
	}

	public JavaMailSender getMailSender() {
		return mailSender;
	}

}



```
