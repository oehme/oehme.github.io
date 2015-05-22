---
layout: post
title: Xtext for IntelliJ  - A first Beta
author: Stefan Oehme
---
Today we released the first public Beta of IntelliJ integration for Xtext. In this post you'll learn about our progress so far, the hurdles we have faced and how you can take a look yourself.

## What have we achieved so far?

We have primarily focused on integrating traditional Xtext languages (i.e. without Xbase) into IntelliJ. This already proved to be a sizable task. There are  fundamental architectural differences between how Xtext, EMF and Eclipse work and how IntelliJ goes about its job. Yet we wanted you to be able to reuse runtime services like validation, scoping and indexing. So we had to create a bridge between both worlds.

For instance, IntelliJ expects languages to provide an AST that is based on their own modeling infrastructure. This made it impossible to reuse our existing parsers and added the overhead of maintaining two ASTs. Also, heavy concurrency and an aggressive cancellation policy enforce IntelliJ's high performance goals. But it is not a good fit for the non-threadsafe model that EMF provides. Thus, a big part of our time was spent learning about the lifecycle of editors and files in IntelliJ.

With this week's Beta we have reached a first milestone that we feel comfortable with giving to early adopters. So if you are curious and brave, here is what you can expect from an editor for your language.

### Basic Syntax Highlighting

Lexer-based highlighting works out of the box. So things like Keywords, Strings and Numbers are colored correctly. Semantic highlighting is not yet implemented. We will have to refactor some Eclipse-UI dependent code to make that reusable over different platforms.

![Syntax Highlighting]({{site.baseurl}}/images/xtext-intellij/highlighting.png)

### Validation

All validations that you are used to from Eclipse are displayed just the same in IntelliJ. This is achieved by converting the IntelliJ-based AST to an EMF-model that looks exactly the same like you would get in Eclipse.

### Auto Completion

Reference proposals work purely based on IntelliJ's modeling infrastructure. You can also get keyword proposals by binding the Antlr-based content-assist parser in you IdeaModule. This will result in a slight performance hit, as it adds another parsing step.

![Auto Completion]({{site.baseurl}}/images/xtext-intellij/completion.gif)

### Rename Refactoring

By implementing IntelliJ's NamedElement API, we benefit from rename refactoring support out of the box.

![Rename Refactoring]({{site.baseurl}}/images/xtext-intellij/rename.gif)

### Structural View

Just like the Outline in Eclipse, IntelliJ offers a structural view of your code. By default, the structure is exactly the same, but they can be customized independently. This is because you probably want different icons to fit in with IntelliJ's design.

![Structural View]({{site.baseurl}}/images/xtext-intellij/structure.png)

### Incremental Compiler

We have created an all-new incremental generator infrastructure that is independent from Eclipse. IntelliJ is the first platform to benefit from this. If you click on "Make Project", the build request will be send to a compiler daemon which greatly speeds up build times.

## What about Xbase and Xtend?

Hooking into IntelliJ's Java type system turned out to be much harder than anticipated. Our first approach of using so called LightElements did not work out, as many of IntelliJ APIs expect Java types to be backed up by actual Java source code. Our second try was to compile to Java on the fly and let IntelliJ parse that. But since reparsing is a very frequent activity in IntelliJ, this resulted in catastrophic performance.

The next approach that we are going to explore is a less invasive integration, that is generating Java code in the background and letting IntelliJ's Java tooling link against the generated code. This is a similar approach we use for integration with JDT in Eclipse, where it has proven to work nicely. The challenge here will be to find appropriate APIs to change navigation and other UI services, so the user is taken back to his DSL files and not to generated Java code.

## Can I try it myself?

To develop an IntelliJ plugin for your own Xtext language, install the Beta from our [update site](http://download.eclipse.org/modeling/tmf/xtext/updates/composite/milestones/). You will notice that the `New Project Wizard` now has a second page. Deselect the Eclipse Plugin and select the IntelliJ Plugin instead.

![New Xtext Wizard]({{site.baseurl}}/images/xtext-intellij/wizard.png)

This will create the necessary projects for you. Since building IntelliJ plugins requires quite a lot of setup and repetitive tasks, we created a [Gradle](gradle.org) build plugin to make this as easy as possible. After generating the language infrastructure, you can use `gradle runIdea` in your IDEA plugin's folder to start InteliiJ with your plugin installed.

![Running IDEA with Gradle]({{site.baseurl}}/images/xtext-intellij/run-idea.png)

You can also run `gradle eclipse` to generate a classpath containing all the necessary libraries, so you can browse and edit your plugin's source code.

## So what's next?

For the 2.9 release in autumn this year we plan to stabilize the APIs. They won't be public by that time, since we expect more changes as the first real world use cases are implemented. But we want to stop moving around fundamental concepts by then. We will also focus on solving the remaining issues for Xbase languages and the Xtend plug-in.

If you want to help us improve faster, take the Beta version for a spin and tell us about your experience in the forum or via Bugzilla. Depending on demand and progress we will put out more beta releases in the coming weeks.
