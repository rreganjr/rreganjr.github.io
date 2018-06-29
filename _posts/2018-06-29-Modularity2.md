---
layout: post
title: Modularity, ManyToAny, and Bidirectional Associations Follow Up
---

## The Solution

This is the solution to the problem described in [Modularity, ManyToAny, and Bidirectional Associations]({% link _posts/2018-06-18-Modularity.md %})

I ended up creating a new module that only contains a package-info.java file where I define the AnyMetaDef with a name, added dependencies on the requel-project and annotation modules, but *no dependency* on this module except for a runtime dependency in the main requel module so it gets included in the spring boot jar and is available in the classpath on startup.

```
@AnyMetaDef(name = "annotatables", idType = "long", metaType = "string", metaValues = {
    @MetaValue(value = "com.rreganjr.requel.annotation.Annotation", targetEntity = AbstractAnnotation.class),
    @MetaValue(value = "com.rreganjr.requel.project.Project", targetEntity = ProjectImpl.class ),
    @MetaValue(value = "com.rreganjr.requel.project.ProjectTeam", targetEntity = ProjectTeamImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.Goal", targetEntity = GoalImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.GoalRelation", targetEntity = GoalRelationImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.UseCase", targetEntity = UseCaseImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.Scenario", targetEntity = ScenarioImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.Step", targetEntity = StepImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.Story", targetEntity = StoryImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.Actor", targetEntity = ActorImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.GlossaryTerm", targetEntity = GlossaryTermImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.NonUserStakeholder", targetEntity = NonUserStakeholderImpl.class),
    @MetaValue(value = "com.rreganjr.requel.project.UserStakeholder", targetEntity = UserStakeholderImpl.class)
})
package com.rreganjr.requel.annotation;

    import com.rreganjr.requel.annotation.impl.AbstractAnnotation;
    import com.rreganjr.requel.project.impl.*;
    import org.hibernate.annotations.AnyMetaDef;
    import org.hibernate.annotations.MetaValue;
```

In the AbstractAnnotation class in the annotation module I reference the metaDef by name.

```
	@ManyToAny(metaDef = "annotatables", fetch = FetchType.LAZY, metaColumn = @Column(name = "annotatable_type", length = 255, nullable = false))
	@JoinTable(name = "annotation_annotatable", joinColumns = { @JoinColumn(name = "annotation_id") }, inverseJoinColumns = { @JoinColumn(name = "annotatable_id") })
	public Set<Annotatable> getAnnotatables() {
		return annotatables;
	}
```

I will still need to manually maintain the mapping file, which is unfortunate, but doable.

## The Solution I Wanted

 My first thought to the problem was to remove the MetaDef completely by hooking into the hibernate metamodel creation process and dynamically adding classes by scanning available classes that implement the interface Annotatable and programmatically add the MetaDef. I spent a few days debugging Hibernate bootstrapping and couldn't figure out a way. Using MetadataContributor seems promising, but I couldn't find any examples on the web of someone trying to do something similar.

 The stack trace of of where I ended up (hiberate 5.2 source)

 ```
   LocalContainerEntityManagerFactoryBean afterPropertiesSet()
     AbstractEntityManagerFactoryBean buildNativeEntityManagerFactory()
 	  LocalContainerEntityManagerFactoryBean createNativeEntityManagerFactory()
 	    SpringHibernateJpaPersistenceProvider createContainerEntityManagerFactory()
           EntityManagerFactoryBuilderImpl build()
             metadata()
               MetadataBuildingProcess complete()
                 InFlightMetadataCollectorImpl buildMetadataInstance()
                   new MetadataImpl()
```

The documentation I found was light:

* [Native Bootstrapping](http://docs.jboss.org/hibernate/orm/5.0/topical/html/bootstrap/NativeBootstrapping.html)
* [Hibernate5 migration question](http://hibernate-dev.jboss.narkive.com/1t0vw8rS/hibernate5-migration)
* [MetaValue Cache](https://gist.github.com/danigulyas/d070df922234d6259f4f96f2a54236b7)

This seems promising, but it didn't have details and the source referenced was a groovy project

* [Grails Hibernate Filters support for Hibernate 5](http://www.alexckramer.com/grails/2016/12/18/grails-hibernate5-filter-support.html)

### Next Step: Scan through github

If I have any free time I may come back to this type of solution by looking in gitlab:

* [MetadataContributor in:file](https://github.com/search?q=MetadataContributor+in%3Afile&type=Code)