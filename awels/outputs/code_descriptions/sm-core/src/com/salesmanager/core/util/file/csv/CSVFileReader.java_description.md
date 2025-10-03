# CSVFileReader.java

## Review

## 1. Summary  

**Purpose**  
`CSVFileReader` is a tiny utility that reads a CSV file and returns its content as a `List<List<String>>` – each inner list represents a row, each string a cell. It uses a regular‑expression based parser (configured via `CSV_PATTERN` from `CSVConstants`) to handle quoted fields and trailing commas.

**Key Components**  
| Component | Role |
|-----------|------|
| `csvRE` | A `Pattern` compiled from `CSV_PATTERN`.  It is static so all instances share the same regex. |
| `processCSV(File)` | Main entry point; opens a file, reads it line‑by‑line, and delegates each line to `parseRow`. |
| `parseRow(String)` | Uses the regex matcher to extract individual columns from a CSV line, handling commas, quotes, and empty fields. |
| `CSVConstants` | External interface defining the regex (`CSV_PATTERN`), and string constants (`COMMA_STR`, `DOUBLEQUOTE_STR`). |

**Design Notes**  
- Very lightweight; no external CSV libraries are used.  
- Regular‑expression based parsing can be fragile for complex CSVs (e.g., embedded newlines).  
- The implementation is *read‑only* and *stateless* aside from the shared pattern.  
- Uses `org.apache.commons.lang.StringUtils` only for `isBlank`.  

---

## 2. Detailed Description  

### Core Flow  
1. **Construction**  
   - On instantiation, the constructor compiles the regex pattern from `CSV_PATTERN` and assigns it to the static field `csvRE`.  
   - Because the field is static, subsequent objects reuse the compiled pattern, avoiding recompilation overhead.

2. **Reading the File** (`processCSV`)  
   - Opens the file with a `BufferedReader` wrapped around a `FileReader`.  
   - Iterates over each line with `readLine()`.  
   - Skips blank lines using `StringUtils.isBlank`.  
   - For every non‑blank line, calls `parseRow` to split it into columns and appends the resulting list to `rowList`.  
   - Ensures the reader is closed in a `finally` block (pre‑Java‑7 style).

3. **Parsing a Line** (`parseRow`)  
   - Creates a new `Matcher` for the current line.  
   - In a `while (m.find())` loop, each match is a CSV field (including its trailing comma).  
   - Trims the trailing comma (`COMMA_STR`).  
   - Strips surrounding double quotes (`DOUBLEQUOTE_STR`).  
   - Empty fields are converted to `null`.  
   - Adds each cleaned field to the list, which is returned to the caller.

### Assumptions & Constraints  
- **No support for multiline fields**: The regex and line‑by‑line reading assume a single line per CSV record.  
- **Quoted fields** may contain commas but **not** escaped quotes or newlines.  
- **Trailing commas** are allowed but optional; empty trailing columns become `null`.  
- **Header handling** is not built‑in; the caller must decide how to interpret the first row.  
- **Character encoding** defaults to platform default via `FileReader`; this may lead to mis‑decoded data on non‑UTF‑8 systems.

### Architecture & Design Choices  
- **Regex‑based parsing** is chosen over a character‑by‑character scan for simplicity, but it trades robustness for speed.  
- **Static pattern** avoids re‑compilation but also makes the class not thread‑safe if the underlying `Pattern` is mutated (though it isn’t).  
- **List of lists** representation is generic but could be replaced with a more domain‑specific data structure (e.g., `List<String[]>` or a custom Row type).

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `CSVFileReader()` | Constructor – compiles the CSV regex. | None | Instantiates object | Sets static `csvRE`. |
| `processCSV(File csvFile)` | Reads a CSV file, line‑by‑line, returning all rows. | `csvFile` – the CSV file to parse. | `List<List<String>>` – rows of columns. | Opens/reads file, closes reader. |
| `parseRow(String line)` | Parses a single CSV line into a list of column values. | `line` – raw line string. | `List<String>` – parsed columns. | None. |
| `List<List<String>>` | Container of rows. | – | – | – |

