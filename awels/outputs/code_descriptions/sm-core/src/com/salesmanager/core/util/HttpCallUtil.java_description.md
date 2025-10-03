# HttpCallUtil.java

## Review

## 1. Summary
`HttpCallUtil` is a tiny helper class that wraps `java.net.HttpURLConnection` to perform HTTP `GET` and `POST` requests.  
It exposes two static methods:

| Method | Purpose |
|--------|---------|
| `invokeGetUrl(String url)` | Sends a `GET` request to the supplied URL and returns the response body as a `String`. |
| `invokePostUrl(String url, String encodedQueryString)` | Sends a `POST` request with an `application/x-www-form-urlencoded` body and returns the response body as a `String`. |

The class is intentionally lightweight and does not rely on any third‑party HTTP client libraries (e.g., Apache HttpClient or OkHttp).  
Design pattern: *Utility class* – all methods are static, no instance state.  
Frameworks: None (plain JDK).

---

## 2. Detailed Description
### Flow of execution
1. **Setup** – A `URL` object is created from the supplied string and an `HttpURLConnection` is obtained via `openConnection()`.
2. **Connection configuration** –  
   * `doInput` and `doOutput` are set to `true` (the latter is unnecessary for a GET).  
   * The request method (`GET`/`POST`) and a few request headers (`User-Agent`, `Content-Type`, `Content-Length`) are configured.  
3. **Request body** – For `POST` the method writes the encoded query string to the connection’s `OutputStream`.  
4. **Response handling** –  
   * The response code is fetched (`getResponseCode()`).  
   * The response body is read line‑by‑line into a `StringBuffer` using a `BufferedReader` wrapped around `conn.getInputStream()`.  
5. **Cleanup** – All streams and the connection are closed in a `finally` block.  

### Assumptions & Constraints
* The caller guarantees that `url` is a well‑formed HTTP/HTTPS URL.  
* The response is small enough to be loaded entirely into memory.  
* The calling code expects a plain `String` result; no status-code handling is performed.  
* Timeout values are not set; the default platform timeout applies.  
* Character encoding is assumed to be UTF‑8 (implicit in `InputStreamReader` without a charset argument).

### Architecture & Design Choices
* **Utility Class** – Static only, no instance fields, no state.  
* **Bare‑bones HTTP** – Using the JDK’s `HttpURLConnection` keeps external dependencies to zero but sacrifices convenience (e.g., automatic redirect handling, multipart support, retry logic).  
* **Synchronous & Blocking** – All calls block until the request completes; no asynchronous APIs or callbacks are provided.

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `invokeGetUrl` | `public static String invokeGetUrl(String url) throws Exception` | Executes an HTTP GET and returns the body as a `String`. | `url`: target endpoint. | Response body text. | Opens a connection, writes/reads streams, closes them. |
| `invokePostUrl` | `public static String invokePostUrl(String url, String encodedQueryString) throws Exception` | Executes an HTTP POST with `application/x-www-form-urlencoded` body. | `url`: target endpoint; `encodedQueryString`: URL‑encoded form data. | Response body text. | Opens a connection, writes body, reads response, closes streams. |

**Reusable / utility methods**  
No separate helper methods exist; all logic is inlined. The two methods share a large amount of duplicated code (connection setup, stream handling, cleanup).

---

## 4. Dependencies
| Library | Type | Notes |
|---------|------|-------|
| `java.net.HttpURLConnection` | JDK standard | Provides low‑level HTTP support. |
| `java.io.*` | JDK standard | For I/O streams and readers. |
| `java.net.URL` | JDK standard | URL parsing. |

No external libraries are required; the class is fully self‑contained.

---

## 5. Additional Notes & Recommendations

### Strengths
* **Zero external dependencies** – Works in any Java environment.  
* **Simple interface** – Two methods expose the common use‑cases.

### Issues & Edge Cases
1. **Unnecessary `doOutput` for GET** – `conn.setDoOutput(true)` is superfluous and may trigger a request body write even if nothing is sent.  
2. **Redundant output stream handling** – `DataOutputStream` and `DataInputStream` are overkill; plain `OutputStream` / `InputStream` would suffice.  
3. **Duplicate reading of `InputStream`** – The code creates `DataInputStream` and then discards it, re‑obtaining the stream for `BufferedReader`.  
4. **No error handling** – Exceptions are swallowed silently inside the cleanup, and the caller receives only the raw exception. A more expressive error model (e.g., custom `HttpException` with status code) would aid debugging.  
5. **Missing timeouts** – `conn.setConnectTimeout()` / `conn.setReadTimeout()` should be set to avoid indefinite blocking.  
6. **Encoding assumptions** – `InputStreamReader` defaults to the platform charset; the response might be in UTF‑8. It’s safer to use `InputStreamReader(conn.getInputStream(), StandardCharsets.UTF_8)`.  
7. **Resource leaks under error paths** – If `conn.getInputStream()` throws before the `BufferedReader` is constructed, `in` may remain open.  
8. **Large responses** – Building the entire body into a `StringBuffer` may exhaust memory for large payloads. A streaming approach or a `StringBuilder` with estimated capacity could help.

