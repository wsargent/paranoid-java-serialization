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
    String enabled = java.security.Security.getProperty("paranoid.serialization.enabled");
    if (! Boolean.parseBoolean(enabled)) {
        throw new InvalidClassException("Object deserialization is disabled!");
    }

    ...
}
```

## Other libraries

Take a look at https://github.com/ikkisoft/SerialKiller by [Luca Carettoni](mailto:luca.carettoni@ikkisoft.com).

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
