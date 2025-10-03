# SimpleSmtpMailModule.java

## Review

## 1. Summary

`SimpleSmtpMailModule` is a utility that prepares an e‑mail message using **FreeMarker** templates and the **JavaMail API**.  
The class extends `EmailUtilImpl` (presumably a base class that supplies configuration such as SMTP host, port, credentials, and a FreeMarker `Configuration`).  
The `send()` method:

1. Creates a JavaMail `Session` and a `MimeMessage`.
2. Sets the **To** recipient and the **From** address using the base class’ getters.
3. Generates two body parts:
   * a **plain‑text** part (processed from a FreeMarker template)
   * an **HTML** part (also from a template, wrapped in a `related` multipart)
4. Adds both parts to an `alternative` multipart and sets that as the message content.
5. (Intended) would send the message, but the method ends by throwing a `Not Implemented` exception.

The implementation is **incomplete** – the message is never sent and several modern Java practices are not followed.

---

## 2. Detailed Description

### Core components

| Component | Responsibility |
|-----------|----------------|
| `Session` | Holds JavaMail configuration (here obtained via `Session.getDefaultInstance(new Properties())`). |
| `MimeMessage` | The mail message being constructed. |
| `Multipart` | Holds the body parts (text + HTML). |
| `FreeMarker Configuration` (via `super.getConfiguration()`) | Loads templates for the mail body. |
| `DataSource` (anonymous inner class) | Wraps the template output into a data handler for the MIME body. |

### Execution flow

1. **Session creation** – uses a *default* session with an empty `Properties` object.  
   No host, port, or authentication data is supplied, so the session will be unable to connect to an SMTP server unless a global default is already configured.
2. **Message construction** – the recipient, sender, and subject are set.
3. **Text part** –  
   * A FreeMarker template is fetched (`super.getFreemarkerTemplate()`).  
   * The template is processed into a `StringWriter`.  
   * An anonymous `DataSource` provides a `StringBufferInputStream` (deprecated) as the data stream.  
   * This part is added to the `alternative` multipart.
4. **HTML part** – follows the same process, but the MIME type is set to `text/html` and wrapped in a `related` multipart (though no related resources are actually added).  
5. **Final message** – the multipart is set as the content of the `MimeMessage`.  
6. **Unimplemented send** – a generic `Exception` is thrown, signalling that the code is still a stub.

### Assumptions & constraints

* The FreeMarker template name is retrieved via `super.getFreemarkerTemplate()`; it must exist and be compatible with both plain‑text and HTML rendering (which is unusual – normally two separate templates are used).
* The map of `entries` is untyped (raw `Map`), so runtime `ClassCastException`s can surface if the template expects a typed model.
* The method does not configure any SMTP settings; it relies on a pre‑configured default `Session`.
* No support for attachments or inline resources (the commented code hints at future work).

### Design choices

* **Template‑first**: The mail body is entirely generated from FreeMarker, keeping content separate from code.
* **Multipart alternative**: Provides both plain‑text and HTML to clients that can display either.
* **Manual `DataSource`**: Avoids creating a custom `StringDataSource` or using `ByteArrayDataSource` from Apache Commons.

However, these choices are somewhat dated (e.g., `StringBufferInputStream` is deprecated, and manual data source construction is error‑prone).

---

## 3. Functions/Methods

| Method | Parameters | Returns | Side‑effects | Comments |
|--------|------------|---------|--------------|----------|
| `public void send(final String email, final String subject, final Map entries)` | `email` – recipient address; `subject` – mail subject; `entries` – data model for the template | `void` | Constructs a `MimeMessage` and (supposedly) sends it | The method ends by throwing an exception, so no email is actually sent. |

No other public methods are present. The helper classes (`DataSource`, `BodyPart`) are used only within this method.

---

## 4. Dependencies

| Dependency | Category | Notes |
|------------|----------|-------|
| `javax.mail.*` | Standard (JavaMail) | Required for session, message, MIME handling. |
| `javax.activation.*` | Standard (JavaBeans Activation Framework) | Provides `DataHandler` and `DataSource`. |
| `freemarker.template.*` | Third‑party | FreeMarker templating engine. |
| `org.springframework.mail.MailPreparationException` | Third‑party | Spring framework exception (only used for wrapping `TemplateException`). |
| `com.salesmanager.core.util.EmailUtilImpl` | Project internal | Base class providing configuration (SMTP credentials, FreeMarker config). |

No database or web frameworks are referenced directly. The class is platform‑agnostic but expects a JavaMail provider on the classpath.

---

## 5. Additional Notes

### Strengths
* Clear separation of concerns: templating vs. MIME construction.
* Supports both plain‑text and HTML in a single message.
* Uses Spring’s `MailPreparationException` to surface template errors.

### Weaknesses & Edge Cases
1. **Incomplete implementation** – the method never actually sends the mail; it throws a generic `Exception`.  
   *Potential fix:* Integrate with Spring’s `JavaMailSender` or `Transport.send(mimeMessage)` after configuring the session.
