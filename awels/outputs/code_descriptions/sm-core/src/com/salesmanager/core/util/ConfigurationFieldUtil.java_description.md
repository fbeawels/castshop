# ConfigurationFieldUtil.java

## Review

## 1. Summary  

**Purpose** – `ConfigurationFieldUtil` is a helper that converts JSON definitions of configuration fields (text, select, radio, checkbox) into a map of custom `Field` objects and vice‑versa. It also constructs a JSON string from a map of field values.

**Key components**  
| Component | Role |
|-----------|------|
| `getMerchantConfigurationKey` / `getMerchantConfigurationKeyLike` | Build predictable keys for configuration storage. |
| `parseFields(String)` | Deserialises a JSON blob that contains a `fields` array into `Map<String, Field>`. |
| `parseFieldsValues(List<String>)` & `parseFieldsValues(String)` | Convert a list or a single JSON string that contains module‑specific field values into `Map<String, List<Field>>`. |
| `buildFieldValuesString(Map<String, List<Field>>)` | Serialises a map of field values back into a JSON string. |

The class relies heavily on **Jackson** (`org.codehaus.jackson.map.ObjectMapper`) for JSON parsing, **Log4j** for diagnostics, and a few custom domain classes (`Field`, `FieldOption`).  

It follows a *utility* pattern: all public members are `static`, there is no state, and it is designed to be used by other layers (e.g., service or DAO) that need to persist or read configuration metadata.

---

## 2. Detailed Description  

### Execution Flow  

1. **Key Generation** – The two `getMerchantConfigurationKey*` methods simply concatenate a constant prefix with the supplied page and module names.  
2. **Deserialisation (`parseFields`)**  
   * The JSON string is parsed into a raw `Map` (`HashMap`).
   * The method expects a top‑level key `"fields"` whose value is a `List`.  
   * Each list element is assumed to be a `LinkedHashMap` containing a single key `"field"` that maps to a map describing the field (`type`, `name`, `label`, optional `values`).  
   * For each field, a `Field` instance is created, its properties set, and added to the return map keyed by `name`.  
   * If the field has a `values` array, each entry is turned into a `FieldOption` and attached to the field.  
3. **Deserialisation of field values** (`parseFieldsValues`) – two overloads.  
   * The first one accepts a list of JSON strings, concatenates them into a single JSON object with a dummy `"paage"` key and a `"modules"` array.  
   * The second one parses a JSON that contains a top‑level `"fields"` key, where each element contains a `"module"` key and a `"values"` array of `{name, value}` objects.  
   * In both cases, a `Field` is created for each `{name, value}` pair, and the resulting list is associated with the module name in the returned map.  
4. **Serialisation (`buildFieldValuesString`)** – builds a JSON string from a map of module‑to‑field‑lists.  
   * For each module it emits `"module":"<name>"` followed by a `"values"` array.  
   * Each field is rendered as `{ "name":"<name>", "value":"<value>" }`.  

### Design & Architecture  

* **Utility‑only** – Stateless and static.  
* **No type safety** – Raw collections and unchecked casts dominate the code.  
* **Manual JSON building** – The builder methods use string concatenation, which is error‑prone and makes the output fragile.  
* **Error handling** – All methods throw `Exception`, a broad catch‑all that forces callers to handle a very generic failure.  
* **Logging** – Only the `parseFields` method logs malformed `isDefault` values; no other method logs issues.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `getMerchantConfigurationKey(String page, String module)` | Build a storage key for a specific page/module. | `page`, `module` | `String` | none |
| `getMerchantConfigurationKeyLike(String page)` | Build a key prefix for all modules on a page. | `page` | `String` | none |
| `parseFields(String fields)` | Parse a JSON string containing an array of field definitions into a map. | `fields` – JSON string | `Map<String, Field>` | creates `Field`/`FieldOption` objects |
| `parseFieldsValues(List<String> stringFields)` | Convert a list of JSON strings into a module‑to‑field‑list map. | `stringFields` – list of JSON strings | `Map<String, List<Field>>` | creates `Field` objects |
| `parseFieldsValues(String fields)` | Parse a single JSON string of module‑values into a map. | `fields` – JSON string | `Map<String, List<Field>>` | creates `Field` objects |
| `buildFieldValuesString(Map<String, List<Field>> fieldValues)` | Serialize a module‑to‑field‑list map into a JSON string. | `fieldValues` | `String` | none |

