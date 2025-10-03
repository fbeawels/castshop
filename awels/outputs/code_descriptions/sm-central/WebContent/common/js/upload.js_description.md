# upload.js

## Review

## 1. Summary  
This snippet implements a **file‑upload progress bar** in vanilla JavaScript.  
- **Purpose**: To periodically query an `UploadMonitor` object for the current upload status, update a UI progress bar, and disable/enable form controls while an upload is in progress.  
- **Key components**  
  - `refreshProgress` – kick‑off routine that requests upload status.  
  - `updateProgress` – callback that receives the status, calculates a percentage, updates the UI, and schedules the next refresh.  
  - `startProgress` – UI helper that shows the progress bar and disables the submit button before the first status poll.  
  - (Commented out) `startPostProgress`, `updatePostProgress`, `refreshPostProgress` – similar logic for a post‑upload processing step.  
- **Design patterns & libraries**  
  - The code relies on a **callback‑based polling** approach.  
  - No external framework is used; the code assumes the presence of global objects `UploadMonitor` and (in the commented section) `ProcessMonitor`.  
  - The UI is manipulated directly via `document.getElementById`.

---

## 2. Detailed Description  
1. **Initialization** – `startProgress` is the entry point. It shows the progress bar container and disables the submit button.  
2. **Polling loop** – `refreshProgress` invokes `UploadMonitor.getUploadInfo(updateProgress)`.  
   - `UploadMonitor` is expected to perform an asynchronous request (e.g., XHR or fetch) and invoke the supplied callback with an `uploadInfo` object that contains:  
     - `bytesRead` – how many bytes have been uploaded so far.  
     - `totalSize` – total number of bytes to be sent.  
     - `fileIndex` – index of the file in a multi‑file upload.  
3. **UI update** – `updateProgress` calculates the percent complete, updates the UI (progress bar text/width), disables the file inputs, and schedules the next poll via `window.setTimeout('refreshProgress()', 1000)`.  
4. **Termination** – The polling loop never stops in the current code; it will continue until the page is unloaded or the callback is overridden.  
5. **Commented post‑processing** – The same pattern is intended for a secondary progress indicator after the upload is complete.

### Assumptions & Constraints  
- Elements with IDs `uploadbutton`, `file1–file4`, `progressBar`, `uploadproduct_button_label_submit`, etc., exist in the DOM.  
- `UploadMonitor` (and `ProcessMonitor` if uncommented) provide the described API and are globally available.  
- The upload status can be polled at a one‑second interval without affecting performance.  
- The upload will finish before the polling interval causes race conditions (e.g., the upload might finish in the middle of a call).

### Architectural Choices  
- **Polling over event callbacks**: Simpler to implement but can be wasteful; a push‑based approach (e.g., `XMLHttpRequest` upload progress events or WebSockets) would reduce overhead.  
- **String arguments to `setTimeout`**: The code uses `window.setTimeout('refreshProgress()', 1000)` which evaluates a string; this is slower, harder to debug, and considered a bad practice compared to passing a function reference.  

---

## 3. Functions/Methods  

| Function | Purpose | Inputs | Outputs | Side‑effects |
|---|---|---|---|---|
| `refreshProgress()` | Initiates a status query on the upload monitor. | None | None (async callback) | Calls `UploadMonitor.getUploadInfo` |
| `updateProgress(uploadInfo)` | Callback that receives progress data and updates the UI. | `uploadInfo` object (`bytesRead`, `totalSize`, `fileIndex`) | `true` | Disables submit button, updates progress bar, schedules next poll |
| `startProgress()` | Prepares UI for upload start. | None | `true` | Shows progress bar, disables submit button |
| *(commented)* `startPostProgress()` | Shows post‑upload progress UI. | None | `true` | Shows process bar, sets initial text, starts poll |
| *(commented)* `updatePostProgress()` | Callback for post‑upload progress. | `processInfo` (assumed) | `true` | Updates UI, re‑enables upload button on completion |
| *(commented)* `refreshPostProgress()` | Triggers post‑upload polling. | None | None | Calls `ProcessMonitor.getProcessInfo` |

**Reusable utilities** – None are abstracted; all logic is tightly coupled to specific element IDs.  

---

## 4. Dependencies  
| Dependency | Type | Notes |
|---|---|---|
| `UploadMonitor` | Third‑party global | Must expose `getUploadInfo(callback)` |
| `ProcessMonitor` | Third‑party global | (commented out) Must expose `getProcessInfo(callback)` |
| Browser DOM APIs (`document.getElementById`, `window.setTimeout`) | Standard | No polyfills required for modern browsers |
| CSS classes/IDs (`uploadbutton`, `progressBar`, etc.) | Application | Assumes certain markup exists |

