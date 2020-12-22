+++
title = "Serializing and deserializing IDs and other value objects to Java records with Jackson"
date = 2020-12-22
+++

There are many good reasons to [model IDs as value objects](https://buildplease.com/pages/vo-ids/). Since Java 16 [Records](https://openjdk.java.net/jeps/395) it is very easy to use value objects, but how do you (de-)serialize JSON to these records with Jackson? 

```java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonValue;

record MyId(String value) {

    @JsonCreator(mode = JsonCreator.Mode.DELEGATING)
    public MyId {}

    @JsonValue
    public String value() {
        return value;
    }

}
```
