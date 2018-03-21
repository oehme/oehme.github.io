---
layout: post
title: "The build-changing magic of tidying up"
---

Organizing your build logic properly can make the difference between a tool that your developers will love or a mess of spaghetti code that no one dares to touch.

Gradle offers different ways to organize build logic:

1. Scripting in a project's build script
2. Writing reusable scripts
3. Writing binary plugins in [`buildSrc`](https://docs.gradle.org/current/userguide/organizing_build_logic.html#sec:build_sources)
4. Writing binary plugins shared through a repository with other teams

Unfortunately, I often see users not making the leap from step 2 to step 3 as early as they should.
Scripts should only contain declarative configuration.
As soon as you need control flow, complex computations or external dependencies, you should move that logic to `buildSrc`

`buildSrc` is a special project which is built before your settings and build scripts are executed.
This means it can contribute additional plugins and tasks that your build scripts can use.
You can write these types in any JVM language you like.
The dependencies of the `buildSrc` project are visible to all build scripts in your project.

Let's see how these properties make `buildSrc` such a great tool.

## Plugin classpath management

### Without buildSrc

If you have ever tried splitting a complex build script into multiple smaller ones you may have run into [classloading problems](https://github.com/gradle/gradle/issues/1262).
Project build scripts cannot see classes loaded by scripts that they apply.

For instance, you may have a `bintray.gradle` script that loads the Bintray plugin and configures some defaults.
Already there are some oddities - you have to apply the plugin by its class name instead of id.

<figure>
  <figcaption>bintray.gradle</figcaption>
{% highlight groovy %}
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7.3'
    }
}

apply plugin: com.jfrog.bintray.gradle.BintrayPlugin

tasks.withType(BintrayUploadTask) {
  //set some defaults
}
{% endhighlight %}
</figure>

But even worse - when you try referencing any of the Bintray plugin's types in a project script, you'll
be greeted with script compilation errors.

<figure>
  <figcaption>some-project.gradle</figcaption>
{% highlight groovy %}
apply from: 'bintray.gradle'

tasks.withType(BintrayUploadTask) { //won't compile :(
  //some project-specific config
}
{% endhighlight %}
</figure>

### With buildSrc

`buildSrc` on the other hand exposes all its types and dependencies to all scripts in your project.
This means that `buildSrc` is the perfect place to centrally manage all your plugin dependencies.

<figure>
  <figcaption>buildSrc/build.gradle</figcaption>
{% highlight groovy %}
apply plugin: 'java-library'
apply plugin: 'java-gradle-plugin'

repositories {
    jcenter()
}
dependencies {
    api 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7.3'
}
{% endhighlight %}
</figure>

The two scripts from earlier can now be simplified and will work just as you'd expect.

<figure>
  <figcaption>bintray.gradle</figcaption>
{% highlight groovy %}
apply plugin: 'com.jfrog.bintray'

tasks.withType(BintrayUploadTask) {
  //set some defaults
}
{% endhighlight %}
</figure>

<figure>
  <figcaption>some-project.gradle</figcaption>
{% highlight groovy %}
apply from: 'bintray.gradle'

tasks.withType(BintrayUploadTask) {
  //some project-specific config
}
{% endhighlight %}
</figure>

## IDE support

The example above still uses a script for configuring the Bintray defaults.
I highly recommend moving this logic into `buildSrc` as soon as it gets more complex.
Writing plugins in `buildSrc` gives you much better IDE support than writing scripts.
Your IDE will offer you syntax highlighting, auto completion, source navigation and refactoring.

Don't be afraid to make the move. Both scripts and plugins use the same Gradle API.
A script is just a plugin without the class/method boilerplate.
Here is how that plugin could look like when copy-pasted to `buildSrc`.

<figure>
  <figcaption>buildSrc/src/main/groovy/MyBintrayPlugin.groovy</figcaption>
{% highlight groovy %}
class MyBintrayPlugin implements Plugin<Project> {
  void apply(Project project) {
    project.with {
      apply plugin: 'com.jfrog.bintray'
      tasks.withType(BintrayUploadTask) {
        //set some defaults
      }
    }
  }
}
{% endhighlight %}
</figure>

You can then give this plugin an id and use it in your project like so:

<figure>
  <figcaption>buildSrc/build.gradle</figcaption>
{% highlight groovy %}
gradlePlugin {
    plugins {
        bintray {
            id = 'bintray'
            implementationClass = 'MyBintrayPlugin'
        }
    }
}
{% endhighlight %}
</figure>


<figure>
  <figcaption>some-project.gradle</figcaption>
{% highlight groovy %}
apply plugin: 'bintray'

tasks.withType(BintrayUploadTask) {
  //some project-specific config
}
{% endhighlight %}
</figure>

## Testing

Another reason to keep complex logic out of scripts is that scripts cannot be tested.
`buildSrc` allows you to write unit tests for low-level implementation details and functional [tests](https://guides.gradle.org/testing-gradle-plugins/) for the user-facing behavior
of your plugins and tasks. This is essential as your build grows more complex, as a good test suite
allows you to refactor without fear of breaking your developers' workflow.

## Performance

While dynamic Groovy allows you to build very succinct and readable DSLs, it also has a rather high execution overhead.
If you have a large project and scripts that you apply to all subprojects, consider using statically compiled `buildSrc` plugins instead.
Of course, you could just switch the script over to the statically compiled Kotlin DSL.
But moving it to `buildSrc` and turning it into a proper plugin gives you all the other benefits mentioned above.
And if neither Kotlin nor Groovy are your cup of tea when it comes to writing complex logic,
you can write it in plain old Java too. Any statically compiled language is great for performance.

For instance, here is the plugin from earlier, translated to Java.
<figure>
  <figcaption>buildSrc/src/main/groovy/MyBintrayPlugin.java</figcaption>
{% highlight groovy %}
public class MyBintrayPlugin implements Plugin<Project> {
  public void apply(Project project) {
    project.plugins.apply('com.jfrog.bintray')
    project.tasks.withType(BintrayUploadTask, () -> {
      //set some defaults
    })
  }
}
{% endhighlight %}
</figure>

## Sharing

Once you have extracted plugins to `buildSrc`, the next step towards making them a standalone project shared with other
teams is much simpler. You'll already have the project structure in place and a good suite of tests to show others what
to expect of your plugin.

## Readability

Extracting complex logic into `buildSrc` and only leaving declarative logic in your build scripts makes it much easier
for others to customize the few details they care about.
Don't make your team members read a bunch of unrelated code just to change a dependency or add another subproject.

## Conclusion

Properly structuring your build takes some discipline.
You need to treat your build logic the same way you would write your production code.
Complex control flow should be well-tested and remain separate from declarative configuration.
In return you don't just get a more manageable build, but better IDE support, performance and convenience.
I hope I could inspire you to give your build a fresh look and ask yourself: "What will I extract today?"
