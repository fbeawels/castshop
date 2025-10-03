# ProductImageUtil.java

## Review

## 1. Summary
ProductImageUtil is a utility class that handles the ingestion, resizing, cropping, and storage of product images in a commerce application.  
Key responsibilities:

| Component | Role |
|-----------|------|
| **Image ingestion** – `uploadProductImages`, `uploadCropedProductImages` | Accepts an uploaded file, generates thumbnails/large variants, stores them via a `FileModule`. |
| **Image manipulation** – `resize`, `blurImage`, `getCroppedImage`, `resizeImage` | Performs in‑memory scaling, optional blurring, and cropping. |
| **Cropping logic** – `determineCropeable`, `determineBaseline`, `determineCropArea` | Calculates whether an image can be cropped, which dimension is the baseline, and the final crop area. |
| **Configuration** – `getDefaultConfigMap`, `getValue` | Reads image dimension defaults from a central `PropertiesUtil` configuration. |

The class is **not** thread‑safe (state is stored in instance fields). It relies on several third‑party helpers (`FileModule`, `SpringUtil`, `PropertiesUtil`) and a Sun internal API (`BufferedImageGraphicsConfig`).  

---

## 2. Detailed Description
### 2.1 Flow of execution
1. **Upload**  
   * `uploadProductImages()` is invoked when a new product image is uploaded.  
   * The image file is read into a `BufferedImage`.  
   * A *unique* product‑specific name is constructed and the file is persisted using `FileModule.uploadFile`.  
   * Cropping feasibility is checked (`determineCropeable`).  
   * Two resized variants are generated (small & large) via `resizeImage`, then uploaded and the temporary files are deleted.

2. **Cropping**  
   * `uploadCropedProductImages()` behaves similarly but skips the original‑file upload step; it assumes the product already has an image.  
   * The cropped image is not stored in this method – the assumption is that the caller will use `getCroppedImage()` to obtain a cropped temp file for further processing.

3. **Utility helpers**  
   * `resize()` uses a high‑quality bilinear interpolation to generate a new `BufferedImage`.  
   * `blurImage()` applies a 3×3 averaging kernel via `ConvolveOp`.  
   * `createCompatibleImage()` (currently unused) would create a hardware‑accelerated image.

