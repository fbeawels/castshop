# EasySSLProtocolSocketFactory.java

## Review

## 1. Summary

**Purpose**  
`EasySSLProtocolSocketFactory` is a custom socket factory for Apache HttpClient that produces SSL sockets capable of accepting *self‑signed* certificates. It is intended as a quick‑and‑dirty solution for development or internal testing environments where a full PKI is not available.

**Key components**

| Class | Responsibility |
|-------|----------------|
| `EasySSLProtocolSocketFactory` | Implements `SecureProtocolSocketFactory`. Delegates socket creation to a lazily‑created `SSLContext` that uses a permissive trust manager. |
| `EasyX509TrustManager` | A thin wrapper around the JRE’s standard `X509TrustManager`. Overrides `checkServerTrusted` to accept a single self‑signed certificate while delegating all other cases to the default manager. |

**Design patterns / libraries**

* Uses the **Factory** pattern (via the `SecureProtocolSocketFactory` interface).
* Relies on the **TrustManager** pattern for SSL certificate validation.
* Uses Apache Commons HttpClient (3.x) and Commons Logging.

---

## 2. Detailed Description

### Overall Flow

1. **Lazy SSLContext Creation**  
   `getSSLContext()` checks if `sslcontext` is `null` and, if so, calls `createEasySSLContext()` which:
   * Obtains an `SSLContext` for the protocol `"SSL"`.
   * Instantiates a `TrustManager[]` containing a single `EasyX509TrustManager`.
   * Initializes the context with this trust manager.

2. **Socket Creation**  
   Every public `createSocket` method simply forwards to `getSSLContext().getSocketFactory()` with the appropriate parameters.  
   * The overload that accepts `HttpConnectionParams` handles connection timeouts manually to work around older JREs that lack native support.

3. **Trust Decision**  
   `EasyX509TrustManager`:
   * Delegates client certificate checks to the standard manager.
   * For server certs, if **exactly one** certificate is presented, it only checks the validity dates and silently accepts it.
   * If more than one cert is presented (e.g., a certificate chain), it falls back to the standard manager, which will enforce proper chain validation.

### Assumptions & Constraints

* The code assumes the default trust store will be used (`keystore` is `null`).
* It targets **Apache HttpClient 3.x**, which is now EOL; newer projects should use HttpClient 4.x or Java’s `HttpClient`.
* TLS protocol defaults to the JRE’s “SSL” algorithm, which may expose legacy protocols (SSLv3, TLSv1.0, TLSv1.1) on modern Java runtimes.
* The class is *not* designed for production use; it bypasses important security checks.

### Architecture & Design Choices

* **Lazy initialization** of the SSL context keeps the factory lightweight but is not thread‑safe (though the cost of double‑instantiation is minor).
* **Explicit timeout handling** in `createSocket` reflects historical JRE limitations; modern code would delegate to the socket’s own timeout APIs.
* **TrustManager wrapping** keeps the code minimal while allowing selective relaxation of server certificate validation.

---

## 3. Functions/Methods

### `EasySSLProtocolSocketFactory`

| Method | Signature | Purpose | Notes |
|--------|-----------|---------|-------|
| `createSocket(String, int, InetAddress, int)` | `Socket` | Delegates to the SSL socket factory to create a socket bound to a local address/port. | Uses default SSL context. |
| `createSocket(String, int)` | `Socket` | Standard hostname/port socket creation. | Delegates to SSL context. |
| `createSocket(Socket, String, int, boolean)` | `Socket` | Wraps an existing socket into an SSL socket. | Delegates to SSL context. |
| `createSocket(String, int, InetAddress, int, HttpConnectionParams)` | `Socket` | Creates a socket with a connection timeout. | Implements custom timeout logic for older JREs. |
| `equals(Object)` | `boolean` | Compares against the same class. | Simple implementation; not necessary for most uses. |
| `hashCode()` | `int` | Returns the class hash code. | Consistent with `equals`. |
| `getSSLContext()` | `SSLContext` | Lazily creates (or returns cached) SSLContext. | Not synchronized; potential double‑init. |
| `createEasySSLContext()` | `SSLContext` (static) | Instantiates SSLContext with `EasyX509TrustManager`. | Wraps all exceptions into `HttpClientError`. |

