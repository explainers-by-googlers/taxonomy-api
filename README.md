# Explainer for the Taxonomy API

**Instructions for the explainer author: Search for "todo" in this repository and update all the instances as appropriate. For the instances in `index.bs`, update the repository name, but you can leave the rest until you start the specification. Then delete the TODOs and this block of text.**

This proposal is an early design sketch by the Chrome Built-in AI Team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Google Chrome Team

## Participate

- [Discussion forum](https://github.com/explainers-by-googlers/taxonomy-api/issues)


## Introduction

The **Taxonomy API** is a specialized, high-performance JavaScript API designed to classify input text strictly into a predefined taxonomy (specifically **IAB Content Taxonomy V3.1**).

Unlike the generic [Prompt API](https://github.com/webmachinelearning/prompt-api), which exposes general-purpose Large Language Models (LLMs), this API is backed by a dedicated "tiny model" optimized solely for classification. This architecture allows for drastic reductions in inference latency and resource consumption, making it feasible for real-time synchronous operations like ad auctions, where milliseconds determine viability.

## Goals

The primary goal is to provide a privacy-preserving, on-device mechanism for Ad Tech partners (e.g., Google Ads, Prebid.js) to classify page content.

*   **Low Latency:** Provide classification results fast enough to participate in real-time bidding flows.
*   **Privacy:** Keep page content processing entirely local to the device; no text is sent to a server for classification.
*   **Standardization:** Adherence to the IAB Content Taxonomy V3.1 standard.
*   **Efficiency:** Minimize CPU and memory impact using a specialized model.

## Non-goals

*   **Custom Taxonomies:** Support for user-defined categories or custom models is **out of scope** for the initial experiment. We may revisit this based on ecosystem demand.
*   **Generic NLP:** This API is not intended for summarization, translation, or sentiment analysis.
*   **Human-Readable Strings:** The API returns stable IAB Unique IDs. Mapping these to human-readable names (and localization) is the responsibility of the developer.

## Use cases

### Use case 1: Real-time Contextual Advertising

Ad scripts executing on a publisher's page need to determine the topic of the current content immediately to bid on relevant ads. Currently, this often requires sending page text to a third-party server (high latency, privacy risk) or loading heavy JS libraries. This API allows the script to classify the content locally and inject the category IDs into the ad request before the auction closes.

### Use case 2: Brand Safety Verification

Publishers and advertisers need to verify that content does not fall into sensitive categories (e.g., "Hate Speech" or "Adult Content") before rendering ads. The Taxonomy API allows a script to quickly "gut check" the content against the IAB standard taxonomy to prevent brand suitability violations without leaking the page content to external verifiers.

## Potential Solution

We propose a new interface, `Taxonomizer`, which exposes the classification capabilities.

```js
// 1. Check availability
// Returns: "available", "downloadable", "downloading" or "unavailable"
const status = await Taxonomizer.availability();

if (status == "available" || status == “downloadable”) {
  // 2. Create the categorizer. If status is "downloadable", triggers the model download.
  // 'iab-taxonomy-v3.1' is the implicit default for the experiment.
  const taxo = await Taxonomizer.create();

  // 3. Classify content
  const textContent = document.body.innerText; 
  
  // Returns a flat list of categories that met a confidence threshold.
  const categories = await taxo.categorize(textContent);

  // Output: Simple flat array of IDs and scores, sorted by confidence.
  console.log(categories);
  
  /* Example Output (using IAB V3.1 Unique IDs): 
   [
    { id: "602", confidence: 0.98 }, // Represents "Consumer Electronics"
    { id: "597", confidence: 0.95 }, // Represents "Technology & Computing"
    { id: "45",  confidence: 0.82 }  // Represents "Automotive" 
  ]
  */
}
```



## Detailed design discussion

### Specialized Model vs. Generic Prompt API

A generic LLM (like Gemini Nano via the Prompt API) can classify text, but it is "overkill" for this task. It requires significant memory, drains more battery, and has higher latency.
By using a smaller model trained specifically on the IAB dataset, we can achieve high accuracy with a fraction of the resources, enabling the API to be used more aggressively on mobile devices and in performance-critical paths.

### IAB V3.1 and ID-based Outputs

To ensure stability and reduce API bloat, the API returns the **Unique ID** (string) defined in the IAB V3.1 spec, rather than the category name.
*   **Stability:** Names might change or be localized; IDs remain constant.
*   **Size:** Reduces the memory footprint of the result object.
*   **Flexibility:** Developers can map IDs to their own internal naming conventions or preferred languages.

### Ergonomics and Resource Management

To keep the initial implementation lightweight and focused on the core value proposition, we will NOT support the following ergonomic features UNLESS they are trivial to implement in chrome:

1.  **Cancellation (`AbortSignal`):** Allowed in `categorize()` to stop processing long text if the user navigates away or the ad auction times out.
2.  **Resource Cleanup (`destroy()`):** A method to explicitly free up the model memory. However, the API is designed to allow parallel repeated usage (a created `taxonomizer` instance can be used on different inputs multiple times).
3.  **Download Progress:** Standard events to monitor the download of the model weights if `availability` is `after-download`.
4.  **Quota Management:** We will not expose a complex token counting API (`measureInputQuota`). Instead, if the input text exceeds the model's context window, the `categorize()` method will simply throw a standard Error.
5.  **Streaming:** Given that classification is an atomic operation (the result is a set of categories, not a generated sentence), we do not plan to support streaming outputs.
6.  **Multi-lingual:** The model will only support English.

## Considered alternatives

### Server-side Classification

*   **Approach:** Send `document.body.innerText` to an ad-tech server.
*   **Pros:** Access to massive models; easy to update.
*   **Cons:** Extremely high privacy risk (sending user browsing data); high latency; bandwidth costs.

### Client-side WASM/JS Libraries

*   **Approach:** Bundle a TensorFlow.js or ONNX model in the website's JavaScript.
*   **Pros:** Works in all browsers today.
*   **Cons:** Increases initial page load size (megabytes of weights); parsing JS/WASM is slower than native execution; difficult to cache models across different websites (each site re-downloads the library).

## Security and Privacy Considerations

*   **Local Execution:** No user text leaves the device.
*   **Fingerprinting:** As with any API backed by hardware acceleration or specific model versions, there is a risk of fingerprinting based on inference speed or minute differences in numerical precision. We will mitigate this by standardizing the model weights and precision across the specific browser version.
*   **Updates:** Model updates are managed by the browser component updater, ensuring security patches and taxonomy version consistency are applied automatically.

## Stakeholder Feedback / Opposition

- **Ad Tech Partners (Google Ads, Prebid):** Expressed strong interest in a local solution to reduce latency and server costs.
- **Publishers:** Interested in better ad targeting but concerned about the performance impact of running inference on the main thread (we generally recommend using `await` in a non-blocking manner or running in a Worker).
