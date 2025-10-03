# SmtpMailModule.java

## Review

## 1. Summary  

**Purpose** – `SmtpMailModule` is a utility for sending templated e‑mails (plain‑text + HTML) through an SMTP server.  
It inherits configuration, mail‑sender, and template helpers from `EmailUtilImpl` and uses **FreeMarker** to render the message body.  

**Key components**  
| Component | Responsibility |
|-----------|----------------|
| `SmtpMailModule` | Builds a multipart e‑mail and delegates sending to Spring’s `JavaMailSender`. |
| `MimeMessagePreparator` (anonymous) | Configures the `MimeMessage` (recipients, subject, sender, content). |
| FreeMarker `Configuration` | Loads the template used for both text and HTML parts. |
| `MimeMultipart` | Holds the alternative/plain‑text and HTML parts. |

**Design patterns / libraries**  
* Uses **Strategy** – the `MimeMessagePreparator` encapsulates the message‑building logic.  
* Relies on Spring’s **JavaMailSender** and **MimeMessagePreparator** abstractions.  
* Uses **FreeMarker** for template processing.  
* Implements a very lightweight, no‑framework‑specific “adapter” pattern to integrate FreeMarker with Spring’s mail API.

---

## 2. Detailed Description  

1. **Method entry (`send`)**  
   * Parameters: target address (`email`), message subject (`subject`), and a map of values (`entries`) used for template substitution.  
   * Local variables are pulled from the parent class: the e‑mail sender, default from address, and FreeMarker configuration.

2. **Message preparation**  
   * An anonymous `MimeMessagePreparator` is created.  
   * Inside `prepare()` the message is configured:  
     * Recipient, from address (with a personal name), subject.  
     * A `MimeMultipart("alternative")` is built to hold the two parts.  
     * **Text part** – the template is rendered to a `StringWriter`, wrapped in a `DataSource` that streams the plain‑text content.  
     * **HTML part** – the same template is rendered again (to a different writer) and wrapped in a nested `MimeMultipart("related")`.  
     * The HTML multipart is then added as a body part of the main alternative multipart.  

3. **Sending**  
   * The prepared message is passed to `super.getMailSender().send(preparator);`.  
   * No attachments are currently supported (the relevant code is commented out).  

4. **Cleanup** – no explicit resource cleanup; the underlying mail sender handles it.  

**Assumptions / constraints**  
* The supplied template can produce both a plain‑text and an HTML version – the code currently renders it twice.  
* The caller guarantees that `entries` contains all placeholders required by the template.  
* No validation of the e‑mail address format is performed.  
* The application runs on a Java version that still supports `StringBufferInputStream` (pre‑Java 9).  

**Architectural choices**  
* The code chooses *manual* multipart construction over Spring’s `MimeMessageHelper`.  
* It uses an anonymous inner class rather than a separate strategy implementation.  
* Direct use of `StringBufferInputStream` is a quick way to feed the message body but is deprecated and not portable to newer Java releases.

---

## 3. Functions / Methods  

| Method | Parameters | Return | Side‑Effects | Notes |
|--------|------------|--------|--------------|-------|
| `send(String email, String subject, Map entries)` | *`email`* – recipient address<br>*`subject`* – e‑mail subject<br>*`entries`* – template data | `void` | - Builds and sends a MIME e‑mail.<br>- May throw `Exception` if template processing or mail sending fails. | Uses raw `Map`; no generics. |
| `prepare(MimeMessage mimeMessage)` (anonymous) | `mimeMessage` – object to configure | `void` | Sets recipients, from, subject, and multipart content. | Implemented inside `send`; not reusable outside this class. |
| `process(Template template, Writer writer)` (FreeMarker) | — | — | Renders template into writer; throws `TemplateException` if processing fails. | Called twice (for text & HTML). |

The only public API exposed by this class is the `send(...)` method. All other logic is encapsulated within the anonymous preparator.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `freemarker.template.*` | Third‑party | FreeMarker core for template handling. |
| `javax.mail.*`, `javax.mail.internet.*` | Standard Java EE | JavaMail API. |
| `org.springframework.mail.*` | Third‑party | Spring Mail support (`JavaMailSender`, `MimeMessagePreparator`). |
| `javax.activation.*` | Standard Java SE / EE | For `DataHandler` and `DataSource`. |
| `com.salesmanager.core.util.EmailUtilImpl` | Internal | Provides mail sender, configuration, and address helpers. |
| `StringBufferInputStream` | Deprecated (Java 1.4‑1.7) | Removed from Java 9+. |

No external runtime frameworks beyond Spring are required, but the class is tightly coupled to Spring’s mail abstraction.

---

## 5. Additional Notes & Recommendations  

### 5.1. Deprecated / Platform‑specific issues  
* **`StringBufferInputStream`** – removed in Java 9. Replace with `ByteArrayInputStream` or a custom `ByteArrayDataSource`.  
* **Raw `Map`** – generics should be used (`Map<String, Object>`).  

### 5.2. Template rendering duplication  
The code renders the same template twice – once for plain text and once for HTML.  
*If the template is designed to produce both formats in a single pass (e.g., contains `<#if isHtml>` blocks), rendering it twice wastes CPU and memory.*  
Consider using a single rendered string and splitting it, or keeping two separate templates.

### 5.3. Multipart construction  
The inner `MimeMultipart("related")` wrapping the HTML content is unnecessary unless you intend to embed resources (e.g., images).  
If you only need plain text + HTML, simply add two body parts to the `alternative` multipart:
```java
BodyPart htmlPart = new MimeBodyPart();
htmlPart.setContent(htmlWriter.toString(), "text/html; charset=UTF-8");
mp.addBodyPart(htmlPart);
```