### `EasyX509TrustManager`

| Method | Signature | Purpose | Notes |
|--------|-----------|---------|-------|
| `EasyX509TrustManager(KeyStore)` | Constructor | Initializes with a standard SunX509 `TrustManager`. | Passes `null` to use default keystore. |
| `checkClientTrusted(X509Certificate[], String)` | `void` | Delegates to the standard manager. | No custom logic. |
| `checkServerTrusted(X509Certificate[], String)` | `void` | Accepts a single self‑signed cert; otherwise delegates. | Swallows `CertificateException` via `printStackTrace()` – highly insecure. |
| `getAcceptedIssuers()` | `X509Certificate[]` | Delegates to the standard manager. | No custom logic. |

---

## 4. Dependencies

| Library | Version | Role |
|---------|---------|------|
| `org.apache.commons.httpclient` | 3.x | Provides `SecureProtocolSocketFactory`, `HttpConnectionParams`, `ConnectTimeoutException`, `HttpClientError`. |
| `org.apache.commons.logging` | 1.x | Logging abstraction used by the factory. |
| Java SE | 1.5+ | JDK APIs (`SSLContext`, `TrustManager`, `X509Certificate`, etc.). |

*All dependencies are third‑party except for the Java SE runtime.*  
*The code is platform‑agnostic but is tightly coupled to the old HttpClient API.*

---

## 5. Additional Notes

### Security Issues

| Issue | Impact | Recommendation |
|-------|--------|----------------|
| **Unconditional acceptance of single self‑signed certs** | Bypasses trust chain validation, making the client vulnerable to MITM attacks. | Replace the permissive logic with a whitelist or proper PKI. |
| **Swallowing `CertificateException`** | An invalid cert is silently accepted. | Re‑throw the exception or at least log it as an error. |
| **Use of protocol “SSL”** | May enable legacy, insecure protocols (SSLv3, TLSv1.0). | Use `"TLS"` or a specific TLS version (`TLSv1.2`, `TLSv1.3`). |
| **No thread‑safety on `sslcontext`** | Rare double‑initialization but not harmful; still can expose race‑conditions if other mutable state were added. | Mark `sslcontext` as `volatile` or synchronize the getter. |
| **Deprecated HttpClient 3.x** | The library is unmaintained and lacks modern features (e.g., HTTP/2). | Migrate to HttpClient 4.x+ or Java’s built‑in `HttpClient`. |

### Edge Cases & Limitations

* **Multiple‑cert chains**: If a server presents a chain longer than one cert, the trust manager defers to the default manager, which will reject a self‑signed chain unless the CA is already trusted.
* **Timeout handling**: The custom timeout logic only covers the connect phase; read/write timeouts are not managed.
* **Logging**: Errors in `createEasySSLContext` are logged but then wrapped in a generic `HttpClientError`; callers may lose the original exception context.

### Potential Enhancements

1. **Configurable Trust Strategy** – allow callers to supply a custom `TrustManager` or a set of trusted certificates instead of the hard‑coded permissive logic.
2. **TLS Version Control** – expose a configuration option for the SSL/TLS protocol (e.g., `"TLSv1.2"`).
3. **Better Exception Handling** – propagate `CertificateException` rather than swallowing it.
4. **Thread‑Safety** – make `sslcontext` `volatile` or lazily initialize it using `Holder` idiom.
5. **Modern HTTP Client** – rewrite the factory for Apache HttpClient 4.x (`HttpClientBuilder`, `SSLConnectionSocketFactory`) or Java 11+ `HttpClient`.
6. **Unit Tests** – add tests to verify behavior with self‑signed certs, proper rejection of invalid certs, and timeout handling.

---

### Verdict

The code is a pragmatic, short‑term solution for environments that cannot rely on a PKI. It demonstrates how to inject a custom `TrustManager` into an `SSLContext` and expose it via `SecureProtocolSocketFactory`. However, the implementation is **inherently insecure** and **outdated**. For any production or even semi‑production use, the trust‑accepting logic must be replaced with a proper certificate validation strategy, and the codebase should be migrated to a supported HTTP client library.

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
package com.salesmanager.core.service.common.impl;

