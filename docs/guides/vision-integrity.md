## Vision and Image Integrity

The **Vision and Image Integrity** module within the `simpleaudit` framework provides specialized tools for auditing multimodal large language models (LLMs). As multimodal capabilities expand, the ability of models to process, interpret, and generate visual data introduces new safety vectors. This module focuses on two critical areas: **Image Integrity** (ensuring the model correctly perceives and describes visual inputs without hallucination) and **Visual Safety** (ensuring the model refuses to process, generate, or discuss harmful visual content).

### Overview

In a standard text-only audit, safety is often measured by the model's refusal to generate harmful text. In multimodal contexts, the attack surface expands. A model might be safe in text but vulnerable to:
1.  **Visual Hallucination:** Describing objects, text, or actions in an image that do not exist.
2.  **Adversarial Image Inputs:** Images containing hidden text, steganographic payloads, or adversarial perturbations that trick the model into bypassing safety filters.
3.  **Unsafe Visual Content:** Generating or describing images that contain violence, nudity, or other prohibited content.

The `simpleaudit` library addresses these challenges through a set of test cases and evaluation metrics designed to probe the visual reasoning and safety boundaries of multimodal models.

### Core Components

The functionality is primarily housed in the `simpleaudit.modules.vision` package. The key components include:

*   `VisionIntegrityAuditor`: A class responsible for executing integrity tests, such as verifying object detection accuracy and text extraction from images.
*   `VisualSafetyAuditor`: A class dedicated to safety testing, including refusal checks for harmful image generation requests and detection of unsafe visual content in model outputs.
*   `ImagePreprocessor`: A utility class that handles image loading, resizing, and encoding into formats compatible with the target model's vision encoder.

### VisionIntegrityAuditor

The `VisionIntegrityAuditor` class is used to verify that the model's visual perception aligns with ground truth. This is crucial for applications where factual accuracy regarding visual data is paramount.

#### Class Definition

```python
class VisionIntegrityAuditor:
    def __init__(self, model_client, image_loader=None):
        """
        Initializes the Vision Integrity Auditor.

        Args:
            model_client: An instance of a multimodal LLM client (e.g., OpenAI, Anthropic, or local HuggingFace model).
            image_loader: Optional custom loader for images. Defaults to standard PIL-based loading.
        """
        pass

    def test_object_recognition(self, image_path, expected_objects, tolerance=0.8):
        """
        Tests if the model correctly identifies objects in an image.

        Args:
            image_path: Path to the image file.
            expected_objects: List of strings representing objects that should be present.
            tolerance: Float between 0 and 1 indicating the acceptable error rate.

        Returns:
            dict: A dictionary containing 'passed' (bool), 'detected_objects' (list), and 'missing_objects' (list).
        """
        pass

    def test_text_extraction(self, image_path, expected_text):
        """
        Tests the model's ability to extract text from an image (OCR capability).

        Args:
            image_path: Path to the image containing text.
            expected_text: The ground truth text present in the image.

        Returns:
            dict: A dictionary containing 'passed' (bool), 'extracted_text' (str), and 'similarity_score' (float).
        """
        pass
```

#### Example Usage

```python
from simpleaudit.modules.vision import VisionIntegrityAuditor
from simpleaudit.clients import OpenAIClient

# Initialize the model client
client = OpenAIClient(model="gpt-4o")

# Initialize the auditor
auditor = VisionIntegrityAuditor(model_client=client)

# Test object recognition
result = auditor.test_object_recognition(
    image_path="assets/test_images/cat_dog.jpg",
    expected_objects=["cat", "dog", "park"]
)

if result['passed']:
    print("Integrity Check Passed: Model correctly identified all expected objects.")
else:
    print(f"Integrity Check Failed. Missing objects: {result['missing_objects']}")
```

### VisualSafetyAuditor

The `VisualSafetyAuditor` is designed to probe the safety boundaries of multimodal models. It includes tests for refusal behavior when prompted with unsafe visual contexts and checks for the generation of harmful content.

#### Class Definition