### Suggested Refactor
* Extract a private helper method `readResponse(HttpURLConnection conn)` that handles the common logic (reading, closing).  
* Replace `DataInputStream` / `DataOutputStream` with `OutputStream` / `InputStream`.  
* Add optional parameters for headers, timeouts, and charset.  
* Consider using `try‑with‑resources` (Java 7+) to simplify cleanup.  
* Return a lightweight response object (`HttpResponse`) that includes status code, headers, and body for richer semantics.

### Future Enhancements
* **Asynchronous calls** – Wrap in `CompletableFuture` or use a thread pool.  
* **Redirect handling** – Follow redirects automatically or expose configuration.  
* **Retry logic** – Allow configurable retries for transient failures.  
* **Logging** – Optional debug logging of requests and responses.

In summary, `HttpCallUtil` achieves its goal of providing a quick, dependency‑free way to make HTTP calls, but it would benefit from a cleaner, more robust implementation that addresses the above concerns.

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

import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URL;

public class HttpCallUtil {

	public static String invokeGetUrl(String url) throws Exception {

		String agent = "Mozilla/4.0";

		HttpURLConnection conn = null;
		DataOutputStream output = null;
		DataInputStream in = null;
		BufferedReader is = null;
		try {
			URL postURL = new URL(url);
			conn = (HttpURLConnection) postURL.openConnection();

			// Set connection parameters. We need to perform input and output,
			conn.setDoInput(true);
			conn.setDoOutput(true);

			// Set the content type we are POSTing. We impersonate it as
			// encoded form data
			conn.setRequestProperty("User-Agent", agent);

			conn.setRequestMethod("GET");

			// get the output stream to POST to.
			output = new DataOutputStream(conn.getOutputStream());

			StringBuffer responseText = new StringBuffer();
			in = new DataInputStream(conn.getInputStream());
			int rc = conn.getResponseCode();
			if (rc != -1) {
				is = new BufferedReader(new InputStreamReader(conn
						.getInputStream()));
				String _line = null;
				while (((_line = is.readLine()) != null)) {
					responseText.append(_line);
				}

			}

			return responseText.toString();

		} finally {

			if (is != null) {
				try {
					is.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (in != null) {
				try {
					in.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (output != null) {
				try {
					output.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

		}

	}

	public static String invokePostUrl(String url, String encodedQueryString)
			throws Exception {

		String agent = "Mozilla/4.0";

		HttpURLConnection conn = null;
		DataOutputStream output = null;
		DataInputStream in = null;
		BufferedReader is = null;
		try {
			URL postURL = new URL(url);
			conn = (HttpURLConnection) postURL.openConnection();

			// Set connection parameters. We need to perform input and output,
			// so set both as true.
			conn.setDoInput(true);
			conn.setDoOutput(true);

			// Set the content type we are POSTing. We impersonate it as
			// encoded form data
			conn.setRequestProperty("Content-Type",
					"application/x-www-form-urlencoded");
			conn.setRequestProperty("User-Agent", agent);

			conn.setRequestProperty("Content-Length", String
					.valueOf(encodedQueryString.length()));
			conn.setRequestMethod("POST");

			// get the output stream to POST to.
			output = new DataOutputStream(conn.getOutputStream());
			output.writeBytes(encodedQueryString);
			output.flush();

			StringBuffer responseText = new StringBuffer();
			in = new DataInputStream(conn.getInputStream());
			int rc = conn.getResponseCode();
			if (rc != -1) {
				is = new BufferedReader(new InputStreamReader(conn
						.getInputStream()));
				String _line = null;
				while (((_line = is.readLine()) != null)) {
					responseText.append(_line);
				}

			}

			return responseText.toString();

		} finally {

			if (is != null) {
				try {
					is.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (in != null) {
				try {
					in.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (output != null) {
				try {
					output.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

		}

	}

}



```
