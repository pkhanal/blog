---
date: 2017-06-28T20:47:03-07:00
subtitle: "Leverage IBM Watson Java SDK from Play Application"
tags: ["java", "cognitive", "watson", "play", "akka", "lightbend"]
title: Add Cognitive Capability into Akka Play Application
---

As a Software Engineer and Dev Advocate for IBM Cloud and Watson platform, we are always looking for new ways to build cloud-native cognitive solution leveraging IBM Watson Platform in collaboration with external technologies. I am really excited about this morning’s <a href="https://developer.ibm.com/dwblog/2017/ibm-lightbend-scala/"><u>announcement</u></a> from IBM regarding a new collaborative development initiative with <a href="https://www.lightbend.com/"><u>Lightbend</u></a>. The joint effort will provide developers a single stop solution for building and deploying distributed cognitive application both on premises and in the cloud. 

<!--more-->

Although I use IBM Cloud & Watson Platform day in and day out, I haven’t really worked on Lightbend Reactive Platform that comprises of <a href="https://www.lightbend.com/platform/development/akka">Akka</a>, <a href="https://www.lightbend.com/platform/development/play-framework">Play</a> and <a href="https://www.lightbend.com/platform/development/lagom-framework">Lagom</a>. As a developer, the best way to find out is to start building something simple out of those technologies and take it from there. The rest of the blog discusses the technical recipes to add a cognitive capability to Java based Play application with the help of online documentation and other resources about Lightbend Reactive Platform. For the purpose of this blog, we will use <a href="https://github.com/watson-developer-cloud/java-sdk">Watson Java SDK</a> from within the Play application to add the cognitive capability.

The sample project for this blog post is available in <a href="https://github.com/pkhanal/lightbend-watson-integration">Github</a>.

First, let’s review the architecture.

![Architecture](/img/architecture.jpg?raw=true "Architecture")

The play application exposes following REST endpoints:

<strong>/imageClassification?imageUrl=\<image_url\></strong><BR />
classify an image i.e. identify objects in an image leveraging <a href="https://www.ibm.com/watson/developercloud/visual-recognition.html">Watson Visual Recognition</a>. Watson Visual Recognition enables developer to classify image content using machine learning.

<strong>/translation?text=\<text\>&from=\<source_langauge\>&to=\<target_langauge\></strong><BR />
translates given text to target language using <a href="">Watson Lanugage Translator</a>
 
In the above application, each asynchronous call to a watson cognitive service is encapsulated to an Akka Actor. Actor is capable of much more than just encapsulating a behavior to process a message that it receives. For more details on Akka Actor, I would recommend going though the following <a href="http://doc.akka.io/docs/akka/current/java/actors.html">doc</a>. The following <a href="https://www.youtube.com/watch?v=6Cb1wSVRI-Q">talk</a> from Jonas Bonér provides a great explanation on Akka Actor from the perspective of reactive system.

### Get Started
The following <a href="https://github.com/playframework/play-java-starter-example/tree/2.6.x">starter</a> project is a great way to get started with Java based Play application. You can clone the project and import into your IDE (Intellij, Eclipse..) of your choice. For details on how to import the project into your IDE, take a look the the following <a href="https://www.playframework.com/documentation/2.6.x/IDE">doc</a>. Once you follow the steps, you should be able to run/debug the application from your IDE.

### Add Watson Java SDK Dependency
In order to add a dependency to Watson Visual Recognition and Language, we will add the dependencies to Java SDK for <a href="https://github.com/watson-developer-cloud/java-sdk/tree/develop/visual-recognition">Visual Recognition Service</a> and <a href="https://github.com/watson-developer-cloud/java-sdk/tree/develop/language-translation">Language Translator Service</a> in the <a href="https://github.com/pkhanal/lightbend-watson-integration/blob/master/build.sbt">build.sbt</a> file as shown below.

{{< highlight java >}}

libraryDependencies ++= Seq(
  "com.ibm.watson.developer_cloud" % "language-translator" % "3.8.0",
  "com.ibm.watson.developer_cloud" % "visual-recognition" % "3.8.0"
)
{{</ highlight >}}

### Add Watson Credentials to the Application Settings
Follow the steps provided in the following link to obtain the credentials of the services (Visual Recognition & Language Translator) for the project. Then create a watson.conf file under the <a href="https://github.com/pkhanal/lightbend-watson-integration/tree/master/conf">conf</a> directory of the starter project where we will specify the service credentials/key as shown below.

