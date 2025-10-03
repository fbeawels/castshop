# CalculateBoxPackingModule.java

## Review

## 1. Summary

**Purpose**  
`CalculateBoxPackingModule` is a shipping‑packaging component used in an e‑commerce platform.  
It parses merchant‑defined box configuration values, transforms product lists into individual shipment units, and assigns those units to the smallest possible number of boxes while respecting weight and dimensional constraints.

**Key components**

| Class | Role |
|-------|------|
| `CalculateBoxPackingModule` | Implements `CalculatePackingModule` – exposes configuration UI, persists configuration, and performs the packing algorithm. |
| `PackageDetail` | Simple DTO representing the resulting box dimensions, weight and currency. |
| `PackingBox` | Internal helper storing remaining volume/weight for a box and the cumulative weight of items already packed. |

**Design patterns & libraries**

- **Strategy/Strategy‑like** – The module implements the `CalculatePackingModule` interface, allowing multiple packing strategies to be swapped.
- **Factory** – `ServiceFactory` is used to obtain a `MerchantService`.
- **Commons‑Lang** – `StringUtils` for blank‑string checks.
- **Custom utilities** – `CurrencyUtil`, `LabelUtil`, `LocaleUtil`, `LogMerchantUtil`.
- **XWork2** – `ValidationException` for input validation.

## 2. Detailed Description

1. **Configuration**  
   - UI file: `packing-box.jsp`.  
   - Values are stored as a single string separated by `"|"` in `MerchantConfiguration.configurationValue1`.  
   - `getConfigurationOptions` parses this string into a `PackageDetail` object.  
   - `storeConfiguration` validates each input (width, height, length, weight, max weight, threshold) via `CurrencyUtil.validateMeasure` and persists the concatenated string.

2. **Packing Algorithm** (`calculatePacking`)  
   - Validates the product collection is non‑null.  
   - Extracts box limits (size, max weight, threshold) from configuration.  
   - Expands each product into a unit of quantity = 1, preserving attributes that affect weight.  
   - Checks if the product dimension/weight fits in the box; if not, it aborts and throws an exception (intended to fall back to per‑item shipping).  
   - Uses a greedy approach:  
     * Iterates over each product and tries to fit it into any existing box (by checking remaining volume & weight).  
     * If no existing box fits, a new `PackingBox` is created.  
     * `maxBox` (max 100) limits the number of boxes; when reached the algorithm stops silently (no error).  
   - After assignment, it builds a `PackageDetail` for each used box, adding the total weight of packed items to the base box weight.

3. **Assumptions / constraints**  
   - Box dimensions/weight must be > 0; otherwise a warning is logged but the algorithm proceeds.  
   - The algorithm assumes product dimensions are expressed in the same unit as box dimensions.  
   - Threshold is a simple minimum count of products; no advanced capacity optimisation.  
   - The algorithm is **greedy**; it does not explore combinations or attempt to minimise the number of boxes beyond the first fit.

4. **Architecture / design choices**  
   - The packing logic is tightly coupled with the data format of `MerchantConfiguration`.  
   - No separation of concerns: configuration parsing, persistence, and packing are all in the same class.  
   - Uses raw `Collection` / `List` without generics (raw types).  
   - No logging framework (uses `LogMerchantUtil.log`).  
   - Exception handling is primitive; many business logic errors are communicated by throwing generic `Exception`.

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs / Side effects |
|--------|---------|--------|------------------------|
| `getConfigurationOptionsFileName(Locale)` | Returns the JSP name for configuration UI. | `Locale` | `"packing-box.jsp"` |
| `getConfigurationOptions(MerchantConfiguration, String)` | Parses config string into `PackageDetail`. | `config`, `currency` | `PackageDetail` |
| `calculatePacking(Collection<OrderProduct>, MerchantConfiguration, int)` | Main packing algorithm. | `products`, `config`, `merchantId` | `Collection<PackageDetail>` |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Validates request parameters and persists config. | `merchantId`, `vo`, `request` | Persists via `MerchantService` |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | No‑op – returns passed `vo`. | `configurations`, `vo` | `vo` |
| **Inner helper `PackingBox`** | Stores remaining volume/weight per box. | – | – |