### 2.2 Design decisions & constraints
* **Instance state**: `cropeable`, `cropeBaseline`, `cropAreaWidth`, `cropAreaHeight` are stored per instance. The class is therefore not reusable across threads unless each thread gets its own instance.  
* **Hard‑coded paths & prefixes**: Prefixes for small/large images are fetched from the configuration; the original image path is derived from a hard‑coded folder (`FileUtil.getProductFilePath()`).
* **Temp file usage**: Temporary files are created for each resize/crop operation (`deleteOnExit` is used for the crop temp). This may lead to file‑handle exhaustion in high‑traffic scenarios.  
* **Sun internal API**: `sun.awt.image.BufferedImageGraphicsConfig` is used in `createCompatibleImage()`. This is non‑portable and may break on other JVMs or future Java releases.  
* **Error handling**: Exceptions are propagated (`throws Exception`) but no fine‑grained recovery or logging is performed.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `initCropImage(Product, Map)` | Pre‑computes cropping parameters for an existing product image. | `product`, `moduleConfigMap` | none | Sets internal state (`cropeable`, `cropeBaseline`, etc.) |
| `uploadProductImages(File, String, String, Product, Map)` | Handles a new image upload: stores original, small & large versions, updates product entity. | `image`, `imageName`, `imageContentType`, `product`, `moduleConfigMap` | none | Persists files, updates `product` fields, writes temp files |
| `uploadCropedProductImages(File, String, String, Product, Map)` | Similar to above but assumes image already exists. | `image`, `imageName`, `imageContentType`, `product`, `moduleConfigMap` | none | Persists resized variants |
| `determineCropeable(int,int,int,int)` | Sets `cropeable` based on whether the image can be cropped to requested specs. | `width`, `specWidth`, `height`, `specHeight` | none | Sets instance field |
| `determineBaseline(int,int)` | Chooses baseline dimension (width=0, height=1) for crop area calculation. | `width`, `height` | none | Sets `cropeBaseline` |
| `determineCropArea(int,int,int,int)` | Computes crop area (`cropAreaWidth`, `cropAreaHeight`) based on baseline and scaling factor. | `width`, `specWidth`, `height`, `specHeight` | none | Sets instance fields |
| `getCroppedImage(File,int,int,int,int)` | Returns a temporary file containing a sub‑image. | `originalFile`, `x1`, `y1`, `width`, `height` | `File` (temp) | Creates temp file, writes JPEG |
| `getValue(String,Map,Map)` | Resolves a dimension value from the module config or defaults. | `key`, `moduleConfigMap`, `defaultConfigMap` | `Integer` | none |
| `getDefaultConfigMap()` | Builds a map of default image dimension settings from the global configuration. | none | `Map<String,String>` | none |
| `resize(BufferedImage,int,int)` | Scales a `BufferedImage` to new dimensions using bilinear interpolation. | `image`, `width`, `height` | `BufferedImage` | none |
| `blurImage(BufferedImage)` | Applies a 3×3 averaging blur. | `image` | `BufferedImage` | none |
| `createCompatibleImage(BufferedImage)` | Creates a graphics‑compatible image (unused). | `image` | `BufferedImage` | none |
| `resizeImage(BufferedImage,int,int)` | Convenience wrapper: resizes and writes to a temp PNG file. | `image`, `width`, `height` | `File` | Creates temp file, writes PNG |
| `isCropeable()` / `setCropeable(boolean)` | Accessors for `cropeable`. | none / `boolean` | `boolean` / none | none |
| `getCropeBaseline() / setCropeBaseline(int)` | Accessors for `cropeBaseline`. | none / `int` | `int` / none | none |
| `getCropAreaWidth() / setCropAreaWidth(int)` | Accessors for `cropAreaWidth`. | none / `int` | `int` / none | none |
| `getCropAreaHeight() / setCropAreaHeight(int)` | Accessors for `cropAreaHeight`. | none / `int` | `int` / none | none |

---

## 4. Dependencies
| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Central configuration holder. |
| `javax.imageio.ImageIO` | Standard | Image read/write. |
| `sun.awt.image.BufferedImageGraphicsConfig` | **Internal** | Not portable; should be replaced with `GraphicsEnvironment.getLocalGraphicsEnvironment()` approach. |
| `com.salesmanager.core.util.PropertiesUtil` | Custom | Reads global config. |
| `com.salesmanager.core.util.FileUtil` | Custom | Provides file path helpers. |
| `com.salesmanager.core.module.model.application.FileModule` | Custom | Abstracts file storage (local or remote). |
| `com.salesmanager.core.util.SpringUtil` | Custom | Retrieves beans. |
| `com.salesmanager.core.entity.catalog.Product` | Custom | Entity representing a product. |
| `com.salesmanager.core.module.model.application.FileModule` | Custom | Handles upload to storage. |

All dependencies are either standard JDK or internal project libraries. No external image processing libraries are used; only the built‑in `ConvolveOp` for blurring.

---

## 5. Additional Notes & Recommendations

### 5.1 Code quality & maintainability
* **Redundant / unused code** – `createCompatibleImage()` and the commented `shrinkResize()` are never invoked. Consider removing or documenting their intended use.  
* **Magic numbers** – The baseline logic uses `0` and `1`. Replace with named constants (`BASELINE_WIDTH`, `BASELINE_HEIGHT`) for clarity.  
* **Error handling** – All public methods throw `Exception`. It would be cleaner to define a custom `ImageProcessingException` and catch lower‑level exceptions (e.g., I/O, `NullPointerException`) inside the methods.  
* **Logging** – No logging is performed. In production, each I/O or failure point should log at least `WARN` or `ERROR` level.  

### 5.2 Thread safety
The class holds mutable state per instance. If a single `ProductImageUtil` instance is shared across threads (e.g., as a Spring bean), concurrent calls will corrupt the state (`cropeable`, `cropAreaWidth`, …).  
* **Solution**:  
  * Make the class stateless – compute all parameters locally and return a DTO.  
  * Or synchronize the methods that modify shared fields, though this would serialize access.