No external libraries (jQuery, React, etc.) are used.

---

## 5. Additional Notes  

### Code‑Quality & Maintainability  
- **String `setTimeout`**: Replace with function references (`setTimeout(refreshProgress, 1000)`).  
- **Hard‑coded element IDs**: Consider passing a configuration object or using data attributes to make the component reusable.  
- **Disabled inputs**: The code disables several file inputs but these lines are commented out; if multiple files are allowed, you’d want to disable each accordingly.  
- **No cleanup**: The polling loop never stops. If the page navigates away or the user aborts the upload, `setTimeout` continues to fire. A clear interval/timeout mechanism or a cancellation token would prevent memory leaks.  
- **Error handling**: There’s no fallback for network errors, `UploadMonitor` failures, or malformed `uploadInfo`.  
- **Accessibility**: The progress bar’s width is set via inline styles, but there’s no ARIA role or `aria-valuenow`/`aria-valuemin`/`aria-valuemax` attributes.  

### Edge Cases  
- **Large files**: The percent calculation may be imprecise if `bytesRead` is an integer but `totalSize` is huge (though JS numbers can handle up to 2^53).  
- **Concurrent uploads**: If two uploads start simultaneously, the same global `progressBar` will be overwritten.  
- **Network hiccups**: If `UploadMonitor` stalls or returns stale data, the UI will freeze until the next poll.  
- **User abort**: There’s no way for the user to cancel the upload; disabling the button alone does not abort the network request.  

### Potential Enhancements  
1. **Event‑driven progress** – Use `XMLHttpRequest.upload.onprogress` or the Fetch API’s `ReadableStream` to react to progress events instead of polling.  
2. **Modular component** – Wrap the logic in an ES6 class or a function factory to accept options (container selector, poll interval, etc.).  
3. **Promises / async–await** – Convert callback‑style polling into a promise chain for cleaner control flow.  
4. **AbortController** – Expose a cancel API that aborts the upload request and stops polling.  
5. **Accessibility** – Add ARIA roles, `aria-live` region for progress updates, and support for screen readers.  
6. **Styling** – Separate presentation logic from script; use CSS classes instead of inline `style.width`.  
7. **Testing** – Provide unit tests for `updateProgress` by mocking `UploadMonitor` and checking DOM changes.  

---  

**Bottom line:**  
The code achieves its primary goal of displaying a progress bar for a single file upload, but it would benefit from modern JavaScript practices, better separation of concerns, robust error handling, and accessibility considerations. Refactoring to an event‑driven model and encapsulating the logic into a reusable component would greatly improve maintainability and scalability.

## Code Critique



## Code Preview

```javascript
/**
* Updates the progress bar during file upload
**/
function refreshProgress()
{
    UploadMonitor.getUploadInfo(updateProgress);
}

function updateProgress(uploadInfo)
{

        document.getElementById('uploadbutton').disabled = true;
        //document.getElementById('file1').disabled = true;
        //document.getElementById('file2').disabled = true;
        //document.getElementById('file3').disabled = true;
        //document.getElementById('file4').disabled = true;

        var fileIndex = uploadInfo.fileIndex;

        var progressPercent = Math.ceil((uploadInfo.bytesRead / uploadInfo.totalSize) * 100);

        //document.getElementById('progressBarText').innerHTML = 'upload in progress: ' + progressPercent + '%';
        //document.getElementById('progressBarBoxContent').style.width = parseInt(progressPercent * 3.5) + 'px';


	    //myJsProgressBarHandler.setPercentage('element-progress',progressPercent);


        window.setTimeout('refreshProgress()', 1000);



    	return true;
}

function startProgress()
{

    //myJsProgressBarHandler.setPercentage('element-progress','0');
    document.getElementById('progressBar').style.display = 'block';
    document.getElementById('uploadproduct_button_label_submit').disabled = true;

    // wait a little while to make sure the upload has started ..
    //window.setTimeout("refreshProgress()", 1500);
    return true;

}

/**
function startPostProgress()
{
	document.getElementById('processBar').style.display = 'block';
	document.getElementById('processText').innerHTML = 'Processing file...';
	window.setTimeout("refreshPostProgress()", 1500);
      return true;


}

function updatePostProgress()
{
    if (!processInfo.done)
    {
    	window.setTimeout('refreshPostProgress()', 1000);
    } else {
    	document.getElementById('uploadbutton').disabled = false;
    	document.getElementById('processText').innerHTML = 'Done';
    }
    return true;

}

function refreshPostProgress()
{
	ProcessMonitor.getProcessInfo(updatePostProgress);
}
**/



```