### Notable helper logic

- **Dimension validation** – early exit if a product exceeds box dimensions or weight.
- **Box creation** – new `PackingBox` when no existing box can accommodate the product.
- **Volume & weight calculation** – simple multiplication, no packing algorithm for orientation.

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard | For configuration form handling. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Simple blank check. |
| `com.opensymphony.xwork2.validator.ValidationException` | Third‑party | XWork validation. |
| `com.salesmanager.core.*` | Internal | Entity, service, util classes. |
| `java.math.BigDecimal`, `java.util.*` | Standard | Core collections, number handling. |

Platform assumption: Java EE (servlet container) and the SalesManager core framework.

## 5. Additional Notes & Recommendations

### Strengths

- **Self‑contained**: All packing logic resides in one class, making it straightforward to understand in isolation.
- **Extensible interface**: By implementing `CalculatePackingModule`, other packing strategies could be plugged in.
- **Input validation**: Uses `CurrencyUtil.validateMeasure` to ensure numeric inputs are parsed correctly.

### Issues & Risks

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`Collection`, `List`, `Map`, `Iterator`) | Compile‑time type safety lost; possible `ClassCastException`. | Use generics throughout (`Collection<OrderProduct>`, `List<PackingBox>`, etc.). |
| **Hard‑coded limits** (`maxBox = 100`) | May silently fail on large orders. | Expose as configuration or throw exception when exceeded. |
| **Greedy packing** | Not optimal; may use more boxes than necessary. | Consider implementing a bin‑packing algorithm (first‑fit decreasing, etc.). |
| **Dimension units mismatch** | If product or box dimensions use different units, calculations break. | Explicitly document unit expectations; convert if needed. |
| **No concurrency handling** | `calculatePacking` is stateless, but `storeConfiguration` may race on configuration writes. | Use proper transaction isolation in `MerchantService`. |
| **Exception handling** | Generic `Exception` thrown for many business errors, making it hard for callers to react. | Define custom exceptions (e.g., `InvalidProductDimensionException`). |
| **Logging** | Uses a custom `LogMerchantUtil` without log levels or context. | Integrate SLF4J/Log4j and log at appropriate levels. |
| **No unit tests** | Hard to validate correctness or regression. | Add tests covering: parsing, packing edge cases, threshold logic. |
| **UI coupling** | JSP name returned directly; could be externalised. | Store UI paths in configuration or constants. |
| **Performance** | For each product, iterates over all boxes; O(n*m). | Optimize by tracking free space or using spatial data structures. |

### Future Enhancements

1. **Refactor to separate concerns**  
   - Extract configuration parsing into a dedicated class (`BoxConfigurationParser`).  
   - Move persistence to a DAO/service layer.  
   - Isolate packing logic into a pure, testable class (`BoxPackingAlgorithm`).

2. **Introduce a better packing algorithm**  
   - First‑Fit Decreasing (FFD) or Best‑Fit Decreasing (BFD) to reduce box count.  
   - Optionally support orientation changes (rotating items).

3. **Improve validation and error reporting**  
   - Return detailed error objects instead of throwing generic exceptions.  
   - Provide user‑friendly messages for each failure case.

4. **Enhance configurability**  
   - Allow different packing strategies per merchant.  
   - Expose box dimensions and weight limits as separate configuration keys.

5. **Add internationalisation and unit support**  
   - Store dimensions in a standard unit (e.g., centimeters) and convert user input accordingly.

6. **Testing & CI**  
   - Unit tests for packing scenarios (small/large products, weight limits).  
   - Integration tests for configuration persistence.

---

**Overall verdict**: The module fulfills its basic functional requirements but would benefit significantly from refactoring for type safety, maintainability, and packing optimality. Addressing the highlighted issues will make the code more robust, easier to test, and ready for future extensions.

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
package com.salesmanager.core.module.impl.application.shipping;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang.StringUtils;

import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductAttribute;
import com.salesmanager.core.entity.shipping.PackageDetail;
import com.salesmanager.core.module.model.application.CalculatePackingModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.LogMerchantUtil;

public class CalculateBoxPackingModule implements CalculatePackingModule {