/**
 *
 * $HeadURL$
 * $Revision$
 * $Date$
 * 
 * ====================================================================
 *
 *  Licensed to the Apache Software Foundation (ASF) under one or more
 *  contributor license agreements.  See the NOTICE file distributed with
 *  this work for additional information regarding copyright ownership.
 *  The ASF licenses this file to You under the Apache License, Version 2.0
 *  (the "License"); you may not use this file except in compliance with
 *  the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 * ====================================================================
 *
 * This software consists of voluntary contributions made by many
 * individuals on behalf of the Apache Software Foundation.  For more
 * information on the Apache Software Foundation, please see
 * <http://www.apache.org/>.
 *
 */

import java.io.IOException;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.net.SocketAddress;
import java.net.UnknownHostException;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;

import javax.net.SocketFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManager;
import javax.net.ssl.TrustManagerFactory;
import javax.net.ssl.X509TrustManager;

import org.apache.commons.httpclient.ConnectTimeoutException;
import org.apache.commons.httpclient.HttpClientError;
import org.apache.commons.httpclient.params.HttpConnectionParams;
import org.apache.commons.httpclient.protocol.SecureProtocolSocketFactory;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

/**
 * <p>
 * EasySSLProtocolSocketFactory can be used to creats SSL {@link Socket}s that
 * accept self-signed certificates.
 * </p>
 * <p>
 * This socket factory SHOULD NOT be used for productive systems due to security
 * reasons, unless it is a concious decision and you are perfectly aware of
 * security implications of accepting self-signed certificates
 * </p>
 * 
 * <p>
 * Example of using custom protocol socket factory for a specific host:
 * 
 * <pre>
 * Protocol easyhttps = new Protocol(&quot;https&quot;, new EasySSLProtocolSocketFactory(),
 * 		443);
 * 
 * HttpClient client = new HttpClient();
 * client.getHostConfiguration().setHost(&quot;localhost&quot;, 443, easyhttps);
 * // use relative url only
 * GetMethod httpget = new GetMethod(&quot;/&quot;);
 * client.executeMethod(httpget);
 * </pre>
 * 
 * </p>
 * <p>
 * Example of using custom protocol socket factory per default instead of the
 * standard one:
 * 
 * <pre>
 * Protocol easyhttps = new Protocol(&quot;https&quot;, new EasySSLProtocolSocketFactory(),
 * 		443);
 * Protocol.registerProtocol(&quot;https&quot;, easyhttps);
 * 
 * HttpClient client = new HttpClient();
 * GetMethod httpget = new GetMethod(&quot;https://localhost/&quot;);
 * client.executeMethod(httpget);
 * </pre>
 * 
 * </p>
 * 
 * @author <a href="mailto:oleg -at- ural.ru">Oleg Kalnichevski</a>
 * 
 *         <p>
 *         DISCLAIMER: HttpClient developers DO NOT actively support this
 *         component. The component is provided as a reference material, which
 *         may be inappropriate for use without additional customization.
 *         </p>
 */

