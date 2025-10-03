# FacebookIntegrationFactory.java

## Review

## 1. Summary  

The file **`FacebookIntegrationFactory.java`** is a helper that:

* Creates or retrieves a `FacebookUser` object from the HTTP session.
* Parses the Facebook signed request (passed as a request parameter) to extract OAuth data and verify the request signature.
* Builds a Facebook OAuth authorization URL for a given page.

Key components  

| Component | Role |
|-----------|------|
| `FacebookUser` | Holds the Facebook credentials, OAuth token, and state flags (e.g., `likedPage`, `authorized`). |
| `Page` | Entity that stores the Facebook page configuration (client id, secret, key, URL). |
| `getFacebookUser()` | Session‑based initialization and signed‑request handling. |
| `getAuthorizationUrl()` | Generates the user‑agent OAuth URL with required permissions. |

The code uses a handful of third‑party libraries:

* Apache Commons Lang (`StringUtils`) – string utilities.
* Apache Commons Codec – Base64 encoding/decoding.
* Jackson – JSON deserialization.
* `restfb` – Facebook Java SDK (only the `FacebookClient` import is present, but not used).
* Log4j – logging.

No particular design pattern is evident; the class is essentially a static façade.

---

## 2. Detailed Description  

### 2.1 Flow of execution  

1. **Session handling**  
   * `getFacebookUser()` first pulls a `FacebookUser` from the session using a fixed key.  
   * If none exists, a new one is created, populated with data from the `Page` object, and stored in the session.

2. **Error handling**  
   * If the request contains an `error_reason` parameter, the user is reset (token cleared, authorized flag cleared).

3. **Signed request parsing**  
   * When `signed_request` is present, the code:
     * Decodes the base64‑encoded JSON payload.
     * Deserialises it into a `Map<String,String>`.
     * Extracts `oauth_token` and stores it on the `FacebookUser`.
     * Looks for a nested `page` map to set the `likesPage` flag.
   * Independently verifies the signature:
     * Splits the signed request into the encoded signature and payload.
     * Recomputes an HMAC‑SHA256 hash using a hard‑coded secret key.
     * Compares the recomputed hash with the original signature; sets `authorized` if they match.

4. **Return**  
   * The populated `FacebookUser` is returned.

5. **Authorization URL generation**  
   * `getAuthorizationUrl()` uses the `FacebookUser` client ID, the page’s redirect URI, and the hard‑coded permission set to build the OAuth URL.

### 2.2 Assumptions & Constraints  

| Assumption | Implication |
|------------|-------------|
| `Page` properties (2,4,5,6) contain correct values. | No validation – errors propagate silently. |
| The signed request is always correctly formatted. | Parsing errors are swallowed into a generic `Exception`. |
| The Facebook app secret is hard‑coded in the source. | Security risk; must be externalised. |
| The application runs in a servlet container that provides a per‑request `HttpServletRequest`. | Not thread‑safe if the factory is used elsewhere. |
| `FacebookUser` is serialisable and safe to store in the HTTP session. | No session‑replication handling shown. |

### 2.3 Architecture & Design Choices  

* **Static façade**: All methods are static – easy to call but hard to test/mock.  
* **Session‑centric**: The factory is tightly coupled to servlet sessions, limiting reuse in non‑web contexts.  
* **Hard‑coded strings**: Permissions and secrets are inline, reducing flexibility.  
* **No separation of concerns**: Parsing, verification, and session handling are intermingled in a single method.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `public static FacebookUser getFacebookUser(HttpServletRequest request, Page page)` | Retrieves or creates a `FacebookUser` from the HTTP session and populates it with data from the signed request. | `request` – current HTTP request. <br>`page` – Facebook page configuration. | `FacebookUser` instance – fully populated. | Stores or updates `FacebookUser` in the session; logs debug information. |
| `public static String getAuthorizationUrl(FacebookUser user, Page page) throws Exception` | Builds the OAuth URL that the front‑end should redirect the user to. | `user` – contains client ID.<br>`page` – contains redirect URI. | String – the full URL. | None. |

### 3.1 Reusable or Utility Methods  

None are defined; the logic is embedded directly inside the two public methods. Extracting the signed request parsing and signature verification into private helpers would improve readability and testability.

---

## 4. Dependencies  