2. **Deprecated API usage** – `StringBufferInputStream` was removed in Java 1.4/5 and is a security risk.  
   *Potential fix:* Replace with `ByteArrayInputStream` or use a proper `StringDataSource` implementation.
3. **Hard‑coded template** – same template used for both text and HTML. This can lead to formatting issues or XSS if the template is not carefully crafted.  
   *Potential fix:* Use two distinct templates (`subject.txt.ftl`, `subject.html.ftl`).
4. **Unnecessary complexity** – the anonymous `DataSource` wrapper could be replaced with a single utility method or library class, improving readability and testability.
5. **No attachment support** – commented code suggests future intention, but the current implementation is limited.
6. **Raw `Map` usage** – loses type safety and may cause runtime errors if the template expects specific types.  
   *Potential fix:* Use generics (`Map<String, Object>`) or a dedicated POJO.
7. **SMTP session misconfiguration** – a default `Session` with empty properties will not connect to any real SMTP server.  
   *Potential fix:* Pull SMTP configuration from `EmailUtilImpl` or pass a pre‑configured `Session` to the method.

### Suggested Enhancements
* **Refactor to Spring Mail** – inject a `JavaMailSender` and delegate sending, simplifying session handling.
* **Add Attachment Handling** – expose a method to add `File` or `InputStream` attachments.
* **Error Handling** – catch `MessagingException` and convert to a domain‑specific exception.
* **Unit Tests** – mock the JavaMail API and validate that the MIME structure matches expectations.
* **Template Separation** – allow the caller to specify distinct templates for text and HTML.
* **Security** – validate email addresses and sanitize template output to prevent injection attacks.

--- 

**Verdict:**  
The code demonstrates a reasonable architectural approach for composing emails with FreeMarker and JavaMail, but it is unfinished and uses deprecated patterns. Refactoring for modern APIs, proper exception handling, and a clear sending strategy will be required before it can be used in production.

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
import java.util.Properties;

import javax.mail.BodyPart;
import javax.mail.Message;
import javax.mail.Multipart;
import javax.mail.Session;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeBodyPart;
import javax.mail.internet.MimeMessage;
import javax.mail.internet.MimeMultipart;

import org.springframework.mail.MailPreparationException;

import com.salesmanager.core.util.EmailUtilImpl;

import freemarker.template.Template;
import freemarker.template.TemplateException;

public class SimpleSmtpMailModule extends EmailUtilImpl {

	public void send(final String email, final String subject, final Map entries)
			throws Exception {

		Session session = Session.getDefaultInstance(new Properties());

		MimeMessage mimeMessage = new MimeMessage(session);

		mimeMessage.setRecipient(Message.RecipientType.TO, new InternetAddress(
				email));

		InternetAddress inetAddress = new InternetAddress();

		inetAddress.setPersonal(super.getFromEmail());
		inetAddress.setAddress(super.getFromAddress());

		mimeMessage.setFrom(inetAddress);
		mimeMessage.setSubject(subject);

		Multipart mp = new MimeMultipart("alternative");

		// Create a "text" Multipart message
		BodyPart textPart = new MimeBodyPart();
		Template textTemplate = super.getConfiguration().getTemplate(
				super.getFreemarkerTemplate());
		final StringWriter textWriter = new StringWriter();
		try {
			textTemplate.process(entries, textWriter);
		} catch (TemplateException e) {
			throw new MailPreparationException("Can't generate text mail", e);
		}
		textPart.setDataHandler(new javax.activation.DataHandler(
				new javax.activation.DataSource() {
					public InputStream getInputStream() throws IOException {
						return new StringBufferInputStream(textWriter
								.toString());
					}

					public OutputStream getOutputStream() throws IOException {
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
		Template htmlTemplate = super.getConfiguration().getTemplate(
				super.getFreemarkerTemplate());
		final StringWriter htmlWriter = new StringWriter();
		try {
			htmlTemplate.process(entries, htmlWriter);
		} catch (TemplateException e) {
			throw new MailPreparationException("Can't generate HTML mail", e);
		}
		htmlPage.setDataHandler(new javax.activation.DataHandler(
				new javax.activation.DataSource() {
					public InputStream getInputStream() throws IOException {
						return new StringBufferInputStream(htmlWriter
								.toString());
					}

					public OutputStream getOutputStream() throws IOException {
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
		// MimeMessageHelper messageHelper = new MimeMessageHelper(mimeMessage,
		// true);
		// messageHelper.addAttachment(attachmentFileName, attachment);
		// }

		throw new Exception(
				"Not Implemented, needs to connect to an implementation");
		// simple http server sends emails on port 25, no configuration required
		// https://aspirin.dev.java.net/ (2 jars are required dnsjava and
		// aspirin)
		// org.masukomi.aspirin.core.MailQue.queMail(mimeMessage);

	}

}



```
