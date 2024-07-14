+++
categories = "imageprocessing"
disableToc = true
title = "Extraction of Odia Text From Images"
weight = 1

+++

---

**Introduction:**
This blog's focus is on integrating Tesseract OCR for extracting Odia text in Java.

**Prerequisites:**
- macOS installed on your system
- Basic familiarity with terminal commands
- Java Development Kit (JDK) installed
- Maven installed for managing Java dependencies

**Installing Tesseract OCR:**
1. **Using Homebrew:**
    - Installation of Tesseract using Homebrew:
      ```bash
      brew install tesseract
      ```

2. **Verifying Installation:**
    - Command to verify Tesseract installation:
      ```bash
      tesseract --version
      ```

**Installing Odia Language Support (tessdata):**
1. **Downloading tessdata for Odia:**
    - Execute below command to download the trained data for Odia language:
      ```bash
      brew install tesseract-lang
      ```
      This downloads additional trained data for other supported languages as well.

### Creating a Maven Project

1. **Create a New Maven Project:**
   Open your terminal and run the following Maven command to generate a new project using the Maven Quickstart archetype:
   ```bash
   mvn archetype:generate -DgroupId=io.azar.examples -DartifactId=odia-ocr-demo -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
   ```
   This creates a basic Maven project structure with a simple Java class.

2. **Navigate to the Project Directory:**
   ```bash
   cd odia-ocr-demo
   ```

### Adding Dependencies to `pom.xml`

Next, add dependencies for Tesseract OCR in `pom.xml` file:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
         
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.azar.examples</groupId>
    <artifactId>odia-ocr-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- PDFBox for PDF processing -->
        <dependency>
            <groupId>org.apache.pdfbox</groupId>
            <artifactId>pdfbox</artifactId>
            <version>2.0.27</version>
        </dependency>

        <!-- Apache POI for Word document creation -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>5.3.0</version>
        </dependency>

        <!-- Log4j2 for logging -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.17.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.17.2</version>
        </dependency>
        
        <!-- https://mvnrepository.com/artifact/org.bytedeco/tesseract-platform -->
        <dependency>
            <groupId>org.bytedeco</groupId>
            <artifactId>tesseract-platform</artifactId>
            <version>5.3.4-1.5.10</version>
        </dependency>

    </dependencies>

    <build>
        <sourceDirectory>.</sourceDirectory>
    </build>
</project>
```

### Writing Java Code to Parse an Image

Now, let's create a Java class `ImageToOdiaTextReader.java` within `src/main/java/io/azar/examples/ocr/` directory to extract text from an image using Tesseract OCR:

```java
package io.azar.examples.ocr;

import org.apache.poi.xwpf.usermodel.*;

import org.bytedeco.javacpp.BytePointer;
import org.bytedeco.leptonica.*;
import org.bytedeco.tesseract.*;

import java.io.*;

import static org.bytedeco.leptonica.global.leptonica.*;
import static org.bytedeco.tesseract.global.tesseract.*;

public class ImageToOdiaTextReader {
    
    public static void main(String[] args) {
        // Path to tessdata directory containing language data
        String tessDataPath = "/opt/homebrew/share/tessdata";

        // Output Word document path
        String outputWordPath = "odia.docx";

        // Initialize Tesseract API
        try (TessBaseAPI api = new TessBaseAPI()) {
            // Initialize Tesseract with Oriya language and tessdata path
            if (api.Init(tessDataPath, "ori") != 0) {
                System.err.println("Could not initialize Tesseract.");
                System.exit(1);
            }

            // Read input image using Leptonica PIX
            PIX pixImage = pixRead(args.length > 0 ? args[0] : "/Users/azaruddin/IdeaProjects/ocr-scanner-img-to-txt/src/main/resources/odia.png");
            api.SetImage(pixImage);

            // Get OCR result
            BytePointer outText = api.GetUTF8Text();
            String extractedText = outText.getString().trim();

            // Create a new Word document
            XWPFDocument wordDocument = new XWPFDocument();

            // Create paragraph and add extracted text
            XWPFParagraph paragraph = wordDocument.createParagraph();
            XWPFRun run = paragraph.createRun();
            run.setText(extractedText);

            // Save the Word document
            try (FileOutputStream out = new FileOutputStream(outputWordPath)) {
                wordDocument.write(out);
            }

            // Clean up resources
            outText.deallocate();
            api.End();
            pixDestroy(pixImage);

            System.out.println("Text extracted and saved to " + outputWordPath);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

### Conclusion

This setup allows you to effectively integrate Tesseract OCR into your Maven Java project on macOS for extracting text from images, particularly in the Odia language context. Adjust paths and configurations as per your specific environment and requirements.