### 5.3 Temporary file handling
`File.createTempFile()` is used for every resize/crop operation.  
* **Potential issue** – Under high load, the temp directory may fill up or hit file‑descriptor limits.  
* **Recommendation** – Return a `BufferedImage` directly or use a pooling mechanism for temp files. Clean up with `deleteOnExit()` only for truly temporary files.

### 5.4 Image format consistency
* `getCroppedImage()` writes JPEG; `resizeImage()` writes PNG. Inconsistent formats may cause downstream consumers to handle images differently.  
* **Suggestion** – Expose a single image format (e.g., PNG) or let the caller specify the desired format.

### 5.5 Use of internal APIs
`sun.awt.image.BufferedImageGraphicsConfig` is a Sun/Oracle internal class and is not part of the public API. It may be removed in future Java releases.  
* **Replace** with standard JDK calls:  
  ```java
  GraphicsEnvironment ge = GraphicsEnvironment.getLocalGraphicsEnvironment();
  GraphicsDevice gd = ge.getDefaultScreenDevice();
  GraphicsConfiguration gc = gd.getDefaultConfiguration();
  BufferedImage result = gc.createCompatibleImage(w, h, Transparency.TRANSLUCENT);
  ```
  or simply avoid hardware acceleration if not required.

### 5.6 Resizing algorithm
The current resize uses bilinear interpolation (`VALUE_INTERPOLATION_BILINEAR`).  
* **Quality trade‑off** – For product images, a higher‑quality algorithm (bicubic) may provide better visual fidelity.  
* **Performance** – If resizing large images frequently, consider caching or using a dedicated image‑processing library (e.g., `imgscalr`, `Thumbnailator`) that optimizes for speed.

### 5.7 API exposure
* **Encapsulation** – Expose only the operations that are truly useful to callers. Many setters/getters for internal fields are unnecessary.  
* **DTO** – Return a simple data object (e.g., `ImageVariants` containing URLs/paths for original, small, large) instead of mutating the `Product` entity inside the util.

### 5.8 Testing
* **Unit tests** – Verify that cropping logic works correctly for images of various aspect ratios.  
* **Integration tests** – Test file upload to `FileModule` with mocked storage to ensure correct filenames and prefixes.  
* **Performance tests** – Benchmark resizing under load to detect bottlenecks.

---

### TL;DR Checklist for Refactoring
1. Remove internal‑API usage (`BufferedImageGraphicsConfig`).  
2. Make the class stateless or guard mutable fields.  
3. Replace magic numbers with named constants.  
4. Add robust error handling and logging.  
5. Standardize image format outputs.  
6. Replace temp‑file patterns with in‑memory processing or resource pooling.  
7. Replace `Exception` with a domain‑specific exception type.  
8. Consolidate duplicated logic (e.g., `uploadProductImages` & `uploadCropedProductImages`).  
9. Add unit tests for all public methods.  

Implementing these changes will result in a cleaner, safer, and more maintainable image‑handling utility.

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

import java.awt.AlphaComposite;
import java.awt.Graphics2D;
import java.awt.GraphicsConfiguration;
import java.awt.RenderingHints;
import java.awt.Transparency;
import java.awt.RenderingHints.Key;
import java.awt.image.BufferedImage;
import java.awt.image.BufferedImageOp;
import java.awt.image.ConvolveOp;
import java.awt.image.Kernel;
import java.io.File;
import java.util.HashMap;
import java.util.Map;

import javax.imageio.ImageIO;

import org.apache.commons.configuration.Configuration;

import sun.awt.image.BufferedImageGraphicsConfig;

import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.module.model.application.FileModule;

public class ProductImageUtil {

	private boolean cropeable = true;
	private int cropeBaseline = 0;// o is width, 1 is height

	public int getCropeBaseline() {
		return cropeBaseline;
	}

	public void setCropeBaseline(int cropeBaseline) {
		this.cropeBaseline = cropeBaseline;
	}

	private int cropAreaWidth;
	private int cropAreaHeight;

	public int getCropAreaWidth() {
		return cropAreaWidth;
	}

	public void setCropAreaWidth(int cropAreaWidth) {
		this.cropAreaWidth = cropAreaWidth;
	}