### Reusable/Utility Methods  
- **`parseRow`** is a clean, pure function that could be reused independently (e.g., for parsing a string buffer).  
- The regex constants in `CSVConstants` enable swapping the parser for alternative patterns without touching the code.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.io.*` | Standard JDK | File I/O. |
| `java.util.*` | Standard JDK | Collections, regex. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Only used for `isBlank`. |
| `CSVConstants` | Custom | Provides regex and string constants; must be available on the classpath. |

No other external libraries or platform‑specific APIs are used.

---

## 5. Additional Notes  

### Edge Cases & Limitations  
1. **Multiline fields** – CSV may legally include embedded line breaks inside quoted fields. This implementation would treat each line as a separate record, breaking the data.  
2. **Escaped quotes** – CSV allows `""` inside quoted fields to represent a single `"`. The current regex does not handle this; such data will be truncated.  
3. **Encoding** – Using `FileReader` assumes default system encoding. For UTF‑8 or other encodings, a `InputStreamReader` with an explicit charset should be used.  
4. **Large files** – The method reads the entire file into memory (list of lists). For very large CSVs, a streaming approach (e.g., iterator) would be preferable.  
5. **Trailing commas** – Empty trailing columns become `null`; callers must handle potential `null` values.

### Potential Enhancements  
- **Switch to `BufferedInputStream` + `InputStreamReader`** with configurable charset.  
- **Implement a streaming API** (`Iterator<List<String>>`) to process very large files without memory blow‑up.  
- **Add support for escaped quotes and multiline fields** by enhancing the regex or moving to a state machine parser.  
- **Provide optional header processing** (e.g., return `Map<String, String>` rows).  
- **Unit tests** covering all edge cases, including quoted commas, empty fields, and malformed lines.  
- **Use `try-with-resources`** (Java 7+) to simplify resource management.  
- **Replace `StringUtils.isBlank`** with `line.trim().isEmpty()` if the extra dependency is undesirable.

Overall, the class serves its basic purpose for simple CSVs, but would need significant adjustments to handle the full CSV specification robustly.

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
package com.salesmanager.core.util.file.csv;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang.StringUtils;

/**
 * CSVFileReader class parses the supplied *.csv file to return a list of
 * rows,each consisting of a list of corresponding columns.
 * 
 * @author anilkumar.talla
 */
public class CSVFileReader implements CSVConstants {

	private static Pattern csvRE;

	/**
	 * CSVFileReader constructor.
	 */
	public CSVFileReader() {
		csvRE = Pattern.compile(CSV_PATTERN);
	}

	/**
	 * processCSV method takes the csv file argument and processes csv file to
	 * return a list of rows having corresponding column list.
	 * 
	 * @param csvFile
	 *            File
	 * @return List<List<String>>
	 * @throws IOException
	 */
	public List<List<String>> processCSV(File csvFile) throws IOException {
		BufferedReader reader = null;
		List<List<String>> rowList = new ArrayList<List<String>>();
		try {
			reader = new BufferedReader(new FileReader(csvFile));
			String line;
			while ((line = reader.readLine()) != null) {
				if (!StringUtils.isBlank(line)) {
					List<String> columns = parseRow(line);
					rowList.add(columns);
				}
			}
		} finally {
			if (reader != null) {
				reader.close();
			}
		}

		return rowList;
	}

	/**
	 * parseRow method parses the supplied line to split in to columns by
	 * matching the corresponding columns with regular expression.
	 * 
	 * @param line
	 *            String
	 * @return List<String>
	 */
	private List<String> parseRow(String line) {
		List<String> list = new ArrayList<String>();
		Matcher m = csvRE.matcher(line);
		// For each field
		while (m.find()) {
			String match = m.group();
			if (match == null)
				break;
			if (match.endsWith(COMMA_STR)) {
				match = match.substring(0, match.length() - 1);
			}
			if (match.startsWith(DOUBLEQUOTE_STR)) {
				match = match.substring(1, match.length() - 1);
			}
			if (match.length() == 0)
				match = null;
			list.add(match);
		}
		return list;
	}
}


```
