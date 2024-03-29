---
title: "Making springdoc-openapi works with protobuf"
date: 2022-05-20 23:00:00 +0800
categories: Tech
---

# What is going on?
Recently at work, we are trying to be fancy just like any other cool devs and decide to switch from boring vanilla JSON based RESTful API to using protobuf as our payload carrier.

To be even fancier and make our API easy to explore for our dear downstream users, we also want to keep using springdoc-openapi to automate API documentation generation. This way, our users can use the Swagger UI that comes with it to play around with the API.

# Google Protocol Buffer, a.k.a. protobuf
protobuf, developed by Google as the native payload carrier for their gRPC protocol, is a data format for carrying structured data like JSON and the accompanying serialization mechanism. Unlike JSON, the data is stored in binary form instead of plaintext which makes it much smaller and faster. 

You can refer to its [official page](https://developers.google.com/protocol-buffers/docs/overview) for more details.

## proto file for demonstration
Instead of showing you our lengthy proto file. I have created a simple one for this demonstration as below.

```proto
syntax = "proto3";

package eternalwind;

option java_package = "io.eternalwind.springdocprotobuf";

message ProtoRequest {
    string name = 1;
    repeated string values = 2;
}
```

# springdoc-openapi and Swagger UI
springdoc-openapi is a Java library for automate the generation of API documentation in OpenAPI standard at runtime for Spring Boot applications. 

It is also conveniently bundled with Swagger UI which renders a basic web UI based on the generated documentation for people to try out your API without having to write any code or install any RESTful API agent e.q. Postman.

Again, you can visit their [homepage](https://springdoc.org/) for more details.

# Sounds great except it does not work
You hear it right. Each of these components is fantastic on its own except it simply refuses to work when you combine them together.

When we switch our API to use the data model generated by the protobuf compiler, the Swagger UI will freeze upon inspecting the API like below, showing you the dreadful message "This page isn't responding". In addition to this, the "Schemas" section also displays a bunch of random stuff.
![not_working](/assets/2022-05-20-Making-springdoc-openapi-works-with-protobuf/not_working.png)

The controller for this demonstration is rather simply and can be found below.

```java
@Slf4j
@RestController
public class HomeController {
    @PostMapping
    public ProtoResponse post(@RequestBody ProtoRequest request) {
        log.info("Received {}", request);
        
        return ProtoResponse.newBuilder()
            .setResult("hello")
            .build();
    }
}
```

You might naively think it's because the Swagger UI expects something in plaintext like JSON while protobuf is in binary. However even with the mighty [jackson-datatype-protobuf](https://github.com/HubSpot/jackson-datatype-protobuf) applied to help with converting between JSON and protobuf, this dreadful message persists.

```java
@Bean
public ObjectMapper objectMapper() {
    final ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.registerModule(new ProtobufModule());

    return objectMapper;
}
```

# An almost perfect solution
At the time of writing this post, I have been Googling the entire internet with little success. Eventually I was able to stumble upon a seemingly perfect [solution from innogames](https://github.com/innogames/springfox-protobuf). 

Except, this solution is developed for springfox instead of springdoc.

I still gave it a try following their instructions but not working unfortunately.

```java
@Bean
public ObjectMapper objectMapper() {
    final ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.registerModule(new ProtobufPropertiesModule());
    objectMapper.registerModule(new ProtobufModule());

    return objectMapper;
}
```

# What is springfox?
So what the heck is springfox and what is its relationship with springdoc?

Well, springfox does the almost exact same thing as springdoc and is more or less of a predecessor of springdoc. However, they codebase are totally different and the developers state there is no relation between the two projects.

You might ask, why not just migrate to springfox and use that solution already?!

Good question. Unfortunately, there has been no new release/update for springfox for more than 2 years and the project is lacking support for many new features for Swagger and the OpenAPI standard.

We certainly don't want to rely on yesterday's technology.

# Making it works with springdoc-openapi
As you can see for yourself, innogames' solution isn't really doing a lot of black magic. It is basically just telling Jackson to ignore unknown fields internally used by protobuf and only try to serialize/deserialize those fields defined explicitly in the *.proto file when the data model is a protobuf object. 

So it must be these internally fields having some sort of cyclic references and causing the Swagger UI to freeze.

```java
@Override
public BasicBeanDescription forDeserialization(DeserializationConfig cfg, JavaType type, MixInResolver r) {
	BasicBeanDescription desc = super.forDeserialization(cfg, type, r);

	if (Message.class.isAssignableFrom(type.getRawClass())) {
		return protobufBeanDescription(cfg, type, r, desc);
	}

	return desc;
}


@Override
public BasicBeanDescription forSerialization(SerializationConfig cfg, JavaType type, MixInResolver r) {
	BasicBeanDescription desc = super.forSerialization(cfg, type, r);

	if (Message.class.isAssignableFrom(type.getRawClass())) {
		return protobufBeanDescription(cfg, type, r, desc);
	}

	return desc;
}

private BasicBeanDescription protobufBeanDescription(MapperConfig<?> cfg, JavaType type, MixInResolver r, BasicBeanDescription baseDesc) {
	Map<String, FieldDescriptor> types = cache.computeIfAbsent(type.getRawClass(), this::getDescriptorForType);

	AnnotatedClass ac = AnnotatedClassResolver.resolve(cfg, type, r);

	List<BeanPropertyDefinition> props = new ArrayList<>();


	for (BeanPropertyDefinition p : baseDesc.findProperties()) {
		String name = p.getName();
		if (!types.containsKey(name)) {
			continue;
		}

		if (p.hasField() && p.getField().getType().isJavaLangObject()
			&& types.get(name).getType().equals(com.google.protobuf.Descriptors.FieldDescriptor.Type.STRING)) {
			addStringFormatAnnotation(p);
		}

		props.add(p.withSimpleName(name));
	}


	return new BasicBeanDescription(cfg, type, ac, new ArrayList<>(props)) {};
}
```

Lucky for us, after digging a little into both springfox's and springdoc's source code, I found both of them rely on Jackson for serialization so in theory innogames' solution should work for springdoc as well.

However at the moment, it is not working.

Since we have tried to call our API with our own RESTful API agent by sending and receiving data in JSON, we are sure the ProtobufModule module is doing its magic as intended so should innogames' ProtobufPropertiesModule. So the only possible explanation is that springdoc-openapi and Swagger UI are using their own ObjectMapper instance rather than the one we instantiated.

Now the question becomes how to tell springdoc to use our ObjectMapper.

Fortunately, this is relatively straightforward and more well-known than "making springdoc works protobuf".

With below code, we are able to configure springdoc to use our ObjectMapper instead of creating its own.

```java
@Bean
public ModelResolver modelResolver(final ObjectMapper objectMapper) {
    return new ModelResolver(objectMapper);
}
```

And it works.
![now_working](/assets/2022-05-20-Making-springdoc-openapi-works-with-protobuf/now_working.png)

I was really surprised that it's this simply and there has not been a single post/thread suggesting this. 

So here I decided to document it down saving anyone who might be scratching their heads for the same issue in the future.

# Where to find the source code for this example?
The sample project for this post can be found at [here](https://github.com/EternalWind/springdoc-protobuf-example/tree/master). 

Since innogames' package is not available at maven central, you might need to build from source and install his package locally first. You will also need protobuf compiler installed and have its executable, i.e. protoc, accessible from your system PATH.

# More on this...
You will soon find there are still some minor issues to be tackled with even having applied this solution. You can find out what they are and how to squeeze them out in [my next post](/tech/2022/05/29/Fixing-the-openapi-schema-for-protobuf-message.html)!