public class EasySSLProtocolSocketFactory implements
		SecureProtocolSocketFactory {

	/** Log object for this class. */
	private static final Log LOG = LogFactory
			.getLog(EasySSLProtocolSocketFactory.class);

	private SSLContext sslcontext = null;

	/**
	 * Constructor for EasySSLProtocolSocketFactory.
	 */
	public EasySSLProtocolSocketFactory() {
		super();
	}

	private static SSLContext createEasySSLContext() {
		try {
			SSLContext context = SSLContext.getInstance("SSL");
			context.init(null, new TrustManager[] { new EasyX509TrustManager(
					null) }, null);
			return context;
		} catch (Exception e) {
			LOG.error(e.getMessage(), e);
			throw new HttpClientError(e.toString());
		}
	}

	private SSLContext getSSLContext() {
		if (this.sslcontext == null) {
			this.sslcontext = createEasySSLContext();
		}
		return this.sslcontext;
	}

	/**
	 * @see SecureProtocolSocketFactory#createSocket(java.lang.String,int,java.net.InetAddress,int)
	 */
	public Socket createSocket(String host, int port, InetAddress clientHost,
			int clientPort) throws IOException, UnknownHostException {

		return getSSLContext().getSocketFactory().createSocket(host, port,
				clientHost, clientPort);
	}

	/**
	 * Attempts to get a new socket connection to the given host within the
	 * given time limit.
	 * <p>
	 * To circumvent the limitations of older JREs that do not support connect
	 * timeout a controller thread is executed. The controller thread attempts
	 * to create a new socket within the given limit of time. If socket
	 * constructor does not return until the timeout expires, the controller
	 * terminates and throws an {@link ConnectTimeoutException}
	 * </p>
	 * 
	 * @param host
	 *            the host name/IP
	 * @param port
	 *            the port on the host
	 * @param clientHost
	 *            the local host name/IP to bind the socket to
	 * @param clientPort
	 *            the port on the local machine
	 * @param params
	 *            {@link HttpConnectionParams Http connection parameters}
	 * 
	 * @return Socket a new socket
	 * 
	 * @throws IOException
	 *             if an I/O error occurs while creating the socket
	 * @throws UnknownHostException
	 *             if the IP address of the host cannot be determined
	 */
	public Socket createSocket(final String host, final int port,
			final InetAddress localAddress, final int localPort,
			final HttpConnectionParams params) throws IOException,
			UnknownHostException, ConnectTimeoutException {
		if (params == null) {
			throw new IllegalArgumentException("Parameters may not be null");
		}
		int timeout = params.getConnectionTimeout();
		SocketFactory socketfactory = getSSLContext().getSocketFactory();
		if (timeout == 0) {
			return socketfactory.createSocket(host, port, localAddress,
					localPort);
		} else {
			Socket socket = socketfactory.createSocket();
			SocketAddress localaddr = new InetSocketAddress(localAddress,
					localPort);
			SocketAddress remoteaddr = new InetSocketAddress(host, port);
			socket.bind(localaddr);
			socket.connect(remoteaddr, timeout);
			return socket;
		}
	}

	/**
	 * @see SecureProtocolSocketFactory#createSocket(java.lang.String,int)
	 */
	public Socket createSocket(String host, int port) throws IOException,
			UnknownHostException {
		return getSSLContext().getSocketFactory().createSocket(host, port);
	}

	/**
	 * @see SecureProtocolSocketFactory#createSocket(java.net.Socket,java.lang.String,int,boolean)
	 */
	public Socket createSocket(Socket socket, String host, int port,
			boolean autoClose) throws IOException, UnknownHostException {
		return getSSLContext().getSocketFactory().createSocket(socket, host,
				port, autoClose);
	}

	public boolean equals(Object obj) {
		return ((obj != null) && obj.getClass().equals(
				EasySSLProtocolSocketFactory.class));
	}

	public int hashCode() {
		return EasySSLProtocolSocketFactory.class.hashCode();
	}

}

class EasyX509TrustManager implements X509TrustManager {

	private X509TrustManager standardTrustManager = null;

	public EasyX509TrustManager(KeyStore keystore)
			throws NoSuchAlgorithmException, KeyStoreException {

		super();

		TrustManagerFactory factory = TrustManagerFactory
				.getInstance("SunX509");
		factory.init(keystore);

		TrustManager[] trustmanagers = factory.getTrustManagers();

		if (trustmanagers.length == 0) {
			throw new NoSuchAlgorithmException(
					"SunX509 trust manager not supported");
		}

		this.standardTrustManager = (X509TrustManager) trustmanagers[0];
	}

	public void checkClientTrusted(X509Certificate[] certificates, String string)
			throws CertificateException {

		this.standardTrustManager.checkClientTrusted(certificates, string);
	}

	public void checkServerTrusted(X509Certificate[] certificates, String string)
			throws CertificateException {

		if ((certificates != null) && (certificates.length == 1)) {
			X509Certificate certificate = certificates[0];

			try {
				certificate.checkValidity();
			} catch (CertificateException e) {
				e.printStackTrace();
			}
		} else {
			this.standardTrustManager.checkServerTrusted(certificates, string);
		}
	}

	public X509Certificate[] getAcceptedIssuers() {
		return this.standardTrustManager.getAcceptedIssuers();
	}
}



```