**Reusable utilities** – None; all logic is tightly coupled to the JSON format.

---

## 4. Dependencies  

| External | Version/Notes | Type |
|----------|---------------|------|
| `org.codehaus.jackson.map.ObjectMapper` | Jackson 1.x | JSON parsing |
| `org.apache.log4j.Logger` | Log4j 1.x | Logging |
| `com.salesmanager.core.constants.ConfigurationConstants` | Custom constants class | Configuration |
| `com.salesmanager.core.entity.system.Field` / `FieldOption` | Custom domain objects | Business model |

All dependencies are third‑party libraries, except for the custom `Field`/`FieldOption` classes.

---

## 5. Additional Notes  

### Strengths  

* Provides a single place for handling configuration field JSON, reducing duplication.  
* Uses Jackson (albeit old 1.x) for JSON parsing, which is robust for the simple schemas involved.  
* Logging is present for at least one error case.

### Weaknesses / Risks  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types and unchecked casts** | Compile‑time warnings, possible `ClassCastException` at runtime. | Add generic types (`Map<String, Object>`, `List<Map<String,Object>>`) and use `ObjectMapper.readValue(..., new TypeReference<Map<String,Object>>() {})`. |
| **Hard‑coded JSON structure** | Fragile against schema changes; no validation. | Create POJOs that match the JSON shape and let Jackson deserialize directly. |
| **Manual JSON string construction** | Bugs (e.g., missing commas, wrong braces). | Use Jackson’s `ObjectNode`/`ArrayNode` or a builder library; or serialize POJOs directly. |
| **Broad `throws Exception`** | Forces callers to handle all exceptions; hides specific failures. | Define custom checked exceptions or return `Optional<Map<...>>` with proper error handling. |
| **Inconsistent JSON keys** – e.g., `"paage":"p"` typo, `"fields"` vs `"modules"`. | Incompatible with clients. | Standardise key names; provide documentation. |
| **No thread safety discussion** – Not an issue as stateless, but raw static `ObjectMapper` might be reused; ensure it's thread‑safe (Jackson 1.x is). | Minor. | Keep `ObjectMapper` as a static final instance. |
| **Logging only in one place** | Incomplete diagnostics. | Add logs for other methods or error paths. |
| **`buildFieldValuesString`** – never emits the outer `"fields"` array and does not correctly delimit modules. | Malformed JSON output. | Wrap the module objects in an array and emit the `"fields"` key; use a JSON library. |

### Suggested Enhancements  

1. **Refactor to POJOs** – Define `FieldDef`, `FieldOptionDef`, `ModuleValues`, `FieldsWrapper` classes and let Jackson handle both directions.  
2. **Centralize JSON handling** – A single `ObjectMapper` instance, possibly configured once.  
3. **Add unit tests** – Verify round‑trip conversion, error handling, and edge cases (empty fields, missing keys).  
4. **Use newer Jackson (2.x)** – Modernize the library; it's backward compatible but has better API and performance.  
5. **Return typed collections** – Replace raw `Map` with `Map<String, Field>` etc.  
6. **Document the expected JSON schema** – In Javadoc, include sample payloads and explain optional fields.  
7. **Improve error reporting** – Throw domain‑specific exceptions with messages that can be surfaced to the UI.  

Overall, the class achieves its core goal but suffers from maintainability and robustness issues. Refactoring towards a type‑safe, library‑centric implementation would greatly improve its quality and future‑proof it against changes in the configuration schema.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Dec 7, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import org.apache.log4j.Logger;
import org.codehaus.jackson.map.ObjectMapper;

import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.entity.system.FieldOption;



/**
 * JSON to String helper class
 * @author Carl Samson
 *
 */
public class ConfigurationFieldUtil {
	
	private static Logger log = Logger.getLogger(ConfigurationFieldUtil.class);
	
	
	
	public static String getMerchantConfigurationKey(String page, String module) {
		
		return new StringBuilder().append(ConfigurationConstants.PAGE_PORTLET_PREFIX).append(page).append("_").append(module).toString();
	}
	
