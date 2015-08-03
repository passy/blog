# Simpler Android APIs with AutoParcel

I have made a small change to one of the Android libraries I maintain and was quite happy with how it turned out, so I wanted to write a few words about it.

## Strings! Strings everywhere!

[![5560914199_f1219deea4_o.jpg](https://d23f6h5jpj26xu.cloudfront.net/h5axmlqrfonog_small.jpg)](http://img.svbtle.com/h5axmlqrfonog.jpg)

One of the things I like the least about the Android framework is its tendency to use Strings in all the wrong places. Java already has a (rightfully) terrible reputation for its type system. The Android SDK, however, goes a step further by forcing people to serialize their well-typed data structures and handing back nothing but a String in return. If you’ve ever accidentally called `Intent#getStringExtra` on an Integer you’ll know what I’m talking about. There is nothing preventing you from doing this at compile time. The only mitigation is manual careful checks that you’ve passed the corresponding constants on both sides of the serialization boundary.

## Breaking Changes

My library [Android-DirectoryChooser](https://github.com/passy/Android-DirectoryChooser/) has received several wonderful contributions lately, but with the unfortunate side-effect that the factory method grew from a reasonable 

```java
newInstance(@NonNull final String newDirectoryName)
```

to an unwieldy 

```java
newInstance(
    @NonNull final String newDirectoryName,
    @Nullable final String initialDirectory,
    final boolean allowReadOnlyDirectory,
    final boolean allowNewDirectoryNameModification)
```

Positional boolean parameters are among my least favorite things, right after slow-moving people on the Tube. All arguments were then used to prepare a `Bundle` that was set on a `Fragment`. To maintain backwards-compatibility, there were extra overloaded versions of the method for three, two and one argument. The time had come for a refactor.

## Enter AutoParcelable

[![5396608804_6a18cbecc4_o.jpg](https://d23f6h5jpj26xu.cloudfront.net/jqia19uyglryda_small.jpg)](http://img.svbtle.com/jqia19uyglryda.jpg)

So how do we fix this? The idiomatic way in Java is to extract these parameters into their own class. We could just slam an `implements Serializable` on it and go home early, but we know that it comes with a [non-negligable cost on Android](http://www.3pillarglobal.com/insights/parcelable-vs-java-serialization-in-android-app-development).

 An alternative is to implement a `Parcelable`, but doing this by hand is both tedious and error-prone. Luckily, there are plenty of ways these days to free us from this manual task and my favorite is [AutoParcel](https://github.com/frankiesardo/auto-parcel) which builds on Google’s [AutoValue](https://github.com/google/auto/tree/master/value). After including the necessary dependencies as described in the project’s [README](https://github.com/frankiesardo/auto-parcel/blob/master/README.md), all we need to do is declare a small abstract class like this:

```java
@AutoParcel
public abstract class ChooserConfig implements Parcelable {
    abstract String newDirectoryName();
    abstract String initialDirectory();
    abstract boolean allowReadOnlyDirectory();
    abstract boolean allowNewDirectoryNameModification();
}
```

The AutoValue library utilizes annotation preprocessors to generate code for you at compile time and comes with zero runtime overhead (as compared to hand-written code). After a single javac run, you get a `AutoParcel_ChooserConfig` on your classpath with a `Parcelable` implementation. You even get null checks in its constructor for all non-primitive parameters. Just bear in mind how much boilerplate this would normally require, cluttering your code reviews.

This is already pretty great, but if we look at the constructor, it hasn’t helped our boolean blindness problem much:

```java
AutoParcel_ChooserConfig(
      String newDirectoryName,
      String initialDirectory,
      boolean allowReadOnlyDirectory,
      boolean allowNewDirectoryNameModification)
```

In fact, we’ve actually added *more* syntactical noise as we now have to create a new object that we must pass to the `newInstance` method. But the internal method logic is already drastically simplified because we can directly store the config object as an argument instead of having to deal with strings.

## Auto, the builder

We can do better. Just as AutoParcel can generate `Parcelable`s for you, it can also generate Builders. And again, it’s simple. All it takes is an annotation and some abstract methods. We add the Builder nested inside our previous `ChooserConfig` class:

```java
@AutoParcel.Builder
public abstract static class Builder {
    public abstract Builder newDirectoryName(String s);
    public abstract Builder initialDirectory(String s);
    public abstract Builder allowReadOnlyDirectory(boolean b);
    public abstract Builder allowNewDirectoryNameModification(boolean b);
    public abstract ChooserConfig build();
}
```

This may look a bit repetitive, but this way we have more control about the fields we actually want to expose through the builder to the user. In this example, there’s a 1:1 correspondence, but you can easily imagine cases where you have private state that the caller of your builder is not supposed to control.

Remember all of this happens at compile-time, so you’ll see compilation errors rather than runtime exceptions if you make a typo while writing your builder, for instance.

Another `javac` and you’ll get a `AutoParcel_ChooserConfig.Builder` class generated with an empty constructor as well as a copy-constructor that takes an existing `ChooserConfig` object.

## Sssh, private!

[![4687262861_2e98963e69_o.jpg](https://d23f6h5jpj26xu.cloudfront.net/priz0km8uwic4g_small.jpg)](http://img.svbtle.com/priz0km8uwic4g.jpg)

I’ve neglected to tell you that you cannot actually use `AutoParcel_` prefixed classes directly in your application code because they’re marked as package-local. Sorry.

This is by design so you have flexibility in the interface you expose. This is very useful for us because we’d much rather clients used the builder than unsightly constructors directly.

We do this by adding a concrete method to our otherwise abstract `ChooserConfig` class:

```java
@AutoParcel.Builder
public abstract static class Builder {
    public abstract Builder newDirectoryName(String s);
    public abstract Builder initialDirectory(String s);
    public abstract Builder allowReadOnlyDirectory(boolean b);
    public abstract Builder allowNewDirectoryNameModification(boolean b);
    public abstract ChooserConfig build();
}
```

This may look a bit repetitive, but this way we have more control about the fields we actually want to expose through the builder to the user. In this example, there’s a 1:1 correspondence, but you can easily imagine cases where you have private state that the caller of your builder is not supposed to control.

Remember all of this happens at compile-time, so you’ll see compilation errors rather than runtime exceptions if you make a typo while writing your builder, for instance.

## Aaand, action!

[![6344955477_92cc14316d_o.jpg](https://d23f6h5jpj26xu.cloudfront.net/gmslfcat82icng_small.jpg)](http://img.svbtle.com/gmslfcat82icng.jpg)

Let’s put this into action and compare it with what we had originally.

```java
// Before
mDialog = DirectoryChooserFragment.newInstance(“DialogSample”, “”, false, true);

// After
final DirectoryChooserConfig config = DirectoryChooserConfig.builder()
    .newDirectoryName("DialogSample")
    .allowReadOnlyDirectory(true)
    .build();
mDialog = DirectoryChooserFragment.newInstance(config);
```

And behind the scenes, instead of having to juggle with four different `EXTRA_*` Strings, we have a single Parcelable instead. For this particular example, the savings are greater because there is an Activity interface that mirrors the Fragment, which can reuse the config object. You can find the [full changeset on GitHub](https://github.com/passy/Android-DirectoryChooser/pull/60/files).

## Conclusion

AutoValue is a brilliant little library that takes away a lot of the pains we face in Android development. Immutable configuration objects and builders have several benefits over the the flawed pattern of passing extras via strings to Activities and Fragments:

- **Simpler constructors.** No more positional arguments, no more hard-to-see bugs from accidental boolean argument swapping and no more chains of `null, null, null, …`.
- **Auto-completion.** When using builders, our IDE can suggest possible completion options from pressing `<Ctrl/Cmd>-<Space>`. This construction information is at a much closer point to the edit than reading docs or searching the class we’re trying to instantiate for `static final String EXTRA_` fields.
- **Easier integration testing.** You no longer have to write tests intercepting Activity or Fragment construction to manually check their bundles include the correct arguments, with the possibility of missing one of the fields. AutoValue generates an `equals()` method for you, so you can compare full objects instead.
- **Forward-compatibility.** To add a new construction argument you can simply add a new method to your builder and set a default value. No existing code is broken.
- **Immutability.** Objects created with Auto Value are immutable and able to be shared across different threads without defensive copying.

This is a great pattern and one I will definitely be using more in the future. Let me know what you think about this by tweeting me at [@passy](https://twitter.com/passy). Want to chat without a 140 character limit? Why don’t you [join my team at Twitter](https://about.twitter.com/careers/locations/london)?

---

Photo credits:

- https://flic.kr/p/48N5j8
- https://flic.kr/p/9dT2Mo
- https://flic.kr/p/89crWV
- https://flic.kr/p/aEFyd6

With special thanks to [Daithí Ó Crualaoich](https://twitter.com/twtrdaithi) for thoroughly proofreading and editing this article.