{{< highlight java >}}

# application specific settings
# copy credentials from visual recognition & language translation services
watson {
  translation {
    username = ""
    password = ""
  },
  visual_recognition {
    key = ""
  }
}
{{</ highlight >}}

We will then import the watson.conf file in the <a href="https://github.com/pkhanal/lightbend-watson-integration/blob/master/conf/application.conf">application.conf</a> file as shown below.

```
include "watson.conf"
```

You can find detail information on the Akka configuration from the following link.
http://doc.akka.io/docs/akka/2.5.3/java/general/configuration.html

As recommended by the Akka doc, we will place application <a href="http://doc.akka.io/docs/akka/2.5.3/java/extending-akka.html#extending-akka-settings">specific settings in an Extension.</a>

For that, we will create <a href="https://github.com/pkhanal/lightbend-watson-integration/blob/master/app/settings/SettingsImpl.java">SettingsImpl.java</a> class under the package settings as shown below.

{{< highlight java >}}

package settings;

import akka.actor.Extension;
import com.typesafe.config.Config;


public class SettingsImpl implements Extension {

    public final String WATSON_TRANSLATOR_USERNAME;
    public final String WATSON_TRANSLATOR_PASSWORD;
    public final String WATSON_VISUAL_RECOGNITION_KEY;

    public SettingsImpl(Config config) {
        WATSON_TRANSLATOR_USERNAME = config.getString("watson.translation.username");
        WATSON_TRANSLATOR_PASSWORD = config.getString("watson.translation.password");
        WATSON_VISUAL_RECOGNITION_KEY = config.getString("watson.visual_recognition.key");
    }

}

{{</ highlight >}}

Add <a href="https://github.com/pkhanal/lightbend-watson-integration/blob/master/app/settings/Settings.java">Settings.java</a> class as well under the package settings as shown below: 

{{< highlight java >}}

package settings;

import akka.actor.AbstractExtensionId;
import akka.actor.ExtendedActorSystem;
import akka.actor.ExtensionIdProvider;

public class Settings extends AbstractExtensionId<SettingsImpl>
        implements ExtensionIdProvider {

    public final static Settings SettingsProvider = new Settings();

    private Settings() {}

    public Settings lookup() {
        return Settings.SettingsProvider;
    }

    public SettingsImpl createExtension(ExtendedActorSystem system) {
        return new SettingsImpl(system.settings().config());
    }
}

{{</ highlight >}}

We will later use the application settings defined in configuration file through **SettingsImpl** & **Settings** in Actors.

### Define Actors
We will create Actor that encapsulates behavior of corresponding Watson Service. Below is a <a href="https://github.com/pkhanal/lightbend-watson-integration/blob/master/app/actors/ImageClassifier.java">visual recognition actor</a> that given an image url, will classify the image and detect faces in the image using Watson Java SDK for visual recognition.

{{< highlight java >}}

package actors;

import akka.actor.AbstractActor;
import akka.actor.Props;
import com.ibm.watson.developer_cloud.visual_recognition.v3.VisualRecognition;
import com.ibm.watson.developer_cloud.visual_recognition.v3.model.ClassifyImagesOptions;
import com.ibm.watson.developer_cloud.visual_recognition.v3.model.VisualClassification;
import settings.Settings;
import settings.SettingsImpl;

import javax.inject.Singleton;

@Singleton
public class ImageClassifier extends AbstractActor {

    private final VisualRecognition service;

    {
        final SettingsImpl settings =
                Settings.SettingsProvider.get(getContext().getSystem());

        // https://github.com/watson-developer-cloud/java-sdk/tree/develop/language-translator
        service = new VisualRecognition(VisualRecognition.VERSION_DATE_2016_05_20);
        service.setApiKey(settings.WATSON_VISUAL_RECOGNITION_KEY);
    }

    public static Props getProps() {
        return Props.create(ImageClassifier.class);
    }

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .match(Message.class, message -> {
                    ClassifyImagesOptions options = new ClassifyImagesOptions.Builder()
                            .url(message.imageUrl)
                            .build();
                    VisualClassification result = service.classify(options).execute();
                    getSender().tell(result.toString(), getSelf());
                })
                .build();
    }

    // protocol
    public static class Message {
        private final String imageUrl;

        public Message(String imageUrl) {
            this.imageUrl = imageUrl;
        }

        public String toString() {
            return "Image Url: " + imageUrl;
        }
    }
}
{{</ highlight >}}