| Library | Role | Standard / 3rd‑party | Remarks |
|---------|------|-----------------------|---------|
| `javax.servlet.http.*` | Servlet API | Standard | Required for session handling. |
| `org.apache.commons.codec.binary.Base64` | Base64 encoder/decoder | 3rd‑party | Legacy (Apache Commons Codec). |
| `org.apache.commons.lang.StringUtils` | String utilities | 3rd‑party | Could replace with Java 8 `String` methods. |
| `org.codehaus.jackson.map.ObjectMapper` | JSON deserialization | 3rd‑party (Jackson 1.x) | Jackson 1 is deprecated; consider Jackson 2. |
| `org.apache.log4j.Logger` | Logging | 3rd‑party | Log4j 1 is end‑of‑life; upgrade to Log4j 2 or SLF4J. |
| `com.restfb.FacebookClient` | Facebook SDK client | 3rd‑party | Imported but never used. |
| `com.salesmanager.core.entity.reference.Page` | Domain entity | Internal | Holds configuration for the Facebook page. |
| `com.salesmanager.core.util.www.integration.fb.FacebookUser` | Domain DTO | Internal | Holds OAuth state. |

**Platform specific**: Runs in a Java EE servlet container. The use of `HttpServletRequest` and `HttpSession` ties the code to a web context.

---

## 5. Additional Notes & Recommendations  

### 5.1 Security Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| Hard‑coded App Secret (`"93b47625ec3dcc4172fda796899ae42d"`) | Exposure if the repo is shared; credential leaks. | Externalise the secret (environment variable, JNDI, or secure vault). |
| Base64 decoding with `new Base64(true)` – the `true` flag allows non‑standard decoding. | Potential decoding bugs. | Use standard Base64 (`java.util.Base64.getDecoder()`) or Apache Commons `Base64.decodeBase64()`. |
| Signature verification uses `Mac.getInstance("HMACSHA256")` but the key is derived from the app secret directly. | Should use the Facebook app secret as the HMAC key. | Confirm that the key used matches Facebook’s spec; ensure correct UTF‑8 encoding. |
| No replay‑attack protection: the signed request is not timestamp‑checked. | Could be reused maliciously. | Verify the `issued_at` or `expires` field. |

### 5.2 Error Handling  

* The code catches generic `Exception` and simply logs it.  
* Returning a partially populated `FacebookUser` on failure may lead to subtle bugs downstream.  
* Recommendation: Throw a custom checked exception (`FacebookIntegrationException`) and let callers decide how to handle it.  

### 5.3 Code Quality  

* **Magic strings** (`"facebook.user.client"`, permission list, index constants).  
  * Extract to constants.  
* **Unnecessary imports** (`FacebookClient` is unused).  
* **Hard‑coded index checks** (`signed_request.indexOf(".")`).  
  * Use `String.split("\\.", 2)` for clarity.  
* **Logging** – debug messages are verbose and may leak sensitive data (`oauth_token`).  
  * Mask tokens or avoid logging them.  
* **Iteration over Map** – use generics (`Map<String, Object>`) and avoid raw types.  
* **Base64 handling** – the code decodes `encPayload` and then re‑decodes the raw payload for signature verification; this could be simplified.  
* **Thread safety** – static methods are fine because they only use request/session data, but if used in other contexts, concurrency issues may arise.  

### 5.4 Maintainability  

* Splitting the logic into smaller, focused private methods (`parseSignedRequest`, `verifySignature`, `extractPageLikeStatus`) would make the code easier to read and unit‑test.  
* Replace the deprecated Jackson 1.x `ObjectMapper` with Jackson 2 (`com.fasterxml.jackson.databind.ObjectMapper`).  
* Move the permissions to a configurable set (e.g., `facebook.permissions`).  
* Consider using a dedicated Facebook SDK (e.g., `restfb`) to handle OAuth flows and signed request parsing, reducing custom implementation and potential bugs.

### 5.5 Potential Enhancements  

1. **Unit tests** for the parsing and verification logic (mock `HttpServletRequest`/`HttpSession`).  
2. **Support for `client_credentials` flow** if needed.  
3. **Automatic refresh of expired tokens** – the current code never checks or refreshes the token.  
4. **Support for page access tokens** – if the integration needs to post to a page.  
5. **Rate‑limit handling** – adding retries with exponential backoff for Graph API calls.  
6. **Better session cleanup** – remove the user object on logout or session expiry.  

---

### Verdict  

The class achieves its basic goal—parsing Facebook signed requests and generating an OAuth URL—but does so in a brittle, hard‑coded, and insecure manner. Refactoring to decouple concerns, externalise secrets, improve error handling, and adopt modern libraries would significantly raise the quality, security, and maintainability of the code.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.util.www.integration.fb;

import java.net.URLEncoder;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.codec.binary.Base64;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.codehaus.jackson.map.ObjectMapper; 


import com.restfb.FacebookClient;
import com.salesmanager.core.entity.reference.Page;



public class FacebookIntegrationFactory {
	
	private static Logger log = Logger.getLogger(FacebookIntegrationFactory.class);
	private static final String FACEBOOK_USER_CLIENT = "facebook.user.client";
	