	public String getConfigurationOptionsFileName(Locale locale)
			throws Exception {

		return "packing-box.jsp";
	}

	public PackageDetail getConfigurationOptions(MerchantConfiguration config,
			String currency) throws Exception {

		if (config == null || config.getConfigurationValue1() == null) {
			return null;
		}

		PackageDetail details = new PackageDetail();
		StringTokenizer st = new StringTokenizer(config
				.getConfigurationValue1(), "|");

		Map parseTokens = new HashMap();

		int i = 1;
		while (st.hasMoreTokens()) {
			String token = st.nextToken();
			if (i == 1) {

				details.setShippingWidth(new BigDecimal(token).doubleValue());

			} else if (i == 2) {
				details.setShippingHeight(new BigDecimal(token).doubleValue());

			} else if (i == 3) {
				details.setShippingLength(new BigDecimal(token).doubleValue());

			} else if (i == 4) {
				details.setShippingWeight(new BigDecimal(token).doubleValue());

			} else if (i == 5) {
				details.setShippingMaxWeight(new BigDecimal(token)
						.doubleValue());

			} else if (i == 6) {
				details.setTreshold(Integer.parseInt(token));

			}
			i++;
		}
		details.setCurrency(currency);
		return details;

	}

	public Collection<PackageDetail> calculatePacking(
			Collection<OrderProduct> products, MerchantConfiguration config,
			int merchantId) throws Exception {

		if (products == null) {
			throw new Exception("Product list cannot be null !!");
		}

		double width = 0;
		double length = 0;
		double height = 0;
		double weight = 0;
		double maxweight = 0;

		int treshold = 0;

		// get box details from merchantconfiguration
		String values = config.getConfigurationValue1();
		if (!StringUtils.isBlank(values)) {
			StringTokenizer st = new StringTokenizer(config
					.getConfigurationValue1(), "|");

			Map parseTokens = new HashMap();

			int i = 1;
			while (st.hasMoreTokens()) {
				String token = st.nextToken();
				if (i == 1) {

					width = new BigDecimal(token).doubleValue();

				} else if (i == 2) {
					height = new BigDecimal(token).doubleValue();

				} else if (i == 3) {
					length = new BigDecimal(token).doubleValue();

				} else if (i == 4) {
					weight = new BigDecimal(token).doubleValue();

				} else if (i == 5) {

					maxweight = new BigDecimal(token).doubleValue();

				} else if (i == 6) {

					treshold = Integer.parseInt(token);

				}
				i++;
			}

		} else {
			LogMerchantUtil.log(merchantId,
					"Shipping Box information is not configured adequatly");
			throw new Exception("Cannot determine box size");
		}

		List boxes = new ArrayList();

		// maximum number of boxes
		int maxBox = 100;
		int iterCount = 0;

		Collection leftProducts = new ArrayList();

		// need to put items individualy
		Iterator prodIter = products.iterator();
		while (prodIter.hasNext()) {
			OrderProduct op = (OrderProduct) prodIter.next();

			if (!op.isShipping()) {
				continue;
			}

			int qty = op.getProductQuantity();

			Set attrs = op.getOrderattributes();

			// set attributes values
			BigDecimal w = op.getProductWeight();
			if (attrs != null && attrs.size() > 0) {
				Iterator attributesIterator = attrs.iterator();
				OrderProductAttribute opa = (OrderProductAttribute) attributesIterator
						.next();
				w = w.add(opa.getProductAttributeWeight());
			}

			if (qty > 1) {

				for (int i = 1; i <= qty; i++) {
					OrderProduct tempop = new OrderProduct();
					tempop.setProductHeight(op.getProductHeight());
					tempop.setProductLength(op.getProductLength());
					tempop.setProductWidth(op.getProductWidth());
					tempop.setProductWeight(w);
					tempop.setProductQuantity(1);
					tempop.setOrderattributes(attrs);
					leftProducts.add(tempop);
				}
			} else {
				op.setProductWeight(w);
				leftProducts.add(op);
			}
			iterCount++;
		}

		if (iterCount == 0) {
			return null;
		}

		int productCount = leftProducts.size();

		if (productCount < treshold) {
			throw new Exception("Number of item smaller than treshold");
		}

		List usedBoxesList = new ArrayList();

		PackingBox b = new PackingBox();
		// set box max volume
		double maxVolume = width * length * height;

		if (maxVolume == 0 || maxweight == 0) {
			LogMerchantUtil.log(merchantId,
					"Check shipping box configuration, it has a volume of "
							+ maxVolume + " and a maximum weight of "
							+ maxweight
							+ ". Those values must be greater than 0.");
		}
		b.setVolumeLeft(maxVolume);
		b.setWeightLeft(maxweight);

		usedBoxesList.add(b);

		int boxCount = 1;
		Collection assignedProducts = new ArrayList();

		// calculate the volume for the next object
		if (assignedProducts.size() > 0) {
			leftProducts.removeAll(assignedProducts);
			assignedProducts = new ArrayList();
		}
		Iterator prodIterator = leftProducts.iterator();

		boolean productAssigned = false;

		while (prodIterator.hasNext()) {
			OrderProduct op = (OrderProduct) prodIterator.next();
			Collection attributes = op.getOrderattributes();
			productAssigned = false;

			double productWeight = op.getProductWeight().doubleValue();


			// validate if product fits in the box
			if (op.getProductWidth().doubleValue() > width
					|| op.getProductHeight().doubleValue() > height
					|| op.getProductLength().doubleValue() > length) {
				// log message to customer
				LogMerchantUtil
						.log(
								merchantId,
								"Product "
										+ op.getProductId()
										+ " has a demension larger than the box size specified. Will use per item calculation.");
				// exit this process and let shipping calculator calculate
				// individual items
				throw new Exception(
						"Product configuration exceeds box configuraton");
			}

			if (productWeight > maxweight) {
				LogMerchantUtil
						.log(
								merchantId,
								"Product "
										+ op.getProductId()
										+ " has a weight larger than the box maximum weight specified. Will use per item calculation.");
				throw new Exception("Product weight exceeds box maximum weight");
			}

			double productVolume = (op.getProductWidth().doubleValue()
					* op.getProductHeight().doubleValue() * op
					.getProductLength().doubleValue());

			if (productVolume == 0) {
				LogMerchantUtil
						.log(
								merchantId,
								"Product "
										+ op.getProductId()
										+ " has one of the dimension set to 0 and therefore cannot calculate the volume");
				throw new Exception("Cannot calculate volume");
			}

			List boxesList = usedBoxesList;

			// try each box
			Iterator boxIter = boxesList.iterator();
			while (boxIter.hasNext()) {
				PackingBox pb = (PackingBox) boxIter.next();
				double volumeLeft = pb.getVolumeLeft();
				double weightLeft = pb.getWeightLeft();

				if (pb.getVolumeLeft() >= productVolume
						&& pb.getWeightLeft() >= productWeight) {// fit the item
																	// in this
																	// box
					// fit in the current box
					volumeLeft = volumeLeft - productVolume;
					pb.setVolumeLeft(volumeLeft);
					weightLeft = weightLeft - productWeight;
					pb.setWeightLeft(weightLeft);

					assignedProducts.add(op);
					productCount--;

					double w = pb.getWeight();
					w = w + productWeight;
					pb.setWeight(w);
					productAssigned = true;
					maxBox--;
					break;

				}

			}

			if (!productAssigned) {// create a new box

				b = new PackingBox();
				// set box max volume
				b.setVolumeLeft(maxVolume);
				b.setWeightLeft(maxweight);

				usedBoxesList.add(b);

				double volumeLeft = b.getVolumeLeft() - productVolume;
				b.setVolumeLeft(volumeLeft);
				double weightLeft = b.getWeightLeft() - productWeight;
				b.setWeightLeft(weightLeft);
				assignedProducts.add(op);
				productCount--;
				double w = b.getWeight();
				w = w + productWeight;
				b.setWeight(w);
				maxBox--;
			}

		}

		// now prepare the shipping info

		// number of boxes

		Iterator ubIt = usedBoxesList.iterator();

		System.out.println("###################################");
		System.out.println("Number of boxex " + usedBoxesList.size());
		System.out.println("###################################");

		while (ubIt.hasNext()) {
			PackingBox box = (PackingBox) ubIt.next();
			PackageDetail details = new PackageDetail();
			details.setShippingHeight(height);
			details.setShippingLength(length);
			details.setShippingWeight(weight + box.getWeight());
			details.setShippingWidth(width);
			boxes.add(details);
		}

		return boxes;

	}

