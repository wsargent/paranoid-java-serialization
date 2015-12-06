# Paranoid Java Serialization

> **NOTE**: This project modifies the boot classpath, which is fine locally, but cannot be deployed, per the "Oracle Binary Code License Agreement".  If you see the java executable [Non-Standard Options](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html#BABHDABI), there is a note saying "Do not deploy applications that use this option to override a class in rt.jar because this violates the Java Runtime Environment binary code license."

This is a proof of concept that hacks `java.io.ObjectInputStream` to provide JVM level control over Java object serialization.  Other solutions are user level -- they will work individually, but they don't change the behavior of internal libraries or application servers.  This will enforce behavior at the lowest level.

See [the blog post](https://tersesystems.com/2015/11/08/closing-the-open-door-of-java-object-serialization/) and the [original talk](https://frohoff.github.io/appseccali-marshalling-pickles/) for details.

## Building

```
mvn package
```

## Running

Create a file `deserialization.properties` in your local application directory:

```
paranoid.serialization.enabled=true
#paranoid.serialization.blacklist=com.evil.Nastygram
#paranoid.serialization.whitelist=com.good.VirtualPackage
```

And copy the `paranoid-java-serialization-1.0-SNAPSHOT.jar` created from the package to your local application.

Then run Java with:

```
java \
   -Xbootclasspath/p:paranoid-java-serialization-1.0-SNAPSHOT.jar \
   -Djava.security.properties=deserialization.properties
```

## Code

Because this is a hack of ObjectInputStream, it's harder to see what's changed between this and the stock version.

First, you can disable serialization entirely:

``` java
public final Object readObject()
            throws IOException, ClassNotFoundException
{
    String enabled = java.security.Security.getProperty("paranoid.serialization.enabled");
    if (! Boolean.parseBoolean(enabled)) {
        throw new InvalidClassException("Object deserialization is disabled!");
    }

    ...
}
```

Second: you can whitelist and blacklist based on class name:


``` java
/** Set of blacklisted class name patterns. */
private static final java.util.Set<Pattern> blacklistPatterns;

/** Set of whitelisted class name patterns. */
private static final java.util.Set<Pattern> whitelistPatterns;

// JUL isn't ideal, but it's the out of the box one.
private static final Logger logger = Logger.getLogger("java.io.ObjectInputStream");

static {
    ...

    final String blacklist = Security.getProperty("paranoid.serialization.blacklist");
    blacklistPatterns = parsePatterns(blacklist);

    final String whitelist = Security.getProperty("paranoid.serialization.whitelist");
    whitelistPatterns = parsePatterns(whitelist);
}

private static Set<Pattern> parsePatterns(String listString) {
    final Set<Pattern> listSet = new HashSet<>();
    if (listString != null) {
        final String[] regexArray = listString.split(",\\s*");
        for (String regex : regexArray) {
            Pattern whitePattern = Pattern.compile(regex);
            listSet.add(whitePattern);
        }
    }
    return java.util.Collections.unmodifiableSet(listSet);
}

...

protected Class<?> resolveClass(ObjectStreamClass desc)
            throws IOException, ClassNotFoundException
{
    String name = desc.getName();
    logger.fine("resolveClass: resolving " + name);

    // From https://github.com/ikkisoft/SerialKiller by luca.carettoni@ikkisoft.com

    //Enforce blacklist
    for (Pattern blackPattern : blacklistPatterns) {
        Matcher blackMatcher = blackPattern.matcher(name);
        if (blackMatcher.find()) {
            logger.warning("resolveClass: rejecting blacklisted class " + name);
            String msg = "[!] Blocked by blacklist '"
                    + blackPattern.pattern() + "'. Match found for '" + name + "'";
            throw new InvalidClassException(msg);
        }
    }

    //Enforce whitelist if it exists.
    if (! whitelistPatterns.isEmpty()) {
        boolean safeClass = false;
        for (Pattern whitePattern: whitelistPatterns) {
            Matcher whiteMatcher = whitePattern.matcher(name);
            if (whiteMatcher.find()) {
                safeClass = true;
                break;
            }
        }

        if (!safeClass) {
            logger.warning("resolveClass: rejecting class " + name + " not found in whitelist.");
            String msg = "[!] Blocked by whitelist. No match found for '" + name + "'";
            throw new InvalidClassException(msg);
        }
    }

    try {
        logger.fine("resolveClass: accepting class " + name);
        return Class.forName(name, false, latestUserDefinedLoader());
    } catch (ClassNotFoundException ex) {
        Class<?> cl = primClasses.get(name);
        if (cl != null) {
            return cl;
        } else {
            throw ex;
        }
    }
}
```

## Logging

If you are whitelisting, it can be helpful to run through the application first with a log of all the existing classes.

While you can use configure `java.util.logging` through a properties file, it's probably better to use SLF4J and the [JUL to SLF4J bridge](http://mvnrepository.com/artifact/org.slf4j/jul-to-slf4j), after which you can use [Logback](http://mvnrepository.com/artifact/ch.qos.logback).  You will need to add the following as initialization:

``` java
LogManager.getLogManager().reset();
SLF4JBridgeHandler.removeHandlersForRootLogger();
SLF4JBridgeHandler.install();
Logger.getLogger("global").setLevel(Level.FINEST);
```


## Other libraries

Take a look at https://github.com/ikkisoft/SerialKiller by [Luca Carettoni](mailto:luca.carettoni@ikkisoft.com) and [Invoker defender](https://github.com/kantega/invoker-defender/).

## License

Because this is a modification of OpenJDK code, the [GPL v2 license](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html) applies to all of this code as well.

Also, because this is a modification of Oracle code, I will not be providing a binary distribution and uploading it to Maven unless I have their express permission.  Sorry about that.

```
/*
 * Copyright (c) 1996, 2013, Oracle and/or its affiliates. All rights reserved.
 * DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS FILE HEADER.
 *
 * This code is free software; you can redistribute it and/or modify it
 * under the terms of the GNU General Public License version 2 only, as
 * published by the Free Software Foundation.  Oracle designates this
 * particular file as subject to the "Classpath" exception as provided
 * by Oracle in the LICENSE file that accompanied this code.
 *
 * This code is distributed in the hope that it will be useful, but WITHOUT
 * ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
 * FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
 * version 2 for more details (a copy is included in the LICENSE file that
 * accompanied this code).
 *
 * You should have received a copy of the GNU General Public License version
 * 2 along with this work; if not, write to the Free Software Foundation,
 * Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301 USA.
 *
 * Please contact Oracle, 500 Oracle Parkway, Redwood Shores, CA 94065 USA
 * or visit www.oracle.com if you need additional information or have any
 * questions.
 */
```

## Questions

Email [will.sargent+github@gmail.com](mailto:will.sargent+github@gmail.com).