	private static final String perms = "publish_stream,email,user_online_presence,user_likes";
	
	/**
	 * - Check if exist in http session
	 * - Check if authenticated
	 * @param request
	 * @param page
	 */
	public static FacebookUser getFacebookUser(HttpServletRequest request, Page page) {
		
		FacebookUser user = new FacebookUser();
		try {
			
			HttpSession session = request.getSession();
			user = (FacebookUser)session.getAttribute(FACEBOOK_USER_CLIENT);
			
			if(user==null) {
				
				user = new FacebookUser();
				user.setClientId(page.getProperty2());
				user.setApplicationSecret(page.getProperty4());
				user.setApplicationKey(page.getProperty5());
				session.setAttribute(FACEBOOK_USER_CLIENT, user);
				
			}
			
			String error_reason = request.getParameter("error_reason");
			
			if(!StringUtils.isBlank(error_reason)) {
				user.setOauth_token(null);
				user.setExpires(null);
				user.setAuthorized(false);
				return user;
			}
			
			String signed_request = request.getParameter("signed_request");
			if(!StringUtils.isBlank(signed_request)) {
				
				user.setAuthenticated(true);
					
                if (signed_request == null) 
                    throw new Exception("Invalid signature."); 


                String[] parts = signed_request.split("\\."); 
                if (parts.length != 2) 
                    throw new Exception("Invalid signature."); 


                String encSig = parts[0]; 
                String encPayload = parts[1]; 


                Base64 decoder = new Base64(true); 
                Map<String, String> data; 
                try {
                	

    				
    				String o = new String(decoder.decode(encPayload.getBytes()));
    				
    				o = o.trim();

    				
    				data = new ObjectMapper().readValue(o, HashMap.class);
    				String oauthtoken = data.get("oauth_token");
    				

    			    Iterator entries = data.entrySet().iterator(); 
    				while (entries.hasNext()) {   
    					Entry thisEntry = (Entry) entries.next();   
    					Object key = thisEntry.getKey();   
    					Object value = thisEntry.getValue();
    					log.debug("Got key " + key + " and value " + value);
    					if(key.equals("paage")) {
    						if(value instanceof Map) {
    							Iterator ientries = ((Map)value).entrySet().iterator(); 
    		    				while (ientries.hasNext()) { 
    		    					Entry iEntry = (Entry) ientries.next(); 
    		    					Object ikey = iEntry.getKey();   
    		    					Object ivalue = iEntry.getValue();
    		    					log.debug("Got paage key " + ikey + " and value " + ivalue + " type " + ivalue.getClass().getName());
    		    					if(ikey.equals("liked")) {
    	    								if((Boolean)ivalue==true) {
    	    									log.debug("User likes paage [Boolean]");
        	    								user.setLikesPage(true);
    	    								}
    	    							break;
    		    					}
    		    				}

    						} else {
    							log.debug("value instanceof " + value.getClass().toString());
    						}
    					}
    				}
    				
    				//String expires = data.get("expires");
    				
    				//System.out.println(oauthtoken);
    				//System.out.println(expires);
    				user.setOauth_token(oauthtoken);
    				//user.setExpires(expires);
  

                } catch (Exception e) { 
                    throw new Exception("Failed to parse JSON session.", e); 
                } 

                

                try {
                	

					int idx = signed_request.indexOf(".");
					byte[] sig = new Base64(true).decode(signed_request.substring(0, idx).getBytes());
					String rawpayload = signed_request.substring(idx+1);
					

					SecretKeySpec secretKeySpec = new SecretKeySpec("93b47625ec3dcc4172fda796899ae42d".getBytes(), "HMACSHA256");
					Mac mac2 = Mac.getInstance("HMACSHA256");
					mac2.init(secretKeySpec);
					byte[] mysig = mac2.doFinal(rawpayload.getBytes());


					if (Arrays.equals(mysig, sig)) {
						user.setAuthorized(true);
					}

                    

                } catch (Exception e) { 
                    throw new Exception("Failed to perform crypt operation.", e); 
                } 


				
			}
			
		} catch (Exception e) {
			log.error(e);
		}
		
		return user;
		
	}
	
	public static String getAuthorizationUrl(FacebookUser user, Page page) throws Exception {
		
		//facebook page url
		String url = URLEncoder.encode(page.getProperty6(), "UTF-8");
		
		StringBuffer requestUrl = new StringBuffer();
		requestUrl.append("https://graph.facebook.com/oauth/authorize?type=user_agent");
		requestUrl.append("&client_id=");
		requestUrl.append(user.getClientId());
		requestUrl.append("&redirect_uri=");
		requestUrl.append(url);
		requestUrl.append("&scope=");
		requestUrl.append(perms);
		   

		return requestUrl.toString();
		
	}
	


	
}





```
