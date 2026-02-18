# SwiftyTesseract

![SPM compatible](https://img.shields.io/badge/SPM-compatible-blueviolet.svg) ![swift-version](https://img.shields.io/badge/Swift-5.7-orange.svg) ![platforms](https://img.shields.io/badge/Platforms-iOS%2016.0%2B%20|%20macOS%2013.0%2B-lightgrey.svg)

A Swift wrapper around Google's [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) engine for Apple platforms. This is an actively maintained fork of [SwiftyTesseract/SwiftyTesseract](https://github.com/SwiftyTesseract/SwiftyTesseract), upgraded to **Tesseract 5.5.2** and **Leptonica 1.84.1**.

## Table of Contents
* [Version Compatibility](#version-compatibility)
* [What's New in 5.x](#whats-new-in-5x)
* [Using SwiftyTesseract in Your Project](#using-swiftytesseract-in-your-project)
  * [Performing OCR](#performing-ocr)
    * [Platform Agnostic](#platform-agnostic)
    * [Combine](#combine)
    * [UIKit](#uikit)
    * [AppKit](#appkit)
  * [Extensibility](#extensibility)
    * [Tesseract Variable Configuration](#tesseract-variable-configuration)
    * [Tesseract.Variable](#tesseractvariable)
    * [perform(action:)](#performaction)
  * [Initializer Defaults](#a-note-on-initializer-defaults)
* [Installation](#installation)
* [Additional Configuration](#additional-configuration)
  * [Shipping language training files in an application bundle](#shipping-language-training-files-as-part-of-an-application-bundle)
  * [Shipping language training files as part of a Swift Package](#shipping-language-training-files-as-part-of-a-swift-package)
  * [Custom location for language files](#custom-location)
* [Language Training Data Considerations](#language-training-data-considerations)
* [Custom Trained Data](#custom-trained-data)
* [Recognition Results](#recognition-results)
* [Attributions](#attributions)

## Version Compatibility
| SwiftyTesseract Version | Tesseract Engine | Platforms Supported | Swift Version |
|-------------------------|-----------------|:-------------------:|--------------:|
| 5.x.x                  | 5.5.2           | **iOS** **macOS**   | 5.7           |
| 4.x.x                  | 4.1.3           | **iOS** **macOS** **Linux** | 5.3  |

## What's New in 5.x

- **Tesseract 5.5.2** with LSTM-based OCR engine
- **Leptonica 1.84.1**, libpng 1.6.44, libjpeg 9f, libtiff 4.7.0
- Minimum deployment targets raised to **iOS 16.0** and **macOS 13.0**
- Swift tools version bumped to **5.7**
- **Legacy engine mode is disabled.** The Tesseract binary is built with `--disable-legacy`, meaning only `EngineMode.lstmOnly` is supported. Use training data from [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) or [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast). The [tessdata](https://github.com/tesseract-ocr/tessdata) repo also works (its files contain LSTM models), but `.tesseractOnly` and `.tesseractLstmCombined` engine modes will fail at runtime.
- **Accelerate.framework** is now a required linker dependency (used by Leptonica)
- Linux support has been removed from this fork (Apple platforms only)

## Using SwiftyTesseract in Your Project
Import the module:
```swift
import SwiftyTesseract
```
Create an instance with one language:
```swift
let tesseract = Tesseract(language: .english)
```
Or with multiple languages:
```swift
let tesseract = Tesseract(languages: [.english, .french, .italian])
```

### Performing OCR
#### Platform Agnostic
Pass an instance of `Data` derived from an image to `performOCR(on:)`:
```swift
let imageData = try Data(contentsOf: urlOfYourImage)
let result: Result<String, Tesseract.Error> = tesseract.performOCR(on: imageData)
```

#### Combine
Pass an instance of `Data` derived from an image to `performOCRPublisher(on:)`:
```swift
let imageData = try Data(contentsOf: urlOfYourImage)
let result: AnyPublisher<String, Tesseract.Error> = tesseract.performOCRPublisher(on: imageData)
```

#### UIKit
Pass a `UIImage` to the `performOCR(on:)` _or_ `performOCRPublisher(on:)` methods:
```swift
let image = UIImage(named: "someImageWithText.jpg")!
let result: Result<String, Error> = tesseract.performOCR(on: image)
let publisher: AnyPublisher<String, Error> = tesseract.performOCRPublisher(on: image)
```

#### AppKit
Pass a `NSImage` to the `performOCR(on:)` _or_ `performOCRPublisher(on:)` methods:
```swift
let image = NSImage(named: "someImageWithText.jpg")!
let result: Result<String, Error> = tesseract.performOCR(on: image)
let publisher: AnyPublisher<String, Error> = tesseract.performOCRPublisher(on: image)
```

#### Conclusion
For a synchronous call, `performOCR(on:)` returns a `Result<String, Error>` and blocks on the calling thread.

The `performOCRPublisher(on:)` publisher makes it easy to perform OCR on a background thread and receive results on the main thread:
```swift
let cancellable = tesseract.performOCRPublisher(on: image)
  .subscribe(on: backgroundQueue)
  .receive(on: DispatchQueue.main)
  .sink(
    receiveCompletion: { completion in 
      // do something with completion
    },
    receiveValue: { string in
      // do something with string
    }
  )
```
The publisher is **cold** — it does not perform any work until subscribed to.

### Extensibility
The 4.0+ API is designed to be extensible. If you need to set a variable or call a function that exists in the Google Tesseract API but isn't exposed by SwiftyTesseract, you can do so directly.

#### Tesseract Variable Configuration
All public instance variables have been replaced with a declarative configuration API:
```swift
let tesseract = Tesseract(language: .english) {
  set(.disallowlist, "@#$%^&*")
  set(.minimumCharacterHeight, .integer(35))
  set(.preserveInterwordSpaces, .true)
}
// or post-initialization
tesseract.configure {
  set(.disallowlist, "@#$%^&*")
  set(.minimumCharacterHeight, .integer(35))
  set(.preserveInterwordSpaces, .true)
}
```

#### Tesseract.Variable
`Tesseract.Variable` is a simple struct backed by a `String` raw value. The library ships with a few built-in variables:
```swift
public extension Tesseract.Variable {
  static let allowlist: Tesseract.Variable = "tessedit_char_whitelist"
  static let disallowlist: Tesseract.Variable = "tessedit_char_blacklist"
  static let preserveInterwordSpaces: Tesseract.Variable = "preserve_interword_spaces"
  static let minimumCharacterHeight: Tesseract.Variable = "textord_min_xheight"
  static let oldCharacterHeight: Tesseract.Variable = "textord_old_xheight"
}
```

You can extend it with any Tesseract variable:
```swift
extension Tesseract.Variable {
  static let numericMode: Tesseract.Variable = "classify_bln_numeric_mode"
}

tesseract.configure {
  set(.numericMode, .true)
}
```

#### `perform(action:)`
The `perform(action:)` method gives you full access to the Tesseract C API in a thread-safe manner.

**Important:** When using the C API directly, you are responsible for managing memory. ARC does not apply to C pointers. Use `defer` to release memory — see [`Sources/SwiftyTesseract/Tesseract+OCR.swift`](https://github.com/caetanonetodev/SwiftyTesseract/blob/develop/Sources/SwiftyTesseract/Tesseract%2BOCR.swift) for examples.

All library methods other than `perform(action:)` and `configure(_:)` are implemented as extensions using `perform(action:)`. Here's an example implementing page segmentation mode:
```swift
import SwiftyTesseract
import libtesseract

public extension Tesseract {
  var pageSegmentationMode: TessPageSegMode {
    get {
      perform { tessPointer in
        TessBaseAPIGetPageSegMode(tessPointer)
      }
    }
    set {
      perform { tessPointer in
        TessBaseAPISetPageSegMode(tessPointer, newValue)
      }
    }
  }
}

// usage
tesseract.pageSegmentationMode = PSM_SINGLE_COLUMN
```

#### ConfigurationBuilder
You can also create functions with a return signature of `(TessBaseAPI) -> Void` to use with the declarative configuration block:
```swift
import SwiftyTesseract
import libtesseract

func setPageSegMode(_ pageSegMode: TessPageSegMode) -> (TessBaseAPI) -> Void {
  return { tessPointer in
    TessBaseAPISetPageSegMode(tessPointer, pageSegMode)
  }
}

let tesseract = Tesseract(language: .english) {
  setPageSegMode(PSM_SINGLE_COLUMN)
}
```

### A Note on Initializer Defaults
The full signature of the primary `Tesseract` initializer is:
```swift
public init(
  languages: [RecognitionLanguage], 
  dataSource: LanguageModelDataSource = Bundle.main, 
  engineMode: EngineMode = .lstmOnly,
  @ConfigurationBuilder configure: () -> (TessBaseAPI) -> Void = { { _ in } }
)
```
The `dataSource` parameter locates the `tessdata` folder. Change this if `Tesseract` is not being used in your application bundle (e.g. use `Bundle.module` for Swift Package targets).

**Note:** Since this build uses `--disable-legacy`, only `.lstmOnly` is functional. Passing `.tesseractOnly` or `.tesseractLstmCombined` will fail at runtime.

## Installation
Swift Package Manager is the only supported dependency manager.

```swift
// Package.swift
// swift-tools-version:5.7
import PackageDescription

let package = Package(
  name: "AwesomePackage",
  platforms: [
    .macOS(.v13),
    .iOS(.v16),
  ],
  products: [
    .library(
      name: "AwesomePackage",
      targets: ["AwesomePackage"]
    ),
  ],
  dependencies: [
    .package(url: "https://github.com/caetanonetodev/SwiftyTesseract.git", from: "5.0.0")
  ],
  targets: [
    .target(
      name: "AwesomePackage",
      dependencies: ["SwiftyTesseract"]
    ),
  ]
)
```

### libtesseract
Tesseract and its dependencies are built and distributed as an xcframework under the [caetanonetodev/libtesseract](https://github.com/caetanonetodev/libtesseract) repository. Any issues regarding the build configuration should be raised there.

## Additional Configuration
### Shipping language training files as part of an application bundle
1. Download the appropriate language training files from [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) or [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast).
2. Place them into a folder named `tessdata`.
3. Drag the folder into your Xcode project. You **must** select "Create folder references" or `Tesseract` will **not** initialize successfully.

### Shipping language training files as part of a Swift Package
If you keep training data files under source control, copy your tessdata directory as a package resource:
```swift
let package = Package(
  // ...
  targets: [
    .target(
      name: "App",
      dependencies: ["SwiftyTesseract"],
      resources: [.copy("tessdata")],
    )
  ]
)
```

If you prefer not to keep training data under source control, see the custom location instructions below.

### Custom Location
You can provide a custom location for training data files by conforming to:
```swift
public protocol LanguageModelDataSource {
  var pathToTrainedData: String { get }
}
```

Then pass it to the initializer:
```swift
let customDataSource = CustomDataSource()
let tesseract = Tesseract(
  language: .english, 
  dataSource: customDataSource, 
  engineMode: .lstmOnly
)
```

## Language Training Data Considerations
There are two recommended sources for `.traineddata` files:

| Repository | Speed | Accuracy | Notes |
|-----------|-------|----------|-------|
| [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast) | Fastest | Good | Recommended for most use cases |
| [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) | Slower | Best | Use when accuracy is critical |

The [tessdata](https://github.com/tesseract-ocr/tessdata) repository also works (its files include LSTM models), but since this build has the legacy engine disabled, there is no advantage to using it over `tessdata_fast` or `tessdata_best`.

For most cases, `tessdata_fast` will be the best option — it provides fast results with good accuracy.

## Custom Trained Data
To use custom `.traineddata` files, use the `.custom(String)` case of `RecognitionLanguage`:
```swift
let tesseract = Tesseract(language: .custom("custom-traineddata-file-prefix"))
```
You can combine custom training files with built-in languages:
```swift
let tesseract = Tesseract(languages: [.custom("MyCustomModel"), .english])
```

**Important:** Custom training data must be LSTM-compatible since the legacy engine is disabled in this build.

## Recognition Results
OCR follows the "garbage in, garbage out" principle. The Tesseract engine will process the image and return anything it believes is text. For best results, pre-process your images to isolate the text regions and ensure proper orientation. The quality of your input image directly affects recognition accuracy.

## Attributions
SwiftyTesseract would not be possible without the work done by the [Tesseract](https://github.com/tesseract-ocr/tesseract) team.

See the [Attributions section](https://github.com/caetanonetodev/libtesseract#attributions) in the [libtesseract repo](https://github.com/caetanonetodev/libtesseract) for a full list of vendored dependencies and their licenses.
