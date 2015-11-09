# Paranoid Java Serialization

This is a proof of concept that takes `java.io.ObjectInputStream` and disables blind object deserialization.

See [the blog post](https://tersesystems.com/2015/11/08/closing-the-open-door-of-java-object-serialization/) and the [original talk](https://frohoff.github.io/appseccali-marshalling-pickles/) for details.

## Building
 
```
mvn package
```

## Running

Create a file `deserialization.properties`

```
java.io.enableObjectDeserialization=true
```

And take the `paranoid-java-serialization-1.0-SNAPSHOT.jar` created from the package.

Then run Java with:

```
java \
   -Xbootclasspath/p:paranoid-java-serialization-1.0-SNAPSHOT.jar \
   -Djava.security.properties=deserialization.properties
```

## Code 

The only code that's changed in ObjectInputStream (modulo OpenJDK vs Oracle JDK):

```
public final Object readObject()
            throws IOException, ClassNotFoundException
{
    String enabled = java.security.Security.getProperty("java.io.enableObjectDeserialization");
    if (! Boolean.parseBoolean(enabled)) {
        throw new InvalidClassException("Object deserialization is disabled!");
    }

    ...
}
```

## Questions

Email [will.sargent+github@gmail.com](mailto:will.sargent+github@gmail.com).