Notice here that the **ImageClassifier** defines a static method called **getProps**, this method returns a **Props** object that describes how to create the actor. This is a good Akka convention, to separate the instantiation logic from the code that creates the actor. Another best practice shown here is that the messages that ImageClassifier receives are defined as static inner class called **Message**. It serves as a protocol of the message to send to the ImageClassifier actor. All the details on the Akka Actor can be read from the following doc.

http://doc.akka.io/docs/akka/current/java/actors.html

I was blown away by the API design backed by strong documentation while working on this project. The functional approach to the API design made things lot simpler to write. The documentation is well writtern and provides various recommended practices for different situations when developing Actors. I would highly recommend going through the doc when getting started.

### Define REST Endpoint

Now that we have defined an Actor and encapsulated a behavior, we can now create a REST endpoint for client to consume Actor’s behavior over the web through HTTP Request. We can create a REST Endpoint by defining a <a href="https://github.com/pkhanal/lightbend-watson-integration/blob/master/app/controllers/VisualRecognitionController.java">Controller</a> that creates and uses the ImageClassifier actor.

{{< highlight java>}}
package controllers;


import actors.ImageClassifier;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import play.mvc.Controller;
import play.mvc.Result;

import javax.inject.Inject;
import java.util.concurrent.CompletionStage;

import static akka.pattern.PatternsCS.ask;

public class VisualRecognitionController extends Controller {

    final ActorRef imageClassifer;

    @Inject
    public VisualRecognitionController(ActorSystem system) {
        imageClassifer = system.actorOf(ImageClassifier.getProps());
    }

    public CompletionStage<Result> classifyImage(String imageUrl) {
        ImageClassifier.Message message = new ImageClassifier.Message(imageUrl);
        return ask(imageClassifer, message, 5000L)
                .thenApply(response -> ok((String) response));
    }

}
{{</ highlight >}}

The ActorSystem is used to create and/or use an actor. The basic functionality available with Actor is to send a message for computation or processing. There are two patterns for that:

- **tell pattern (Fire-forget)** where there is no response from actor.
  I would recommend reading the following doc on **tell pattern**:
  http://doc.akka.io/docs/akka/current/java/actors.html#tell-fire-forget

- **ask pattern (Send-And-Receive-Future)** where actor sends response. This pattern is used when sending message to actor from a REST endpoint where client expects the response.
I would recommend reading the following doc on the **ask pattern**:
http://doc.akka.io/docs/akka/current/java/actors.html#ask-send-and-receive-future


### Add Routes

The final step is to add routes for the rest endpoint. Add the following route in routes file for image classification as shown below: 

```
# Image Classification controller that classifies image in the given image url and detect faces
GET    /imageClassification         controllers.VisualRecognitionController.classifyImage(imageUrl:String)
```

Here we have created a REST endpoint ``/imageClassification`` that can be accessed by ``GET`` method and it requires a query parameter called ``imageUrl``. For more details on routing, you can read the following <a href="https://www.playframework.com/documentation/2.6.x/JavaRouting">doc</a>.

### Test REST endpoint with curl command

Now we can run the play application and test image classification REST endpoint through following command:

```
curl -G "http://localhost:9000/imageClassification” --data-urlencode “imageUrl=https://visual-recognition-demo.mybluemix.net/images/samples/5.jpg"
```

You will see the following response message in the console:

```

  "images": [
    {
      "classifiers": [
        {
          "classes": [
            {
              "class": "Chihuahua dog",
              "score": 0.944,
              "type_hierarchy": "/domestic animal/small dog/Chihuahua dog"
            },
           {
              "class": "small dog",
              "score": 0.961
            },
            {
              "class": "dog",
              "score": 0.962
            }
            …

```

### Conclusion

All in all, I had fun trying this out since my goal was to get something up and running in a quick time. Akka & Play are mature frameworks widely used in the industry and backed by strong documentation and developer community. That really helped me to get my hands dirty in quick time and be able to build a simple application. Although, the recipe may not be the production ready but it did help me to understand the bits and pieces of Lightbend Reactive Platform and gave me a good foundation take it to the next level.

### Resources

https://github.com/pkhanal/lightbend-watson-integration
http://doc.akka.io/docs/akka/current/java/actors.html
https://www.playframework.com/documentation/2.5.x/IDE
https://www.playframework.com/documentation/2.6.x/JavaAkka
https://www.ibm.com/watson/developercloud/visual-recognition.html
https://www.ibm.com/watson/developercloud/language-translator.html
