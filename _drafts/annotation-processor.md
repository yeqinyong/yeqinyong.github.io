---
title: "annotation processor"
date: "2017-05-19 14:05"
---
An annotation processor is no more than a class that implements javax.annotation.processing.Processor interface

javax.annotation.processing.AbstractProcessor.

The custom processor may use three annotations to configure itself:

javax.annotation.processing.SupportedAnnotationTypes: This annotation is used to register the annotations that the processor supports. Valid values are fully qualified names of annotation types – wildcards are allowed.
javax.annotation.processing.SupportedSourceVersion: This annotation is used to register the source version that the processor supports.
javax.annotation.processing.SupportedOptions: This annotation is used to register allowed custom options that may be passed through the command line.


## WRITING OUR FIRST ANNOTATION PROCESSOR
```java
@SupportedAnnotationTypes("sdc.assets.annotations.Complexity")
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class ComplexityProcessor extends AbstractProcessor {

    public ComplexityProcessor() {
        super();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        return true;
    }
}
```

This incomplete class, although does nothing when called, is registered to support annotations of type sdc.assets.annotations.Complexity. Therefore, each time the Java Compiler founds a class annotated with that type will execute the processor, given that the process is available in the classpath   

## PACKAGING AND REGISTERING THE ANNOTATION PROCESSOR

The final step needed to finish our custom Annotation Processor, is to package and register it so the Java Compiler or other tools can find it.

The easiest way to register the processor is to leverage the standard Java services mechanism:

Package your Annotation Processor in a Jar file.
Include in the Jar file a directory META-INF/services.
Include in the directory a file named javax.annotation.processing.Processor.
Write in the file the fully qualified names of the processors contained in the Jar file, one per line.

The Java Compiler and other tools will search for this file in all provided classpath elements and make use of the registered processors.