	public void storeConfiguration(int merchantId, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {

		// get the store information
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(merchantId);

		// id - merchantId - SHP_PACK - packing-item/packing/box - [values] -
		// null - packing-item/packing/box

		// validate submited values
		// box_maxweight
		// box_weight
		// box_length
		// box_height
		// box_width
		
		Locale locale = LocaleUtil.getLocale(request);

		StringBuffer buf = new StringBuffer();

		try {
			BigDecimal width = CurrencyUtil.validateMeasure(request
					.getParameter("box_width"), store.getCurrency());
			// int width = Integer.parseInt();
			buf.append(width.toString()).append("|");
		} catch (Exception e) {
			throw new ValidationException(LabelUtil.getInstance().getText(
					locale, "module.box.invalid.width"));
		}

		try {
			BigDecimal height = CurrencyUtil.validateMeasure(request
					.getParameter("box_height"), store.getCurrency());
			// int height =
			// Integer.parseInt(request.getParameter("box_height"));
			buf.append(height.toString()).append("|");
		} catch (Exception e) {
			throw new ValidationException(LabelUtil.getInstance().getText(
					locale, "module.box.invalid.height"));
		}

		try {
			BigDecimal length = CurrencyUtil.validateMeasure(request
					.getParameter("box_length"), store.getCurrency());
			// int length =
			// Integer.parseInt(request.getParameter("box_length"));
			buf.append(length.toString()).append("|");
		} catch (Exception e) {
			throw new ValidationException(LabelUtil.getInstance().getText(
					locale, "module.box.invalid.length"));
		}

		try {
			BigDecimal weight = CurrencyUtil.validateMeasure(request
					.getParameter("box_weight"), store.getCurrency());

			buf.append(weight.toString()).append("|");
		} catch (Exception e) {
			throw new ValidationException(LabelUtil.getInstance().getText(
					locale, "module.box.invalid.weight"));
		}

		try {
			BigDecimal maxweight = CurrencyUtil.validateMeasure(request
					.getParameter("box_maxweight"), store.getCurrency());

			buf.append(maxweight.toString()).append("|");
		} catch (Exception e) {
			throw new ValidationException(LabelUtil.getInstance().getText(
					locale, "module.box.invalid.maxweight"));
		}

		try {
			int treshold = Integer.parseInt(request
					.getParameter("box_treshold"));
			buf.append(treshold);
		} catch (Exception e) {
			throw new ValidationException(LabelUtil.getInstance().getText(
					locale, "module.box.invalid.treshold"));
		}

		ConfigurationRequest vr = new ConfigurationRequest(merchantId,
				"SHP_PACK");
		ConfigurationResponse resp = mservice.getConfiguration(vr);

		MerchantConfiguration conf = null;
		if (resp == null || resp.getMerchantConfiguration("SHP_PACK") == null) {

			conf = new MerchantConfiguration();

		} else {
			conf = resp.getMerchantConfiguration("SHP_PACK");
		}

		conf.setMerchantId(merchantId);
		conf.setConfigurationKey("SHP_PACK");
		conf.setConfigurationValue("packing-box");
		conf.setConfigurationValue1(buf.toString());

		mservice.saveOrUpdateMerchantConfiguration(conf);

	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo) {

		// nothing specific

		return vo;
	}

}

class PackingBox {

	private double volumeLeft;
	private double weightLeft;
	private double weight;

	public double getVolumeLeft() {
		return volumeLeft;
	}

	public void setVolumeLeft(double volumeLeft) {
		this.volumeLeft = volumeLeft;
	}

	public double getWeight() {
		return weight;
	}

	public void setWeight(double weight) {
		this.weight = weight;
	}

	public double getWeightLeft() {
		return weightLeft;
	}

	public void setWeightLeft(double weightLeft) {
		this.weightLeft = weightLeft;
	}

}



```
