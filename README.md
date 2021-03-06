Moshi
=====

Moshi is a modern JSON library for Android and Java. It makes it easy to parse JSON into Java
objects:

```java
String json = ...;

Moshi moshi = new Moshi.Builder().build();
JsonAdapter<BlackjackHand> jsonAdapter = moshi.adapter(BlackjackHand.class);

BlackjackHand blackjackHand = jsonAdapter.fromJson(json);
System.out.println(blackjackHand);
```

And it can just as easily serialize Java objects as JSON:

```java
BlackjackHand blackjackHand = new BlackjackHand(
    new Card('6', SPADES),
    Arrays.asList(new Card('4', CLUBS), new Card('A', HEARTS)));

Moshi moshi = new Moshi.Builder().build();
JsonAdapter<BlackjackHand> jsonAdapter = moshi.adapter(BlackjackHand.class);

String json = jsonAdapter.toJson(blackjackHand);
System.out.println(json);
```

### Built-in Type Adapters

Moshi has built-in support for reading and writing Java’s core data types:

 * Primitives (int, float, char...) and their boxed counterparts (Integer, Float, Character...).
 * Arrays, Collections, Lists, Sets, and Maps
 * Strings
 * Enums

It supports your model classes by writing them out field-by-field. In the example above Moshi uses
these classes:

```java
class BlackjackHand {
  public final Card hidden_card;
  public final List<Card> visible_cards;
  ...
}

class Card {
  public final char rank;
  public final Suit suit;
  ...
}

enum Suit {
  CLUBS, DIAMONDS, HEARTS, SPADES;
}
```

to read and write this JSON:

```json
{
  "hidden_card": {
    "rank": "6",
    "suit": "SPADES"
  },
  "visible_cards": [
    {
      "rank": "4",
      "suit": "CLUBS"
    },
    {
      "rank": "A",
      "suit": "HEARTS"
    }
  ]
}
```

The [Javadoc][javadoc] catalogs the complete Moshi API, which we explore below.

### Custom Type Adapters

With Moshi, it’s particularly easy to customize how values are converted to and from JSON. A type
adapter is any class that has methods annotated `@ToJson` and `@FromJson`.

For example, Moshi’s default encoding of a playing card is verbose: the JSON defines the rank and
suit in separate fields: `{"rank":"A","suit":"HEARTS"}`. With a type adapter, we can change the
encoding to something more compact: `"4H"` for the four of hearts or `"JD"` for the jack of
diamonds:

```java
class CardAdapter {
  @ToJson String toJson(Card card) {
    return card.rank + card.suit.name().substring(0, 1);
  }

  @FromJson Card fromJson(String card) {
    if (card.length() != 2) throw new JsonDataException("Unknown card: " + card);

    char rank = card.charAt(0);
    switch (card.charAt(1)) {
      case 'C': return new Card(rank, Suit.CLUBS);
      case 'D': return new Card(rank, Suit.DIAMONDS);
      case 'H': return new Card(rank, Suit.HEARTS);
      case 'S': return new Card(rank, Suit.SPADES);
      default: throw new JsonDataException("unknown suit: " + card);
    }
  }
}
```

Register the type adapter with the `Moshi.Builder` and we’re good to go.

```java
Moshi moshi = new Moshi.Builder()
    .add(new CardAdapter())
    .build();
```

Voila:

```json
{
  "hidden_card": "6S",
  "visible_cards": [
    "4C",
    "AH"
  ]
}
```

#### Another example

Note that the method annotated with `@FromJson` does not need to take a String as an argument. Rather it can take
input of any type and Moshi will first parse the JSON to an object of that type and then use the `@FromJson`
method to produce the desired final value. Conversely, the method annotated with `@ToJson` does not have to produce
a String.

Assume, for example, that we have to parse a JSON in which the date and time of an event are represented as two
separate strings.

```json
{
  "title": "Blackjack tournament",
  "begin_date": "20151010",
  "begin_time": "17:04"
}
```

We would like to combine these two fields into one string to facilitate the date parsing at a
later point. Also, we would like to have all variable names in CamelCase. Therefore, the `Event` class we
want Moshi to produce like this:

```java
class Event {
    String title;
    String beginDateAndTime;
}
```

Instead of manually parsing the JSON line per line (which we could also do) we can have Moshi
do the transformation automatically. We simply define another class `EventJson` that directly corresponds to the JSON structure:

```java
class EventJson {
    String title;
    String begin_date;
    String begin_time;
}
```

