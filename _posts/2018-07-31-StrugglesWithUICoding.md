---
layout: post
title: 'My Struggles with UI Coding: JavaScript and Angular'
---

## History

In 2005 I started working for a company that focused on mobile apps. I decided to stay focused on server side coding because I thought in time the need for specialized phone specific code would go away. But I didn't keep up with web coding. When we built a web UI for administration, my team hadn't used javascript in a while so I looked for a Java solution. I compared Google Web Toolkit and the Echo2 framework. I went with Echo2 because it was much more like Java's AWT/Swing and we had a single language to work with. 

I didn't get back to "real" client web coding until around 2010 or 2011. Five or six web years is like 100 human years. Even then mobile still wasn't the same as desktop so I didn't catch on to the new web frameworks. Now I find myself at a disadvantage. The way JavaScript was used back in the early 2000s is completely different. The programming model on the client side has advanced to complex multi-file modules. I feel like I missed something big somewhere as I struggle with seemingly simple issues.

## Struggles

For example I was creating a new form for collecting reporting criteria that was very similar to another report page so I started by copying the files for the other report, renaming them and then editing the contents. At first nothing worked, but I got passed that relatively easily by restarting the buld/server. Then when I loaded the page most things were looking good except the first two date input fields. The labels for the start/end dates in the schedule select failed to display, both showed as `{{model.label}}` and I had the following stack trace.

```
angular.js:15288 TypeError: Cannot read property 'label' of undefined
    at activate (pqScheduleWidgetDirective.js:31)
    at pqScheduleWidgetController (pqScheduleWidgetDirective.js:22)
    at Object.invoke (angular.js:4898)
    at O.instance (angular.js:11101)
    at q (angular.js:9867)
    at angular.js:10305
    at angular.js:17099
    at m.$digest (angular.js:18238)
    at m.$apply (angular.js:18604)
    at l (angular.js:12404)
```

This made it seem like there was a problem with how I was configuring our custom schedule control. I can't debug in the browser because the references to code in the stack trace open as empty files. Our build process obfuscates and merges the code into a single file. There is a mapping file that suppose to get loaded in the browser so the compressed code can be viewed and debugged as the source code, but it doesn't work, so I have to guess my way around the code with alerts or commenting out changes I made until I happen on the problem.

Long story short, I had a typo in my model name in the html template. there was no indication in the file to the error. I apparently cut and pasted here and left an errant 'P' in the model reference, after removing it things worked fine. This cost me 2 hours. 
	
```
<formly-form model="vm.reportPCMEOutcomesView" fields="vm.formFields" options="vm.formOptions" ng-if="vm.reportCMEOutcomesView"></formly-form>
```

This is were I feel like I'm missing something. Debugging in Java is such a joy compared to Visual Basic circa 1997 or C via gdb in the early 90s. Debugging in Chrome isn't bad with un-obfuscated code, I can set break points and view data values. But stepping into library code is terrible. 

There are other things about UI development. For example I lost time with an AngularJS module issue. Again I was copying and pasting a file someone else made to my reporting module. I changed the module name to match the name of the module I was putting it.

```
  angular
    .module('some.module', [])
    .directive(...)
  ...
```

After I did this I started getting errors that my controller wasn't defined, making me think I messed up something else. What really happened is that I didn't remove the second parameter to the module function `[]` and now I redefining the module, blowing away all the other things I had added to it. I didn't get an error that I was redefining an existing module which seems like something you shouldn't be able to do. My new file should have started like this 
  
```
  angular
    .module('some.module')
    .directive(...)
  ...
```

Again this took me a while to figure out. Am I struggling because I don't understand a tenet of JavaScript development or is this just how it is, and if so why do people work this way?

## Coda

Before our last UI expert left we decided to switch to using Typescript. In most cases we just simply renamed some files, but in others we did use real Typescript and that helped with compile time errors, until we moved to Intellij Ultimate with real Typescript support. That was awesome. Unfortunately he left and the new UI expert isn't in to Typescript. Intellij Ultimate is still much better with regular JavaScript identifying typos and being able to find where functions are defined, so I have that. Debugging is still a pain.