# UploadListener.java

## Review

## 1. Summary  

The `UploadListener` class is a small utility that tracks the progress of an HTTP file upload.  It implements the `OutputStreamListener` interface (not shown) and is meant to be invoked by an upload framework that streams data to the server.  Each time a chunk of data is written to the output stream, the listener updates session‑level information (`uploadInfo`) so that the front‑end can display progress, total bytes, elapsed time, etc.  

Key responsibilities  
* **Track progress** – keep count of total bytes read and the number of files uploaded.  
* **Provide timing information** – compute elapsed time since the first `start()` call.  
* **Delay optionally** – inject a configurable pause (`debugDelay`) after each read to make debugging easier.  
* **Session state** – store a lightweight `UploadInfo` bean in the HTTP session so that the UI can read it.  

The class is straightforward, but its design is tightly coupled to the Servlet API (HttpServletRequest/Session) and it relies on a custom `UploadInfo` bean (not included here).

---

## 2. Detailed Description  

### Core Components  
| Component | Purpose |
|-----------|---------|
| `HttpServletRequest request` | Access to the HTTP session for persisting upload progress. |
| `long delay` | Millisecond pause injected after each `bytesRead` call (debug helper). |
| `long startTime` | Timestamp when the first upload started; used for elapsed‑time calculations. |
| `int totalToRead` | Content‑length of the entire request (bytes to be read). |
| `int totalBytesRead` | Cumulative bytes that have been read so far. |
| `int totalFiles` | Counter of how many file uploads have been initiated in the current request. |
| `UploadInfo` | Plain‑old Java object (POJO) that holds the progress snapshot; stored in the session. |

### Flow of Execution  

1. **Construction** – The listener is created with a `HttpServletRequest` and a `debugDelay`.  
   * `totalToRead` is read from the request (`Content‑Length`).  
   * `startTime` is initialized to the current system time.

2. **`start()`** – Called when a new file upload begins.  
   * Increments `totalFiles`.  
   * Calls `updateUploadInfo("start")` to persist an initial snapshot.

3. **`bytesRead(int)`** – Called each time the output stream receives a chunk.  
   * Adds the chunk size to `totalBytesRead`.  
   * Calls `updateUploadInfo("progress")`.  
   * Sleeps for `delay` ms if a non‑zero value was supplied.

4. **`error(String)`** – Invoked on failure.  
   * Persists the state with a status of `"error"`.  

5. **`done()`** – Called when the upload finishes successfully.  
   * Persists the final state with status `"done"`.

6. **`updateUploadInfo(String)`** – Creates a new `UploadInfo` instance containing:
   * Number of files processed (`totalFiles`)
   * Total bytes expected (`totalToRead`)
   * Bytes already read (`totalBytesRead`)
   * Elapsed seconds (`delta`)
   * Current status (`status`)

   Then stores this object in the current HTTP session under the key `"uploadInfo"`.

### Assumptions & Constraints  

* The request has a **fixed `Content-Length`**. If the client sends chunked transfer encoding, `request.getContentLength()` returns `-1` and the progress logic will be inaccurate.  
* Progress is only available per **HTTP session**; concurrent uploads in the same session will overwrite each other.  
* No concurrency control is present – if multiple uploads are processed simultaneously in the same session, data races can occur.  
* The listener relies on a **mutable session attribute** (`uploadInfo`) that is overwritten on each call; consumers must retrieve and interpret it immediately.

### Architecture & Design Choices  

The class follows a **callback/listener** pattern: the actual I/O component calls these methods to report progress.  It is *stateful* (keeps counters) and *session‑centric*.  While this works for simple scenarios, it couples the listener to the Servlet API, making it hard to test in isolation and limiting reuse outside a web container.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `UploadListener(HttpServletRequest, long)` | Constructor | Initialize listener; capture request metadata. | `request`, `debugDelay` | None | Sets `request`, `delay`, `totalToRead`, `startTime` |
| `start()` | `void` | Signals start of a file upload. | None | None | Increments `totalFiles`; updates session |
| `bytesRead(int)` | `void` | Called on each data chunk; updates counters. | `bytesRead` | None | Adds to `totalBytesRead`; updates session; sleeps `delay` |
| `error(String)` | `void` | Called when an error occurs. | `message` | None | Updates session status `"error"` |
| `done()` | `void` | Called when upload finishes. | None | None | Updates session status `"done"` |
| `getDelta()` | `private long` | Compute seconds elapsed since first start. | None | Delta seconds | None |
| `updateUploadInfo(String)` | `private void` | Create and store `UploadInfo` snapshot. | `status` | None | Sets session attribute `"uploadInfo"` |

