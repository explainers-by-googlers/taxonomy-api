# Explainer for the Taxonomy API

**Instructions for the explainer author: Search for "todo" in this repository and update all the instances as appropriate. For the instances in `index.bs`, update the repository name, but you can leave the rest until you start the specification. Then delete the TODOs and this block of text.**

This proposal describes a specialized, high-performance JavaScript API designed to enable Dynamic Content Classification entirely on-device. This work is a tentative and early design sketch by the Chrome Built-in AI Team to solicit feedback on privacy-preserving semantic utilities.

## Proponents

- Google Chrome Team

## Participate

- [Discussion forum](https://github.com/explainers-by-googlers/taxonomy-api/issues)


## Introduction


The **Taxonomy API** is a purpose-built interface that allows browsers to categorize text content into a structured, interoperable schema without ever exposing that content over the network.

Unlike general-purpose Large Language Models (LLMs) and associated APIs like the [Prompt API](https://github.com/webmachinelearning/prompt-api), this API utilizes a dedicated, on-device expert model optimized for high-speed classification, entirely on-device. By moving semantic understanding from the cloud to the client, developers can ensure **data sovereignty** and **stronger privacy**: raw text, including dynamic or authenticated content, never leaves the user’s device. This architecture makes high-fidelity classification feasible for latency-sensitive applications where milliseconds determine viability.

## Goals

*   **Privacy-First Design & Statelessness**: Ensure page content processing is strictly local.
    *   **Data sovereignty**: Raw text never leaves the browser, preventing the leakage of sensitive user data to third-party servers.
    *   **No Memory/State**: The API is stateless by design. It does not "learn" from user behavior, nor does it store historical classification data. Each request is a discrete, isolated event: the model takes a text input, returns a classification, and retains no record of the interaction.
*   **Contextual Relevance**: Empower developers to understand the high-level context of a page, or any piece of content, dynamically to provide more relevant, or lower-friction, user experiences.
*   **Low Latency:** Provide classification results in milliseconds to power real-time, interactive use cases that require immediate responsiveness.
*   **Ecosystem interoperability:** Adherence to standardized taxonomies to produce consistent signals that can be used across the web ecosystem.
*   **Performance Efficiency**: Minimize CPU and memory impact by using a specialized expert model rather than a resource-heavy Large Language Model (LLM).

## Non-goals

*   **Generic NLP:** This API is not intended for summarization, translation, or sentiment analysis.
*   **Human-Readable Strings:** The API returns stable Unique IDs. Mapping these to human-readable names (and localization) is the responsibility of the developer.
*   **Exclusive Taxonomies:** While the initial experiment intends to use the IAB Content Taxonomy V3.1 which we understand to have broad applications and interops appeal, we explicitly aim to design the API so that it can support additional taxonomies depending on demand and the impact these would unlock. To that extent, we welcome feedback on use cases that would require alternative classification schemas, and pointers to other popular taxonomies for consideration.

## Use cases

### Use case 1: Intelligent Accessibility & Personalization.

Developers can use the API to automatically detect the topic of a document to adjust the browsing environment. For example: 
 - Cognitive Load Reduction: An extension could group open tabs by topic (e.g., "Automotive," "Cooking") to help users organize their research.
 - Dynamic UI: A site could automatically surface relevant "Deep Dive" tools or accessibility shortcuts based on whether a user is reading a highly technical topic versus a common news article.

### Use case 2: Privacy-Preserving Contextual Signals.

Publishers can determine the broad topic of a page to provide relevant content recommendations or serve contextually-aligned advertising. Because this occurs on-device, it provides a functional replacement for more intrusive practices while maintaining the user’s privacy. This is particularly valuable for dynamic or authenticated pages where traditional server-side crawlers cannot operate.

### Use Case 3: Streamlined User Contribution

Platforms can use the API to assist users when submitting content. For instance, a forum or Q&A site could suggest the most relevant categories for a user's post in real-time, reducing manual effort and improving site organization without sending the draft to a server before the user hits "submit".

### Use Case 4: Enhanced User Safety & Protection
The API can act as a local "early warning system" for security-sensitive pages.

 - **Proactive Protection**: A browser extension could use the API to identify if a page is related to Personal Finance or high-risk transaction environments.
 - **Friction with Purpose**: Detecting these categories can trigger more thorough local heuristic checks or surfacing tailored security advice before a user interacts with sensitive fields, without needing to send the page content to a security cloud server.

## Potential Solution

We propose a new interface, `Taxonomizer`, which exposes the classification capabilities.
See the "Minimal Viable Prototype (MVP) Scope" section for important context on the current shape of the API.

```js
// 1. Check availability
// Returns: "available", "downloadable", "downloading" or "unavailable"
const status = await Taxonomizer.availability();

if (status == "available" || status == “downloadable”) {
  // 2. Create the categorizer. If status is "downloadable", triggers the model download.
  // 'iab-taxonomy-v3.1' is the temporary and implicit default for the experimental phase.
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

A generic LLM is "overkill" for classification. It requires significant memory, drains more battery, and has higher latency. By using a specialized model, trained for the taxonomy(ies) of interest, we achieve:
 - Better latency: Millisecond-level inference suitable for synchronous page-load events.
 - Better ergonomics and interops: Consistent ID-based outputs that are easier for developers to handle than unpredictable natural language strings.
 - Better device coverage and usage of resources: These smaller expert models can run on many more devices with significantly less resources (hardware, energy).

### ID-based Outputs

To ensure good ergonomics and interoperability, the API returns the **Unique ID** (string) defined in the relevant taxonomy, rather than the category name.
*   **Interoperability:** Names might change or be localized; IDs remain constant.
*   **Size:** Reduces the memory footprint of the result object.
*   **Flexibility:** Developers can map IDs to their own internal naming conventions or preferred languages.
*   **Ergonomics**: It's easier to work with IDs rather than natural language strings (i.e. category names).

### Minimal Viable Prototype (MVP) Scope

To keep the initial implementation lightweight and focused on verifying the core value proposition, we will NOT support the following ergonomic features UNLESS they are trivial to implement. Our goal is to verify, with the help of the web community, that this capability and the proposed design, can help solve compelling use cases before fully investing in a production-grade API path.

1.  **Cancellation (`AbortSignal`):** Allowed in `categorize()` to stop processing long text if the user navigates away or the ad auction times out.
2.  **Resource Cleanup (`destroy()`):** A method to explicitly free up the model memory. However, the API is designed to allow parallel repeated usage (a created `taxonomizer` instance can be used on different inputs multiple times).
3.  **Download Progress:** Standard events to monitor the download of the model weights if `availability` is `after-download`.
4.  **Quota Management:** We will not expose a complex token counting API (`measureInputQuota`). Instead, if the input text exceeds the model's context window, the `categorize()` method will simply throw a standard Error.
5.  **Streaming:** Given that classification is an atomic operation (the result is a set of categories, not a generated sentence), we do not plan to support streaming outputs.
6.  **Multi-lingual:** The model will only support English.

## Considered alternatives

### Server-side Classification

*   **Approach:** Developers send `document.body.innerText`, or other sources of raw content, to a Centralized Cloud Classification.
*   **Pros:** Access to massive models; simplified updates.
*   **Cons:** Significant privacy risk (browsing data leaves the device); high latency; increased bandwidth and infrastructure costs.

### Client-side WASM/JS Libraries

*   **Approach:** Developers bundle their own TensorFlow.js or ONNX model in their bundles.
*   **Pros:** Works in all browsers today.
*   **Cons:** Significant "page weight" (megabytes of downloads); inefficient resource usage (each site downloads its own model); slower execution compared to a native browser-optimized engine; responsibility for handling device hardware complexity; Harder for users to express their preferences over a non-standardized capability.

## Security and Privacy Considerations

*   **Local Execution & Statelessness**: No user text is ever transmitted, and no historical data is retained between calls.
*   **Fingerprinting:** We will mitigate hardware-based fingerprinting by standardizing model weights and execution precision across browser versions.
*   **Updates:** Model updates are managed by the browser component updater, ensuring security patches and taxonomy version consistency are applied automatically.
*   **Sensitive category suppression**: Browser implementation may choose to suppress specific high-risk taxonomy branches (either at the API or model level), or offer user controls over sensitive categories. This is an area that we would love to discuss further with the ecosystem given the potential implications for the viability of various use cases.

## Feedback / Interest

 - **Web Developers & Publishers**: Expressed interest in localized, low-latency contextual signals that offer a privacy-preserving alternative to cross-site tracking and server-side scraping.
 - **Community Performance Concerns**: Members of the web community have noted that while general AI (like the Prompt API) is powerful, its resource footprint is significant. The Taxonomy API is a direct response to this feedback, offering a "lean" alternative that minimizes battery and memory drain for specific, high-frequency classification tasks.
 - **Accessibility & Utility Authors**: Implicit supportive signals for easier solution toward semantic context for "Smart History", "tab management", and automated UI adaptations to help users navigate complex information environments more efficiently.