	public int getCropAreaHeight() {
		return cropAreaHeight;
	}

	public void setCropAreaHeight(int cropAreaHeight) {
		this.cropAreaHeight = cropAreaHeight;
	}

	public void initCropImage(Product product,
			Map<String, String> moduleConfigMap) throws Exception {

		Configuration conf = PropertiesUtil.getConfiguration();
		//String folder = conf.getString("core.product.image.filefolder") 
		String folder = FileUtil.getProductFilePath()
				+ "/"
				+ product.getMerchantId() + "/";
		File image = new File(folder + product.getProductImage());

		Map<String, String> defaultConfigMap = getDefaultConfigMap();
		// Save the Large Image
		// get specifications
		int largeImageHeight = getValue("largeimageheight", moduleConfigMap,
				defaultConfigMap);
		int largeImageWidth = getValue("largeimagewidth", moduleConfigMap,
				defaultConfigMap);

		/** Original Image **/
		// get original image size
		BufferedImage originalImage = ImageIO.read(image);
		int width = originalImage.getWidth();
		int height = originalImage.getHeight();

		/*** determine if image can be cropped ***/
		determineCropeable(width, largeImageWidth, height, largeImageHeight);

		/*** determine crop area calculation baseline ***/
		this.determineBaseline(width, height);

		determineCropArea(width, largeImageWidth, height, largeImageHeight);
	}

	public void uploadProductImages(File image, String imageName,
			String imageContentType, Product product,
			Map<String, String> moduleConfigMap) throws Exception {
		FileModule fh = (FileModule) SpringUtil.getBean("localfile");
		Configuration conf = PropertiesUtil.getConfiguration();
		Map<String, String> defaultConfigMap = getDefaultConfigMap();
		// Save the Large Image
		// get specifications
		int largeImageHeight = getValue("largeimageheight", moduleConfigMap,
				defaultConfigMap);
		int largeImageWidth = getValue("largeimagewidth", moduleConfigMap,
				defaultConfigMap);

		/** Original Image **/
		// get original image size
		BufferedImage originalImage = ImageIO.read(image);
		int width = originalImage.getWidth();
		int height = originalImage.getHeight();

		// original image
		StringBuffer imgName = new StringBuffer();
		imgName.append(product.getProductId()).append("-").append(imageName);
		// store renamed image in products_image column
		product.setProductImage(imgName.toString());

		// upload anyway
		fh.uploadFile(product.getMerchantId(), "core.product.image", image,
				imgName.toString(), imageContentType);

		/*** determine if image can be cropped ***/
		determineCropeable(width, largeImageWidth, height, largeImageHeight);

		product.setProductImageCrop(this.isCropeable());

		/*** determine crop area calculation baseline ***/
		this.determineBaseline(width, height);

		// Save the small Image
		int smallImageHeight = getValue("smallimageheight", moduleConfigMap,
				defaultConfigMap);
		int smallImageWidth = getValue("smallimagewidth", moduleConfigMap,
				defaultConfigMap);
		File resizedSmallImage = resizeImage(originalImage, smallImageWidth,
				smallImageHeight);
		StringBuffer smallImgName = new StringBuffer();
		smallImgName.append(conf.getString("core.product.image.small.prefix"))
				.append("-").append(imgName.toString());
		fh.uploadFile(product.getMerchantId(), "core.product.image",
				resizedSmallImage, smallImgName.toString(), imageContentType);
		resizedSmallImage.delete();

		// Save large Image
		int listingImageHeight = getValue("largeimageheight", moduleConfigMap,
				defaultConfigMap);
		int listingImageWidth = getValue("largeimagewidth", moduleConfigMap,
				defaultConfigMap);
		File resizedListingImage = resizeImage(originalImage,
				listingImageWidth, listingImageHeight);
		StringBuffer largeImgName = new StringBuffer();
		largeImgName.append(conf.getString("core.product.image.large.prefix"))
				.append("-").append(imgName.toString());

		fh.uploadFile(product.getMerchantId(), "core.product.image",
				resizedListingImage, largeImgName.toString(), imageContentType);
		resizedListingImage.delete();

		determineCropArea(width, largeImageWidth, height, largeImageHeight);

	}

