---
date: 2017-05-25T19:10:32-07:00
subtitle: "Watson Developer Cloud Java SDK"
tags: ["java", "cognitive", "watson"]
title: Reactive Cognitive API
---

Starting <strong>v3.0.1</strong>, <a href="https://github.com/watson-developer-cloud/java-sdk" target="_blank">Watson Developer Cloud Java SDK</a> has introduced <a href="https://github.com/watson-developer-cloud/java-sdk#introduce-reactive-api-call-for-v301" target="_blank">Reactive API</a> that allows developers to compose and combine asynchronous congitive operations in a reactive fashion.

<!--more-->

For each Service class (TextToSpeech, LanguageTranslator, SpeechToText...) now has <strong>``.rx()``</strong> method that returns <strong>``CompletableFuture``</strong>. ``CompletableFuture`` provides powerful APIs to build asynchronous system.

Let us consider a scenario and try to achive that using Reactive Watson API. Let's say we want to achieve following use cases in our cognitive system:

- text translation from English to French
- speech synthesis of tranlsated text

For this, we will use <a href="https://github.com/watson-developer-cloud/java-sdk/tree/develop/language-translator">LanguageTranslator</a> and <a href="https://github.com/watson-developer-cloud/java-sdk/blob/develop/text-to-speech">TextToSpeech</a> API from Watson Developer Cloud Java SDK. We will leverage Reactive API to chain two asynchronous operations (translation and speech synthesis) and gather result.

Assuming that the bluemix account is already created and both <a href="https://console.ng.bluemix.net/catalog/services/language-translator">Language Translator</a> and <a href="https://console.ng.bluemix.net/catalog/services/text-to-speech">Text to Speech</a> services are available for use, let's jump into the code section that shows how the two asynchronous operations are chained together.

{{< highlight java "linenos=inline">}}
    ...

    LanguageTranslator translator = new LanguageTranslator();
    translator.setUsernameAndPassword("<username>", "<password>");

    TextToSpeech tts = new TextToSpeech();
    tts.setUsernameAndPassword("<username>", "<password>");

    translator
      .translate("hello", Language.ENGLISH, Language.FRENCH)
      .rx()
      .thenApply(translationResult -> translationResult.getFirstTranslation())
      .thenApply(translation -> tts.synthesize(translation, Voice.FR_RENEE, AudioFormat.WAV).rx())
      .thenAccept(App::processSpeechSynthesis);

    ...
{{</ highlight >}}

Although it may look intimidating at first glance but it actually makes more sense once we get to know the Reactive API and what it does. On line **11**, we are grabbing the ``CompletableFuture`` for the translation operation and we are chaining it with the text to speech API on line **13**. Once the translation result is available, it is passed to the text to speech operation. ``thenApply`` is used to pass in the result from one asynchronous operation to another operation. On line **11**, we are grabbing the translated text from the result object of the translation and passing it to the text to speech operation. Finally, we use ``thenAccept`` to mark an end of chaining which tells that we are now ready to consume the result.

On line **14**, ``App::processSpeechSynthesis`` is a reference to the method that handles the result of speech synthesis. Method reference is a feature introduced in Java 8 that allows us to reference constructors or methods without executing them. ``processSpeechSynthesis`` is a static method that plays the audio data receivied from text to speech service.


{{< highlight java "linenos=inline">}}
    private static void processSpeechSynthesis(CompletableFuture<InputStream> result) {
        ...
        InputStream stream = result.get();
        stream = WaveUtils.reWriteWaveHeader(stream);
        AudioStream audioStream = new AudioStream(stream);
        AudioPlayer.player.start(audioStream);
        ...
    }
{{</ highlight >}}

The asynchronous pattern discussed above is just a tip of iceberg and it can be applied in many scenarios to build asynchronous system.

The code sample is available in <a href="https://github.com/pkhanal/watson-reactive-translation-tts" target="_blank">Github</a>.
