---
layout: post
title: 10 Usability Heuristics for User Interface Design
---

### 10 Usability Heuristics for User Interface Design


These are my notes for the [Youtube playlist The 10 Usability Heuristics](https://www.youtube.com/playlist?list=PLJOFJ3Ok_idtb2YeifXlG1-TYoMBLoG6I)

also see [Nielsen Norman Group's page on 10 Usability Heuristics](https://www.nngroup.com/articles/ten-usability-heuristics/)


1. Visibility - keep users informed with what is going on in the system in a reasonable time
* what they need to in a timely manner
* durable - always on - current state - ex: battery charge or network strength
* as a response to an action 
  * acknowledge success or a problem
  * indicate the system is working = progress bar
  * give a feeling of control
    * don't provide updates if not helpful - key pieces of info = user can't do anything with that feedback.
    * open and continuous communication
* reliable and predictable - creates trust

1. Match between the system and real world
* use language and concepts from real world
  * terms familiar to the user rather than system oriented 
  * instead of marketing jargon use regular language
  * mental models 
  * skeuomorphic - digital experience to match/align a physical experience.
  * real world conventions 
  * use user's language
* design for them and not you.

1. User Control and Freedom
* need a way to back out of a decision
  * back/cancel button
  * undo
  * must be visible to the user
* foster a sense of freedom 

1. Consistency and Standards
* internal and external
  * internal - in a product or family 
    * color for call to action buttons 
  * external - conventions for industry or in general
    * shopping cart - standard across most sites - upper right corner
  * don't force users to learn something new - unless it is worth the cost

1. Error Prevention
* don't let users do something catastrophic by accident
  * high cost effects
* location of a button
* confirmation dialog?
* support undo function so error doesn't become permanent
  * reply all - timer before actually sending?

1. Recognition vs. Recall
* why is recognition easier than recall
  * is the information provided correct
* memory retrieval - the more ques we have the easier to retreive
* show the commands and the user can recognize what they want versus remembering the command on a command line
* minimize user working

1. Flexibility and Efficiency of Use
* cntrl-c to copy versus right click copy versus edit menu copy option
* let the user select their prefered way to work
* accelerator - speeds up interaction for experienced users
* macros
* visibility for new users so they aren't overwelmed.

1. Aesthetic and Minimalist Design
* focus on the essentials
  * doesn't mean flat or monochromatic colors
  * keeping the content in the visual design focused on essentials
* signal to noise ration - ratio of relevant to irrelevant information
  * text content, animation
* every extra piece of information in the UI takes away from the relevant information
* visual - graphics or photos - support the primary goals of users - 
* communicate don't decorate

1. Recognize, Diagnose and Recover from Errors
* clearly inform users when an error has occurred
  * visual treatments (colors) and error message
  * what is the problem, in regular language
  * offer users a solution to solve the problem
  * best - link to solve the problem 
* gmail - supports undoing sending an email

1. Help and Documentation
* should be as intuitive as possible not to need helpful, but...
* consider:
  * is it easy to search/find helpful
  * focused on user's task
  * list steps to be carried out
* search suggestions
* minimize the interaction cost to the user
* make help contextual
  * pop over
  * screen shots
* use analytics to see it works
* don't overload users with unnecessary information

#### Bonus Material :)

1. Fitts's Law  [youtube](https://www.youtube.com/watch?v=M-9FbUJk6tI)

* the time to acquire a target is a function of the size of the target and the distance to the target

```
MT = a + b * log2( 2D / W)

MT = time to select target 
a & b = constants set by type of device
D = distance from starting point to target
W = width of target along axis of motion
```

* buttons 
  * submit button for a form - close to the bottom of the input fields
* important actions should be larger
* lists - shorter is better


1. Hick's Law - [youtube](https://www.youtube.com/watch?v=pbbTOzArcWQ)

designing long menu lists

* two criteria
  * in order (alphabetically)
  * items are known to the user - they know what the terms mean/are
* example - list of US States - no matter where a user focuses on the list, they know what they are looking for and where it is in relation to where they are.

*Should the whole list  be visible to scan?*

