# ShippingPricePound.java

## Review

## 1. Summary  
`ShippingPricePound` is a simple value‑object that maps a weight limit (in pounds) to a shipping cost (`BigDecimal`).  
It is intended to be stored in sorted collections – e.g. a `TreeSet` – where items are ordered by the weight limit.  
The class implements the raw `Comparable` interface and provides basic getters/setters.  
No external libraries or frameworks are used; the code relies only on JDK types (`int`, `BigDecimal`, `String`).

---

## 2. Detailed Description  
The class is composed of three private fields:  

| Field | Type | Purpose |
|-------|------|---------|
| `maxpound` | `int` | The maximum weight in pounds for which the price applies. |
| `price` | `BigDecimal` | The cost associated with the weight bracket. |

### Flow of execution
1. **Construction** – An instance is created (typically via default constructor).  
2. **Population** – Caller sets `maxpound` and `price` through the provided setters.  
3. **Sorting** – When inserted into a sorted collection, the `compareTo` method orders the objects by `maxpound` in *descending* order (higher weight limits come first).  
4. **String Representation** – The overridden `toString` simply returns `"NOT IMPLEMENTED"`; therefore, debugging output is not helpful.  

### Assumptions & Constraints
- `maxpound` is a non‑negative integer; the code does not validate this.  
- `price` should not be `null`; a `NullPointerException` will be thrown if it is.  
- `compareTo` assumes the argument is a `ShippingPricePound`; otherwise it throws `ClassCastException`.  

### Design choices
- Implements the raw `Comparable` interface, which forces manual type checks.  
- The ordering is *descending* on `maxpound`, a design decision that should be documented (or inverted, depending on business needs).  
- No `equals`/`hashCode` overrides – equality is determined by reference only, which is acceptable if the objects are immutable after insertion into a set but not if they are reused elsewhere.  

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `getMaxpound()` | Accessor for weight limit | none | `int` | none |
| `setMaxpound(int)` | Mutator for weight limit | `maxpound` | void | changes internal state |
| `getPrice()` | Accessor for price | none | `BigDecimal` | none |
| `setPrice(BigDecimal)` | Mutator for price | `price` | void | changes internal state |
| `compareTo(Object)` | Implements ordering (descending by `maxpound`) | any `Object` | `int` | throws `ClassCastException` if wrong type |
| `toString()` | Debug/printing helper | none | `String` | none (currently unhelpful) |

### Reusable / Utility methods  
None. All methods are straightforward getters/setters except for `compareTo` and `toString`.  

---

## 4. Dependencies

| Library | Type | Purpose |
|---------|------|---------|
| `java.math.BigDecimal` | JDK | Holds the monetary amount. |
| `java.lang.Object` | JDK | Inheritance base class. |
| `java.lang.Comparable` | JDK | Enables sorting. |

No third‑party or platform‑specific dependencies are used.

---

## 5. Additional Notes & Recommendations

### Edge cases / shortcomings
1. **`compareTo` implementation**  
   - Uses raw `Comparable`; better to use `Comparable<ShippingPricePound>` and remove the `instanceof` guard.  
   - Manual `ClassCastException` is unnecessary – the compiler will enforce the type.  
   - Current logic sorts *descending*; if ascending order is desired, swap the return values.

2. **Null handling**  
   - `price` can be `null`. If a `BigDecimal` is required, either validate in setters or provide a default value.  
   - `compareTo` does not check for `null` in the incoming object – although `instanceof` guards it, a `NullPointerException` will still occur if `o` is `null`.

3. **`equals` / `hashCode`**  
   - Adding these would allow the object to behave correctly in hash‑based collections (`HashSet`, `HashMap`).  
   - If immutability is desired, make the fields `final` and provide a constructor that sets both values.

4. **`toString`**  
   - Currently returns `"NOT IMPLEMENTED"`. Replace with a meaningful representation, e.g.  
     ```java
     @Override
     public String toString() {
         return String.format("ShippingPricePound{maxpound=%d, price=%s}", maxpound, price);
     }
     ```

5. **Immutability**  
   - The class could be made immutable: remove setters, declare fields `final`, provide a constructor. This improves thread safety and simplifies usage in collections.

6. **Documentation**  
   - Add Javadoc comments to clarify the purpose of the class and the semantics of the ordering.

### Suggested refactor (minimal)

```java
public class ShippingPricePound implements Comparable<ShippingPricePound> {

    private final int maxPound;
    private final BigDecimal price;

    public ShippingPricePound(int maxPound, BigDecimal price) {
        if (price == null) throw new IllegalArgumentException("price cannot be null");
        this.maxPound = maxPound;
        this.price = price;
    }

    public int getMaxPound() { return maxPound; }
    public BigDecimal getPrice() { return price; }

    @Override
    public int compareTo(ShippingPricePound other) {
        // descending order by maxPound
        return Integer.compare(other.maxPound, this.maxPound);
    }

    @Override
    public String toString() {
        return String.format("ShippingPricePound{maxPound=%d, price=%s}", maxPound, price);
    }

    @Override
    public boolean equals(Object o) { /* compare fields */ }
    @Override
    public int hashCode() { /* hash fields */ }
}
```

### Future Enhancements
- **Range handling** – Currently only a single max pound value is stored. If shipping rates need lower bounds, add a `minPound` field.
- **Currency support** – `price` could be wrapped in a dedicated `Money` class that includes currency information.
- **Validation** – Enforce business rules (e.g., non‑negative weights, positive prices) during construction.
- **Unit tests** – Provide tests for ordering, equality, and string representation.

---

**Conclusion**  
The class fulfills a very narrow purpose but could benefit from a few small refactorings to improve type safety, readability, and robustness. Making it immutable and properly implementing `Comparable`, `equals`, and `hashCode` would align it with common Java best practices.

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
package com.salesmanager.core.entity.shipping;

import java.math.BigDecimal;

public class ShippingPricePound implements Comparable {

	private int maxpound;
	private BigDecimal price;

	public int getMaxpound() {
		return maxpound;
	}

	public void setMaxpound(int maxpound) {
		this.maxpound = maxpound;
	}

	public BigDecimal getPrice() {
		return price;
	}

	public void setPrice(BigDecimal price) {
		this.price = price;
	}

	public int compareTo(Object o) {
		if (!(o instanceof ShippingPricePound))
			throw new ClassCastException();
		if (((ShippingPricePound) o).getMaxpound() < this.getMaxpound())
			return 1;
		if (((ShippingPricePound) o).getMaxpound() > this.getMaxpound())
			return -1;
		else
			return 0;
	}

	public String toString() {
		// return new
		// StringBuffer().append(maxpound).append(",").append(price.doubleValue()).toString();
		return "NOT IMPLEMENTED";
	}

}



```