### 5.4. Use `MimeMessageHelper`  
Spring’s `MimeMessageHelper` dramatically simplifies this logic:
```java
MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true, "UTF-8");
helper.setTo(email);
helper.setFrom(new InternetAddress(from, eml));
helper.setSubject(subject);
helper.setText(textWriter.toString(), htmlWriter.toString());
```
This handles the `alternative` multipart for you, avoids manual `DataSource` creation, and guarantees proper charset handling.

### 5.5. Attachment handling  
The commented block suggests future support. When adding attachments, use `MimeMessageHelper.addAttachment(...)` instead of manually building `MimeBodyPart`s.

### 5.6. Validation & Error handling  
* Validate `email` (RFC 5322 compliant) before sending.  
* Catch `MessagingException` and log a meaningful message (currently re‑throws as `Exception`).  

### 5.7. Resource management  
* `StringWriter` does not need closing, but if you switch to `StringBuilder` or other streams, ensure they’re closed via try‑with‑resources.  

### 5.8. Encoding & charset  
Always specify the charset explicitly (`"UTF-8"`) when setting text or content types to avoid default platform issues.

### 5.9. Testability  
* Extract template rendering into a separate method so it can be unit‑tested independently of the mail sending logic.  
* Inject a mock `JavaMailSender` in tests to verify message construction without hitting a real SMTP server.

### 5.10. Future enhancements  
| Feature | Why it matters |
|---------|----------------|
| **HTML resource embedding** | If inline images are needed. |
| **Multiple recipients / CC/BCC** | Expand use‑case. |
| **Scheduling / queueing** | Decouple from request thread. |
| **Template selection** | Different email types (welcome, reset‑password, etc.). |
| **Internationalization** | Load locale‑specific templates. |

---

### Bottom line  
The class achieves its core goal – sending a templated plain‑text + HTML e‑mail – but it relies on deprecated APIs, manual multipart handling, and duplicate template rendering. Refactoring to use Spring’s `MimeMessageHelper`, eliminating `StringBufferInputStream`, and applying generics will make the code more robust, future‑proof, and easier to maintain.

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
package com.salesmanager.core.module.impl.application.utils;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.StringBufferInputStream;
import java.io.StringWriter;
import java.util.Map;
import freemarker.template.Configuration;

import javax.mail.BodyPart;
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Multipart;
import javax.mail.Session;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeBodyPart;
import javax.mail.internet.MimeMessage;
import javax.mail.internet.MimeMultipart;

import org.springframework.mail.MailPreparationException;
import org.springframework.mail.javamail.MimeMessagePreparator;

import com.salesmanager.core.util.EmailUtilImpl;

import freemarker.template.Template;
import freemarker.template.TemplateException;

public class SmtpMailModule extends EmailUtilImpl {

	public void send(final String email, final String subject, final Map entries)
			throws Exception {

		final String eml = super.getFromEmail();
		final String from = super.getFromAddress();
		final Configuration conf = super.getConfiguration();
		final String tmpl = super.getFreemarkerTemplate();

		MimeMessagePreparator preparator = new MimeMessagePreparator() {
			public void prepare(MimeMessage mimeMessage)
					throws MessagingException, IOException {

				mimeMessage.setRecipient(Message.RecipientType.TO,
						new InternetAddress(email));

				InternetAddress inetAddress = new InternetAddress();

				inetAddress.setPersonal(eml);
				inetAddress.setAddress(from);

				mimeMessage.setFrom(inetAddress);
				mimeMessage.setSubject(subject);

				Multipart mp = new MimeMultipart("alternative");

				// Create a "text" Multipart message
				BodyPart textPart = new MimeBodyPart();
				Template textTemplate = conf.getTemplate(tmpl);
				final StringWriter textWriter = new StringWriter();
				try {
					textTemplate.process(entries, textWriter);
				} catch (TemplateException e) {
					throw new MailPreparationException(
							"Can't generate text mail", e);
				}
				textPart.setDataHandler(new javax.activation.DataHandler(
						new javax.activation.DataSource() {
							public InputStream getInputStream()
									throws IOException {
								return new StringBufferInputStream(textWriter
										.toString());
							}

							public OutputStream getOutputStream()
									throws IOException {
								throw new IOException("Read-only data");
							}

							public String getContentType() {
								return "text/plain";
							}

							public String getName() {
								return "main";
							}
						}));
				mp.addBodyPart(textPart);

				// Create a "HTML" Multipart message
				Multipart htmlContent = new MimeMultipart("related");
				BodyPart htmlPage = new MimeBodyPart();
				Template htmlTemplate = conf.getTemplate(tmpl);
				final StringWriter htmlWriter = new StringWriter();
				try {
					htmlTemplate.process(entries, htmlWriter);
				} catch (TemplateException e) {
					throw new MailPreparationException(
							"Can't generate HTML mail", e);
				}
				htmlPage.setDataHandler(new javax.activation.DataHandler(
						new javax.activation.DataSource() {
							public InputStream getInputStream()
									throws IOException {
								return new StringBufferInputStream(htmlWriter
										.toString());
							}

							public OutputStream getOutputStream()
									throws IOException {
								throw new IOException("Read-only data");
							}

							public String getContentType() {
								return "text/html";
							}

							public String getName() {
								return "main";
							}
						}));
				htmlContent.addBodyPart(htmlPage);
				BodyPart htmlPart = new MimeBodyPart();
				htmlPart.setContent(htmlContent);
				mp.addBodyPart(htmlPart);

				mimeMessage.setContent(mp);

				// if(attachment!=null) {
				// MimeMessageHelper messageHelper = new
				// MimeMessageHelper(mimeMessage, true);
				// messageHelper.addAttachment(attachmentFileName, attachment);
				// }

			}
		};

		super.getMailSender().send(preparator);

	}

}



```