	public void uploadCropedProductImages(File image, String imageName,
			String imageContentType, Product product,
			Map<String, String> moduleConfigMap) throws Exception {
		FileModule fh = (FileModule) SpringUtil.getBean("localfile");
		Configuration conf = PropertiesUtil.getConfiguration();
		Map<String, String> defaultConfigMap = getDefaultConfigMap();

		/** Original Image **/
		// get original image size
		BufferedImage originalImage = ImageIO.read(image);

		// Save the small Image
		int smallImageHeight = getValue("smallimageheight", moduleConfigMap,
				defaultConfigMap);
		int smallImageWidth = getValue("smallimagewidth", moduleConfigMap,
				defaultConfigMap);
		File resizedSmallImage = resizeImage(originalImage, smallImageWidth,
				smallImageHeight);
		StringBuffer smallImgName = new StringBuffer();
		smallImgName.append(conf.getString("core.product.image.small.prefix"))
				.append("-").append(product.getProductImage());
		fh.uploadFile(product.getMerchantId(), "core.product.image",
				resizedSmallImage, smallImgName.toString(), imageContentType);
		resizedSmallImage.delete();

		// Save large Image
		int listingImageHeight = getValue("largeimageheight", moduleConfigMap,
				defaultConfigMap);
		int listingImageWidth = getValue("largeimagewidth", moduleConfigMap,
				defaultConfigMap);
		File resizedListingImage = resizeImage(originalImage,
				listingImageWidth, listingImageHeight);
		StringBuffer largeImgName = new StringBuffer();
		largeImgName.append(conf.getString("core.product.image.large.prefix"))
				.append("-").append(product.getProductImage());

		fh.uploadFile(product.getMerchantId(), "core.product.image",
				resizedListingImage, largeImgName.toString(), imageContentType);
		resizedListingImage.delete();

	}

	private void determineCropeable(int width, int specificationsWidth,
			int height, int specificationsHeight) {
		/*** determine if image can be cropped ***/
		// height
		int y = height - specificationsHeight;
		// width
		int x = width - specificationsWidth;

		if (x < 0 || y < 0) {
			cropeable = false;
		}

		if (x == 0 && y == 0) {
			cropeable = false;
		}
	}

	private void determineBaseline(int width, int height) {
		/*** determine crop area calculation baseline ***/
		if (width < height) {
			this.setCropeBaseline(0);// width
		}
		if (height < width) {
			this.setCropeBaseline(1);// height
		}
		if (width == height) {
			this.setCropeBaseline(0);
		}
	}

	private void determineCropArea(int width, int specificationsWidth,
			int height, int specificationsHeight) {

		cropAreaWidth = specificationsWidth;
		cropAreaHeight = specificationsHeight;

		// crop factor
		double factor = 1;
		if (this.getCropeBaseline() == 0) {// width
			factor = new Integer(width).doubleValue()
					/ new Integer(specificationsWidth).doubleValue();
		} else {// height
			factor = new Integer(height).doubleValue()
					/ new Integer(specificationsHeight).doubleValue();
		}

		double w = factor * specificationsWidth;
		double h = factor * specificationsHeight;

		cropAreaWidth = (int) w;
		cropAreaHeight = (int) h;

		/*
		 * if(factor>1) { //determine croping section for(double
		 * i=factor;i>1;i--) { //multiply specifications by factor int newWidth
		 * = (int)(i * specificationsWidth); int newHeight = (int)(i *
		 * specificationsHeight); //check if new size >= original image
		 * if(width>=newWidth && height>=newHeight) { cropAreaWidth = newWidth;
		 * cropAreaHeight = newHeight; break; } } }
		 */

	}

	public File getCroppedImage(File originalFile, int x1, int y1, int width,
			int height) throws Exception {

		BufferedImage image = ImageIO.read(originalFile);
		BufferedImage out = image.getSubimage(x1, y1, width, height);
		File tempFile = File.createTempFile("temp", ".jpg");
		tempFile.deleteOnExit();
		ImageIO.write(out, "jpg", tempFile);
		return tempFile;
	}