	public static String getMerchantConfigurationKeyLike(String page) {
		
		return new StringBuilder().append(ConfigurationConstants.PAGE_PORTLET_PREFIX).append(page).append("_").toString();
	}
	
	/**
	 * This method de-serializes fields instructions formated as JSON objects
	 * Multiple fields instructions can be de-serialized using this class
	 * The string must be formated as an array of fields
	 * {"fields":[field,field]}
	 * The system currently supports Text, Select, Radio and checkbox fields
	 * Text field:
	 * {"field":{"type":"text","name":"nameOfTheField","label":"Field Label"}}
	 * Select field
	 * {"field":{"type":"select","name":"option",,"label":"Field Label","values":[{"name":"option label","value":"option value","isDefault":"false"},{"name":"option label2","value":"option value2","isDefault":"false"}]}}
	 * Radio field
	 * {"field":{"type":"radio","name":"option","label":"Field Label","values":[{"name":"option label","value":"option value","isDefault":"false"},{"name":"option label2","value":"option value2","isDefault":"false"}]}}
	 * Returns a Map of String[fieldName] and Field field content
	 * 
	 * Full example
	 * {"fields":[{"field":{"type":"text","label":"Invitation type text","name":"invitationType"}},{"field":{"type":"text","label":"Action text","name":"actionText"}},{"field":{"type":"select","label":"Select Action text","name":"selectActionText","values":[{"name":"select1","value":"option value1","isDefault":"false"},{"name":"select2","value":"option value2","isDefault":"false"}]}},{"field":{"type":"radio","label":"Radio Action text","name":"radioActionText","values":[{"name":"radioActionText","value":"yes","isDefault":"false"},{"name":"radioActionText","value":"no","isDefault":"true"}]}},{"field":{"type":"checkbox","label":"Checkbox Action text","name":"checkboxActionText"}}]}
	 * 
	 * Returns Map<String->fieldName,Field>
	 * @author Carl Samson
	 * @param fields
	 * @return
	 * @throws Exception
	 */
	public static Map<String,Field> parseFields(String fields) throws Exception {
		
		Map returnMap = new HashMap();

			Map<String, String> data = new ObjectMapper().readValue(fields, HashMap.class);
			
			if(data!=null) {
				for(Object o: data.keySet()) {

					if(o instanceof String && ((String)o).equals("fields")) {
						// can parse
						
						Object oo = data.get(o);
						if(oo instanceof List) {//List 
							
							for(Object ooo:(List)oo) {

								if(ooo instanceof LinkedHashMap) {
									
									Map m = (Map)ooo;
									
									//get each fields
									Map field = (Map)m.get("field");
									
									//get field type
									String type = (String)field.get("type");
									String name = (String)field.get("name");
									String label = (String)field.get("label");
									
									//System.out.println("This field is of type [" + type + "] and name [" + name + "]");
									
									
									Field f = new Field();
									f.setName(name);
									f.setType(type);
									f.setLabel(label);
									returnMap.put(name, f);
									
									
									List valuesList = (List)field.get("values");
									
									if(valuesList!=null) {
										
										for(Object oooo:valuesList) {
											
											FieldOption fo = new FieldOption();
											
											
											
											Map values = (Map)oooo;
											String valueName = (String)values.get("name");
											String valueValue = (String)values.get("value");
											String defaultOption = (String)values.get("isDefault");
											
											boolean def = false;
											
											try {
												def = new Boolean(defaultOption).booleanValue();
											} catch (Exception e) {
												log.error("Invalid value for isDefault " + name);
											}
											
											//System.out.println("This field is of type [" + type + "] has option [" + valueName + "] and value [" + valueValue + "]");
											
											fo.setName(valueName);
											fo.setValue(valueValue);
											fo.setDefaultOption(def);
											
											
											
											f.addFieldOption(fo);
											
										}
										
									}
									
								}
							}
						}
					}
						
				}

			}
			
			return returnMap;
	}
	