And another class with the appropriate `@FromJson` and `@ToJson` methods that are telling Moshi how to convert
an `EventJson` to an `Event` and back. Now, whenever we are asking Moshi to parse a JSON to an `Event` it will first parse it to an `EventJson` as an intermediate step. Conversely, to serialize an `Event` Moshi will first
create an `EventJson` object and then serialize that object as usual.

```java
class EventJsonAdapter {

    @FromJson
    Event eventFromJson(EventJson eventJson) {
        Event event = new Event();
        event.title = eventJson.title;
        event.beginDateAndTime = eventJson.begin_date + " " + eventJson.begin_time;
        return event;
    }

    @ToJson
    EventJson eventToJson(Event event) {
        EventJson json = new EventJson();
        json.title = event.title;
        json.begin_date = event.beginDateAndTime.substring(0, 8);
        json.begin_time = event.beginDateAndTime.substring(9, 14);
        return json;
    }

}
```

Again we register the adapter with Moshi.

```java
Moshi moshi = new Moshi.Builder()
    .add(new EventJsonAdapter())
    .build();
```

We can now use Moshi to parse the JSON directly to an `Event`.

```java
JsonAdapter<Event> jsonAdapter = moshi.adapter(Event.class);
Event event = jsonAdapter.fromJson(json);
```

### Fails Gracefully

Automatic databinding almost feels like magic. But unlike the black magic that typically accompanies
reflection, Moshi is designed to help you out when things go wrong.

```
JsonDataException: Expected one of [CLUBS, DIAMONDS, HEARTS, SPADES] but was ANCHOR at path $.visible_cards[2].suit
  at com.squareup.moshi.JsonAdapters$11.fromJson(JsonAdapters.java:188)
  at com.squareup.moshi.JsonAdapters$11.fromJson(JsonAdapters.java:180)
	...
```

Moshi always throws a standard `java.io.IOException` if there is an error reading the JSON document,
or if it is malformed. It throws a `JsonDataException` if the JSON document is well-formed, but
doesn’t match the expected format.

### Built on Okio

Moshi uses [Okio][okio] for simple and powerful I/O. It’s a fine complement to [OkHttp][okhttp],
which can share buffer segments for maximum efficiency.

### Borrows from Gson

Moshi uses the same streaming and binding mechanisms as [Gson][gson]. If you’re a Gson user you’ll
find Moshi works similarly. If you try Moshi and don’t love it, you can even migrate to Gson without
much violence!

But the two libraries have a few important differences:

 * **Moshi has fewer built-in type adapters.** For example, you need to configure your own date
   adapter. Most binding libraries will encode whatever you throw at them. Moshi refuses to
   serialize platform types (`java.*`, `javax.*`, and `android.*`) without a user-provided type
   adapter. This is intended to prevent you from accidentally locking yourself to a specific JDK or
   Android release.
 * **Moshi is less configurable.** There’s no field naming strategy, versioning, instance creators,
   or long serialization policy. Instead of naming a field `visibleCards` and using a policy class
   to convert that to `visible_cards`, Moshi wants you to just name the field `visible_cards` as it
   appears in the JSON.
 * **Moshi doesn’t have a `JsonElement` model.** Instead it just uses built-in types like `List` and
   `Map`.
 * **No HTML-safe escaping.** Gson encodes `=` as `\u003d` by default so that it can be safely
   encoded in HTML without additional escaping. Moshi encodes it naturally (as `=`) and assumes that
   the HTML encoder – if there is one – will do its job.


Download
--------

Download [the latest JAR][dl] or depend via Maven:
```xml
<dependency>
  <groupId>com.squareup.moshi</groupId>
  <artifactId>moshi</artifactId>
  <version>1.0.0</version>
</dependency>
```
or Gradle:
```groovy
compile 'com.squareup.moshi:moshi:1.0.0'
```

Snapshots of the development version are available in [Sonatype's `snapshots` repository][snap].


License
--------

    Copyright 2015 Square, Inc.

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


 [dl]: https://search.maven.org/remote_content?g=com.squareup.moshi&a=moshi&v=LATEST
 [snap]: https://oss.sonatype.org/content/repositories/snapshots/com/squareup/moshi/
 [okio]: https://github.com/square/okio/
 [okhttp]: https://github.com/square/okhttp/
 [gson]: https://github.com/google/gson/
 [javadoc]: http://square.github.io/moshi/1.x/moshi/
