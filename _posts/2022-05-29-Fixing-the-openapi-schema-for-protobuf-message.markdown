---
title: "Fixing the OpenAPI schema for protobuf message"
date: 2022-05-29 23:00:00 +0800
categories: Tech
---

In my last post, I have explored how to make springdoc-openapi works with protobuf.
However despite working, you will soon realize the solution was not perfect and there are some minor issues if you look at the generated schema carefully.

So, let's try to tackle them today!

# What are these minor issues?

To demonstrate these issues, let me update our example .proto file to include some enum fields.

```protobuf
syntax = "proto3";

package eternalwind;

option java_package = "io.eternalwind.springdocprotobufexample";

message ProtoRequest {
    string name = 1;
    repeated string values = 2;

    ProtoEnum protoEnum = 3;
    repeated ProtoEnum protoEnumList = 4;
}

enum ProtoEnum {
    ENUM_VALUE_1 = 0;
    ENUM_VALUE_2 = 1;
}
```

Now, take a look at the generated example values and schema carefully. 

![example_value_issues](/assets/2022-05-29-Fixing-the-openapi-schema-for-protobuf-message/example_value_issues.png)

![schema_issues](/assets/2022-05-29-Fixing-the-openapi-schema-for-protobuf-message/schema_issues.png)

I am sure below issues are not difficult to notice.

* The default values for our string field 'name' and repeated string field 'values' are 'string' which is not the default value for protobuf string fields.
* The repeated enum field 'protoEnumList' is shown as a list of integers rather than a list of enum strings.
* The enum field 'protoEnum' is shown to have 3 possible values including an extra one 'UNRECOGNIZED' which is not defined in our .proto file.

Obviously Swagger UI must be generating the schema for our protobuf message incorrectly. 

# The default value

This is simply the default values from our protobuf message has not been applied to the properties' default values in the schema.

# The extra enum value 'UNRECOGNIZED'

If we take a look at the generated java file for our protobuf message, we can see protobuf actually generates an extra one named, guess what, 'UNRECOGNIZED' for its internal usage.

```java
/**
 * Protobuf enum {@code eternalwind.ProtoEnum}
 */
public enum ProtoEnum
    implements com.google.protobuf.ProtocolMessageEnum {
  /**
   * <code>ENUM_VALUE_1 = 0;</code>
   */
  ENUM_VALUE_1(0),
  /**
   * <code>ENUM_VALUE_2 = 1;</code>
   */
  ENUM_VALUE_2(1),
  UNRECOGNIZED(-1),
  ;

  // More generated code...
}
```

# Enum list being treated as integer list

This one is not so straightforward to explain.

To make the serialized message size smaller and more efficient to transfer, protobuf actually stores its enums as integers rather than plain Java enums and only converts it to Java enum through its getter.

```java
/**
 * Protobuf type {@code eternalwind.ProtoRequest}
 */
public static final class ProtoRequest extends
    com.google.protobuf.GeneratedMessageV3 implements
    // @@protoc_insertion_point(message_implements:eternalwind.ProtoRequest)
    ProtoRequestOrBuilder {
    // More generated code...

    private ProtoRequest() {
      name_ = "";
      values_ = com.google.protobuf.LazyStringArrayList.EMPTY;
      protoEnum_ = 0;
      protoEnumList_ = java.util.Collections.emptyList();
    }

    private int protoEnum_ = 0;
    private java.util.List<java.lang.Integer> protoEnumList_ =
        java.util.Collections.emptyList();

    /**
     * <code>.eternalwind.ProtoEnum protoEnum = 3;</code>
     * @return The protoEnum.
     */
    @java.lang.Override public io.eternalwind.springdocprotobufexample.ProtoRequestOuterClass.ProtoEnum getProtoEnum() {
      @SuppressWarnings("deprecation")
      io.eternalwind.springdocprotobufexample.ProtoRequestOuterClass.ProtoEnum result = io.eternalwind.springdocprotobufexample.ProtoRequestOuterClass.ProtoEnum.valueOf(protoEnum_);
      return result == null ? io.eternalwind.springdocprotobufexample.ProtoRequestOuterClass.ProtoEnum.UNRECOGNIZED : result;
    }

    /**
     * <code>repeated .eternalwind.ProtoEnum protoEnumList = 4;</code>
     * @return A list containing the protoEnumList.
     */
    @java.lang.Override
    public java.util.List<io.eternalwind.springdocprotobufexample.ProtoRequestOuterClass.ProtoEnum> getProtoEnumListList() {
      return new com.google.protobuf.Internal.ListAdapter<
          java.lang.Integer, io.eternalwind.springdocprotobufexample.ProtoRequestOuterClass.ProtoEnum>(protoEnumList_, protoEnumList_converter_);
    }

    // More generated code...
}
```

For non-repeated enum fields, their getters that return them as Java enums match their names. So the Swagger UI can generate their counterparts in the schema correctly as enum strings.

For repeated enum fields, unfortunately their getters' naming is not so standard, i.e. being in the format of *'get[FIELD_NAME]List()'*. As a result, the Swagger UI cannot recognize it and falls back to generating the property in the schema based on the field itself which is indeed a list of integers.

# How do we fix this?

Lucky for us, springdoc-openapi provides a way for us to customize the generated schemas through [the use of OpenApiCustomiser/GlobalOpenApiCustomizer](https://springdoc.org/#how-can-i-customise-the-openapi-object) which is exactly what we need to tackle these issues.

```java
@Bean
public GlobalOpenApiCustomizer openApiCustomizer() {
    return openApi -> {
        // customization code
    };
}
```

Basically we just need to tell the Swagger UI to do the followings.

* Use the default values from our protobuf message's fields as the default values for the properties in the generated schema.
* Only include the enum values we explicitly defined as possible values for the enum fields.
* Treat the enum list fields as lists of enum strings.

```java
private void customiseEnumListFieldSchema(final Schema schema, final FieldDescriptor field) {
    // Only include enum values we have defined explicitly.
    final List<String> enumValues = field.getEnumType().getValues().stream()
            .map(EnumValueDescriptor::getName)
            .toList();

    // Treat the enum list field as a list of enum strings.
    final StringSchema enumSchema = new StringSchema();
    enumSchema.setEnum(enumValues);

    schema.setItems(enumSchema);

    // Apply the default value
    schema.setDefault(field.getDefaultValue());
}

private void customiseEnumFieldSchema(final Schema schema, final FieldDescriptor field) {
    // Only include enum values we have defined explicitly.
    final List<String> enumValues = field.getEnumType().getValues().stream()
            .map(EnumValueDescriptor::getName)
            .toList();

    schema.setEnum(enumValues);

    // Apply the default value
    schema.setDefault(field.getDefaultValue());
}

private void customiseStringListFieldSchema(final Schema schema, final FieldDescriptor field) {
    // Apply the default value
    schema.setDefault(field.getDefaultValue());
}

private void customiseStringFieldSchema(final Schema schema, final FieldDescriptor field) {
    // Apply the default value
    schema.setDefault(field.getDefaultValue());
}
```

With above customizations, now you will see the example values and schemas are shown as expected.

![example_value_fixed](/assets/2022-05-29-Fixing-the-openapi-schema-for-protobuf-message/example_value_fixed.png)

![schema_fixed](/assets/2022-05-29-Fixing-the-openapi-schema-for-protobuf-message/schema_fixed.png)

# Or...

You can either DIY this solution or just grab [the one I have written](https://github.com/EternalWind/springdoc-protobuf-schema). Just add it as your project's dependency and you are done!

I have also updated [my example project](https://github.com/EternalWind/springdoc-protobuf-example) to demonstrate how to use it.