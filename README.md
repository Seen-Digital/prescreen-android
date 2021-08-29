# PreScreen OCR Library

## Prerequisite

- Android SDK Version 21 or Higher
- Google Play Services
- `Internet` and `Camera` Permissions in Manfiest

## Installation Guide

- Add this to your `build.gradle` at the root of the project

  ```
	allprojects {
		repositories {
			...
			maven { url 'https://jitpack.io' }
		}
	}
  ```
- Add your dependency in your app or feature module as below
```
	dependencies {
		implementation 'com.github.Seen-Digital:prescreen-android:<version>'
	}
```

- Sync your gradle project and PreScreen is now available for your application

## HOW-TO

- Initialize SDK by calling `init` function with your API Key

  ```kotlin
  PreScreen.init(this, "API_KEY")
  ```

- Create the `CardImage` object which will be used as an input for OCR

  ```kotlin
  val card = CardImage(this.image!!, imageInfo.rotationDegrees)
  ```

- Call the `scanIDCard` method from `PreScreen` object class which will extract card information from the image. There are 3 required parameters.
  - `card`: Input card image
  - `resultListener`: The listener to receive recognition result. This will provide the information of cards that can be detected in `IDCardResult` object

  ```kotlin
  it.setAnalyzer(cameraExecutor) { imageProxy ->
          imageProxy.run {
            val card = CardImage(this.image!!, imageInfo.rotationDegrees)
            PreScreen.scanIDCard(card) {result ->
              this@MainActivity.displayResult(result)
              imageProxy.close()
            }
          }
        }
   ```
val error: Error?
) {
val isFrontSide: Boolean? = results?.isFrontSide
val fullImage: Bitmap? = detectResult?.fullImage
val croppedImage: Bitmap? = detectResult?.croppedImage
val texts: List<TextResult>? = results?.texts
val confidence: Float =

- The `IDCardResult` object will consist of the following fields:
  - `error`: If the recognition is successful, the `error` will be null. In case of unsuccessful scan, the `error.errorMessage` will contain the problem of the recognition.
  - `isFrontSide`: A boolean flag indicates whether the scan found the front side (`true`) or back side (`false`) of the card ID.
  - `confidence`: A value between 0.0 to 1.0 (higher values mean more likely to be an ID card).
  - `texts`: A list of OCR results. An OCR result consists of `type` and `text`.
    - `type`: Type of information. Right now, PreScreen support 3 types
      - `ID`
      - `SERIAL_NUMBER`
      - `LASER_CODE`
    - `text`: OCR text based on the `type`.
  - `fullImage`: A bitmap image of the full frame used during scanning.
  - `croppedImage`: A bitmap image of the card. This is available if `isFrontSide` is `true`.

  ```kotlin
  private fun displayResult(result: IDCardResult) {
        result.error?.run {
            resultText.text = errorMessage
        }
        result.run {
            // fullImage
            // fullImage is always available.
            if (isFrontSide != null && isFrontSide as Boolean) {
                // croppedImage
                // cropped image is only available for front side scan result.
            }
            confidenceText.text = "%.3f ".format(confidence)
            if (texts != null) {
                resultText.text = "TEXTS -> ${texts!!.joinToString("\n")}, isFrontside -> $isFrontSide"
            } else {
                resultText.text = "TEXTS -> NULL, isFrontside -> $isFrontSide"
            }
        }
    }
    ```

### Example Code

```kotlin
@androidx.camera.core.ExperimentalGetImage
class MainActivity : AppCompatActivity() {

  lateinit var resultText: TextView
  lateinit var confidenceText: TextView

  private var preview: Preview? = null
  private var imageCapture: ImageCapture? = null
  private var imageAnalyzer: ImageAnalysis? = null
  private var camera: Camera? = null
  private var cameraProvider: ProcessCameraProvider? = null
  private lateinit var cameraExecutor: ExecutorService

  companion object {
    private const val RATIO_4_3_VALUE = 4.0 / 3.0
    private const val RATIO_16_9_VALUE = 16.0 / 9.0
  }

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    PreScreen.init(this, "API_KEY")

    resultText = findViewById(R.id.resultTextView)
    confidenceText = findViewById(R.id.confidenceTextView)

    if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED) {
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        requestPermissions(arrayOf(Manifest.permission.CAMERA), 1000)
      }
    }

    cameraExecutor = Executors.newSingleThreadExecutor()
    setupCamera()
  }

  private fun setupCamera() {
    val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
    cameraProviderFuture.addListener({
      // CameraProvider
      cameraProvider = cameraProviderFuture.get()

      // Build and bind the camera use cases
      bindCameraUseCases()
    }, ContextCompat.getMainExecutor(this))
  }

  private fun bindCameraUseCases() {
    val cameraSelector = CameraSelector.Builder().requireLensFacing(CameraSelector.LENS_FACING_BACK).build()

    val metrics = DisplayMetrics().also { previewView.display.getRealMetrics(it) }

    val screenAspectRatio = aspectRatio(metrics.widthPixels, metrics.heightPixels)

    val rotation = previewView.display.rotation

    preview = Preview.Builder()
      .setTargetRotation(rotation)
      .build()

    imageCapture = ImageCapture.Builder()
      .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
      .setTargetAspectRatio(screenAspectRatio)
      .setTargetRotation(rotation)
      .build()

    imageAnalyzer = ImageAnalysis.Builder()
      .setTargetAspectRatio(screenAspectRatio)
      .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
      .build()
      .also {
        it.setAnalyzer(cameraExecutor) { imageProxy ->
          imageProxy.run {
            val card = CardImage(this.image!!, imageInfo.rotationDegrees)
            PreScreen.scanIDCard(card) {result ->
              this@MainActivity.displayResult(result)
              imageProxy.close()
            }
          }
        }
      }

    cameraProvider?.unbindAll()

    try {
      // A variable number of use-cases can be passed here -
      // camera provides access to CameraControl & CameraInfo
      camera = cameraProvider?.bindToLifecycle(
        this, cameraSelector, preview, imageAnalyzer)

      // Attach the viewfinder's surface provider to preview use case
      preview?.setSurfaceProvider(previewView.surfaceProvider)
    } catch (exc: Exception) {
    }
  }

  private fun displayResult(result: IDCardResult) {
    result.error?.run {
      resultText.text = errorMessage
    }
    result.run {
      confidenceText.text = "%.3f ".format(confidence)
      if (texts != null) {
        resultText.text = "TEXTS -> ${texts!!.joinToString("\n")}, isFrontside -> $isFrontSide"
      } else {
        resultText.text = "TEXTS -> NULL, isFrontside -> $isFrontSide"
      }
    }
  }

  private fun aspectRatio(width: Int, height: Int): Int {
    val previewRatio = max(width, height).toDouble() / min(width, height)
    if (abs(previewRatio - RATIO_4_3_VALUE) <= abs(previewRatio - RATIO_16_9_VALUE)) {
      return AspectRatio.RATIO_4_3
    }
    return AspectRatio.RATIO_16_9
  }
}
```