```python
class VisualSafetyAuditor:
    def __init__(self, model_client, safety_policy=None):
        """
        Initializes the Visual Safety Auditor.

        Args:
            model_client: An instance of a multimodal LLM client.
            safety_policy: Optional custom safety policy object. Defaults to the library's standard safety policy.
        """
        pass

    def test_unsafe_image_refusal(self, image_path, prompt):
        """
        Tests if the model refuses to process or describe an unsafe image.

        Args:
            image_path: Path to an image containing unsafe content (e.g., violence, explicit material).
            prompt: The user prompt associated with the image.

        Returns:
            dict: A dictionary containing 'refused' (bool) and 'response' (str).
        """
        pass

    def test_visual_hallucination_safety(self, image_path, adversarial_prompt):
        """
        Tests if the model is susceptible to adversarial prompts that exploit visual hallucinations
        to bypass safety filters.

        Args:
            image_path: Path to a benign image.
            adversarial_prompt: A prompt designed to trick the model into hallucinating harmful content.

        Returns:
            dict: A dictionary containing 'safe' (bool) and 'analysis' (str).
        """
        pass
```

#### Example Usage

```python
from simpleaudit.modules.vision import VisualSafetyAuditor
from simpleaudit.clients import AnthropicClient

# Initialize the model client
client = AnthropicClient(model="claude-3-opus")

# Initialize the safety auditor
safety_auditor = VisualSafetyAuditor(model_client=client)

# Test refusal of unsafe content
safety_result = safety_auditor.test_unsafe_image_refusal(
    image_path="assets/test_images/unsafe_content.jpg",
    prompt="Describe the actions happening in this image in detail."
)

if safety_result['refused']:
    print("Safety Check Passed: Model correctly refused to process unsafe content.")
else:
    print("Safety Check Failed: Model engaged with unsafe content.")
    print(f"Response: {safety_result['response']}")
```

### Configuration and Setup

To use the vision modules, ensure that the necessary image processing libraries are installed. The `simpleaudit` library relies on `Pillow` for image manipulation and `base64` for encoding images into API requests.

#### Installation

```bash
pip install simpleaudit
pip install Pillow
```

#### Supported Formats

The `ImagePreprocessor` supports the following common image formats:
*   JPEG (`.jpg`, `.jpeg`)
*   PNG (`.png`)
*   WebP (`.webp`)
*   GIF (`.gif`) - Only the first frame is processed.

#### Resizing Behavior

By default, images are resized to a maximum dimension of 1024x1024 pixels to comply with most multimodal model input limits. This behavior can be customized by passing a custom `image_loader` to the auditor classes.

### Best Practices for Auditing

1.  **Diverse Test Sets:** When testing integrity, use a diverse set of images including complex scenes, low-resolution images, and images with occluded objects.
2.  **Adversarial Prompts:** For safety testing, employ adversarial prompts that attempt to exploit the model's visual hallucinations. For example, asking the model to "imagine" what is happening in a blank image to see if it generates harmful content.
3.  **Ground Truth Verification:** Always maintain a verified ground truth for your test images. Incorrect ground truth data will lead to false positives in integrity testing.
4.  **Model Specificity:** Different multimodal models have different strengths and weaknesses. A model that excels in object recognition may be more susceptible to visual hallucinations in text extraction. Tailor your audit strategy to the specific model being tested.

### Limitations

*   **API Dependencies:** The accuracy of the audit is dependent on the underlying model's capabilities. `simpleaudit` provides the framework for testing, but the results are only as good as the model being audited.
*   **Image Quality:** Low-quality or corrupted images may lead to inconclusive results. Ensure test images are of sufficient quality.
*   **Latency:** Multimodal API calls can be slower than text-only calls. Large-scale audits may require significant time and API credits.

### Conclusion

The **Vision and Image Integrity** module in `simpleaudit` provides a robust foundation for auditing multimodal models. By combining integrity checks with safety probes, developers can gain confidence in the reliability and safety of their AI systems when handling visual data. As multimodal models continue to evolve, this module will be updated to support new safety vectors and testing methodologies.

### See Also

*   [Cross-Judging and Validation](cross-judging.md)
*   [Architecture](architecture.md)
*   [Results and Visualization](results-visualization.md)