*Reusable/utility methods*: `getDelta()` and `updateUploadInfo()` are internal helpers used by the public callbacks.  No public utilities are exposed beyond the interface implementation.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard (Servlet API) | Required for request/session access. |
| `javax.servlet.http.HttpSession` | Standard | Used implicitly via `request.getSession()`. |
| `OutputStreamListener` | Custom | Interface this class implements (not provided). |
| `UploadInfo` | Custom POJO | Stores progress snapshot; assumed to have an appropriate constructor. |

No external libraries are used; the code relies solely on the Java EE Servlet API and application‑specific classes.

---

## 5. Additional Notes  

### Edge Cases & Limitations  

1. **Chunked Transfer Encoding** – `Content-Length` may be `-1`, leading to a misleading `totalToRead`.  
2. **Concurrent Uploads** – The session attribute is overwritten; simultaneous uploads will corrupt progress data.  
3. **Large File Support** – `int` is used for byte counters; uploads >2 GB will overflow.  
4. **Thread‑Safety** – The listener is not thread‑safe; shared state could be corrupted if the underlying upload framework uses multiple threads per request.  
5. **Error Handling** – The `error` method ignores the message; it might be useful to log or store it for diagnostics.  
6. **Debug Delay** – The `Thread.sleep()` call blocks the request thread; this is fine for debugging but should be disabled in production.  

### Potential Enhancements  

* **Use `long` for byte counters** to support very large uploads.  
* **Introduce synchronization** or redesign to be stateless per upload (e.g., pass an `UploadInfo` object into the constructor).  
* **Support chunked uploads** by removing reliance on `Content-Length` or by using `ServletInputStream` to determine bytes read.  
* **Persist progress to a more robust store** (e.g., database or in‑memory map keyed by session ID) to handle clustering or load‑balanced environments.  
* **Expose the error message** in `UploadInfo` for better UI feedback.  
* **Provide a configurable `ProgressListener` interface** that decouples from `HttpServletRequest`, improving testability.  
* **Use `ScheduledExecutorService`** instead of `Thread.sleep()` for the debug delay to avoid blocking the request thread.  

Overall, `UploadListener` is a lightweight, purpose‑built component that works well in a simple, single‑threaded servlet environment but would require several refactorings for production‑grade, scalable deployments.

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
package com.salesmanager.central.util.upload;

import javax.servlet.http.HttpServletRequest;

/**
 * Created by IntelliJ IDEA.
 * 
 * @author Original : plosson on 06-janv.-2006 15:05:44 - Last modified by
 *         $Author: vde $ on $Date: 2004/11/26 22:43:57 $
 * @version 1.0 - Rev. $Revision: 1.2 $
 */
public class UploadListener implements OutputStreamListener {
	private HttpServletRequest request;
	private long delay = 0;
	private long startTime = 0;
	private int totalToRead = 0;
	private int totalBytesRead = 0;
	private int totalFiles = -1;

	public UploadListener(HttpServletRequest request, long debugDelay) {
		this.request = request;
		this.delay = debugDelay;
		totalToRead = request.getContentLength();
		this.startTime = System.currentTimeMillis();
	}

	public void start() {
		totalFiles++;
		updateUploadInfo("start");
	}

	public void bytesRead(int bytesRead) {
		totalBytesRead = totalBytesRead + bytesRead;
		updateUploadInfo("progress");

		try {
			Thread.sleep(delay);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	public void error(String message) {
		updateUploadInfo("error");
	}

	public void done() {
		updateUploadInfo("done");
	}

	private long getDelta() {
		return (System.currentTimeMillis() - startTime) / 1000;
	}

	private void updateUploadInfo(String status) {
		long delta = (System.currentTimeMillis() - startTime) / 1000;
		request.getSession().setAttribute(
				"uploadInfo",
				new UploadInfo(totalFiles, totalToRead, totalBytesRead, delta,
						status));
	}

}



```
