---
title: "Mocking Apache Beam's pipeline options with mockito"
date: 2022-07-03 22:00:00 +0800
categories: Tech
---

# A short introduction to Apache Beam
Shameless stealing below introduction from [Apache Beam's official page](https://beam.apache.org/about/): 

Apache Beam is an open-source, unified programming model for batch and streaming data processing pipelines that simplifies large-scale data processing dynamics.

During the development of your pipeline, usually it involves writing your very own transform combining several more basic transforms to achieve a specific goal like below.

```java
public class MyTransform extends PTransform<PCollection<String>, PCollection<String>> {

    @Override
    public PCollection<String> expand(PCollection<String> input) {
        return input.apply(transform1)
                    .apply(transform2);
    }
    
}
```

# Testing your PTransform
Naturally, you will want to test your composite transform to make sure it does exactly what you intended it to do afterwards. 

You cannot directly instantiate a PCollection yourself and even if you can, the processing logic won't actually be executed upon calling ```MyTransform.expand()``` which only builds the execution graph until ```Pipeline.run()``` is invoked and your pipeline runs.

Hence you can only be testing your transform by using a ```TestPipeline```(which unfortunately only works with JUnit 4) or an actual ```Pipeline``` with the direct runner(which runs your pipeline locally) that applies your transform to some test input. This is actually also the way suggested by the [official documentation](https://beam.apache.org/documentation/pipelines/test-your-pipeline/#:~:text=references%20and%20guides.-,Testing%20Transforms,-To%20test%20a).

```java
@Test
void testMyPipeline() {
    final MyPipelineOptions pipelineOptions = PipelineOptionsFactory.create().as(MyPipelineOptions.class);

    // Setting your pipeline options...

    final Pipeline pipeline = Pipeline.create(pipelineOptions);

    // Or with TestPipeline...
    // final Pipeline pipeline = TestPipeline.create(pipelineOptions);

    final PCollection<String> result = pipeline.apply(Create.of(MY_TEST_INPUT))
                                               .apply(new MyTransform());

    PAssert.that(result).containsInAnyOrder(EXPECTED_OUTPUT);

    pipeline.run();
}
```

You may also want to use a mocking library like mockito to mock out some pipeline options to isolate your transform for testing. You might wonder why this is needed if your pipeline options only consist of some basic primitive values. Well, at least until you decided to put some complicated objects into it like us, who put a Spring context into it to utilize its DI magic.

```java
final MyPipelineOptions pipelineOptions = PipelineOptionsFactory.create().as(MyPipelineOptions.class);
pipelineOptions.setContext(mock(MyContext.class));
// More setups...
```

Thinking you are all set and you excitingly hit 'run'. To your surprise, Beam do not like mocks at all and just refuse to run by throwing below exception.

![serialization_error](/assets/2022-07-03-Mocking-Apache-Beams-pipeline-options-with-mockito/serialization_error.png)

This is because Beam's [direct runner](https://github.com/apache/beam/blob/85e8149cbcebc4a6b07d09501f96dfaec95c73bc/runners/direct-java/src/main/java/org/apache/beam/runners/direct/DirectRunner.java#:~:text=public%20DirectPipelineResult%20run(Pipeline%20pipeline)%20%7B) will try to serialize your pipeline options into JSON with Jackson and then immediately deserialize it back upon run despite it only runs on your local machine.

![direct_runner_serialization](/assets/2022-07-03-Mocking-Apache-Beams-pipeline-options-with-mockito/direct_runner_serialization.png)

Unfortunately this is also true for [```TestPipeline```](https://github.com/apache/beam/blob/85e8149cbcebc4a6b07d09501f96dfaec95c73bc/sdks/java/core/src/main/java/org/apache/beam/sdk/testing/TestPipeline.java#L385:~:text=public%20PipelineResult%20run(PipelineOptions%20options)%20%7B).

![test_pipeline_serialization](/assets/2022-07-03-Mocking-Apache-Beams-pipeline-options-with-mockito/test_pipeline_serialization.png)

Jackson by default does not support serializing mockito's mocks even if you makes it serializable like below.

```java
final MyPipelineOptions pipelineOptions = PipelineOptionsFactory.create().as(MyPipelineOptions.class);
pipelineOptions.setContext(mock(MyContext.class, withSettings().serializable()));
// More setups...
```

# Making your mocks serializable by Beam
Now that we know the root cause, solving it is not really that difficult. Basically you just need to make Jackson happy with mockito's mocks.

One way for doing this is to define your own serializer/deserializer for Jackson to use with the specific type you are trying to mock. For our case, the MyContext class. 

Since we just want to make our pipeline able to run with mocks and don't really care about the intermediate JSON content, we can easily do with Java's native serialization mechanism.

```java
public class MyContextModule extends SimpleModule {
    public MyContextModule() {
        addSerializer(MyContext.class, new JsonSerializer<MyContext>() {

            @Override
            public void serialize(MyContext value, JsonGenerator gen, SerializerProvider serializers)
                    throws IOException {
                gen.writeStartObject();
                
                final ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
                try (final ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream)) {
                    objectOutputStream.writeObject(value);
                    objectOutputStream.flush();

                    gen.writeBinaryField("myContext", byteArrayOutputStream.toByteArray());
                }

                gen.writeEndObject();
            }

        });
        
        addDeserializer(MyContext.class, new JsonDeserializer<MyContext>() {

            @Override
            public MyContext deserialize(JsonParser p, DeserializationContext ctxt)
                    throws IOException, JacksonException {
                final JsonNode node = p.getCodec().readTree(p);

                final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(
                        node.get("myContext").binaryValue());
                try (final ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream)) {
                    return (MyContext) objectInputStream.readObject();
                } 
                catch (ClassNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }
        });
    }
}
```

Now you might wonder how we can register our serialization module with the Jackson ```ObjectMapper``` used by Beam's direct runner or ```TestPipeline```. There must be some way to supply it with a custom ```ObjectMapper``` or retrieve the one used created by Beam, right?

Turns out the answer is no. Beam only uses the ```ObjectMapper``` it created internally and never exposes it.

But don't just give up yet. It does provide a way to custom this internal ```ObjectMapper``` via a slightly less flexible way. That is, using Java's [ServiceLoader](http://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html?is-external=true) facility as it calls Jackson's [```ObjectMapper.findModules()```](https://fasterxml.github.io/jackson-databind/javadoc/2.9/com/fasterxml/jackson/databind/ObjectMapper.html#findModules-java.lang.ClassLoader-). 

![beam_object_mapper](/assets/2022-07-03-Mocking-Apache-Beams-pipeline-options-with-mockito/beam_object_mapper.png)

What you need to do is to create a file named 'com.fasterxml.jackson.databind.Module' under 'src\test\resources\META-INF\services' and put the fully qualified class name of your module in it.
![service_loader_registration](/assets/2022-07-03-Mocking-Apache-Beams-pipeline-options-with-mockito/service_loader_registration.png)

That's it! Now Beam should no longer hate your mocks and you can just test away!