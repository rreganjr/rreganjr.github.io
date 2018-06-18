---
layout: post
title: Modularity, ManyToAny, and Bidirectional Associations
---

I like to use my old school project to experiment with new development techniques and technology. One thing I've found is that organizing code into Java packages doesn't make code modular and easy to reuse; over time the code falls into the [big ball of mud](http://wiki.c2.com/?BigBallOfMud) pattern. I'm working on splitting my project into multiple modules via Maven and Intellij.

[Requel](https://github.com/rreganjr/Requel) is a collaborative tool for collecting requirements as stories, goals, use cases and such. Users can comment or ask questions about different elements as well as an automatic analyzer that adds its own notes for users to review. One of the bigger issues I've hit is that requirement entities and annotation entities have bidirectional associations such that when a change occurs on an entity or annotation I can detect the event and update the UI and analyzer easily. This setup was fine in one big module, but leads to a cyclical dependency when I split out the annotations and project entities into distinct modules. As they say in computer science, all problems can be solved with one more level of abstraction, I created a third module to split out the concept of Annotatable from the annotations implementation to get around the cycle.

 Now I'm stuck because I use a Hibernate many-to-any relation from the annotations to the requirement entities and the @AnyMetaDef requires linking to the classes that are referenced, and that leads back to a cyclic dependency. I could probably cheat and do the meta def mapping in an xml file, but that is really a technical solution that breaks the independency goal of modules which is the point of this exercise.

 What I really want is an independent annotation service that allows adding notes to any thing, but is also aware of events and changes so it can respond to changes on those tings.

