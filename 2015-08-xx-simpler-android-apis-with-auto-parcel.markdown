# Simpler Android APIs with auto-parcel

I have made a rather small change to one of the Android libraries I maintain that I liked quite a lot, so I wanted to write a few words about it.

## Strings! Strings everywhere!

One of the things I like the least about the Android framework is its tendency to use Strings in all the wrong places. Java already has a (rightfully) terrible reputation for its type system. The Android SDK, however, goes a step further by forcing people to serialize their well-typed data structures and handing back nothing but a String in return. If you’ve ever accidentally called `Intent#getStringExtra` on an Integer you’ll know what I’m talking about. There is nothing at compile time preventing you from doing this you in this [Stringly Typed System](http://c2.com/cgi/wiki?StringlyTyped) except for manual careful checks that you’ve passed in the corresponding constants on both sides of the serialization boundary.

## Breaking Changes

My library [Android-DirectoryChooser](https://github.com/passy/Android-DirectoryChooser/) has received some wonderful contributions lately, but with the unfortunate side-effect that the factory method grew from a reasonable `newInstance(@NonNull final String newDirectoryName)` to an unwieldy `newInstance(@NonNull final String newDirectoryName, @Nullable final String initialDirectory, final boolean allowReadOnlyDirectory, final boolean allowNewDirectoryNameModification)`. Positional boolean parameters are among my least favorite things, right after slow-moving people in The Underground. All arguments were then used to prepare a `Bundle` that was set on a `Fragment`. In order to maintain backwards-compatibility, there were also overloaded versions of the method for three, two and one argument. The time had come for a refactor.

## Enter AutoParcelable

So how do we fix this? The obvious way in Java is to extract those parameters into its own class. We could just slam an `implements Serializable` on it and call it a day, but we know that it comes with a [non-negligable cost on Android](http://www.3pillarglobal.com/insights/parcelable-vs-java-serialization-in-android-app-development). The alternative is to implement a `Parcelable`, but doing this by hand is both tedious and error-prone. Luckily, there are plenty of ways these days to free us from this manual task and among my favorites is [auto-parcel](https://github.com/frankiesardo/auto-parcel) which builds on top of Google’s [auto-value](https://github.com/google/auto/tree/master/value). After including the necessary dependencies as described in the project’s [README](https://github.com/frankiesardo/auto-parcel/blob/master/README.md), all we need to do is declare a small abstract class like this:

```java
@AutoParcel
public abstract class ChooserConfig implements Parcelable {
    abstract String newDirectoryName();
    abstract String initialDirectory();
    abstract boolean allowReadOnlyDirectory();
    abstract boolean allowNewDirectoryNameModification();
}
```

`auto-value` utilizes annotation preprocessors to generate code for you at compile time and comes with zero runtime overhead (as compared to hand-written code). After one javac run, you have a `AutoParcel_ChooserConfig` in your classpath ready with a `Parcelable` implementation you didn’t have to lift a finger for and even null checks in its constructor for all non-primitive parameters.

This is already pretty great, but if we look at the constructor, it didn’t help our boolean blindness much:

```java
AutoParcel_ChooserConfig(
      String newDirectoryName,
      String initialDirectory,
      boolean allowReadOnlyDirectory,
      boolean allowNewDirectoryNameModification)
```

So we’ve actually just added *more* syntactical noise as you now have to create a new object that you then have to pass to the `newInstance` method. But the logic inside already simplifies drastically as we can directly store the config object as argument instead of having to deal with those dreadful strings.

## Auto, the builder

But we can do better. Just as `auto-value` can generate `Parcelable`s for you, it can generate Builders. And again, all it takes is an annotation and some abstract methods. The class is nested inside our previous `ChooserConfig` class:

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

This may look a bit repetitive, but this way we remain in control about the fields we actually want to expose through the builder to the user. In this case, there’s a 1:1 correspondence, but you can easily imagine cases where you have private state that the caller of your builder is not supposed to control. Remember that all of this happens at compile-time. So if you make a typo while writing your builder, you’ll see this at compile time rather than through an exception at runtime.

Another `javac` run, and you get a `AutoParcel_ChooserConfig.Builder` class generated for you with an empty constructor as well as a copy-constructor that takes an existing `ChooserConfig` object.

## Sssh, private!

Something I haven’t told you before is, that you actually cannot use `AutoParcel_` prefixed classes directly in your application code because they’re marked as package-local. This is by design so you remain in charge of which interfaces to expose. In our case, this is already very handy, because we’d much rather people to use the builder than that ugly constructor directly.

We do this by adding a concrete method to our otherwise abstract `ChooserConfig` class:

```java
public static Builder builder() {
    new AutoParcel_ChooserConfig.Builder().newDirectoryName()
    return new AutoParcel_ChooserConfig.Builder()
            .initialDirectory("")
            .allowNewDirectoryNameModification(false)
            .allowReadOnlyDirectory(false);
}
```

As you can see, we can also use this to set some default options for our Builder. One of my favorite things about the generated builders is that they check for non-optional fields at construction time and raise exceptions if you’ve missed one. The generated code for those checks is very efficient and uses a BitSet under the hood to keep track of the values set.

## Aaand, action!

Let’s put this into action and compare it to what we had before.

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

And behind the scenes, instead of having to juggle with four different `EXTRA_*` Strings, we have only one Parcelable to deal with. For this particular example, the savings are even bigger because there’s an Activity interface for this that mirrors the Fragment, which can now reuse the config object. You can find the [full changeset on GitHub](https://github.com/passy/Android-DirectoryChooser/pull/60/files).

## Conclusion

`auto-value` is a brilliant little library that can take away a lot of the pains that we normally face in Android development. By replacing the common pattern of manually passing extras via strings to our Activities or Fragments with immutable configuration objects and builders, we get several benefits:

- Simpler constructors. No more positional arguments and subtle bugs by accidentally swapping boolean arguments and chains of `null, null, null, …`.
- Auto-completion. By using builders, we get the list of possible options from pressing `<Ctrl/Cmd>-<Space>` instead of reading docs or searching the class we’re trying to instantiate for `static final String EXTRA_` fields.
- Easier integration testing. You no longer need to intercept Activity or Fragment construction and manually check their bundles for the correct arguments, possibly missing one of the fields. `auto-value` generates an `equals()` method for you, so you can compare full objects instead.
- Forward-compatibility. If you want to add a new argument, you can simply add a new method to your builder and set a default value without breaking existing code.
- Immutability. Objects created with Auto Value are immutable and can freely be shared across different threads without defensive copying.

I am quite fond of this pattern and will definitely use it more in the future. Let me know what you think about this by tweeting me at [@passy](https://twitter.com/passy).