	public Integer getValue(String key, Map<String, String> moduleConfigMap,
			Map<String, String> defaultConfigMap) {
		if (moduleConfigMap.get(key) != null) {
			return Integer.valueOf(moduleConfigMap.get(key));
		} else {
			return Integer.valueOf(defaultConfigMap.get(key));
		}
	}

	public Map<String, String> getDefaultConfigMap() {
		Configuration conf = PropertiesUtil.getConfiguration();
		Map<String, String> defaultConfigMap = new HashMap<String, String>();
		defaultConfigMap.put("largeimageheight", conf
				.getString("core.product.config.large.image.height"));
		defaultConfigMap.put("largeimagewidth", conf
				.getString("core.product.config.large.image.width"));
		defaultConfigMap.put("smallimageheight", conf
				.getString("core.product.config.small.image.height"));
		defaultConfigMap.put("smallimagewidth", conf
				.getString("core.product.config.small.image.width"));
		defaultConfigMap.put("listingimageheight", conf
				.getString("core.product.config.large.image.height"));
		defaultConfigMap.put("listingimagewidth", conf
				.getString("core.product.config.large.image.width"));
		return defaultConfigMap;
	}

	public BufferedImage resize(BufferedImage image, int width, int height) {
		int type = image.getType() == 0 ? BufferedImage.TYPE_INT_ARGB : image
				.getType();
		BufferedImage resizedImage = new BufferedImage(width, height, type);
		Graphics2D g = resizedImage.createGraphics();
		g.setComposite(AlphaComposite.Src);
		g.setRenderingHint(RenderingHints.KEY_INTERPOLATION,
				RenderingHints.VALUE_INTERPOLATION_BILINEAR);
		g.setRenderingHint(RenderingHints.KEY_RENDERING,
				RenderingHints.VALUE_RENDER_QUALITY);
		g.setRenderingHint(RenderingHints.KEY_ANTIALIASING,
				RenderingHints.VALUE_ANTIALIAS_ON);
		g.drawImage(image, 0, 0, width, height, null);
		g.dispose();
		return resizedImage;
	}

	public BufferedImage blurImage(BufferedImage image) {
		float ninth = 1.0f / 9.0f;
		float[] blurKernel = { ninth, ninth, ninth, ninth, ninth, ninth, ninth,
				ninth, ninth };
		Map<Key, Object> map = new HashMap<Key, Object>();
		map.put(RenderingHints.KEY_INTERPOLATION,
				RenderingHints.VALUE_INTERPOLATION_BILINEAR);
		map.put(RenderingHints.KEY_RENDERING,
				RenderingHints.VALUE_RENDER_QUALITY);
		map.put(RenderingHints.KEY_ANTIALIASING,
				RenderingHints.VALUE_ANTIALIAS_ON);
		RenderingHints hints = new RenderingHints(map);
		BufferedImageOp op = new ConvolveOp(new Kernel(3, 3, blurKernel),
				ConvolveOp.EDGE_NO_OP, hints);
		return op.filter(image, null);
	}

	private BufferedImage createCompatibleImage(BufferedImage image) {
		GraphicsConfiguration gc = BufferedImageGraphicsConfig.getConfig(image);
		int w = image.getWidth();
		int h = image.getHeight();
		BufferedImage result = gc.createCompatibleImage(w, h,
				Transparency.TRANSLUCENT);
		Graphics2D g2 = result.createGraphics();
		g2.drawRenderedImage(image, null);
		g2.dispose();
		return result;
	}

	// To Shrink
	/*
	 * private BufferedImage shrinkResize(BufferedImage image, int width,int
	 * height) { image = createCompatibleImage(image); image = resize(image,
	 * 100, 100); image = blurImage(image); image = resize(image, width,
	 * height); return image; }
	 */

	public File resizeImage(BufferedImage image, int width, int height)
			throws Exception {
		// BufferedImage readImage = ImageIO.read(image);

		BufferedImage resizedImage = resize(image, width, height);
		File temp = File.createTempFile("temp", ".png");
		ImageIO.write(resizedImage, "png", temp);
		return temp;
	}

	public boolean isCropeable() {
		return cropeable;
	}

	public void setCropeable(boolean cropeable) {
		this.cropeable = cropeable;
	}

}



```