	/**
	 * Accepts a list of json string and creates a full json object, then
	 * returns a Map<String->module,List<Field>)
	 * @param fields
	 * @return
	 * @throws Exception
	 */
	public static Map<String,List<Field>> parseFieldsValues(List<String> stringFields) throws Exception {
		
		
		Map returnMap = new HashMap();
		
		StringBuilder sb = new StringBuilder();
		sb.append("{\"paage\":\"p\",\"modules\":[");
		int i = 0;
		for(Object o:stringFields) {
			
			String s = (String)o;
			sb.append(s);
			if(i<stringFields.size()-1) {
				sb.append(",");
			}
			i++;
			
		}
		sb.append("]}");

		Map<String, String> data = new ObjectMapper().readValue(sb.toString(), HashMap.class);
		
		
		if(data!=null) {
			for(Object o: data.keySet()) {

				if(o instanceof String && ((String)o).equals("modules")) {
					// can parse
					
					Object oo = data.get(o);
					if(oo instanceof List) {//List 
						
						for(Object ooo:(List)oo) {

							if(ooo instanceof LinkedHashMap) {
								
								Map m = (Map)ooo;
								
								String module = (String)m.get("module");
							
								List valuesList = (List)m.get("values");
								
								List returnList = new ArrayList();
								
								if(valuesList!=null) {
									
									for(Object oooo:valuesList) {
										
										Field f = new Field();
										
										
										Map values = (Map)oooo;
										String valueName = (String)values.get("name");
										String valueValue = (String)values.get("value");
										
										f.setName(valueName);
										f.setFieldValue(valueValue);

										returnList.add(f);
									}
								}
								
								returnMap.put(module, returnList);
							}
						}
					}
				}
			}
		}
		
		
		
		return returnMap;
	}
	
	/**
	 * {"fields":[{"module":"moduleName","values":[{"name":"fieldName","value":"fieldValue"}...]}...]}
	 * @param fields
	 * @return String->module, List<Field>
	 * @throws Exception
	 */
	public static Map<String,List<Field>> parseFieldsValues(String fields) throws Exception {
		
		Map returnMap = new HashMap();

		Map<String, String> data = new ObjectMapper().readValue(fields, HashMap.class);
		
		if(data!=null) {
			for(Object o: data.keySet()) {

				if(o instanceof String && ((String)o).equals("fields")) {
					// can parse
					
					Object oo = data.get(o);
					if(oo instanceof List) {//List 
						
						for(Object ooo:(List)oo) {

							if(ooo instanceof LinkedHashMap) {
								
								Map m = (Map)ooo;
								
								String module = (String)m.get("module");
							
								List valuesList = (List)m.get("values");
								
								List returnList = new ArrayList();
								
								if(valuesList!=null) {
									
									for(Object oooo:valuesList) {
										
										Field f = new Field();
										
										
										Map values = (Map)oooo;
										String valueName = (String)values.get("name");
										String valueValue = (String)values.get("value");
										
										f.setName(valueName);
										f.setFieldValue(valueValue);

										returnList.add(f);
									}
								}
								
								returnMap.put(module, returnList);
							}
						}
					}
				}
			}
		}
			return returnMap;
	}
	
	/**
	 * Builds a JSON string with all fields values for a given module
	 * @param fieldValues
	 * @return
	 */
	public static String buildFieldValuesString(Map<String,List<Field>> fieldValues) {
		
		
		if(fieldValues==null) 
			return null;
		
		if(fieldValues.size()==0) 
			return null;
		
		StringBuilder sb = new StringBuilder();
		sb.append("{");
		
		for(Object o: fieldValues.keySet()){
			
			String module = (String)o;
			
			//{"fields":[{"module":"moduleName","values":[{"name":"fieldName","value":"fieldValue"}...]}...]}
			sb.append("\"module\":\"").append(module).append("\",");
			List fv = fieldValues.get(module);
			int i = 0;
			if(fv!=null && fv.size()>0) {
				sb.append("\"values\":[");
				for(Object v:fv) {
					
					Field f = (Field)v;
					sb.append("{\"name\":\"").append(f.getName()).append("\",\"value\":\"").append(f.getFieldValue()).append("\"}");
					if(i<fv.size()-1) {
						sb.append(",");
					}
					i++;
				}
				sb.append("]");
			}
		}
		sb.append("}");
		
		return sb.toString();
		
	}

}



```
